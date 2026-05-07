---
title: "ADR-0007: LINE OA — Messaging API integration"
tags: [adr, line, messaging, integration]
status: Accepted
date: 2026-05-08
---

# ADR-0007 — LINE OA: Messaging API integration

## Status

Accepted

## Context

ต้องช่อง messaging กับ user ฝั่ง consumer-facing. LINE = de-facto messenger ในไทย, นักเรียนใช้แทบ 100%. Use case ระยะแรก:

- Notify reminders (todo due, daily summary)
- Customer support chat
- (อนาคต) Login via LINE OAuth

ตัวเลือกหลัก:
1. **LINE Messaging API** (Messaging API + Push)
2. **Email** — มี delivery overhead, นักเรียนไม่ค่อยเปิด
3. **Web Push (PWA)** — รองรับเฉพาะ browser ที่ install app, iOS Safari ติด
4. **SMS** — ค่าส่งสูง, no rich content

## Decision

ใช้ **LINE Messaging API** (free tier 200 msg/mo, 500 push/mo) เป็น primary channel.

**Setup:**
- LINE Official Account = `LungNote`
- Channel = LINE Messaging API channel ผูก OA
- Webhook URL = `https://lungnote.com/api/line/webhook`
- Env (ใน Vercel + `.env.local`):
  - `LINE_CHANNEL_SECRET` — HMAC verify webhook signature
  - `LINE_CHANNEL_ACCESS_TOKEN` — auth header เรียก LINE Messaging API

**Code structure:**
- `src/lib/line/verify.ts` — `verifySignature(body, signature)` ใช้ Web Crypto API (HMAC-SHA256), edge-runtime compatible
- `src/lib/line/client.ts` — `replyMessage(replyToken, messages[])`, `pushMessage(to, messages[])`
- `src/app/api/line/webhook/route.ts` — `POST` handler:
  1. Read raw body (text)
  2. Verify `x-line-signature` header → 401 ถ้าไม่ผ่าน
  3. Parse events
  4. Dispatch handler ตาม `event.type` (message/follow/unfollow/postback)
  5. Respond `200` ภายใน 30 วิ (LINE timeout)

**Reply pattern:**
- ใช้ `reply` (free, ไม่นับ quota) เมื่อ user wrote first
- ใช้ `push` (นับ quota) เมื่อ proactive notification

## Consequences

**Positive**
- ฟรี tier พอ MVP (200 push/mo, reply ไม่จำกัด)
- LINE adoption ในไทยสูงสุด → no install friction
- Rich UI: Flex Messages (cards), Quick Reply, LIFF (in-app browser)
- LINE Login OAuth พร้อมใช้กับ Supabase auth ภายหลัง
- Webhook signature = built-in security

**Negative**
- Vendor lock-in (LINE-specific message format, LIFF)
- Free quota limit — push เพิ่ม >200/mo = $0.05/msg
- LINE Developer Console UX = legacy
- Webhook ต้อง public HTTPS (ใช้ ngrok สำหรับ local dev)
- Secret/token rotation = manual (Console)

**Mitigation**
- Wrap LINE SDK calls ใน `lib/line/client.ts` → swap provider ภายหลังง่าย
- Throttle/queue push messages → กัน burst เกิน quota
- เก็บ `audit_log` ของ webhook events ใน Postgres (debug + replay)
- Use `ngrok http 3000` หรือ Cloudflare Tunnel สำหรับ test webhook ใน dev

## Alternatives ที่ไม่เลือก

- **Email (Resend / SendGrid)** — ผลตอบรับต่ำกับนักเรียน, friction กับ open rate
- **Web Push** — iOS Safari support เพิ่งมี, ต้อง install PWA ก่อน, friction
- **SMS (Twilio / Thai SMS)** — ค่าส่ง 0.20-0.50 บาท/SMS, no rich, ไม่ scale
- **Discord webhook** — audience นักเรียน TH ใช้น้อยกว่า LINE
- **Telegram** — same — น้อยกว่า LINE

## Open questions / TODO

- [x] เพิ่ม env vars ที่ Vercel prod + `.env.local`
- [x] อัพเดท Glossary "LINE OA / Messaging"
- [x] Implement: `src/lib/line/{verify,client,types}.ts` + `src/app/api/line/webhook/route.ts`
- [x] Auto-reply rules เริ่มต้น (สวัสดี/ช่วย/เว็บ/เกี่ยวกับ + echo)
- [ ] **ทดสอบ**: ตั้ง webhook URL `https://lungnote.com/api/line/webhook` ใน LINE Console → toggle "Use webhook" ON → กด Verify → message OA → ตรวจ reply
- [ ] เพิ่ม `audit_log` table (Postgres) เก็บ webhook event payload
- [ ] LINE Login OAuth → Supabase Auth (ADR แยกเมื่อทำ)
- [ ] Push notification queue (rate limit + retry)
- [ ] ตั้ง webhook URL สำหรับ preview deploy แยก (Vercel preview = unique URL ต่อ PR)
- [ ] Decide: ตอบเป็น Flex Message หรือ plain text สำหรับ daily reminder

## See Also

- [[0006-supabase-db-auth]] — DB จะเก็บ audit_log + line_user_id mapping
- LINE Messaging API docs: https://developers.line.biz/en/reference/messaging-api/
- Verify signature: https://developers.line.biz/en/reference/messaging-api/#signature-validation
