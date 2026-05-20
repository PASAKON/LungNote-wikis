---
title: "ADR-0017: Gmail Integration v1 (readonly)"
tags: [adr, integration, gmail, oauth, ai]
status: Proposed
date: 2026-05-20
---

# ADR-0017 — Gmail Integration v1 (readonly)

## Status

Proposed

## Context

User-driven feature request: ผู้ใช้ LungNote (นักเรียน-นักศึกษาไทย ตาม
[[../30-Domain/Glossary|Glossary]]) ต้องการให้แอปดูดเอา to-do จากกล่อง Gmail
ของตัวเองมาใส่ใน `/dashboard/todo` อัตโนมัติ. ตัวอย่างเมลที่นับเป็น to-do:

- เมลเตือนนัดหมาย / consult / interview
- เมลส่งงานจาก ครู / อาจารย์ / TA (deadline)
- เมล lab result / appointment confirmation
- เมล bill / payment due

Constraints:

- [[0008-line-only-auth-account-linking|ADR-0008]] บังคับ **LINE-only primary
  auth**. Gmail OAuth ไม่ใช่ทางเข้า login ของ webapp — เป็น **secondary
  connection** ผูกกับ user ที่ login ผ่าน LINE/LIFF/Web LINE Login แล้ว
- [[0014-llm-default-gemini-2-5-flash|ADR-0014]] บังคับ **Gemini 2.5 Flash**
  เป็น default LLM. การ extract to-do จาก email ใช้ pipeline เดียวกัน
- [[0012-unified-todo-memory-model|ADR-0012]] กำหนด `lungnote_todos.source`
  enum `('chat','web','liff')`. ต้องขยาย → `+ 'email'`
- [[0006-supabase-db-auth|ADR-0006]] กำหนดให้ทุก table มี RLS deny-by-default
- User กำหนดว่า v1 ต้องการแค่ "อ่าน + extract to-do". การร่าง reply / ติด
  label / archive = phase ภายหลัง (ADR แยก)

Gmail OAuth scopes ที่เป็นไปได้:

| Scope | สิทธิ์ | Sensitivity |
|-------|--------|-------------|
| `gmail.metadata` | header เท่านั้น (ไม่อ่าน body / subject) | non-sensitive |
| `gmail.readonly` | อ่านทุกอย่าง (body, attachment) | **Sensitive** |
| `gmail.modify` | อ่าน + เขียน + label + draft + send (ไม่รวม permanent delete) | **Restricted** |
| `gmail.send` | ส่งอย่างเดียว | Sensitive |
| `https://mail.google.com/` | full access รวม permanent delete | Restricted |

Verification ของ Google:

- **Sensitive** scope = ต้องผ่าน OAuth brand verification + privacy policy
  audit + demo video (4-6 สัปดาห์, ฟรี)
- **Restricted** scope = ทั้งด้านบน + **annual CASA security assessment**
  ($300-500 USD ต่อปี, 6-12 สัปดาห์ผ่าน 3rd-party)
- ก่อนผ่าน verification = cap ที่ "test users" 100 คน

## Decision

**v1 ใช้ `gmail.readonly` อย่างเดียว.** เมื่อ user ต้องการ AI ติด label /
ร่าง reply (defer phase ต่อไป) → upgrade เป็น `gmail.modify` ผ่าน ADR ใหม่
+ consent screen รอบสอง.

### Connection Model

ทุก LungNote user ↔ 0..1 Gmail account (v1 จำกัด 1 connection/user):

```
auth.users (LINE-authenticated)
   │
   └─0..1─ lungnote_gmail_connections
              │ (refresh_token_enc, scope, status, last_history_id, ...)
              │
              └─1:N─ lungnote_gmail_synced_messages
                       │ (message_id, is_todo, todo_id?)
                       │
                       └─0..1─ lungnote_todos (source='email')
```

Schema รายละเอียดดู [[0018-gmail-schema-delta|ADR-0018]].

### OAuth Flow

```
1. User → /dashboard/settings/integrations → click "Connect Gmail"
   ↓
2. Server: GET /api/auth/gmail/connect
   a. require Supabase session (LINE-authenticated user)
   b. generate state (32-byte base64url, HttpOnly cookie 5 min TTL)
   c. redirect → Google OAuth consent
      scope = openid email profile https://www.googleapis.com/auth/gmail.readonly
      access_type=offline, prompt=consent (force refresh_token)
   ↓
3. User authorizes (test users mode pre-verification)
   ↓
4. Google → GET /api/auth/gmail/callback?code=...&state=...
   a. verify state (match cookie, single-use, not expired)
   b. exchange code → access_token + refresh_token + id_token
   c. parse id_token → google_user_id (sub), email
   d. encrypt refresh_token using AES-256-GCM with GMAIL_ENC_KEY (env)
   e. upsert lungnote_gmail_connections (user_id = supabase session uid,
      google_user_id, email, refresh_token_enc, scope, status='active')
   f. clear state cookie
   g. redirect → /dashboard/settings/integrations?gmail=connected
```

### Token Storage Security

- `refresh_token_enc` = AES-256-GCM ciphertext stored in `bytea` column
- Encryption key = env `GMAIL_ENC_KEY` (256-bit, base64) — Vercel secret only,
  rotated annually
- IV = 12 bytes random, prepended to ciphertext
- AAD = `lungnote_gmail_connections:<row_id>` (binds ciphertext to row)
- Plaintext refresh_token **เด็ดขาดไม่ log, ไม่ส่ง client**
- Access token cached (encrypted same way) + `access_token_expires_at` —
  refresh เมื่อ < 60s ก่อน expiry
- RLS: user เห็นแค่ connection ของตัวเอง. Service role bypass สำหรับ cron
  sync เท่านั้น

### Sync Trigger (v1) — Hybrid push + reconcile

1. **Primary — Gmail Push Notification via Pub/Sub**
   - On connect: `users.watch({topicName, labelIds:['INBOX']})` → returns
     `historyId` + `expiration` (max 7 days)
   - Gmail → Pub/Sub topic `projects/<gcp_project>/topics/gmail-events` →
     push subscription → POST `/api/gmail/pubsub`
   - Webhook: verify Google ID token (`google-auth-library`) → ack 200
     within < 1s → background job runs sync (history.list since
     `last_history_id` → classify → upsert todos)
   - Target latency: < 30s from email arrival → todo appears
2. **Safety net — Reconcile cron every 1 hour**
   - `0 * * * *` → POST `/api/cron/gmail-reconcile` (Bearer `CRON_SECRET`)
   - Iterates active connections, runs same sync function
   - Catches messages that Pub/Sub dropped or webhook missed (network /
     cold start failures)
   - **Idempotent** via `lungnote_gmail_synced_messages.unique(user_id, message_id)`
3. **Watch renewal cron daily**
   - `0 0 * * *` → POST `/api/cron/gmail-watch-renew` (Bearer `CRON_SECRET`)
   - Iterates connections where `watch_expires_at < now() + 24h` → calls
     `users.watch()` again → updates expiration
4. **Manual "Scan now" button** — defer (cron 1h + push พอสำหรับ MVP)
5. **LINE bot `scan_gmail` tool** — defer phase 2 (ดู [[0019-gmail-extract-agent-tool|ADR-0019]] §LINE bot flow)

Watermark: `last_history_id` (Gmail `historyId` cursor). ครั้งแรก = full
fetch ของ INBOX label 30 วันล่าสุด (cap 200 messages). Subsequent = both
push และ reconcile cron ใช้ cursor เดียวกัน — idempotent ผ่าน unique
constraint.

Pub/Sub dedup: ใช้ Pub/Sub `messageId` ใน webhook handler เป็น dedup key
ใน in-memory cache (5 min TTL) ก่อน enqueue background job. Network จริงๆ
จะมี at-least-once delivery บางครั้ง.

### What gets stored

**Stored**:
- `message_id`, `thread_id`, `internal_date` (Gmail metadata)
- `is_todo` (boolean — AI judgment)
- `todo_id` (FK → lungnote_todos ถ้า is_todo=true)

**NOT stored**:
- Email body raw
- Subject line plaintext (hash only ถ้าต้อง dedup — defer)
- Attachment
- From/To addresses (excepted truncated for trace debugging)

If todo extracted: `lungnote_todos.text` = AI-generated short title +
`source='email'` + `source_external_id=message_id` + `source_url=Gmail permalink`

### Revoke / Disconnect

- User click "Disconnect" → DELETE `/api/auth/gmail/connect`
  a. POST `https://oauth2.googleapis.com/revoke?token=<refresh_token>` (best-effort)
  b. delete row `lungnote_gmail_connections`
  c. **keep** `lungnote_gmail_synced_messages` + `lungnote_todos` already extracted
     (read-only history, no harm)
- Cascade: ถ้า `auth.users` ถูกลบ → connection + synced_messages cascade

### Verification path

- **Dev / staging**: Google Cloud Console "OAuth consent screen" = External /
  Testing → cap 100 test users (manual whitelist by email)
- **Production**:
  1. Privacy policy หน้าเว็บ + ใน Google Cloud Console
  2. Demo video อธิบาย scope usage
  3. ส่ง verification request (Sensitive scope, ~4-6 wk Google review,
     ไม่ต้อง CASA สำหรับ `readonly`)
  4. โดเมน `lungnote.com` ต้อง verified ใน Search Console
- Phase 5 upgrade `gmail.modify` → ต้อง CASA + re-verification (~6-12 wk)

## Consequences

**Positive**

- User เห็นปุ่ม "อ่านอย่างเดียว" = friction ต่ำ click ง่าย
- Scope ขั้นต่ำ = security incident impact ต่ำ
- Verification path เร็วกว่า `modify` (4-6 wk vs 6-12 wk + CASA)
- Reuse infrastructure ที่มี: Supabase Auth (session), Gemini Flash (extract),
  agent tool framework ([[0014-llm-default-gemini-2-5-flash|ADR-0014]],
  [[0015-agent-eval-harness|ADR-0015]])
- Schema เพิ่มแบบ additive — ไม่ break existing migrations

**Negative**

- ไม่สามารถให้ AI ร่าง reply / ติด label / archive ตั้งแต่ v1 — phase 5
  ต้อง re-consent (UX มี friction)
- ไม่มี real-time push (Gmail watch + Pub/Sub = defer phase 2 cron)
- 1 Gmail / user (v1) — multi-account = defer
- Storing refresh_token = risk vector. ต้อง encrypt + key rotation discipline
- Test users cap 100 ก่อน production verification → จำกัด early traffic

**Mitigation**

- Phase 5 ADR จะระบุ migration plan (add scope, force re-consent ครั้งเดียว)
- Cron + watch = ADR แยกเมื่อมี budget approval
- key rotation playbook ใน [[../50-Workflows/Database-Migration]] (เพิ่ม
  Gmail-OAuth-Setup.md)
- Beta launch = test users mode, formal verification ก่อน public

## Alternatives ที่ไม่เลือก

- **`gmail.metadata`** — ขั้นต่ำสุด แต่ extract to-do ไม่ได้ (ไม่เห็น subject/body)
- **`gmail.modify` ตั้งแต่ v1** — เร็วกว่าระยะยาว (1 consent ครั้ง) แต่:
  - friction สูง (สิทธิ์เยอะ user กลัว)
  - ต้อง CASA assessment ทันที = launch ช้า 6-12 wk
  - ถ้า v1 ไม่ใช้ scope `modify` = Google audit จะตั้งคำถาม "ขอเกินจริง?"
- **Cron-only (no push)** — ง่ายกว่า Pub/Sub แต่ user รอ todo นาน 0-1h (cron
  interval). Pub/Sub push เพิ่ม latency < 30s = UX ดีกว่าชัดเจน, infra
  overhead ยอมรับได้
- **MCP server expose** ([[0019-gmail-extract-agent-tool|ADR-0019]] discusses)
  — defer, ไม่ใช่ v1 requirement
- **3rd-party Gmail MCP server** (เช่น `GongRzhe/Gmail-MCP-Server`) — desktop /
  single-user, ไม่ scale multi-tenant SaaS

## Database Naming

ตาราง prefix `lungnote_*` per [[../20-Conventions/Database-Naming]]:

- `lungnote_gmail_connections`
- `lungnote_gmail_synced_messages`

Schema detail ดู [[0018-gmail-schema-delta|ADR-0018]].

## Env Vars (เพิ่ม)

```
GOOGLE_OAUTH_CLIENT_ID=<from Google Cloud Console>
GOOGLE_OAUTH_CLIENT_SECRET=<server-only secret>
GOOGLE_OAUTH_REDIRECT_URI=https://lungnote.com/api/auth/gmail/callback
GMAIL_ENC_KEY=<base64 32 bytes, server-only>

# Pub/Sub real-time push
GCP_PROJECT_ID=<google cloud project id>
GMAIL_PUBSUB_TOPIC=projects/<GCP_PROJECT_ID>/topics/gmail-events
PUBSUB_PUSH_SERVICE_ACCOUNT=<sa email used in push subscription auth>
PUBSUB_AUDIENCE=https://lungnote.com/api/gmail/pubsub

# Cron auth
CRON_SECRET=<base64 32 bytes, server-only>
```

ตั้งทั้งที่ `.env.local` (dev) + Vercel Production. local dev redirect URI =
`http://localhost:3000/api/auth/gmail/callback` (เพิ่มใน Google Console
authorized URIs).

## Open questions / TODO

- [ ] Verify `lungnote.com` ใน Google Search Console (เตรียม production verification)
- [ ] Privacy policy หน้าเว็บ (`/legal/privacy`) ระบุการใช้ Gmail scope
- [ ] Demo video สั้นๆ อธิบาย Connect Gmail flow + scope usage
- [ ] Decide cron schedule + budget (defer phase 2)
- [ ] Rate limit: max 1 scan / user / minute (กัน DoS เปลือง Gmail quota)
- [ ] Gmail API quota tier: default 250 quota units/user/sec, 1B/day —
  monitor + alert ถ้าใกล้ limit
- [ ] Audit log table สำหรับ Gmail actions ใน phase 5 (เผื่อล่วงหน้า)
- [ ] Update [[../10-Architecture/Overview]] เพิ่ม Gmail integration section
- [ ] Update [[../30-Domain/Glossary]] เพิ่ม term: Gmail connection,
  history_id, message_id, scope, INBOX, Sensitive scope, Restricted scope,
  CASA, refresh_token
- [ ] เขียน `50-Workflows/Gmail-OAuth-Setup.md` (Google Cloud Console setup
  + test users add + production verification checklist)

## See Also

- [[0006-supabase-db-auth]]
- [[0008-line-only-auth-account-linking]]
- [[0012-unified-todo-memory-model]]
- [[0014-llm-default-gemini-2-5-flash]]
- [[0018-gmail-schema-delta]]
- [[0019-gmail-extract-agent-tool]]
- Google OAuth scope reference: https://developers.google.com/gmail/api/auth/scopes
- Google API verification: https://support.google.com/cloud/answer/13463073
