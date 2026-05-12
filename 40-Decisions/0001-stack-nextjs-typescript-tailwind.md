---
title: "ADR-0001: Stack — Next.js + TypeScript + Tailwind"
tags: [adr, stack]
status: Accepted
date: 2026-05-07
---

# ADR-0001 — Stack: Next.js + TypeScript + Tailwind

## Status

Accepted

## Context

ต้องการ webapp ที่:
- Mobile-first
- มาตรฐานสากล (i18n, accessibility, SEO)
- Ecosystem ใหญ่ — หา dev ง่าย, library พร้อม
- Deploy ง่าย

## Decision

ใช้ **Next.js 16 (App Router) + TypeScript + Tailwind CSS 4**.

UI: shadcn/ui. i18n: next-intl. Package manager: pnpm.

## Consequences

**Positive**
- React ecosystem ใหญ่ที่สุด
- SSR/RSC → SEO + first-paint ดี
- Vercel deploy 1 command
- shadcn = component สวย ไม่ผูก vendor

**Negative**
- React 19 + Next 16 ใหม่ — บาง lib ยังไม่ support เต็ม
- Tailwind 4 syntax เปลี่ยน

## Alternatives

- **Remix** — web standards ดี แต่ ecosystem เล็ก
- **SvelteKit** — เร็ว เบา หา dev ยาก
- **Astro** — เน้น static, ไม่เหมาะ app interactive

## See Also

- [[0003-i18n-native-not-next-intl]] — i18n decision reversed: native Next 16 dictionaries แทน `next-intl` ที่ระบุไว้เดิม
- [[0013-cardboard-theme-overhaul]] — brand palette ปัจจุบัน (cardboard) — UI Kit ยังคงเป็น shadcn/ui
