---
title: LINE Auth Flow (Account Linking)
tags: [workflow, auth, line, sequence]
---

# LINE Auth Flow (Account Linking)

Per [[../40-Decisions/0008-line-only-auth-account-linking|ADR-0008]] — user authenticates only via LINE OA, web Dashboard accessed through one-time link.

## Sequence

```
┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  User   │   │ LINE OA  │   │ webhook  │   │ /auth/   │   │ Supabase │
│ (LINE)  │   │  (bot)   │   │ /api/... │   │ line     │   │  Auth    │
└────┬────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘
     │ message     │              │              │              │
     │────────────▶│              │              │              │
     │             │ webhook POST │              │              │
     │             │─────────────▶│              │              │
     │             │              │ verify sig   │              │
     │             │              │ mintToken()  │              │
     │             │              │ INSERT       │              │
     │             │              │ lungnote_    │              │
     │             │              │ auth_link_   │              │
     │             │              │ tokens       │              │
     │             │ Flex w/ URL  │              │              │
     │             │◀─────────────│              │              │
     │ link msg    │              │              │              │
     │◀────────────│              │              │              │
     │ tap link    │              │              │              │
     │──────────────────────────────────────────▶│              │
     │             │              │              │ verify token │
     │             │              │              │ fetch LINE   │
     │             │              │              │ profile      │
     │             │              │              │ upsert user  │
     │             │              │              │─────────────▶│
     │             │              │              │ generateLink │
     │             │              │              │◀─────────────│
     │ 302 → magic link                          │              │
     │◀──────────────────────────────────────────│              │
     │ GET magic link                                           │
     │─────────────────────────────────────────────────────────▶│
     │                                           │ set cookie   │
     │ 302 → /dashboard                                         │
     │◀─────────────────────────────────────────────────────────│
     │ GET /dashboard                                           │
     │ (cookie attached)                                         │
     │ → notebook list (RLS-scoped)                             │
```

## Components

| Component | Path | Type |
|---|---|---|
| LINE bot trigger | LINE Console Rich Menu / quick reply postback | Config |
| Webhook handler | `webapp/src/app/api/line/webhook/route.ts` | Route handler |
| Token mint/redeem | `webapp/src/lib/auth/line-link.ts` | Server lib |
| LINE profile fetch | `webapp/src/lib/line/profile.ts` | Server lib |
| Synthetic email helper | `webapp/src/lib/auth/synthetic-email.ts` | Pure fn |
| Admin Supabase client | `webapp/src/lib/supabase/admin.ts` | Service-role |
| Auth callback | `webapp/src/app/auth/line/route.ts` | Route handler |
| Dashboard guard | `webapp/src/app/[locale]/dashboard/layout.tsx` | RSC |
| Notes CRUD | `webapp/src/app/[locale]/dashboard/notes/{page,actions}.tsx` | RSC + Server Action |

## Trigger Patterns

User can request a Dashboard link via:

| Trigger | Content |
|---|---|
| Type "dashboard" / "เปิด" / "ลิงก์" | text message → bot reply Flex |
| Tap Rich Menu "📊 Dashboard" | postback `action=open_dashboard` |
| Type "/login" or "/dash" | text message → same handler |

## Token Lifecycle

```
mint:
  token = randomBytes(32).toString('base64url')        ← 256-bit entropy
  hash  = sha256(token).hex                            ← stored
  INSERT lungnote_auth_link_tokens
    (line_user_id, token_hash, expires_at = now+5min)

redeem:
  hash = sha256(input).hex
  row = SELECT * WHERE token_hash = hash
                    AND used_at IS NULL
                    AND expires_at > now()
  if !row → 403
  UPDATE lungnote_auth_link_tokens SET used_at = now() WHERE id = row.id
  return row.line_user_id
```

## Edge Cases

| Case | Handling |
|---|---|
| Expired token | `403`, reply user "ลิงก์หมดอายุ — พิมพ์ 'dashboard' อีกครั้ง" |
| Used token | `403`, same reply |
| Multiple devices | OK — each device gets a different cookie session |
| User unfollow OA | optional: revoke Supabase session via `unfollow` event handler |
| LINE profile fetch fails | use last cached `lungnote_profiles.line_display_name`; if no cache → use "ผู้ใช้ LINE" |
| First-time user | `auth.admin.createUser` + insert profile, then magic link |
| Returning user | `auth.admin.getUserById` (by email lookup); update profile (displayName/picture refresh) |

## Local Dev

LINE webhook ต้องเป็น public HTTPS. ใช้ `ngrok`:

```bash
# terminal 1
cd webapp && pnpm dev                 # localhost:3000

# terminal 2
ngrok http 3000                       # ngrok HTTPS URL → forward to localhost

# LINE Console: temporarily set webhook URL to ngrok URL
# https://abc123.ngrok-free.app/api/line/webhook
```

Test:
1. add OA in LINE app → bot welcome
2. type "dashboard" → bot reply with link (uses ngrok URL)
3. tap link → ngrok → localhost → Supabase magic link → /dashboard

หลัง dev เสร็จ → revert webhook URL กลับ `https://lungnote.com/api/line/webhook` ใน LINE Console.

## See Also

- [[../40-Decisions/0008-line-only-auth-account-linking]]
- [[../40-Decisions/0007-line-oa-messaging]]
- [[../20-Conventions/Database-Naming]]
- [[Database-Migration]]
