---
title: Architecture Overview
tags: [architecture, overview]
---

# Architecture Overview

## Stack

- **Framework**: Next.js 16 (App Router) + React 19
- **Language**: TypeScript (strict)
- **Styling**: Tailwind CSS 4
- **UI Kit**: shadcn/ui ([[../40-Decisions/0001-stack-nextjs-typescript-tailwind|ADR-0001]])
- **i18n**: Native Next 16 dictionaries + `proxy.ts` ([[../40-Decisions/0003-i18n-native-not-next-intl|ADR-0003]])
- **PWA**: Native `manifest.ts` ([[../40-Decisions/0004-pwa-via-manifest|ADR-0004]])
- **Auth**: Supabase Auth + RLS ([[../40-Decisions/0006-supabase-db-auth|ADR-0006]])
- **DB**: Supabase Postgres + RLS ([[../40-Decisions/0006-supabase-db-auth|ADR-0006]])
- **Deploy**: Vercel + `lungnote.com` ([[../40-Decisions/0005-deploy-vercel|ADR-0005]])

## i18n Routing

```
/           → proxy redirect → /<accept-language-locale>
/en/...     → English
/th/...     → Thai
```

- Locales config: `webapp/src/i18n/config.ts`
- Dictionary loader: `webapp/src/i18n/dictionaries.ts`
- Translation files: `webapp/messages/<locale>.json`
- Routing logic: `webapp/src/proxy.ts`

## Folder Layout (root)

Local parent dir = working aggregation. แต่ละ subfolder = GitHub repo แยก ([[../40-Decisions/0002-split-repos-webapp-wikis-design|ADR-0002]]):

```
LungNote Projects/
├── webapp/        # ↔ PASAKON/LungNote-webapp — Next.js app
├── wikis/         # ↔ PASAKON/LungNote-wikis — Obsidian docs vault
├── design/        # ↔ PASAKON/LungNote-design — HTML mockups
├── CLAUDE.md      # mirror ใน webapp repo (primary)
└── README.md
```

## Boundary Rule

- `webapp/` ไม่อ้าง `wikis/` หรือ `design/` ใน runtime — เป็น dev-only context
- `wikis/` ไม่ commit code snippet ที่ผูกกับ implementation — link GitHub URL ของ `LungNote-webapp` แทน
- Cross-repo refer: ใช้ GitHub URL (เช่น `https://github.com/PASAKON/LungNote-webapp/blob/main/...`) ไม่ใช่ relative path เพราะ clone แยกได้

## Data Flow (skeleton)

```
Client (Browser/PWA)
   ↓ HTTPS
Vercel Edge (proxy.ts → locale routing)
   ↓
Next.js RSC / Route Handlers / Server Actions
   ↓
@supabase/ssr (cookie-based session)
   ↓
Supabase (PostgREST + Auth)
   ↓
Postgres + RLS (per-user isolation)
```

- ทุก query ผ่าน Supabase client → RLS enforce ระดับแถว
- Auth session = HttpOnly cookie, refresh ใน middleware (proxy.ts)
- Mutation ผ่าน Server Action เป็นหลัก, fallback REST เฉพาะที่ต้อง
- Realtime / Storage = ค่อยเพิ่มเมื่อมี use case

## AI Agent

LungNote's LINE bot routes every user message through a tool-using
LLM agent at `webapp/src/lib/agent/`. The agent exposes 11 tools
(save / list / complete / uncomplete / update / delete / save_note
/ dashboard_link / update_memory / send_text_reply / send_flex_reply).

Model selection is layered:

```
LINE webhook
   │
   ▼
runAgent(userText, ctx)
   │
   ├─→ routeModel(userText)      ← heuristic intent router
   │       └─→ { modelId, reason }     (ADR-0016)
   │
   ├─→ resolveModel(modelId)     ← provider routing
   │       ├─ anthropic/*  + ANTHROPIC_API_KEY → Anthropic direct (cache on)
   │       └─ else                              → OpenRouter passthrough
   │
   └─→ generateText(...) → tool loop → flush bubbles → reply
```

- **Default model**: `google/gemini-2.5-flash` ([[../40-Decisions/0014-llm-default-gemini-2-5-flash|ADR-0014]]).
  Set via Vercel env `LLM_MODEL`. ~98% cheaper than Sonnet, 4.4s p50
  latency on the eval corpus.
- **Intent router**: `webapp/src/lib/agent/router.ts`
  ([[../40-Decisions/0016-intent-router|ADR-0016]]). Heuristic
  regex+length classifier. Escalates ~5 high-failure patterns
  (update verbs, profile facts, multi-position, long messages,
  multi-clause) to Gemini 2.5 Pro. Toggle via `LLM_ROUTER_ENABLED=true`.
- **Eval harness**: `webapp/scripts/eval/`
  ([[../40-Decisions/0015-agent-eval-harness|ADR-0015]]). 31-case
  curated Thai corpus, mocked tools (no Supabase / no LINE), LLM
  judge for reply equivalence, Markdown reports. Baseline snapshots
  in `webapp/eval/baselines/`.

**Trace observability**: every turn writes one row to
`lungnote_chat_traces`. The admin viewer at `admin.lungnote.com/traces`
shows model id + a route-reason pill + tool calls + reply.

## See Also

- [[../20-Conventions/Code-Style]]
- [[../40-Decisions/README]]
- [[../40-Decisions/0014-llm-default-gemini-2-5-flash|ADR-0014 — Default LLM]]
- [[../40-Decisions/0015-agent-eval-harness|ADR-0015 — Eval Harness]]
- [[../40-Decisions/0016-intent-router|ADR-0016 — Intent Router]]
