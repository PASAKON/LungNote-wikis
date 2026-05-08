---
title: Domain Glossary
tags: [domain, glossary]
---

# Domain Glossary

ศัพท์ที่ใช้ใน LungNote — product + business + technical alias.

> เพิ่มศัพท์ใหม่เมื่อพบ — อย่าเดา. ถ้าไม่ชัวร์ ใส่ `(?)` ท้ายและถามคนตรวจ.

## Product

| Term | Definition |
|------|------------|
| **Note** | บันทึกข้อความ/ลิสต์/ไอเดีย หน่วยเล็กสุดของเนื้อหา. Table: `lungnote_notes` |
| **Notebook** | สมุดโน้ต — กลุ่ม note หลายอันรวมกัน, มีชื่อ + `cover_color`. Table: `lungnote_notebooks` |
| **Todo / Checklist** | ไอเทม checkbox (done/undone) ภายใน note, ลำดับด้วย `position`. Table: `lungnote_todos` |
| **Folder** | จัดกลุ่ม notebook (nested ได้ผ่าน `parent_folder_id`, app cap ≤ 3 ระดับ). Table: `lungnote_folders` |
| **Tag** | label สี ใช้ filter/search. Many-to-many กับ note ผ่าน junction. Tables: `lungnote_tags` + `lungnote_notes_tags` |

## Business

| Term | Definition |
|------|------------|
| **LungNote** | ชื่อโปรเจค — webapp จดโน้ต/เช็คลิสต์/จัดระเบียบสำหรับนักเรียน-นักศึกษาไทย. Tagline: "จดโน้ต เช็คลิสต์ จัดระเบียบชีวิต" |
| **Production domain** | `lungnote.com` |
| **Plan: Free** | ฟีเจอร์หลัก, สร้าง notebook/note ไม่จำกัด |
| **Plan: Pro** | เพิ่ม sync ข้ามอุปกรณ์ + backup auto |
| **Plan: Education** | สำหรับโรงเรียน/มหาวิทยาลัย, จัดการห้องเรียน, ติดต่อ school@lungnote.app |
| **Wiki** | knowledge vault (`wikis/`), ไม่ใช่ runtime data |

## Technical Alias

| Term | Means |
|------|-------|
| **App** | `webapp/` — Next.js app |
| **Vault** | `wikis/` — Obsidian vault |

## Supabase / DB

| Term | Definition |
|------|------------|
| **Supabase** | DB + Auth + Storage provider ([[../40-Decisions/0006-supabase-db-auth\|ADR-0006]]). Project ref `qkaxvockysyazmtormvf` |
| **RLS** | Row-Level Security — Postgres policy ตัดสินว่า user แต่ละคน SELECT/INSERT/UPDATE/DELETE แถวไหนได้บ้าง. Default = deny |
| **Publishable key** | Supabase client-safe key (`sb_publishable_…`) — โอเค embed ใน browser. แทน legacy `anon` JWT |
| **Secret key** | Supabase server-only key (`sb_secret_…`) — bypass RLS, ห้ามส่ง client. แทน legacy `service_role` JWT |
| **PostgREST** | REST layer ที่ Supabase สร้างจาก schema อัตโนมัติ (auto-generated CRUD endpoint) |
| **`auth.users`** | Supabase built-in table เก็บ user account (managed) |
| **`profiles`** | App-owned table linked 1:1 กับ `auth.users` สำหรับ profile data ฝั่งเรา |
| **JWT** | Token ที่ Supabase Auth ออก, เก็บใน HttpOnly cookie (`sb-<project-ref>-auth-token`) |
| **`@supabase/ssr`** | Lib ที่ wire Supabase auth → Next App Router cookie flow |

## LINE OA / Messaging

| Term | Definition |
|------|------------|
| **LINE OA** | LINE Official Account — channel ที่ LungNote ใช้ส่งข้อความหาผู้ใช้ผ่าน LINE Messaging API |
| **Channel Secret** | Secret 32-char ใช้ verify webhook signature (`x-line-signature` header). Env: `LINE_CHANNEL_SECRET` |
| **Channel Access Token** | Long-lived token เรียก LINE Messaging API (push/reply message). Env: `LINE_CHANNEL_ACCESS_TOKEN` |
| **Webhook URL** | Endpoint ที่ LINE call เมื่อ user message เข้ามา. ตั้งใน LINE Developer Console (TBD path เช่น `/api/line/webhook`) |

## LLM / AI

| Term | Definition |
|------|------------|
| **OpenRouter** | LLM gateway — routes requests to multiple model providers (OpenAI, Anthropic, Google, Mistral, ฯลฯ) ผ่าน API เดียว |
| **OPENROUTER_API_KEY** | Secret key สำหรับเรียก OpenRouter API. Server-only. Format: `sk-or-v1-...` |

## Auth & Identity

| Term | Definition |
|------|------------|
| **Account Linking** | จับคู่ identity ภายนอก (LINE userId) กับ Supabase user ([[../40-Decisions/0008-line-only-auth-account-linking\|ADR-0008]]) |
| **Synthetic Email** | Email ที่สร้างขึ้นเพื่อให้ Supabase Auth พอใจ — `line.<userId>@auth.lungnote.com`. ไม่ใช่ email จริง, ไม่มี MX record |
| **Magic Link** | One-time URL ที่ Supabase generate ผ่าน `auth.admin.generateLink({type:'magiclink'})` — สร้าง session โดยไม่ต้อง password |
| **One-time Token** | random 256-bit string ใช้แลก Supabase session ครั้งเดียว, TTL 5 นาที, store แค่ sha256 hash |
| **Postback** | LINE event type — เมื่อ user แตะปุ่ม Rich Menu / Flex action ที่กำหนด `postback`, bot ได้ payload `data` |
| **Rich Menu** | UI menu ที่ load อยู่ที่ chat ของ OA — image 2500×1686 + tap regions |
| **Flex Message** | Rich card UI ของ LINE — JSON-defined layout, supports buttons/images/typography |
| **LIFF** | LINE Front-end Framework — webapp ที่รันใน LINE in-app browser, มี SDK เข้าถึง userId/profile. ใช้เป็น primary auth ใน LINE app ([[../40-Decisions/0010-liff-auth\|ADR-0010]]) |
| **LIFF ID** | ID ของ LIFF app, สร้างที่ LINE Developer Console. Env: `NEXT_PUBLIC_LINE_LIFF_ID` |
| **id_token** | JWT signed by LINE OIDC. Issued by LIFF (`liff.getIDToken()`) หรือ LINE Login OAuth. Server verify ผ่าน `https://api.line.me/oauth2/v2.1/verify` |
| **LINE Login Channel ID** | Audience ของ id_token. Env: `LINE_LOGIN_CHANNEL_ID`. ต่างจาก Messaging API Channel ID, แต่อยู่ channel เดียวกันได้ถ้า enable ทั้ง 2 product |
| **OAuth Code Flow** | ([[../40-Decisions/0011-web-line-login-oauth\|ADR-0011]]) browser flow: redirect → code → server exchange → id_token. ใช้ `state` (CSRF) + `nonce`. เหมาะ desktop / web ที่ไม่อยู่ใน LINE app |
| **State (OAuth)** | random string ที่ server ส่งไป OAuth provider แล้ว verify match ตอน callback. กัน CSRF. เก็บใน HttpOnly cookie 5 นาที |

## UI / Loading States

| Term | Definition |
|------|------------|
| **Skeleton loader** | Shimmer placeholder ที่ render ก่อน real data — โครงเหมือนหน้าจริง (greeting + stats + list rows) ไม่ใช่ spinner ลอยๆ. ป้องกัน CLS, ทำให้ perceived speed เร็วขึ้น |
| **Shimmer** | animation gradient ไหลผ่าน skeleton 1.5s loop. ใช้สื่อว่า loading. Spec: `linear-gradient(90deg, transparent 0, rgba(255,255,255,0.5) 50%, transparent 100%)` |
| **Next loading.tsx** | Next App Router convention — ไฟล์ `loading.tsx` ที่ root ของ route segment = automatic Suspense boundary, render ระหว่าง async server data resolve |
| **Rich Menu install script** | `webapp/scripts/install-rich-menu.sh` — idempotent installer (delete existing same-name + create + upload PNG + set default). Runbook: [[../50-Workflows/Rich-Menu-Install]] |
