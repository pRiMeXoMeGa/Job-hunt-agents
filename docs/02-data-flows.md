# 02 · Data Flows

> Status: **Draft for review**

This document traces how data moves through the system for each workflow. All chat messages pass through the **Channel Gateway** (normalized to `InboundMessage` / `OutboundMessage`) and the **Orchestrator**. Every user message is also handed to the **Memory Manager write path** asynchronously, so the chat reply never waits on it.

## 0. End-to-end overview

```mermaid
flowchart LR
    A["1 Onboard<br/>resume + profile Q&A"] --> B["2 Add companies<br/>name + careers URL"]
    B --> C["3 Discover roles<br/>ATS API or crawl"]
    C --> D["4 Match and shortlist<br/>score + reason"]
    D --> E["5 Prepare application<br/>resolve every field"]
    E -->|missing info| F["6 Ask you in chat<br/>one batched question"]
    F -->|answer remembered| E
    E --> G{"7 Mode gate"}
    G -->|review| H["You approve<br/>screenshot + summary"]
    G -->|auto, all gates pass| I["8 Submit"]
    H --> I
    I --> J["9 Track + report<br/>status, evidence"]
    J -.->|feedback events| K["Memory consolidation"]
    K -.->|confirmed preferences| D
```

## 1. Onboarding: resume and profile

```mermaid
sequenceDiagram
    autonumber
    actor U as You
    participant GW as Channel Gateway
    participant OR as Orchestrator
    participant PA as Profile Agent
    participant MM as Memory Manager
    participant DB as MongoDB / File store

    U->>GW: Upload resume.pdf
    GW->>DB: Store file (GridFS), create document record
    GW->>OR: InboundMessage(attachment = doc_id)
    OR->>PA: task parse_resume(doc_id)
    PA->>DB: Read file, extract text
    PA->>PA: LLM structured extraction to CandidateProfile
    PA->>MM: propose_memories(facts, source = resume, doc_id)
    MM->>MM: classify, score, reconcile
    MM->>DB: write memories + profile snapshot + audit events
    MM-->>PA: summary of saved and conflicting keys
    PA->>OR: done + gaps list (required keys still missing)
    OR->>U: "Here's what I extracted. Please confirm" [✅ Looks right] [✏️ Fix]
    loop Required onboarding keys still missing
        OR->>U: Batched questions, e.g. notice period, visa status, expected salary
        U->>GW: Answers in free text
        GW->>OR: InboundMessage
        OR->>MM: observe(message, context = onboarding)
        MM->>DB: write classified memories
    end
    OR->>U: "Profile ready ✅ Send me companies whenever you like."
```

**Data produced:** `documents`, `chunks` (resume chunks with embeddings for RAG), `memories` (profile facts with `source = resume`, then `source = user_chat`), `profile_snapshot`.

The **required onboarding keys** are defined in the schema registry (see [04](04-memory-manager.md#4-schema-registry)): legal name, email, phone, current location, work authorization per target country, notice period, expected salary range and currency, target titles, target locations, remote preference, LinkedIn URL.

## 2. Adding companies

```mermaid
sequenceDiagram
    autonumber
    actor U as You
    participant OR as Orchestrator
    participant DB as MongoDB
    participant Q as Task Queue

    U->>OR: "Careem - https://careers.careem.com<br/>Stripe - https://stripe.com/jobs"
    OR->>OR: Parse lines, CSV or file into (name, url)
    OR->>DB: upsert companies (normalized domain = dedup key)
    OR->>Q: enqueue discover(company_id) for each company
    OR->>U: "Added 2 companies. Scanning now, I'll report back."
```

## 3. Discovery: careers page to jobs

```mermaid
sequenceDiagram
    autonumber
    participant Q as Task Queue
    participant DA as Discovery Agent
    participant ATS as ATS public API
    participant BR as Browser Pool
    participant LLM as LLM Gateway
    participant DB as MongoDB

    Q->>DA: discover(company_id)
    DA->>DB: load company (url, detected_ats, last_scan)
    DA->>DA: detect ATS (URL pattern, embedded script, iframe src)
    alt Known ATS with public JSON
        DA->>ATS: GET job board endpoint
        ATS-->>DA: jobs JSON
    else Custom site / Workday
        DA->>BR: render careers page, paginate, follow job links
        BR-->>DA: HTML / text per job page
        DA->>LLM: extract JobPosting schema (treated as untrusted data)
        LLM-->>DA: structured jobs
    end
    DA->>DA: normalize, hash, diff vs previous scan
    DA->>DB: upsert jobs (new / changed), mark missing as closed
    DA->>Q: enqueue match(job_ids = new + changed)
```

Scheduled re-scan: each company every 24 h by default, configurable. Only **new or changed** jobs go on to matching, so you're never notified twice about the same role.

## 4. Matching and shortlist

```mermaid
sequenceDiagram
    autonumber
    participant Q as Task Queue
    participant MA as Matching Agent
    participant MM as Memory Manager
    participant LLM as LLM Gateway
    participant DB as MongoDB
    participant CO as Concierge
    actor U as You

    Q->>MA: match(job_ids)
    MA->>MM: get_profile_block() + get_preferences(confirmed only)
    MM-->>MA: compact profile + hard and soft preferences
    MA->>MA: Stage 1 hard filters (no LLM)
    MA->>DB: Stage 2 vector similarity (job embedding vs profile)
    MA->>LLM: Stage 3 rubric score for survivors
    LLM-->>MA: score, label, reasons, gaps
    MA->>DB: write matches
    MA->>CO: shortlist ready
    CO->>CO: batch into digest (respect quiet hours, max N per message)
    CO->>U: "🎯 5 new matches at Careem" + per-job [Apply] [Skip] [Details]
    U->>CO: tap Skip, reason "too junior" (optional)
    CO->>MM: observe(event = job_skipped, reason)
```

Skip reasons are **episodic memories**. The nightly consolidator may turn repeated patterns into a *proposed* preference, which you confirm before it's used as a filter.

## 5. Application preparation and the missing-info loop

This is where the Memory Manager earns its keep: every form field is resolved from memory, and only truly unknown or unsafe fields get asked.

```mermaid
sequenceDiagram
    autonumber
    actor U as You
    participant CO as Concierge
    participant AP as App Prep Agent
    participant BR as Browser Pool
    participant MM as Memory Manager
    participant DB as MongoDB

    AP->>BR: open apply URL, read form schema (fields, types, required, options)
    BR-->>AP: FormSchema
    loop each field
        AP->>MM: resolve_field(label, type, options, job_ctx)
        MM-->>AP: value + confidence + sensitivity + policy
    end
    Note over AP,MM: policy is one of AUTO_FILL, FILL_AND_FLAG, MUST_ASK, NEVER_FILL
    AP->>AP: draft free-text answers, cover note (grounded in profile, no invention)
    alt Any MUST_ASK fields
        AP->>DB: create or attach pending_questions (deduplicated across applications)
        AP->>DB: application.status = NEEDS_INFO
        AP->>CO: questions ready
        CO->>U: "To apply to 3 roles I need: 1) Expected salary in AED? 2) Can you start within 30 days?"
        U->>CO: "25k AED a month, yes"
        CO->>MM: observe(answer, question_ids)
        MM->>DB: store as memories (reusable next time)
        CO->>AP: resume all applications blocked on these questions
    end
    AP->>DB: application packet saved, status = READY
```

**Question de-duplication:** if five applications all need "expected salary (AED)", one `pending_question` is created and linked to all five. One answer unblocks all of them.

## 6. Submission (review mode and auto mode)

```mermaid
sequenceDiagram
    autonumber
    actor U as You
    participant OR as Orchestrator
    participant AA as Applier Agent
    participant BR as Browser Pool
    participant V as Vault
    participant DB as MongoDB
    participant MM as Memory Manager

    OR->>OR: check mode + auto-gates + daily cap + kill switch
    alt Review mode, or any gate fails
        OR->>U: Summary + filled-form screenshot [✅ Submit] [✏️ Edit] [❌ Cancel]
        U->>OR: ✅ Submit
    end
    OR->>AA: submit(application_id)
    AA->>BR: open form, fill from packet, upload resume
    opt Site needs a login
        AA->>V: get credentials(domain)
        V-->>AA: credentials (go straight to the browser, never to the LLM)
    end
    alt CAPTCHA, OTP or unexpected page
        AA->>DB: status = BLOCKED_HANDOFF
        AA->>U: "Needs you: CAPTCHA on Workday. Link to finish (form is pre-filled where possible)."
    else Normal
        AA->>BR: verify every field against packet, take pre-submit screenshot
        AA->>BR: click Submit, capture confirmation page / ID
        AA->>DB: status = SUBMITTED, evidence (screenshots, confirmation text, timestamp)
        AA->>MM: observe(event = applied) + procedural notes for this ATS
        AA->>U: "✅ Applied: Senior Data Engineer @ Careem (ref #12345)"
    end
```

## 7. Application status state machine

```mermaid
stateDiagram-v2
    [*] --> DISCOVERED
    DISCOVERED --> MATCHED: score >= threshold
    DISCOVERED --> FILTERED_OUT: fails hard filter or low score
    MATCHED --> SHORTLISTED: shown in digest
    SHORTLISTED --> SKIPPED: you tap Skip
    SHORTLISTED --> PREPARING: you tap Apply, or auto mode picks it
    PREPARING --> NEEDS_INFO: MUST_ASK fields
    NEEDS_INFO --> PREPARING: answered
    NEEDS_INFO --> EXPIRED: no answer in N days or job closed
    PREPARING --> READY
    READY --> AWAITING_APPROVAL: review mode or gate failed
    READY --> SUBMITTING: auto mode, all gates pass
    AWAITING_APPROVAL --> SUBMITTING: you approve
    AWAITING_APPROVAL --> CANCELLED: you cancel
    SUBMITTING --> SUBMITTED
    SUBMITTING --> BLOCKED_HANDOFF: CAPTCHA, OTP, login wall
    SUBMITTING --> FAILED: error after retries
    BLOCKED_HANDOFF --> SUBMITTED: you finish manually and confirm
    FAILED --> PREPARING: retry
    SUBMITTED --> [*]
```

## 8. Memory flow on every message (summary)

```mermaid
flowchart LR
    M["Inbound user message<br/>or agent event"] --> G{"Source gate"}
    G -->|user chat / user upload| W["Memory write path<br/>see doc 04"]
    G -->|agent event| E["Episodic / procedural only"]
    G -->|web page, JD, email| X["Never writes user memory"]
    W --> S[("memories + profile_snapshot")]
    E --> S
    S --> R["Read path:<br/>profile block, resolve_field, search"]
    R --> AG["Agents"]
```

## 9. Data inventory by flow

| Flow | Creates / updates | Sensitive data touched |
|------|-------------------|------------------------|
| Onboarding | documents, chunks, memories, profile_snapshot | Contact, salary, visa |
| Add companies | companies | None |
| Discovery | jobs | None (public postings) |
| Matching | matches | Profile summary (sent to LLM) |
| Prep | applications.packet, pending_questions | All fields required by the form |
| Submission | applications.evidence, memories (episodic, procedural) | Credentials (vault to browser only) |
| Consolidation | memories (proposed preferences), answer bank | Same as stored |
