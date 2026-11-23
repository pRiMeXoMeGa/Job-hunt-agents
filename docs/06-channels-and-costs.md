# 06 · Messaging Channels and Running Costs

> Status: **Draft for review** · Prices checked 23 Nov 2026, assuming a recipient in the UAE. Re-check before launch.

## 1. Channel comparison (personal use, about 1,000 messages a month)

| Option | Monthly cost (est.) | Setup effort | Proactive pings | Buttons | Notes |
|--------|--------------------|--------------|-----------------|---------|-------|
| **Telegram Bot API** | **$0** | Minutes (BotFather token) | Any time | Inline keyboards | **Recommended for v1** |
| **WhatsApp: Meta Cloud API direct** | ~$0–2 | Meta Business account + a separate phone number | Template message needed after 24 h of silence | Reply buttons / lists | Since 1 Oct 2026 service messages are billed per message, but each business number gets 1,000 free service messages a month. UAE utility template ≈ $0.0157. |
| **WhatsApp via Twilio** | ~$5–7 | Easier onboarding | Same template rules | Same | $0.005 Twilio fee per message, inbound and outbound, **plus** Meta fees |
| **WhatsApp unofficial (Baileys / whatsapp-web.js)** | $0 | QR scan | Any time | Limited | **Not recommended**: against WhatsApp's terms, risk of losing your number |
| **Slack (free plan)** | $0 | Create a workspace + app | Any time | Block Kit | 90-day visible history, 10-app limit |

**Decision proposed:** Telegram adapter in Phase 1. WhatsApp Cloud API adapter in Phase 3. The rest of the system is unaffected because of the channel-adapter pattern.

### WhatsApp design implications (when added)

* Proactive messages (new matches, "needs info") after 24 h of silence require **pre-approved utility templates**, e.g. `new_matches_v1`: "You have {{1}} new job matches. Reply to view." Tapping it reopens the service window, so the follow-up conversation counts as service messages.
* The bot's number can't be your personal WhatsApp number. Use a second SIM or virtual number.

## 2. Other running costs (estimates)

| Item | Estimate / month | Notes |
|------|------------------|-------|
| LLM usage | $10–30 | Driven by matching + applying. Cheap models for extraction and routing. A hard monthly budget cap in the gateway. |
| Embeddings | < $1 | Jobs + memories |
| MongoDB Atlas | $0 (free tier) → paid dedicated tier later | Free tier is fine to start. Check current Atlas pricing and Vector Search limits before launch. |
| VM / container host | $5–15 | 2 vCPU / 4 GB RAM is enough for one Chromium |
| Redis | $0 (in the compose stack) | |
| Tracing (Langfuse self-host or cloud free tier) | $0 | |
| **Total (start)** | **≈ $15–45** | |
