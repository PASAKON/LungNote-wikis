---
title: "ADR-0014: Default LLM = Gemini 2.5 Flash"
tags: [adr, ai, llm, cost]
status: Accepted
date: 2026-05-14
---

# ADR-0014 — Default LLM = Gemini 2.5 Flash

## Status

Accepted

## Context

LungNote's LINE bot routes every user message through an LLM agent
with 11 tools (save/list/complete/update/delete/profile/dashboard/
reply). The choice of "default" LLM dominates two dimensions we care
about:

- **Cost per turn** — directly maps to monthly OpEx as the friend
  count grows. Today's volume is in the dozens of turns/day; we
  expect 100–1000×.
- **Reply quality** — Thai fluency, tool-call reliability, and
  willingness to act vs. ask.

The agent originally shipped on `anthropic/claude-sonnet-4-5` (set in
the env `LLM_MODEL` default; see [[../../webapp/src/lib/agent/model.ts]]
in the webapp repo). Sonnet was chosen for strongest tool-calling +
native prompt caching, but at \$3 input / \$15 output per 1M tokens
it's the most expensive viable option.

User explicitly asked to lower cost without losing quality. We built
[[0015-agent-eval-harness|ADR-0015 — agent eval harness]] to make
the comparison evidence-based and ran 6 models against a 31-case
curated Thai corpus.

**Eval results** (Sonnet 4.5 as judge, 31 cases):

| Model | Tool match | Judge yes+partial | Cost (31 cases) | Lat p50 |
|---|---|---|---|---|
| Sonnet 4.5 | 100% | — | \$0.7833 | 7.9s |
| Gemini 2.5 Pro | 96.8% | 90.3% | \$0.2376 | 13.1s |
| GPT-4o | 87.1% | 87.1% | \$0.4088 | 3.4s |
| **Gemini 2.5 Flash** | **87.1%** | **83.9%** | **\$0.0111** | **4.4s** |
| Haiku 4.5 | 77.4% | 77.4% | \$0.2129 | 4.2s |
| GPT-4o-mini | 77.4% | 67.7% | \$0.0247 | 3.3s |

Full results checked into the webapp repo at `eval/baselines/`.

## Decision

Default `LLM_MODEL = google/gemini-2.5-flash`, routed via OpenRouter
passthrough (no native cache, doesn't matter at this price tier).

**Why Flash specifically:**

- **98.6% cheaper than Sonnet** for the same corpus — by far the
  best \$/turn at any acceptable quality.
- **84% judge equivalence with Sonnet** — the gap is concentrated in
  ~5 cases that the [[0016-intent-router|intent router]] now
  escalates to Gemini 2.5 Pro.
- **4.4s p50 latency** — comparable to Haiku, faster than Sonnet,
  much faster than Pro (13s). Stays under the LINE reply token's
  30-second window with margin.
- **Tool-calling holds up** at 87.1% match — better than Haiku,
  matches GPT-4o.

## Consequences

**Positive**

- Daily run-rate drops ~50× vs. Sonnet baseline. Friend counts in
  the thousands stay well within Vercel + OpenRouter budgets.
- Latency UX improves (4.4s vs 7.9s p50). Bot feels snappier on
  simple turns.
- Provider risk diversifies — we're not exclusively Anthropic.

**Negative**

- ~16% of turns get a strictly-worse reply vs. Sonnet (judge "no").
  Concentrates in: update-with-new-value, profile facts, multi-step
  list-then-mutate, `list_pending`-followed-by-flex.
- No prompt caching on Gemini (Anthropic-style cache_control is
  Anthropic-only). The full ~3K system prompt gets re-tokenized on
  every turn, but at \$0.075/M input the cost is negligible.
- Gemini's tool-loop sometimes terminates early after a successful
  tool call (visible in eval cases 7, 8 — `list_pending` without
  `send_flex_reply`). Not solvable by prompt alone.

## Mitigations

- [[0015-agent-eval-harness|Eval harness]] in `scripts/eval/` is
  checked into webapp. Re-runnable on every PR that touches the
  agent prompt or tool catalog to catch regressions before merge.
- [[0016-intent-router|Intent router]] sends ~5 high-failure-rate
  patterns to Gemini 2.5 Pro automatically, recovering most of the
  quality gap.
- Env-driven swap: change `LLM_MODEL` on Vercel → next request uses
  the new model. No code deploy needed. Sonnet is a single env edit
  away if Flash misbehaves on a specific cohort.

## Alternatives

- **Stay on Sonnet 4.5** — best quality, but cost makes scaling
  hostile. Friend counts in the hundreds already hit \$10+/day on
  current message volume projections.
- **Haiku 4.5** — middle ground (\$0.21 vs \$0.78 vs \$0.011 — 27%
  Sonnet cost), but eval shows 77% judge equivalence — *worse* than
  Flash on this corpus. Anthropic cache helps Haiku but doesn't
  close the gap.
- **GPT-4o** — quality matches Flash (87%), cost is 35× higher.
  Strictly dominated.
- **GPT-4o-mini** — cheapest USD but 68% judge equivalence is
  unacceptable.
- **Gemini 2.5 Pro everywhere** — best non-Anthropic quality (97%
  tool match, 90% judge), but 22× Flash cost and 3× Flash latency.
  Reserved for the intent router's escalation path.

## References

- Code: `webapp/src/lib/agent/model.ts` (`resolveModel`)
- Env: Vercel `LLM_MODEL` (production)
- Eval baselines: `webapp/eval/baselines/baseline.jsonl`
  (Sonnet) + `candidate.jsonl` (Gemini Flash) + `report.md`
- Webapp PRs: #62 (initial swap), #64 (prompt rule for `update_memory`)
