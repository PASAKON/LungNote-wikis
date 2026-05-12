---
title: "ADR-0017: ClaudeFlow patterns — Anthropic + cache, multi-bubble, DB-stored prompt"
tags: [adr, ai, agent, anthropic, prompt-cache]
status: Accepted
date: 2026-05-12
---

# ADR-0017 — ClaudeFlow patterns: Anthropic prompt cache, multi-bubble Flex, DB-stored prompt

## Status

Accepted (backfilled — bundle landed in [LungNote-webapp#41](https://github.com/PASAKON/LungNote-webapp/pull/41) on 2026-05-09)

## Context

After [[0016-ai-agent-v2|ADR-0016]] settled on Vercel AI SDK + TurnContext, three follow-on needs surfaced:

1. **Latency + cost.** Each turn re-sends the static system prompt (~3KB after the decision tree expanded). At Anthropic list rates that's $0.009/turn for prompt tokens alone. Sonnet's prompt cache can read it for ~10% of write cost.
2. **One-bubble replies are ugly for lists.** A 10-item pending list as one text bubble overflows the LINE preview; sending each item as its own bubble keeps things scannable. LINE allows up to 5 messages per reply.
3. **Prompt iteration friction.** Tuning the system prompt currently requires a Vercel redeploy (3-5 min). For prompt experimentation this is enough friction to discourage tries.

## Decision

Adopt three ClaudeFlow-style patterns. Together they form the agent's "v3 layer" on top of the AI-SDK runtime.

### 1. Default model = Anthropic Sonnet 4.5 + native prompt caching

`src/lib/agent/model.ts → resolveModel()`:

- Default `LLM_MODEL = "anthropic/claude-sonnet-4-5"` (strongest tool-calling + native cache_control)
- If `ANTHROPIC_API_KEY` is set → use `@ai-sdk/anthropic` direct (`providerOptions.anthropic.cacheControl = { type: "ephemeral" }`)
- Else fall back to `@openrouter/ai-sdk-provider` (`providerOptions.openrouter.cacheControl` — works for anthropic/* models via OpenRouter)
- For non-Anthropic models (Gemini, GPT-5) no cache headers are sent

**Split system prompt:**

- **Static block** (cacheable) — static system prompt + tool decision tree. Same across all users.
- **Dynamic block** (NOT cached) — today's BKK date + per-user memory JSON. Different per user.

Result: cache hit rate ≥ 80% in steady state; observed ~50-60% input-token saving.

Cache hit detection by provider:
- Anthropic-direct → `providerMetadata.anthropic.cacheReadInputTokens`
- OpenRouter → `providerMetadata.openrouter.usage.promptTokensDetails.cachedTokens`

Captured into `meta.cacheHit` on the trace row.

### 2. Multi-bubble replies via `send_*_reply` tools

- `send_text_reply(text)` and `send_flex_reply(template, vars)` push onto `TurnContext.replyBuffer` (max 5)
- At end of turn, runtime flushes the buffer in a single `replyMessage` call
- If buffer is empty, fallback to model's free-form `result.text` as one text bubble
- 12 designer-supplied Flex templates (`src/lib/line/flex-templates/`, [#45](https://github.com/PASAKON/LungNote-webapp/pull/45)) — model picks by name (`welcome`, `todo-saved`, `todo-list-carousel`, `daily-digest`, `error-inline`, etc.) and supplies `vars` matching each template's placeholders

### 3. DB-stored system prompt with in-process cache

Table `lungnote_agent_settings` (singleton, `id='default'`):

```sql
create table lungnote_agent_settings (
  id text primary key default 'default' check (id = 'default'),
  system_prompt_override text,
  notes text,
  updated_at timestamptz default now()
);
```

`src/lib/agent/settings.ts → loadAgentSettings()`:

- Reads the row; in-process cache `CACHE_TTL_MS = 60s`
- If `system_prompt_override` is non-null → use it as the static system prompt block
- If null → fall back to hardcoded `buildStaticSystemPrompt()`

Admins can edit the prompt in Supabase Studio → live within 60s on already-warm Vercel instances; cold-starts pick it up immediately. No redeploy.

### 4. Idempotency dedup (carried in from ADR-0014)

Same PR ([#41](https://github.com/PASAKON/LungNote-webapp/pull/41)) wired `checkAlreadyProcessed` into the webhook entry point. See [[0014-observability-chat-traces|ADR-0014]] for the dedup mechanics.

## Consequences

**Positive**
- ~50-60% input-token cost saved on cache-hit turns (Sonnet steady state)
- Multi-bubble replies make list output scannable
- Prompt iteration loop reduced from "edit + commit + push + deploy" (3-5 min) to "edit row + wait ≤60s"
- Provider swap is one env var (`LLM_MODEL=google/gemini-2.5-flash`) — no code change

**Negative**
- Two-block system prompt construction is subtle — order matters for cache hit (static must come first)
- DB prompt = production-write surface for a config that materially changes bot behavior; risk of editor mistake
- In-process cache means warm Vercel instances diverge until TTL elapses — short-lived inconsistency
- OpenRouter "usage:{include:true}" must be set per-call to get cachedTokens — easy to forget on new providers

**Mitigation**
- `notes` column on `lungnote_agent_settings` for change rationale + author
- `invalidateAgentSettingsCache()` exported as a test seam / future "apply now" hook
- Trace meta records `prompt_source = "db_override" | "code"` so admin viewer shows which prompt was active
- Pricing table is hardcoded in `model.ts` per model — needs manual update on provider price changes

## Alternatives ที่ไม่เลือก

- **No prompt cache** — leaves ~50% cost on the table at our message volume
- **Single-bubble replies** — lists overflow LINE preview, UX regression
- **Env-var prompt** — requires redeploy, defeats the iteration goal
- **Edge config / external config service** — more vendors, marginal benefit
- **Per-user prompt customization** — not yet needed; user memory ([[0018-user-memory|ADR-0018]]) covers personalization without prompt sprawl

## Open questions / TODO

- [ ] Add cache hit % to admin home dashboard (currently per-row only)
- [ ] Migrate Gemini path to also set `providerOptions` once Google enables prompt cache more broadly
- [ ] Convert `notes` column into structured `changelog` array

## See Also

- [[0016-ai-agent-v2]] — runtime this layers on
- [[0014-observability-chat-traces]] — cache hit + cost land in `meta` here
- [[0018-user-memory]] — dynamic block carries this
- Code: [`src/lib/agent/{model,settings,prompt}.ts`](https://github.com/PASAKON/LungNote-webapp/tree/main/src/lib/agent), [`src/lib/line/flex-templates/`](https://github.com/PASAKON/LungNote-webapp/tree/main/src/lib/line/flex-templates)
- Anthropic prompt caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
