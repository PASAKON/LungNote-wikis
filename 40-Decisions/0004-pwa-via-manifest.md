---
title: "ADR-0004: PWA via Native Manifest"
tags: [adr, pwa]
status: Accepted
date: 2026-05-07
---

# ADR-0004 — PWA: Native `manifest.ts`

## Status

Accepted

## Context

ต้อง install ได้บนมือถือ (PWA). ตัวเลือก:
1. `next-pwa` (Workbox wrapper)
2. **Native Next 16 `app/manifest.ts`** + เพิ่ม service worker เมื่อจำเป็น

## Decision

เริ่มต้นด้วย `src/app/manifest.ts` (Next built-in). ยังไม่เพิ่ม service worker / offline จนกว่าจะมี use case.

Push notification + offline cache → เพิ่มใน ADR ภายหลังเมื่อจำเป็น.

## Consequences

**Positive**
- Install prompt ทำงานบน Chrome / Safari iOS 16.4+
- ไม่ต้องการ dep
- Manifest สร้างจาก Next route convention → metadata sync

**Negative**
- ยังไม่มี offline support
- Push notification ต้องเพิ่มเอง (Service Worker + Server Action) ภายหลัง

## TODO

- [ ] Generate icons (192, 512, maskable 512) → `public/`
- [ ] Add service worker เมื่อต้อง offline
- [ ] Push notification (ดู Next 16 PWA guide)

## See Also

- Next.js 16 docs: `node_modules/next/dist/docs/01-app/02-guides/progressive-web-apps.md`
