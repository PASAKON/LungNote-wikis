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

ใช้ **Vercel Hobby** (free tier) เป็นจุดเริ่ม. Deploy via GitHub integration — push `main` → auto deploy production.

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

- [ ] Connect Vercel project → GitHub repo `PASAKON/LungNote`
- [ ] Add custom domain `lungnote.com` + `www.lungnote.com`
- [ ] อัพเดท DNS A/CNAME ที่ Namecheap → Vercel
- [ ] Set production env vars (เมื่อมี: `NEXTAUTH_*`, `DATABASE_URL`, ฯลฯ)
- [ ] เปิด Vercel Spend Limit + email alert
- [ ] เพิ่ม `vercel.json` ถ้าต้อง custom routes/headers (ตอนนี้ยังไม่จำเป็น)

## See Also

- [[0001-stack-nextjs-typescript-tailwind]]
- [[0004-pwa-via-manifest]]
