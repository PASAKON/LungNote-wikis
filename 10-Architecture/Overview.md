---
title: Architecture Overview
tags: [architecture, overview]
---

# Architecture Overview

## Stack

- **Framework**: Next.js 16 (App Router) + React 19
- **Language**: TypeScript (strict)
- **Styling**: Tailwind CSS 4 + Cardboard palette ([[../40-Decisions/0013-cardboard-theme-overhaul|ADR-0013]])
- **UI Kit**: shadcn/ui + base-ui + lucide ([[../40-Decisions/0001-stack-nextjs-typescript-tailwind|ADR-0001]])
- **i18n**: Native Next 16 dictionaries + `proxy.ts` ([[../40-Decisions/0003-i18n-native-not-next-intl|ADR-0003]])
- **PWA**: Native `manifest.ts` ([[../40-Decisions/0004-pwa-via-manifest|ADR-0004]])
- **Auth (user)**: LINE-only — Account Linking / LIFF / OAuth → Supabase session ([[../40-Decisions/0008-line-only-auth-account-linking|ADR-0008]] / [[../40-Decisions/0010-liff-auth|ADR-0010]] / [[../40-Decisions/0011-web-line-login-oauth|ADR-0011]])
- **Auth (admin)**: Supabase magic-link email + allowlist on `admin.lungnote.com` ([[../40-Decisions/0015-admin-debug-viewer|ADR-0015]])
- **DB**: Supabase Postgres + RLS ([[../40-Decisions/0006-supabase-db-auth|ADR-0006]])
- **Deploy**: Vercel + `lungnote.com` + `admin.lungnote.com` ([[../40-Decisions/0005-deploy-vercel|ADR-0005]])
- **AI Agent**: Vercel AI SDK + position-aware tools + TurnContext ([[../40-Decisions/0016-ai-agent-v2|ADR-0016]])
- **LLM provider**: Anthropic Sonnet 4.5 (default) via direct `@ai-sdk/anthropic` or OpenRouter passthrough, with prompt caching ([[../40-Decisions/0017-claudeflow-patterns|ADR-0017]])
- **Observability**: Per-turn `lungnote_chat_traces` row + admin viewer ([[../40-Decisions/0014-observability-chat-traces|ADR-0014]])

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

## Data Flow

### Web (browser / LIFF)

```
Client (Browser / LINE in-app WebView)
   ↓ HTTPS
Vercel Edge (proxy.ts → locale routing + session refresh)
   ↓
Next.js RSC / Route Handlers / Server Actions
   ↓
@supabase/ssr (cookie-based session)
   ↓
Supabase (PostgREST + Auth)
   ↓
Postgres + RLS (per-user isolation)
```

- ทุก user query ผ่าน RLS-aware Supabase client → row-level enforcement
- Auth session = HttpOnly cookie (host-scoped per subdomain), refresh ใน `proxy.ts`
- Mutation ผ่าน Server Action เป็นหลัก, fallback REST เฉพาะที่ต้อง
- Realtime / Storage = ค่อยเพิ่มเมื่อมี use case

### LINE webhook → AI Agent

```
LINE user → LINE platform → POST /api/line/webhook
   ↓ verify HMAC (LINE_CHANNEL_SECRET)
TraceCollector (one per turn)
   ↓ checkAlreadyProcessed (dedup via lungnote_chat_traces, 5-min window)
runAgent(text, TurnContext)
   ├─ loadMemory (lungnote_conversation_memory — rolling 5+5)
   ├─ loadUserMemory (lungnote_user_memory — JSONB facts)
   ├─ loadAgentSettings (lungnote_agent_settings, 60s cache)
   ├─ resolveModel (Anthropic direct OR OpenRouter, prompt cache headers)
   ├─ Vercel AI SDK generateText (maxSteps=5)
   │     ↓ tools fire ↑
   │   • save_memory / list_pending / list_done
   │   • complete_by_position / update_by_position / delete_by_position
   │   • update_memory (user facts)
   │   • send_text_reply / send_flex_reply (multi-bubble buffer)
   │   • send_dashboard_link
   ├─ flush replyBuffer (≤5 bubbles) → LINE replyMessage
   └─ trace.finalize → fire-and-forget INSERT lungnote_chat_traces
```

Service-role admin client (`SUPABASE_SECRET_KEY`) is used for the 5 server-only tables. End-user-owned tables (`lungnote_todos` / `lungnote_notes` / `lungnote_profiles`) use the user-scoped client where possible so RLS stays the source of truth.

## See Also

- [[../20-Conventions/Code-Style]]
- [[../40-Decisions/README]]
