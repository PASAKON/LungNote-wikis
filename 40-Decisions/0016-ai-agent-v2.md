---
title: "ADR-0016: AI agent v2 — Vercel AI SDK + position-aware tools + TurnContext"
tags: [adr, ai, agent, line]
status: Accepted
date: 2026-05-12
---

# ADR-0016 — AI agent v2: Vercel AI SDK + position-aware tools

## Status

Accepted (backfilled — feature shipped in [LungNote-webapp#35](https://github.com/PASAKON/LungNote-webapp/pull/35) on 2026-05-09)

## Context

ADR-0012's Phase 2 ("tool-calling AI") was implemented with hand-rolled OpenAI-style tool loops over the OpenRouter REST API. Two pain points emerged:

1. **Provider lock-in to OpenAI tool format.** Anthropic + Google + OpenAI return tool calls in different shapes; the hand-rolled loop branched per provider, complex to maintain.
2. **Stale UUIDs / position confusion.** The agent was passing raw `todo_id` UUIDs around in tool calls. When the user said "ลบอันที่ 3" the model had to remember which UUID was the 3rd item from its last list — it often got this wrong (stale ids, drift across turns).

Need: a provider-agnostic tool runtime, and an architecture where the server (not the model) owns position → ID resolution.

## Decision

### Use Vercel AI SDK (`ai` package)

- `generateText({ model, messages, tools, stopWhen, maxOutputTokens })` — handles tool-call schema differences across Anthropic / OpenAI / Google / OpenRouter
- Tools declared via `tool({ description, inputSchema: z.object(...), execute: async (args) => ... })`
- `maxSteps = 5` (each step = one model call + zero-or-more tool executions)
- `maxOutputTokens = 1024`

Code: `src/lib/agent/runtime.ts` (`runAgent`).

### TurnContext = per-turn server-side working memory

`src/lib/agent/context.ts`. Constructed once per webhook event, passed into every tool's `execute` via the registry. Holds:

- **Pending list cache** — populated by `list_pending` tool; subsequent `*_by_position(n)` tools resolve `n → UUID` server-side
- **Done list cache** — same shape, populated by `list_done`
- **Reply bubble buffer** — multi-bubble (text + flex) pattern; LINE cap = 5 messages per reply
- **`lineUserId`** — scope all DB queries
- **`trace`** — TraceCollector from [[0014-observability-chat-traces|ADR-0014]]

**Key insight:** the model no longer carries UUIDs across tool calls. It says "delete position 3"; the server reads `ctx.getPendingByPosition(3)` and deletes the right row. Stale-id bug class eliminated.

### Tool set (`src/lib/agent/tools/`)

| Tool | Purpose |
|---|---|
| `save_memory(text, due_at?, due_text?)` | Insert into `lungnote_todos` (source='chat') |
| `list_pending()` | Fetch + cache open items |
| `list_done()` | Fetch + cache done items |
| `complete_by_position(position)` | Mark `position`-th open todo done |
| `uncomplete_by_position(position)` | Reverse complete |
| `update_by_position(position, text, due_at?, due_text?)` | Edit |
| `delete_by_position(position)` | Delete |
| `send_text_reply(text)` | Buffer one text bubble |
| `send_flex_reply(template, vars)` | Buffer one flex bubble — see [[0017-claudeflow-patterns]] |
| `send_dashboard_link()` | Mint + reply auth-link Flex |
| `update_memory(action, key, value)` | Persistent user memory — see [[0018-user-memory]] |

**Auto-fetch guard:** `*_by_position` tools that fire before `list_pending`/`list_done` was called auto-trigger a list fetch (single shared promise across parallel calls) so the model can call them in any order. See [`#36`](https://github.com/PASAKON/LungNote-webapp/pull/36).

### Asymmetric bias (from [#37](https://github.com/PASAKON/LungNote-webapp/pull/37))

System prompt instructs:
- **Reads are eager** — when in doubt about what the user has, call `list_pending` rather than ask
- **Writes are cautious** — never `save_memory("ทดสอบ")`; never write on ambiguous intent. Tool layer hard-blocks save of `"ทดสอบ"`, `"test"`, etc. (see [`#39`](https://github.com/PASAKON/LungNote-webapp/pull/39))

### Routing

`AI_AGENT_MODE` env (default `"true"`). When `"true"`, all text events go to the agent — no regex shortcuts. When `"false"`, falls back to ADR-0012 regex+AI hybrid as a rollback safety net.

## Consequences

**Positive**
- Position-based tools eliminated the stale-id bug class entirely
- TurnContext means tools are pure functions of `(args, ctx)` — easy to unit test
- AI SDK abstraction lets us swap LLM provider via `LLM_MODEL` env (see [[0017-claudeflow-patterns|ADR-0017]])
- `maxSteps=5` caps cost per turn deterministically

**Negative**
- One more abstraction layer (registry, context, model resolver) — newcomer ramp ~1 hour
- AI SDK does not yet support every provider quirk natively (cache control needs `providerOptions` plumbing — ADR-0017)
- Multi-bubble reply requires tool calls instead of free-form text → model needs explicit prompt instruction to use `send_*_reply` (drift-prone)

**Mitigation**
- Integration test harness — Claude scores LungNote agent on 4+ regression scenarios (`webapp/src/__tests__/integration/`, [#38](https://github.com/PASAKON/LungNote-webapp/pull/38))
- Trace every turn with `TraceCollector` so regressions surface in admin viewer

## Alternatives ที่ไม่เลือก

- **Hand-rolled OpenAI-style tool loop** — status quo before #35; provider-coupled, hard to maintain
- **LangChain / LangGraph** — heavyweight, opinionated state model, mismatch with our short-turn LINE shape
- **OpenAI Assistants API** — vendor lock-in to OpenAI; we want Anthropic-default + provider swap

## See Also

- [[0012-unified-todo-memory-model]] — original tool list (Phase 2) this supersedes the runtime of
- [[0014-observability-chat-traces]] — every turn produces a trace row
- [[0017-claudeflow-patterns]] — Anthropic prompt cache + multi-bubble + DB-stored prompt sit on top of this runtime
- [[0018-user-memory]] — `update_memory` tool surface
- Code: [`src/lib/agent/`](https://github.com/PASAKON/LungNote-webapp/tree/main/src/lib/agent)
- Vercel AI SDK: https://sdk.vercel.ai/docs
