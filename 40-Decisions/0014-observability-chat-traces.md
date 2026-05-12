---
title: "ADR-0014: Observability — chat traces + idempotency dedup"
tags: [adr, observability, telemetry, line]
status: Accepted
date: 2026-05-12
---

# ADR-0014 — Observability: chat traces + idempotency dedup

## Status

Accepted (backfilled — feature shipped in [LungNote-webapp#25](https://github.com/PASAKON/LungNote-webapp/pull/25) on 2026-05-09)

## Context

Once the LINE bot started running tool-calling AI ([[0016-ai-agent-v2|ADR-0016]] and predecessors), debugging "why did the bot say X to user Y" became hard:

- Vercel function logs are flat text, hard to correlate by user / turn
- Tool calls + arguments + results are lost after the request returns
- LINE webhook redelivers events on timeout — without dedup the bot replies twice and double-saves todos

ADR-0007's TODO mentioned a generic `lungnote_audit_log` table, but row-level mutation logging is wrong shape for AI turns: we want **per-turn** observability — what user said, which path took, what tools fired, what reply went out, how much it cost.

## Decision

Ship two server-side primitives:

### 1. `lungnote_chat_traces` table — one row per webhook turn

Migration: `webapp/supabase/migrations/20260509070000_lungnote_chat_traces.sql`.

```sql
create table lungnote_chat_traces (
  id uuid primary key,
  trace_id text not null,             -- = LINE message.id (unique per turn)
  line_user_id text,
  user_text text not null,
  path text not null check (
    path in ('dashboard','list','memory','regex','ai','error')
  ),
  history_count int default 0,
  ai_iterations int default 0,
  tool_calls jsonb,                   -- [{name, args, result}]
  reply_text text,
  meta jsonb,                         -- {model, tokens_in, tokens_out, cost_usd, latency_ms}
  error_text text,
  created_at timestamptz default now()
);
```

RLS on, **no policies** — reads only via service-role client behind the admin gate ([[0015-admin-debug-viewer|ADR-0015]]).

### 2. `TraceCollector` class — pipeline-wide collector

`src/lib/observability/trace.ts`. One instance per webhook event. API:

- `step(name, data)` — emit structured `console.log` JSON line (cheap, immediate, filterable by `trace_id` in Vercel logs)
- `recordTool(name, args, result)` — buffer tool call detail
- `finalize(opts)` — fire-and-forget insert one trace row at end of turn

Insert is best-effort. DB outage **never** blocks the LINE reply.

### 3. Idempotency dedup

`src/lib/observability/dedup.ts`. `checkAlreadyProcessed(messageId)`:

- Lookup `lungnote_chat_traces` row with `trace_id = messageId`, `created_at` within last **5 minutes** (LINE redelivery window), `reply_text` non-null, `error_text` null
- Match → skip reprocessing (would double-reply / double-save)
- No match / DB error → assume fresh, continue
- Caveat: trace insert is async, so a redelivered event that arrives before the first trace row lands re-processes. Accepted trade-off — double reply is annoying; missed reply during DB outage is worse.

## Consequences

**Positive**
- Single-row per turn = clean admin viewer schema
- Per-turn cost telemetry (`meta.cost_usd`) enables daily/weekly billing review
- `console.log` JSON + filterable `trace_id` works as cheap real-time debug without DB
- Dedup eliminates the most common production bug (LINE retries)

**Negative**
- Trace row adds ~1KB per turn — at 1000 turns/day = 30MB/mo, fits Supabase free tier easily, but watch growth
- `tool_calls` jsonb can balloon if tool args/results aren't trimmed — hard cap at 4000 chars for `user_text` + `reply_text` in `insertTrace`
- Dedup race: redelivery within ~1s of original may slip through (trace not yet inserted)

**Mitigation**
- Add `created_at < now() - interval '30 days'` cleanup job if free tier nears limit
- Dedup race accepted — frequency is low; impact = duplicate reply, not data corruption

## Alternatives ที่ไม่เลือก

- **Vercel logs only** — text-only, no structured query, no cost rollup
- **External APM (Sentry / DataDog)** — overkill for free-tier traffic, vendor lock-in
- **Per-tool row** — too many rows, hard to reconstruct full turn

## See Also

- [[0007-line-oa-messaging]] — webhook handler this hooks into
- [[0015-admin-debug-viewer]] — consumer of these rows
- [[0016-ai-agent-v2]] — agent path generates ai_iterations + tool_calls
- [[../20-Conventions/Database-Naming]] §9 — generic audit_log pattern (not what shipped)
- Code: [`src/lib/observability/{trace,dedup}.ts`](https://github.com/PASAKON/LungNote-webapp/tree/main/src/lib/observability), [`src/lib/admin/traces.ts`](https://github.com/PASAKON/LungNote-webapp/blob/main/src/lib/admin/traces.ts)
