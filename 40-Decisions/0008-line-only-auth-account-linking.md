---
title: "ADR-0008: LINE-only Auth via Account Linking + Synthetic Email"
tags: [adr, auth, line, supabase, account-linking]
status: Accepted
date: 2026-05-08
---

# ADR-0008 — LINE-only Auth via Account Linking + Synthetic Email

## Status

Accepted

## Context

Per [[0007-line-oa-messaging|ADR-0007]] LINE OA = primary user channel. Decision now: **users will live entirely inside LINE.** ไม่มี web sign-up form, ไม่มี email/password, ไม่มี Google OAuth. ทางเข้า web Dashboard อยู่หลังการ chat กับ OA เท่านั้น.

Constraints:
- ไม่ต้องการให้ user กรอก email
- ไม่ต้องการ web Login UI
- Dashboard ที่ `lungnote.com/dashboard` ต้อง require auth
- Supabase Auth ([[0006-supabase-db-auth|ADR-0006]]) เป็น session layer — must mint a real Supabase session
- LINE Messaging API ไม่ส่ง email ของ user — มีแค่ `userId`, `displayName`, `pictureUrl`

ตัวเลือก:
1. **LINE Login OAuth (web button)** — มี "Login with LINE" บนเว็บ. user click → OAuth → ได้ id_token → mint Supabase session
2. **LIFF** — เปิด Dashboard ใน LINE in-app browser ผ่าน LIFF URL, auto-pass userId
3. **Account Linking (one-time token)** — bot ส่ง one-time link, user tap → web verify token → mint session ✅

## Decision

ใช้ **Option 3 (Account Linking) อย่างเดียว** + **synthetic email** เพื่อให้ Supabase Auth user record ถูก format.

### Identity Mapping

ทุก LINE user ↔ 1 Supabase auth.users row:

```
LINE userId  Uxxxx...                  →  auth.users.email = line.Uxxxx...@auth.lungnote.com
                                          auth.users.id    = uuid (primary)
                                          lungnote_profiles.line_user_id = Uxxxx...
                                          lungnote_profiles.id = same uuid (FK to auth.users)
```

### Synthetic Email

- Format: `line.<lowercase-line_user_id>@auth.lungnote.com`
- Subdomain `auth.lungnote.com` = dedicated, **ไม่รับ email จริง** (no MX record)
- ไม่แสดงให้ user เห็น
- ใช้เพื่อให้ Supabase Auth (ที่ require email) มี unique identifier
- ห้าม leak ใน UI — แสดง `line_display_name` แทน

### Account Linking Flow (one-time token)

```
1. User chats LINE OA
   ↓
2. Bot Rich Menu / postback "open_dashboard"
   ↓
3. Webhook handler:
   a. mintToken(line_user_id) → random 32 bytes (base64url) + sha256 hash
   b. INSERT lungnote_auth_link_tokens (line_user_id, token_hash, expires_at = now+5min)
   c. reply Flex Message with URL: https://lungnote.com/auth/line?t=<token>
   ↓
4. User taps URL (LINE in-app browser)
   ↓
5. GET /auth/line?t=<token>:
   a. hash token, lookup lungnote_auth_link_tokens WHERE token_hash = X AND used_at IS NULL AND expires_at > now()
   b. mark used_at = now()
   c. fetch LINE profile (displayName, pictureUrl) via /v2/bot/profile/{userId}
   d. upsert auth.users with synthetic email
   e. upsert lungnote_profiles (id, line_user_id, line_display_name, line_picture_url)
   f. supabase.auth.admin.generateLink({ type: 'magiclink', email: synthetic, options: { redirectTo: '/dashboard' } })
   g. redirect 302 → properties.action_link
   ↓
6. Supabase verify magic link → set auth cookie → redirect /dashboard
   ↓
7. /dashboard reads cookie → Supabase user → query lungnote_notes scoped by RLS
```

### Token Security

- Random: `crypto.randomBytes(32).toString('base64url')` (256-bit entropy)
- Storage: sha256 hash only — **plaintext token never stored**
- TTL: 5 minutes
- Single-use: `used_at` set on first redeem
- Rate limit: max 3 active tokens / line_user_id at any time (clean older first)
- HTTPS only (set in `proxy.ts` matcher / Next route)

### Session

- Supabase auth cookie = HttpOnly, Secure, SameSite=Lax
- Cookie domain: `.lungnote.com` (รับใน in-app browser ของ LINE = WebKit/Blink)
- Default Supabase session lifetime — refresh in middleware (`updateSession` ใน `proxy.ts`, ทำแล้ว)

## Consequences

**Positive**
- 0 friction — user ไม่ต้อง remember email/password
- 0 form — ไม่มี sign-up screen
- LINE userId = stable identifier (LINE ไม่ rotate)
- Synthetic email pattern → ระบุ origin ของ user ได้ทันที
- No PII (email) collected at sign-up
- Bot สามารถ push ถึง user ได้ทันที (มี line_user_id แล้ว)

**Negative**
- ผูกชะตากับ LINE — ถ้า LINE ban OA / user logout LINE / เปลี่ยนประเทศ → user เข้า dashboard ไม่ได้
- ไม่มี email recovery — ลืม login = ต้องไป chat OA ขอ link ใหม่
- ไม่มี multi-device login simultaneously? — มีได้ (session cookie แต่ละ browser แยก) แต่ ทุก device ต้องผ่าน OA link
- Synthetic email = ต้องมี subdomain `auth.lungnote.com` resolvable (เพื่อ Supabase email_confirm validate) — แต่ไม่ต้องรับ email จริง (no MX)
- ถ้าต้องการ web login button ภายหลัง → ต้องเพิ่ม LINE Login OAuth (ADR ใหม่)

**Mitigation**
- Bot welcome message ระบุ "เก็บ link ไว้ส่ง dashboard เสมอ" — user รู้ว่าใช้ LINE = primary
- Rich Menu มีปุ่ม "🔗 รับลิงก์ Dashboard" ตลอด → user gen ใหม่ได้ทุกเมื่อ
- ในอนาคต: optional add LINE Login OAuth button ใน /login fallback page

## Alternatives ที่ไม่เลือก

- **A: LINE Login OAuth (web)** — ต้องการ web Login UI = scope creep
- **B: LIFF** — ผูกกับ LINE app เฉพาะ, ใช้ desktop browser ไม่ได้
- **A+C combined** — ตอนนี้ overkill, ทำ C ก่อน, ขยายภายหลัง
- **Supabase OIDC custom provider** — config ซับซ้อน, ต้อง manage discovery URL, JWKS — overkill สำหรับ single provider
- **Real email collection** — friction สูง, user ไม่ตอบ

## Database Naming

ทุก table ใหม่ใช้ prefix `lungnote_*` per [[../20-Conventions/Database-Naming]]:
- `lungnote_profiles`
- `lungnote_auth_link_tokens`
- `lungnote_notes`

## Open questions / TODO

- [ ] Verify subdomain `auth.lungnote.com` exists ใน DNS (no MX, just A record placeholder)
- [x] Supabase: confirm `auth.users.email` ของ synthetic emails จะไม่ trigger validation/SMTP (`email_confirm: true` set in `lib/auth/line-session.ts`)
- [ ] Decide: ถ้า user block OA → invalidate Supabase session ทันทีไหม? (use `unfollow` event)
- [ ] Decide: rate-limit token mint per IP (กัน abuse OA) — defer until traffic
- [x] Designer: Flex Message template สำหรับ "Dashboard link" + "Welcome card" — landed in [LungNote-design#1](https://github.com/PASAKON/LungNote-design/pull/1)
- [x] Designer: Rich Menu image — landed in [LungNote-design#1](https://github.com/PASAKON/LungNote-design/pull/1), installed via `scripts/install-rich-menu.sh`
- [x] LINE Console: enable postback, set Rich Menu (default + welcome, see [[0010-liff-auth]])

## See Also

- [[0006-supabase-db-auth]]
- [[0007-line-oa-messaging]]
- [[../20-Conventions/Database-Naming]]
- [[../50-Workflows/LINE-Auth-Flow]]
- LINE: https://developers.line.biz/en/reference/messaging-api/#get-profile
- Supabase admin generateLink: https://supabase.com/docs/reference/javascript/auth-admin-generatelink
