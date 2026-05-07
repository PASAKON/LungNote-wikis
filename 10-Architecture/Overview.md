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
- **Auth**: TBD ([[../40-Decisions/README|ADR pending]])
- **DB**: TBD ([[../40-Decisions/README|ADR pending]])
- **Deploy**: TBD (Vercel default)

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

```
LungNote Projects/
├── webapp/        # Next.js app — code only
├── wikis/         # docs vault (Obsidian) — context only
├── CLAUDE.md      # LLM/dev rules
└── README.md
```

## Boundary Rule

- `webapp/` ไม่อ้าง `wikis/` ใน runtime — wikis เป็น dev-only context
- `wikis/` ไม่ commit code snippet ที่ผูกกับ implementation จริง — link ไป file ใน `webapp/` แทน

## Data Flow (skeleton)

ยังไม่กำหนด. เพิ่มเมื่อมี first feature.

## See Also

- [[../20-Conventions/Code-Style]]
- [[../40-Decisions/README]]
