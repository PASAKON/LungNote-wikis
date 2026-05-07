---
title: "ADR-0009: LINE bot AI replies — OpenRouter + Gemini 2.5 Flash"
tags: [adr, line, ai, llm, openrouter, gemini]
status: Proposed
date: 2026-05-07
---

# ADR-0009 — LINE bot AI replies: OpenRouter + Gemini 2.5 Flash

## Status

Proposed

## Context

[[0007-line-oa-messaging|ADR-0007]] established the LINE OA Messaging API channel. The current webhook handler ([github](https://github.com/PASAKON/LungNote-webapp/blob/main/src/app/api/line/webhook/route.ts)) auto-replies via 4 hardcoded regex patterns + an echo fallback. Useful only for keyword commands; off-script messages get a meaningless echo.

We need an LLM-backed reply layer so the bot is genuinely helpful for:
- Customer-support questions about LungNote
- Casual chat with student users
- (Future) note capture, study help, exam-prep generation

Constraints:
- **Cost ceiling at scale.** Target audience = Thai students; planning horizon includes 100K+ MAU. A flagship-tier model would be unaffordable.
- **Thai language quality is non-negotiable.** Audience is native Thai; weak Thai = unusable.
- **Latency.** LINE webhook timeout is 30s, but real-world UX target is <5s.
- **Hot-swappable.** Model market moves fast; must be able to switch providers without re-architecting.
- **Security.** Webhook is unauthenticated (LINE → us, no user JWT). Audit + rate limit needed.
- **CLAUDE.md hard rule #5.** Adding an LLM provider is an architectural dependency change requiring an ADR.

This ADR captures the high-level decision. The full design — schemas, data flow, error handling, testing — is in the companion design spec at [`docs/superpowers/specs/2026-05-07-line-bot-ai-replies-design.md`](https://github.com/PASAKON/LungNote-webapp/blob/main/docs/superpowers/specs/2026-05-07-line-bot-ai-replies-design.md) (webapp repo).

## Decision

Use **OpenRouter** as the LLM gateway, with **Gemini 2.5 Flash** as the default model for v1.

### Provider: OpenRouter

- Single HTTP API exposing many models (Claude, GPT, Gemini, DeepSeek, Mistral, etc.).
- Hard daily spend cap configurable on dashboard.
- Hot-swap models without rewriting client code.
- Pass-through pricing for most providers; modest top-up fee on credit deposits.
- Env: `OPENROUTER_API_KEY` (server-only).

### Model: Gemini 2.5 Flash

- **Thai fluency is excellent.** Google's Thai NLP investment goes back over a decade.
- **Cost.** Approximately $0.075 / 1M input tokens, $0.30 / 1M output tokens. With prompt caching, ~$0.0001 per AI call (650 in / 200 out average).
- **Latency.** Typical 1-2 seconds end-to-end via OpenRouter.
- **Quality.** Smart enough for customer support and casual chat; not meant for hard reasoning (use Sonnet/Opus there if a route ever needs it).

### Architecture (summary; full detail in design spec)

- **Hybrid routing.** Existing regex (`สวัสดี / ช่วย / เว็บ / เกี่ยวกับ` + EN aliases) stays for deterministic keyword commands. Everything else goes to AI.
- **Synchronous inline.** Webhook awaits the AI call and replies via LINE's free `reply` API. Avoids burning `push` quota.
- **8s timeout** on the AI call; on timeout/error, fall back to the existing regex echo (no user-visible failure).
- **Short conversational memory.** Last 5 user + 5 AI messages per LINE userId, stored in a new Supabase table `conversation_memory`. Stateless logic except for this rolling window.
- **Per-user rate limit.** 20 AI messages / user / day, tracked in `rate_limit` table (insert-or-increment). Regex matches are free.
- **Audit.** Every webhook event → `line_webhook_events`. Every AI call → `ai_call_log` (model, tokens, cost estimate, latency, success).
- **Service-role Supabase client.** New `src/lib/supabase/service.ts` using `SUPABASE_SECRET_KEY` for system-internal writes (memory, audit, rate-limit). Existing user-context clients ([`server.ts`](https://github.com/PASAKON/LungNote-webapp/blob/main/src/lib/supabase/server.ts), `client.ts`) unchanged.

### Cost ceiling

OpenRouter daily spend cap = **$20/day** for v1. Realistic 100K-MAU spend (~25% DAU, ~3 msgs/DAU/day, ~70% routed to AI) projects to **~$158/month**. The $20/day cap is a hard ceiling against runaway loops or abuse; not the operating budget.

## Consequences

### Positive

- **Bot becomes useful.** Off-script messages get an actually helpful reply in Thai.
- **Cost-controlled.** ~$0.0001/call, $158/mo at 100K-MAU realistic, $20/day hard cap. Predictable economics.
- **Hot-swappable.** OpenRouter abstracts the provider — switching to GPT-4o mini or Claude Haiku for specific code paths becomes a one-line change in `client.ts`.
- **Defense in depth.** Per-user rate limit + provider spend cap + output token cap + prompt caching = four independent cost levers.
- **Audit-ready foundation.** Establishes service-role Supabase pattern, RLS-on-every-table convention, and migration tooling — all of which the rest of the app needs anyway. This is also the project's first migration; pays down setup tax for future schema work.
- **Hybrid routing keeps deterministic UX** for keyword commands (no risk of AI hallucinating the help menu).

### Negative

- **Vendor surface grows.** OpenRouter + Google Gemini = two new dependencies on top of the existing Supabase + LINE stack. Outage in either degrades replies (graceful — fallback to regex echo).
- **Privacy.** User message text is sent to OpenRouter and from there to Google. Mitigated by minimum disclosure in the welcome message and an opt-out training-data header. Users have plausible expectation that LINE bots use AI now (industry norm), but PDPA-conscious users may object.
- **First migration tax.** This work also bootstraps Supabase CLI + migration workflow in the project. Larger PR than just "wire up an LLM".
- **Quality drift risk.** LLM provider quality changes month-to-month. Need a regression-checking habit (e.g. periodic manual prompts to verify Thai fluency hasn't slipped on cheaper-tier model updates).
- **Cold-start cost.** First request after the function is idle pays a Supabase + OpenRouter network warmup penalty (~500ms additional). Mitigated by Vercel function reuse but not eliminated.

### Mitigation

- **Daily spend cap on OpenRouter** independent of per-user rate limit.
- **Fail-open on rate-limit DB outage.** Don't punish users for our infra problems; document the trade-off (a malicious actor could exploit a Supabase outage to bypass limits, but this is acceptable for v1 — the OpenRouter daily cap is the backstop).
- **Disclosure on follow event** — *"💡 ข้อความของคุณอาจถูกประมวลผลโดย AI เพื่อช่วยตอบ"* — placed in the welcome message users see when they first add the OA.
- **`X-OpenRouter-Allow-Training: false`** header (or equivalent) on every request to opt out of training-data sharing.
- **Hybrid routing** ensures keyword commands stay free + deterministic, so the bot never goes 100% silent if the AI provider is down.
- **Monthly review** of `ai_call_log` for cost trend, latency p50/p95, and error rate. Alert if monthly spend exceeds $300 (well below daily-cap-extrapolated worst case).

## Alternatives

- **GPT-4o mini** (OpenAI via OpenRouter) — Good quality, ~2× the cost of Gemini Flash, slightly weaker Thai. Strong fallback option; will be the first thing tried if Gemini Flash quality regresses.
- **Claude Haiku 4.5** — Substantially smarter than Flash, ~16× the cost. Right answer if budget were not a constraint, or for specific high-value paths (e.g. exam-prep generation when that lands). Not the right default for chat.
- **DeepSeek V3** — Excellent intelligence-per-dollar, but Thai fluency is materially weaker than Gemini Flash. Unsuitable for native-Thai user base.
- **Self-host (Llama 3.x via vLLM / Ollama)** — Eliminates per-call cost but requires GPU infra, much weaker Thai, no SLA. Bad fit for early-stage app on Vercel free tier.
- **Direct Google AI API** (skip OpenRouter) — Slightly cheaper at the margin, but loses the hot-swap benefit. Tied to Google forever for one rounding-error of savings. Not worth it.
- **Async with placeholder reply ("🤖 thinking...")** — Allows AI more time but burns LINE `push` quota (200 free/mo, then $0.05/msg). Two-message UX feels off. Rejected for v1.
- **Status quo (regex only)** — Bot remains a toy. Not viable if the goal is real users.

## Open Questions

- [ ] **OpenRouter Gemini Flash slug** — verify exact model ID against OpenRouter's catalog before merge (likely `google/gemini-2.5-flash`).
- [ ] **Prompt caching syntax** — confirm Gemini path supports OpenRouter's unified prompt-cache parameter; otherwise apply caching via Google's native parameter.
- [ ] **`OPENROUTER_API_KEY` arrived in Vercel env without a corresponding ADR or PR** — sync with `golfmaichai1` to confirm there's no parallel exploration we'd be duplicating.
- [ ] **Rate-limit timezone** — schema uses UTC `current_date`. Thai users see daily reset at 07:00 local. Decide: keep UTC or switch default to `(timezone('Asia/Bangkok', now()))::date`.
- [ ] **Supabase project region** — pending verification per [[0006-supabase-db-auth|ADR-0006]] open questions. Doesn't block this ADR but affects observed latency.
- [ ] **Disclosure copy** — final wording of the AI-disclosure line in the welcome message. Current draft is a placeholder pending legal-friendly polish.

## Implementation TODO

(Detail in companion design spec.)

- [ ] `src/lib/supabase/service.ts` (server-only client using `SUPABASE_SECRET_KEY`)
- [ ] Supabase CLI setup + first migration (`supabase/migrations/20260507120000_create_line_bot_ai_tables.sql`)
- [ ] Four tables with deny-by-default RLS: `line_webhook_events`, `conversation_memory`, `rate_limit`, `ai_call_log`
- [ ] `src/lib/ai/{client,reply,prompts,rate-limit,memory}.ts`
- [ ] Modify `src/app/api/line/webhook/route.ts` for hybrid routing + AI fallback
- [ ] Tests: unit (regex/memory/rate-limit/cost), integration (mocked OpenRouter via MSW), RLS (anon/authenticated cannot access tables)
- [ ] Update [[../30-Domain/Glossary]] — OpenRouter, Gemini 2.5 Flash, prompt caching, RLS service-role
- [ ] Update [[../10-Architecture/Overview]] — add AI reply path to data flow diagram
- [ ] Set OpenRouter daily spend cap to $20 in dashboard
- [ ] Smoke test against prod, monitor `ai_call_log` for first 24h

## See Also

- [[0006-supabase-db-auth]] — Supabase decision; this ADR uses its `SUPABASE_SECRET_KEY` and migration workflow
- [[0007-line-oa-messaging]] — LINE channel decision; this ADR adds the AI reply layer on top
- [[../50-Workflows/Collaboration-Workflow]] — daily flow, pre-push checklist for shipping this work
- Design spec: [`docs/superpowers/specs/2026-05-07-line-bot-ai-replies-design.md`](https://github.com/PASAKON/LungNote-webapp/blob/main/docs/superpowers/specs/2026-05-07-line-bot-ai-replies-design.md)
- OpenRouter Gemini 2.5 Flash: https://openrouter.ai/google/gemini-2.5-flash
- Gemini 2.5 Flash pricing: https://ai.google.dev/gemini-api/docs/pricing
