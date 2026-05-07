---
title: "ADR-0003: i18n — Native Next 16 (no next-intl)"
tags: [adr, i18n]
status: Accepted
date: 2026-05-07
---

# ADR-0003 — i18n: Native Next 16 (no next-intl)

## Status

Accepted

## Context

ต้อง multi-language. ตัวเลือกหลัก:
1. **next-intl** (lib ยอดนิยม)
2. **Native Next 16** (`[locale]` segment + dictionaries + `proxy.ts`)

Next 16 มี breaking change: `params` เป็น async, `middleware` → `proxy`. lib ภายนอกอาจตามไม่ทัน.

## Decision

ใช้ **native Next 16 i18n pattern**:

- `src/i18n/config.ts` — locales list (`en`, `th`)
- `src/i18n/dictionaries.ts` — server-only dictionary loader
- `src/proxy.ts` — locale routing (Accept-Language → redirect)
- `src/app/[locale]/...` — all pages nested
- `messages/<locale>.json` — translation files

Server Components อ่าน dictionary ตรง — bundle ไม่โต.

## Consequences

**Positive**
- 0 dependency เพิ่ม (`next-intl` ถูกถอน)
- ตามแนวทาง Next 16 official
- Server-only translations → ไม่กิน client bundle
- Type-safe ผ่าน `Dictionary` type

**Negative**
- ต้องเขียน plural / date-format เอง (ใช้ `Intl.*` native ได้)
- ไม่มี client-side hook สำหรับ Client Component → ต้อง pass dict ผ่าน prop หรือ context

## Alternatives

- **next-intl** — feature ครบ แต่ต้องรอ Next 16 compat verify, เพิ่ม dep
- **react-i18next** — ใช้ client-heavy, ขัดกับ RSC

## See Also

- [[0001-stack-nextjs-typescript-tailwind]]
- Next.js 16 docs: `node_modules/next/dist/docs/01-app/02-guides/internationalization.md`
