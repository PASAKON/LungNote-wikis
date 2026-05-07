---
title: "ADR-0006: DB + Auth — Supabase"
tags: [adr, db, auth, supabase]
status: Accepted
date: 2026-05-07
---

# ADR-0006 — DB + Auth: Supabase

## Status

Accepted

## Context

Webapp ต้องการ:
- **Database** สำหรับ notebook, note, todo, folder, tag, user profile
- **Auth** สำหรับ sign-up / sign-in (email + OAuth providers)
- **Authorization** ระดับแถว (per-user data isolation) — เพราะข้อมูลโน้ต = private
- รองรับ Next 16 App Router + RSC + Server Actions
- Free tier ที่อยู่ได้ตอน launch (low traffic)
- Migration path ออกได้ถ้าจำเป็น

ตัวเลือกหลัก:
1. **Supabase** — Postgres + Auth + Storage + Realtime
2. **Firebase** — Firestore (NoSQL) + Auth + Storage
3. **Neon + Auth.js** — managed Postgres + DIY auth
4. **Convex** — reactive DB + auth, schema TypeScript-first
5. **Self-host Postgres + Lucia/Better-Auth** — full control

## Decision

ใช้ **Supabase Hobby (free tier)**.

- Project: `qkaxvockysyazmtormvf` (region: TBD-verify)
- API URL: `https://qkaxvockysyazmtormvf.supabase.co`
- ใช้ **API key format ใหม่** (`sb_publishable_...` + `sb_secret_...`) — **ไม่ใช่** legacy `anon` / `service_role` JWT
- Client lib: `@supabase/ssr` (Next App Router cookie flow) + `@supabase/supabase-js`
- Auth: Supabase Auth (email/password เริ่มต้น, เพิ่ม OAuth ภายหลัง)
- Authorization: PostgreSQL **Row-Level Security (RLS)** policies — ทุกตารางเปิด RLS
- Migration: Supabase CLI (`supabase migration new …`) → push ไป remote project

Env vars (ทั้ง local `.env.local` + Vercel Production):
- `SUPABASE_PROJECT_REF`
- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` (client-safe)
- `SUPABASE_SECRET_KEY` (server-only)
- `SUPABASE_DB_PASSWORD`
- `DATABASE_URL` (Postgres connection string สำหรับ migration tool / Drizzle ภายหลัง)

## Consequences

**Positive**
- All-in-one (DB + Auth + Storage + Realtime) → ลด vendor count, devops ต่ำ
- **Postgres เต็ม** → SQL skill โอนได้, schema portable
- **RLS ที่ DB layer** → ปลอดภัยกว่า application-only check (กัน bypass จาก client)
- `@supabase/ssr` คุย cookie flow กับ Next App Router → SSR + middleware refresh token ครบ
- Free tier: 500MB DB, 50K MAU, 1GB storage, 2GB bandwidth — พอ launch + early traffic
- Open-source core → self-host ได้ภายหลังถ้าจำเป็น
- Auth schema (`auth.users`) standard → dump migrate ออกง่าย

**Negative**
- Vendor lock-in **ระดับกลาง**: Auth schema, RLS syntax, Storage API = Supabase-specific (Postgres ส่วนตารางตัวเองโอนได้)
- RLS = **DB-level skill** ที่ต้องเรียน (security model ต่างจาก app-level guard) — เขียนผิด = data leak
- Free tier มี limit: project pause หลัง inactive 7 วัน, no daily backup
- ถ้า scale > free → Pro $25/mo (ต้องตั้ง budget)
- New API key format (`sb_*`) — บาง 3rd-party guide ยังใช้ legacy JWT, ต้องระวัง

**Mitigation**
- เขียน RLS policy ทุกตาราง พร้อม **deny-by-default** (RLS on, no policy = no access)
- มี integration test สำหรับ RLS rules (sign in user A → query → ไม่เห็น data user B)
- Migration ผ่าน CLI + version control (ไม่แก้ schema ใน dashboard ตรงๆ)
- Set Supabase project region ใกล้ user (Singapore สำหรับ TH)
- Backup: enable Point-in-Time Recovery เมื่ออัป Pro

## Alternatives ที่พิจารณา

- **Firebase**
  - ไม่เลือก: NoSQL = ไม่เหมาะ relational (notebook → notes → todos), vendor lock-in แรงกว่า, query language เฉพาะ
- **Neon + Auth.js**
  - ไม่เลือก: 2 vendor (DB + Auth) = devops คู่, Auth.js config ยาก, ไม่มี RLS auto-bridge กับ user_id
- **Convex**
  - ไม่เลือก: vendor lock-in สูงสุด (proprietary DB language), schema portable ยาก, no SQL escape hatch
- **Self-host Postgres + Lucia/Better-Auth (DIY)**
  - ไม่เลือก: devops overhead เกิน MVP — ต้องรัน DB + auth service + email + storage เอง

## Naming

ทุก table ใน `public` schema **ต้อง prefix `lungnote_`** (ดู [[../20-Conventions/Database-Naming]]). กัน collision เผื่อ DB shared + filter ง่าย.

## Open questions / TODO

- [ ] Verify Supabase project region (Singapore แนะนำสำหรับ TH user)
- [ ] **Rotate** `SUPABASE_DB_PASSWORD` (เคยถูกแชร์ผ่าน chat — ทำที่ Supabase dashboard)
- [x] Install `@supabase/ssr` + `@supabase/supabase-js`
- [x] สร้าง `src/lib/supabase/{server,client,middleware}.ts`
- [x] Wire Supabase auth refresh เข้า existing `src/proxy.ts` (`updateSession()`)
- [x] Add `/api/health` endpoint pinging Supabase to verify env vars in deployed env
- [ ] First schema migration: `profiles` (linked to `auth.users`), `notebooks`, `notes`, `todos`, `folders`, `tags`
- [ ] RLS policy ทุกตาราง (deny-by-default + per-owner CRUD)
- [ ] Decide: ใช้ Supabase JS client ตรงๆ vs ใส่ Drizzle ORM ทับ (ADR ต่างหาก ถ้าเลือก Drizzle)
- [x] Update [[../10-Architecture/Overview]] — Auth + DB sections + data flow skeleton
- [x] Update [[../30-Domain/Glossary]] — RLS, Publishable key, Secret key, PostgREST, JWT, `@supabase/ssr`
- [ ] เขียน workflow สำหรับ DB migration → [[../50-Workflows/Dev-Workflow]] (ทำตอนเริ่ม schema)

## See Also

- [[0001-stack-nextjs-typescript-tailwind]]
- [[0005-deploy-vercel]] — env vars ตั้งใน Vercel แล้ว
- Supabase docs: https://supabase.com/docs
- Next App Router + Supabase: https://supabase.com/docs/guides/auth/server-side/nextjs
