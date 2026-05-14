---
title: "ADR-0016: Intent Router — Flash default + Pro escalation"
tags: [adr, ai, llm, cost, routing]
status: Accepted
date: 2026-05-14
---

# ADR-0016 — Intent Router

## Status

Accepted

## Context

[[0014-llm-default-gemini-2-5-flash|ADR-0014]] put Gemini 2.5 Flash
in the hot path because of its huge cost/latency win. But the
[[0015-agent-eval-harness|eval]] showed Flash sits at ~84% judge
equivalence with Sonnet 4.5 over our 31-case corpus, and the misses
cluster in a handful of patterns prompt tuning alone can't move:

1. **Update verbs with a new value** ("เลื่อนข้อ 2 เป็นพฤหัส") —
   Flash often refuses as "ambiguous" even when position + new value
   are both explicit.
2. **Profile facts** ("ฉันอยู่กรุงเทพ") — Flash acknowledges in text
   ("รับทราบครับ") but doesn't actually call `update_memory`.
   The fact is *lost*.
3. **Multi-step "ดูลิสต์ แล้ว ติ๊ก 4"** — Flash sometimes stops
   after the list call without doing the mutation.
4. **`list_pending` with N>0** — Flash sometimes ends the turn
   without rendering the `todo_list` flex card.

Gemini 2.5 Pro handles all four correctly (96.8% tool match in the
eval) but costs ~22× Flash per turn and runs at ~13s p50 instead of
4.4s. Routing every turn to Pro is overkill; routing none of them is
what ADR-0014 alone gives us.

## Decision

A per-turn **heuristic router** picks the model. Default to Flash;
escalate to Pro when the user message matches one of the patterns
Flash empirically fails on. Zero LLM-call overhead — pure regex
+ length heuristics, runs in < 1 ms.

```
User text → routeModel() → { modelId, reason }
                           ↓
                  resolveModel(modelId) → generateText(...)
                                       → log meta.routeReason
                                       → admin trace UI shows pill
```

**Escalation triggers** (in priority order; first match wins):

| Trigger | Pattern | Example |
|---|---|---|
| `update_verb` | เลื่อน / แก้ / เปลี่ยน / ย้าย / update / rename | "เลื่อนข้อ 2 เป็นพฤหัส" |
| `profile_fact` | ฉันชื่อ / ฉันอยู่ / ฉันเรียน / วันเกิด / เรียก{ฉัน,ผม} | "ฉันอยู่กรุงเทพ" |
| `multi_position` | ≥ 2 numbers + connector (และ / กับ / พร้อม / ทั้ง) | "ลบ 1 และ 3" |
| `long_message` | > 150 chars | multi-clause turn |
| `complex_clause` | ดู/ลิสต์ ... แล้ว ... ติ๊ก/ลบ/แก้ (narrowed to avoid the completed-action phrase "เสร็จแล้ว") | "ดูลิสต์ แล้วติ๊ก 4" |

All other turns get `default` and use the fast model.

**Toggle**:

```
LLM_ROUTER_ENABLED=true        # default off (was off pre-deploy)
ROUTER_FAST_MODEL=...          # default google/gemini-2.5-flash
ROUTER_COMPLEX_MODEL=...       # default google/gemini-2.5-pro
```

Off by default keeps the rollout reversible — unset the flag at
Vercel and the runtime ignores the router entirely.

## Eval results

31-case corpus, Sonnet 4.5 as judge. Full report at
`webapp/eval/baselines/router-v2/report.md`.

| Config | Tool match | Judge yes+partial | Yes | Cost / 31 | Lat p50 |
|---|---|---|---|---|---|
| Sonnet 4.5 only | 100% | — | — | \$0.7833 | 7.9s |
| Flash only (ADR-0014) | 87.1% | 83.9% | 20 | \$0.0111 | 4.4s |
| **Router (this ADR)** | **93.5%** | **87.1%** | **20** | **\$0.0310** | 7.5s |
| Pro only | 96.8% | 90.3% | 21 | \$0.2376 | 13.1s |

Router rescued 3 patterns that Flash dropped:
- `update_text_position_2` — Flash refused → Pro updated
- `update_due_position_3` — Flash returned no tool → Pro updated
- `profile_set_timezone` — Flash lied ("จำให้") → Pro stored the fact

**Update (2026-05-15, webapp PR #67) — empty-reply rescue path**

The "still failing" case below got a follow-up fix. The router can
only escalate on patterns it can see in the user text; the
list-then-no-flex failure happens *after* the tool call, so the
router can't reach it. Instead, `runAgent` now retries once with
the complex model when:

- the bubble buffer is empty + `result.text` is empty;
- at least one tool ran and every tool that ran is read-only
  (`list_pending` / `list_done`);
- the model used was the fast tier.

The retry uses the prior tool messages (so it doesn't re-fetch the
list) and a **reply-only tool subset** (`send_text_reply` +
`send_flex_reply`), capped at `maxSteps: 2`. Cost/tokens fold into
the trace totals; `meta.route_reason` becomes `"<reason>+retry"`.

With the rescue path on, the same 31-case eval moves to:

| Config | Tool match | Judge yes+partial | Cost | Lat p50 |
|---|---|---|---|---|
| Router only | 93.5% | 87.1% | \$0.031 | 7.5s |
| **Router + retry** | **96.8%** | **87.1%** | \$0.041 | **3.7s** |
| Pro only | 96.8% | 90.3% | \$0.238 | 13.1s |

Tool match now matches Pro-only quality at ~17× lower cost. Latency
p50 *improves* vs router-only (3.7s vs 7.5s) — most turns succeed
first try; only the dropped-flex stragglers escalate. Toggle via
`LLM_REPLY_RETRY_ENABLED=false`. Eval baseline checked in at
`webapp/eval/baselines/router-retry/`.

## Consequences

**Positive**

- Quality lifts 84% → 87% (judge yes+partial), 87% → 93.5% tool match.
- Cost still 25× cheaper than Sonnet (\$0.031 vs \$0.78 on the
  31-case sweep). 3× Flash spend in absolute terms — within budget.
- Trace observability: every turn records its router decision in
  `meta.route_reason` (default / update_verb / profile_fact / ...).
  The admin trace viewer (`/admin/traces`) shows a pill per row.
- Zero runtime cost for the routing decision itself (heuristic, no
  LLM call). Falls back to fast model on any matcher exception.

**Negative**

- p50 latency lifts 4.4s → 7.5s — escalated turns hit Pro's ~13s
  tail. The bulk of turns still run at Flash speed.
- Brittle regex coverage. Future Thai phrasings not in our patterns
  will silently fall through to Flash. The eval baseline catches
  this on next harness run; nothing catches it pre-deploy.
- Heuristic mispredicts both directions. False positives (escalating
  a simple turn to Pro) cost ~22× per turn; false negatives
  (keeping a complex turn on Flash) silently degrade quality. The
  baselines we checked in let us watch for drift.

## Mitigations

- 13 unit tests in `webapp/src/__tests__/lib/agent/router.test.ts`
  exercise each trigger + the false-positive guards ("เสร็จแล้ว"
  must NOT escalate).
- `eval/baselines/router-v2/` is the regression baseline for the
  router config. New triggers must keep this set passing.
- Toggle is one env edit away from rollback.

## Alternatives

- **LLM classifier** (Flash-Lite as a router) — \$0.0001/turn
  overhead but +500ms-1s latency per turn for the classifier call.
  Slightly smarter on novel phrasings, but heuristic catches the
  known cases for free.
- **Post-hoc retry** (run Flash, if `bubbles.length === 0` retry
  with Pro) — addresses the `list_pending` failure the heuristic
  can't catch, but doubles latency on retries and complicates
  state. Possible future addition layered on top, not instead.
- **Use Pro for everything** — clean code, 96.8% quality, but \$0.24
  per 31 cases vs \$0.031 — 8× the router cost. Latency 13s p50 is
  the dealbreaker.
- **Stay on ADR-0014's prompt-only fix** — proven to plateau at 84%
  on this corpus.

## References

- Code: `webapp/src/lib/agent/router.ts`
- Trace UI: `webapp/src/app/admin/(protected)/traces/[id]/page.tsx`
  ("Model" row shows route_reason pill)
- Unit tests: `webapp/src/__tests__/lib/agent/router.test.ts` (13 cases)
- Eval baseline: `webapp/eval/baselines/router-v2/`
- Env: Vercel `LLM_ROUTER_ENABLED`, `ROUTER_FAST_MODEL`,
  `ROUTER_COMPLEX_MODEL`
- Webapp PR: #65
