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

## See Also

- [[../20-Conventions/Code-Style]]
- [[../40-Decisions/README]]
