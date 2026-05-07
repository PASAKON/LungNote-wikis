---
title: "ADR-0005: Deploy — Vercel + lungnote.com"
tags: [adr, deploy, infra]
status: Accepted
date: 2026-05-07
---

# ADR-0005 — Deploy: Vercel + lungnote.com

## Status

Accepted

## Context

ต้อง host webapp ที่ build บน Next.js 16. Domain `lungnote.com` จดทะเบียน Namecheap, ปัจจุบัน DNS ชี้ park IP `192.64.119.26`.

ตัวเลือก:
1. **Vercel** — ผู้สร้าง Next.js, deploy 1-click, edge network ฟรี hobby
2. **Cloudflare Pages** — เร็ว, ฟรี รวม DDoS protection, แต่ Next 16 server features (RSC, server actions) ต้อง Workers compat layer
3. **Railway / Render / Fly** — VPS-like, ต้องตั้ง Dockerfile, แพงกว่า hobby
4. **Self-host (VPS)** — flexible แต่ devops overhead สูง

## Decision

ใช้ **Vercel Hobby** (free tier) เป็นจุดเริ่ม. Deploy via GitHub integration — push `main` ของ `PASAKON/LungNote-webapp` → auto deploy production. (ดู [[0002-split-repos-webapp-wikis-design]] สำหรับ split-repo layout — Vercel ดู repo เดียว.)

Custom domain `lungnote.com` + `www.lungnote.com` (`www` → 308 → apex). DNS เปลี่ยนจาก Namecheap park → Vercel:
- `A` apex → `76.76.21.21`
- `CNAME www` → `cname.vercel-dns.com`

(หรือ Namecheap delegate NS → Vercel ทั้งโดเมน — ทางเลือกที่สอง)

## Consequences

**Positive**
- Next 16 features ทุกอย่างทำงานเต็ม (RSC, ISR, server actions, image opt, edge proxy)
- Preview deploy ต่อ PR ฟรี
- Edge network global, ไม่ต้องตั้ง CDN เพิ่ม
- Hobby ฟรีสำหรับ traffic ระดับ launch

**Negative**
- Vendor lock-in (Vercel-specific edge config, image opt)
- Hobby tier มี limit (100 GB bandwidth/mo, no team) — ต้องอัพเป็น Pro ($20/mo) ก่อน scale
- ราคาขยายเร็วถ้าโดน botted traffic — ต้องตั้ง budget alert + WAF rule

## Alternatives

- **Cloudflare Pages** — รอ Next 16 compat verified ก่อน
- **Self-host** — over-engineering ระยะนี้

## TODO

- [x] Connect Vercel project → GitHub repo `PASAKON/LungNote-webapp` (root = `/`)
- [x] Add custom domain `lungnote.com` + `www.lungnote.com`
- [x] อัพเดท DNS A `@` + A `www` → `76.76.21.21` ที่ Namecheap
- [x] Set production env vars (Supabase: 6 vars in Production scope)
- [x] HTTPS cert ออก (manual `vercel certs issue` หลัง auto-issue ติดคิว)
- [ ] **เปิด Vercel Spend Limit + email alert** (Settings → Billing → Spend Management)
- [ ] เพิ่ม `vercel.json` ถ้าต้อง custom routes/headers (ตอนนี้ยังไม่จำเป็น)
- [ ] Move from Hobby → Pro plan ก่อน traffic > free tier limit

## See Also

- [[0001-stack-nextjs-typescript-tailwind]]
- [[0004-pwa-via-manifest]]
