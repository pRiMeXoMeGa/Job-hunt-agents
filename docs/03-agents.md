# 03 · Agent Specifications

> Status: **Draft for review**. These contracts are what each agent promises. Prompts and implementation come in the build phase.

Common rules for **all** agents:

* Agents talk to each other only through the **task queue** and **database state**, never directly. This keeps every step resumable and auditable.
* Agents never write user memory directly. They call the Memory Manager (`propose`, `record_event`).
* Any text from the web (job descriptions, careers pages, ATS pages) is wrapped as **untrusted data** in prompts and can never change an agent's instructions.
* Every LLM call goes through the LLM Gateway with a task name, so model choice, cost and traces are per task.
* Idempotency: every task carries an `idempotency_key`, so a retry never double-applies.

---

## 1. Orchestrator (Supervisor)

| | |
|---|---|
| **Purpose** | Single entry point for every inbound message and agent result. Routes work, enforces mode and policy. |
| **Input** | `InboundMessage` (text, attachment, button callback), agent completion events |
| **Output** | Tasks on the queue, replies via the Concierge, workflow state in the checkpointer |
| **Intents** | `upload_resume`, `add_companies`, `answer_question`, `approve_application`, `reject_application`, `skip_job`, `job_details`, `status_query`, `settings_change` (mode, thresholds, quiet hours), `memory_command`, `pause` / `resume`, `smalltalk` |
| **Tools** | `enqueue(task)`, `get_settings`, `set_settings`, `MemoryManager.observe`, `Concierge.send` |
| **Policies enforced** | Mode gate, daily and per-company caps, kill switch, quiet hours |
| **Failure handling** | Unknown intent → asks one clarifying question. Never guesses an approval. |

## 2. Concierge (Communications)

| | |
|---|---|
| **Purpose** | The voice of the system. Turns many small agent needs into few, clear messages. |
| **Responsibilities** | Batch questions (max 5 per message), shortlist digests, approval cards, receipts, error handoffs. Respect quiet hours (default 22:00–08:00 Asia/Dubai). Map replies back to `pending_questions`. |
| **Channel-agnostic output** | `OutboundMessage{text, buttons[], attachments[], reply_to}`. Adapters render buttons as Telegram inline keyboards, WhatsApp interactive buttons/lists, or Slack Block Kit. |
| **Throttling** | At most one digest per hour unless you ask. Urgent items (blocked submission, approval needed with a deadline) bypass batching. |

## 3. Profile Agent

| | |
|---|---|
| **Purpose** | Build and maintain the candidate profile from the resume and onboarding. |
| **Input** | Resume file (PDF/DOCX), onboarding answers |
| **Output** | `CandidateProfile` facts → `MemoryManager.propose(source = resume)`, resume chunks + embeddings for RAG, gap list |
| **Rules** | Extract only what's written. No inference of skills not present. Dates normalized. Each fact keeps a pointer to its resume section for traceability. |
| **Multiple resumes** | Supports resume variants (e.g. "Data Engineer CV", "ML CV"). The Prep Agent picks one per job and tells you which. It never edits the resume content itself in v1. |

## 4. Discovery Agent

| | |
|---|---|
| **Purpose** | Company + careers URL → complete, fresh list of open roles. |
| **Input** | `company_id` |
| **Output** | Upserted `jobs`, scan record, `match` tasks for new or changed jobs |
| **Strategy** | ATS detection → public job-board JSON API (Greenhouse, Lever, Ashby, Workable, SmartRecruiters, Recruitee) → otherwise headless crawl with pagination + LLM extraction to the `JobPosting` schema |
| **Politeness** | Respects robots.txt for crawls, rate limit per domain (e.g. 1 request/2 s), identifiable user agent, cached responses, re-scan every 24 h |
| **Failure handling** | 3 retries with backoff. Then marks the company `scan_failed` and tells you once. |

## 5. Matching Agent

| | |
|---|---|
| **Purpose** | Decide which roles are a real fit. |
| **Input** | `job_ids` |
| **Output** | `matches{score, label, reasons[], gaps[], hard_filter_results}` |
| **Stages** | Hard filters → vector pre-rank → LLM rubric. See [01 §3.2](01-system-architecture.md#32-matching-three-stages-so-its-cheap-and-explainable). |
| **Rubric (weights configurable)** | Core skills overlap 30 · Years / seniority fit 20 · Title and function alignment 20 · Domain / industry 10 · Must-have gaps (negative) −20 · Location / work-mode fit 10 · Preference bonus 10 |
| **Calibration** | You can say "this one should've been higher/lower". That's an episode, used to tune weights later. |

## 6. Application Prep Agent

| | |
|---|---|
| **Purpose** | Produce a complete, honest application packet. |
| **Input** | `match_id` |
| **Output** | `packet{resume_variant, fields[{label, value, policy, source_memory_id}], free_text_answers[], cover_note?}`, `pending_questions` |
| **Rules** | **No fabrication**: every claim must trace to the profile or an approved answer. If a required skill is missing, the answer must not pretend otherwise. Respect character limits. Language matches the posting. |
| **Cover note** | Only if the form has a field or upload for it, or you enable it. 150–250 words, grounded in 2–3 real achievements that match the JD. |

## 7. Applier Agent

| | |
|---|---|
| **Purpose** | Fill and submit forms in a real browser. |
| **Input** | `application_id` (status READY, approved or auto-cleared) |
| **Output** | Submission evidence (pre- and post-submit screenshots, confirmation text / ID, timestamp), status update, procedural notes |
| **Approach** | Deterministic Playwright scripts for known ATS form layouts (Greenhouse, Lever, Ashby), with an **LLM browser agent** (e.g. browser-use) as a fallback for unknown layouts. Before clicking submit, re-read every field from the DOM and compare it to the packet. |
| **Guardrails** | Domain allowlist (only the company's careers domain / known ATS). Never types credentials an LLM produced, only vault values. Never solves CAPTCHAs: hands off to you. Max 1 submission per job, ever (idempotency). |
| **Handoff** | CAPTCHA, OTP, email verification or an unexpected page → status `BLOCKED_HANDOFF` + a message with a link and what's left to do. |
