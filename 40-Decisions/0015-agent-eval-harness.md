---
title: "ADR-0015: Agent Eval Harness"
tags: [adr, ai, testing, eval]
status: Accepted
date: 2026-05-14
---

# ADR-0015 — Agent Eval Harness

## Status

Accepted

## Context

We needed to swap LLMs ([[0014-llm-default-gemini-2-5-flash|ADR-0014]])
without manually retesting the full agent surface every time. Live
A/B isn't a fit here — turn volume is too low and the failure modes
(silent tool drops, wrong reply tone, missing flex bubble) aren't
catchable from production telemetry alone.

Requirements:

- Reproducible: same model → same input → same scoring.
- No prod side effects: replays must not write to Supabase, send LINE
  messages, or touch real user memory.
- Covers every tool ≥ 3× plus adversarial cases (prompt injection,
  ambiguous saves, out-of-range positions, mixed-language input).
- Side-by-side: emit a Markdown report comparing two models with
  per-case diff so the failure list is human-reviewable.

## Decision

A standalone Vercel-AI-SDK-driven eval system in
`webapp/scripts/eval/`, parallel to the prod runtime (not replacing it).

```
scripts/eval/
├── types.ts          TestCase / CaseRun / CaseScore contracts
├── cases.ts          ~30 curated cases (Thai-first)
├── mock-tools.ts     Vercel AI SDK tools mirroring src/lib/agent/tools/*
│                     schema-for-schema. No DB, in-memory state per case.
├── model-factory.ts  Anthropic-direct / OpenRouter resolution
│                     (no `"server-only"` so tsx can import it)
├── runner.ts         CLI: replays cases through baseline + candidate,
│                     writes JSONL
├── judge.ts          LLM judge — uses a stronger model (Sonnet) to
│                     score reply equivalence per case
└── report.ts         Markdown summary + per-case table + failure
                      callouts
```

**Scoring axes**:

| Axis | Type | Method |
|---|---|---|
| `tool_match` | deterministic | set of tool names == expected |
| `tool_args` | deterministic | per-tool predicate over captured args |
| `reply_match` | deterministic | regex must-hit / must-not-hit |
| `bubble_count` | deterministic | exact when asserted |
| `judge` | LLM (Sonnet 4.5) | yes / partial / no equivalence rating |
| `cost / latency` | numeric | strict diff vs baseline |

**Mock strategy**: each tool wraps a small in-memory `MockState`
({ pending todos, done todos, user memory, reply bubbles }). Per-case
preState gives the bot a known starting list. Tool calls captured into
a `CallRecorder` so the runner reconstructs the exact agent sequence
afterwards. No Supabase, no LINE webhook, no real cost beyond model
inference.

**Coverage baseline** (checked in at `webapp/eval/baselines/`):

- `baseline.jsonl` — Sonnet 4.5 over the 31 cases
- `candidate.jsonl` — Gemini 2.5 Flash (post prompt-rule fix)
- `judge.jsonl` — Sonnet's reply-equivalence verdicts
- `report.md` — human-readable summary
- `router-v2/` — same 31 cases with the [[0016-intent-router|intent router]] on

These get re-generated on any PR that touches the agent prompt, the
tool catalog, or `LLM_MODEL`. The artifacts let future contributors
see "did this PR regress against the known-good Sonnet baseline".

## Consequences

**Positive**

- Model swaps become an evidence-based call. ADR-0014 cites the eval
  table directly.
- ~30 known-good test points act as regression coverage for the agent
  surface. Without this, prompt edits ship blind.
- Cost of one full eval run: ~\$0.50. Cheap enough to run on every
  agent-touching PR.
- The mock harness doubles as documentation of the tool schemas
  (re-stating them in `mock-tools.ts` for tsx compat).

**Negative**

- Mocks mirror prod tools at the *schema* level — not 100% of runtime
  quirks (e.g. real `complete_by_position` re-fetches the pending
  list mid-turn from Supabase). Behavioral edges that depend on a
  tool's internal DB access pattern can slip past.
- The LLM judge is opinionated. Single-call equivalence rating
  flips on borderline cases. Acceptable for "is this model
  comparable?" decisions but not for fine-grained quality regression
  catching.
- Maintenance: each new tool needs a matching mock entry. Caught
  once when `save_note` shipped (webapp PR #59) — added retroactively.
- Real-history sampling not yet in. The curated corpus represents
  ~30 patterns; real chat data may surface more.

## Alternatives

- **Vercel staging deploy + manual smoke test** — what we'd done
  before. Slow, missed regressions, blocked the developer for
  ~15 min per attempt. Doesn't scale to multi-model comparison.
- **Unit-test the agent runtime directly** — too coupled to the real
  Supabase admin client + LINE client; tests become integration
  tests with real network. The eval harness mocks the *boundary*
  (tools) cleanly.
- **Use an open-source eval framework** (e.g. promptfoo, Helicone
  Helix) — overkill for our scope. The harness is < 800 LOC,
  understood top-to-bottom by anyone reading the directory. No new
  dependency.

## References

- Code: `webapp/scripts/eval/` (runner, judge, report)
- Baseline artifacts: `webapp/eval/baselines/`
- Run cost: ~\$0.50 per full sweep (Sonnet baseline + Flash candidate
  + Sonnet judge)
- Webapp PR: #63 (initial harness), #64 (prompt fix derived from a
  failing case), #65 ([[0016-intent-router|intent router]])
