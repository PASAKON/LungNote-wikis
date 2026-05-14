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
| **Memory item** | หน่วยความจำหลัก (note ≡ todo) ตั้งแต่ [[../40-Decisions/0012-unified-todo-memory-model\|ADR-0012]]. Row ใน `lungnote_todos` มี `text + done + due_at + due_text + source`. ทุกอย่างที่ user "จด" / "เตือน" / "todo" ผ่าน chat หรือ Web ลงตัวเดียวกัน |
| **Note (container)** | หลัง ADR-0012 เป็นแค่ container/group สำหรับ memory items. Row ใน `lungnote_notes` |
| **Inbox note** | "📥 Inbox" — note อัตโนมัติต่อ user ที่ chat-captured items ลงไป (เพราะ `lungnote_todos.note_id` NOT NULL). Lazy-create ที่ first capture |
| **Notebook** | สมุดโน้ต — กลุ่ม note หลายอันรวมกัน, มีชื่อ + `cover_color`. Table: `lungnote_notebooks` |
| **Todo** | alias ของ Memory item — UI เรียก "Todo" ที่ `/dashboard/todo`. Table: `lungnote_todos`. ลำดับด้วย `position` |
| **due_at / due_text** | timestamp + raw user phrase ("พรุ่งนี้") ที่ AI extract จาก message ตอน capture ([[../40-Decisions/0012-unified-todo-memory-model\|ADR-0012]]) |
| **source (todo)** | enum `'chat' \| 'web' \| 'liff'` — บอก origin ของ memory item เพื่อ analytics + filter |
| **Folder** | จัดกลุ่ม notebook (nested ได้ผ่าน `parent_folder_id`, app cap ≤ 3 ระดับ). Table: `lungnote_folders` |
| **Tag** | label สี ใช้ filter/search. Many-to-many กับ note ผ่าน junction. Tables: `lungnote_tags` + `lungnote_notes_tags` |

## Business

| Term | Definition |
|------|------------|
| **LungNote** | ชื่อโปรเจค — อ่านว่า **"ลังโน้ต"**. "ลัง" = กล่องกระดาษสีเหลือง (the cardboard box mascot). Brand story: เราเป็นลังที่จะคอยช่วยคุณจดโน้ตและเตือนความจำ ไม่ว่าโน้ตจะสั้นหรือยาว — เพื่อให้คุณไม่ลืมสิ่งที่จะต้องทำในอนาคต. Webapp จดโน้ต/เช็คลิสต์/เตือนความจำสำหรับนักเรียน-นักศึกษาไทย. Tagline: "จดโน้ต เช็คลิสต์ จัดระเบียบชีวิต" |
| **Production domain** | `lungnote.com` |
| **Plan: Free** | ฟีเจอร์หลัก, สร้าง notebook/note ไม่จำกัด |
| **Plan: Pro** | เพิ่ม sync ข้ามอุปกรณ์ + backup auto |
| **Plan: Education** | สำหรับโรงเรียน/มหาวิทยาลัย, จัดการห้องเรียน, ติดต่อ school@lungnote.app |
| **Wiki** | knowledge vault (`wikis/`), ไม่ใช่ runtime data |

## Brand

| Term | Definition |
|------|------------|
| **Cardboard palette** | Brand colors ที่ใช้ตั้งแต่ [[../40-Decisions/0013-cardboard-theme-overhaul\|ADR-0013]]. Light: bg `#F5EAD4`, surface `#FAF5E8`, accent `#C9A040`, fg `#3A3020`. Dark: bg `#1A1810`, surface `#2A2618`, accent `#C9A040`, fg `#E8E2D0`. แทน mint palette เดิม |
| **Mascot** | กล่องกระดาษมีปาก (✓-shape) + แขนข้างๆ. ใช้เป็น app icon + OG image. Source: `design/mascot-icon/lungnote-mascot-icon.svg` |
| **WordMark** | "Lung**Note**" — Lung สี fg, Note สี accent. Font: Caveat (display) |
| **Tape** | สี `#D4A855` (light) / `#B89030` (dark) — ใช้เป็น highlight stripe ภายใต้ข้อความ (`.highlight-tape`) เลียนกระดาษเทปไฮไลต์ |

## Technical Alias

| Term | Means |
|------|-------|
| **App** | `webapp/` — Next.js app |
| **Vault** | `wikis/` — Obsidian vault |
| **Agent** | The tool-using LLM that handles every LINE message — `webapp/src/lib/agent/`. Not a generic chatbot; it has a curated tool catalog (11 tools) |
| **Intent router** | Per-turn heuristic at `webapp/src/lib/agent/router.ts` that picks the model based on the user message ([[../40-Decisions/0016-intent-router\|ADR-0016]]). Flash default; Pro on escalation patterns |
| **Eval harness** | `webapp/scripts/eval/` — replays a curated Thai corpus through any LLM with mocked tools and emits a scored Markdown report ([[../40-Decisions/0015-agent-eval-harness\|ADR-0015]]) |
| **Trace** | One row in `lungnote_chat_traces` representing one LINE turn (input, tool calls, reply, meta). Surfaced in the admin viewer at `admin.lungnote.com/traces` |
| **Route reason** | `meta.route_reason` field on a trace — names which intent-router trigger fired (`default`, `update_verb`, `profile_fact`, `multi_position`, `long_message`, `complex_clause`, `router_disabled`). Suffix `+retry` means the [[../40-Decisions/0016-intent-router\|empty-reply rescue]] kicked in |
| **Empty-reply rescue** | Post-hoc retry inside `runAgent` that calls the complex model with a reply-only tool subset when the fast model ends a turn with no bubble after a read-only tool call. Toggle via `LLM_REPLY_RETRY_ENABLED`. See [[../40-Decisions/0016-intent-router\|ADR-0016]] *Update 2026-05-15* |
| **OpenRouter** | LLM provider aggregator we route through for non-Anthropic models (Gemini, GPT). Single `OPENROUTER_API_KEY` env covers all of them |

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
