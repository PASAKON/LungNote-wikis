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
| **Note** | บันทึกข้อความ/ลิสต์/ไอเดีย หน่วยเล็กสุดของเนื้อหา |
| **Notebook** | สมุดโน้ต — กลุ่ม note หลายอันรวมกัน, มีชื่อ + สีปก |
| **Todo / Checklist** | ไอเทมแบบ checkbox (done/undone) ในโน้ต |
| **Folder** | จัดกลุ่ม notebook ตามวิชา/โปรเจค/หมวด |
| **Tag** | label หลายอันต่อ note ใช้ filter/search |

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
