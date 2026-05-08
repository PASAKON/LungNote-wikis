---
title: "ADR-0011: Web LINE Login (OAuth code flow) — third auth path"
tags: [adr, auth, line, oauth, web]
status: Accepted
date: 2026-05-08
---

# ADR-0011 — Web LINE Login (OAuth code flow)

## Status

Accepted

## Context

Two auth paths shipped:
- Account linking via OA chat ([[0008-line-only-auth-account-linking|ADR-0008]]) — user types `dashboard` → bot replies one-time link
- LIFF in-app ([[0010-liff-auth|ADR-0010]]) — user inside LINE app

Both require the user to start in LINE. Landing visitors on desktop browser had no way to log in directly — they had to add OA on phone first.

Want: a "เริ่มเลย" CTA on the landing page that lets a desktop browser visitor log in via LINE Login OAuth.

## Decision

Add **LINE Login OAuth code flow** as the third auth path. Reuses the same LINE Login channel (`2010007749`) and the same OIDC verify endpoint as LIFF.

### Flow

```
landing                 our server               LINE                   browser
────────                ──────────               ────                   ───────
[เริ่มเลย]
  │
  ├──GET /api/auth/line/oauth/start
  │                          │
  │   ╭─state = randomBytes(16).base64url
  │   ╰─Set-Cookie: lungnote_oauth_state=<state>; HttpOnly; SameSite=Lax; 5min
  │                          │
  │  302 → access.line.me/oauth2/v2.1/authorize
  │       ?response_type=code
  │       &client_id=2010007749
  │       &redirect_uri=https://lungnote.com/api/auth/line/oauth/callback
  │       &state=<state>&scope=openid profile&nonce=<nonce>
  │                          │
  │                          │  user signs in to LINE
  │                          │
  │  302 → /api/auth/line/oauth/callback?code=<code>&state=<state>
  │                          │
  │   ╭─Verify cookie state == query state (CSRF)
  │   ├─POST api.line.me/oauth2/v2.1/token
  │   │     code, client_id, client_secret, redirect_uri, grant_type=authorization_code
  │   │   → { access_token, id_token }
  │   ├─POST api.line.me/oauth2/v2.1/verify (existing liff-verify)
  │   │   → { sub, name, picture }
  │   ├─Upsert auth.users + lungnote_profiles (same as LIFF/account-link)
  │   ├─admin.generateLink({type:'magiclink'})
  │   ├─supabase.auth.verifyOtp on cookie-aware ssr client
  │   ╰─clear lungnote_oauth_state cookie
  │                          │
  │  302 → /dashboard
  ▼                          ▼
```

### Env Vars (new)

- `LINE_LOGIN_CHANNEL_SECRET` (server-only) — already added when LIFF channel created
- Reuses existing `LINE_LOGIN_CHANNEL_ID` for both authorize + verify

### LINE Console Setup

Callback URL must be allow-listed in LINE Login channel settings:
- `https://lungnote.com/api/auth/line/oauth/callback`
- (optional) `http://localhost:3000/api/auth/line/oauth/callback` for dev

## Consequences

**Positive**
- Desktop visitors can log in directly from landing CTA — no phone needed
- Reuses existing user model (synthetic email + line_user_id mapping)
- Same Supabase session shape as LIFF + account-linking — no auth model fragmentation
- Standard OAuth code flow — well understood, libraries available if needed later

**Negative**
- Third auth path = more surface area to maintain
- Requires `LINE_LOGIN_CHANNEL_SECRET` (which we have) — secret leak = impersonation
- CSRF state cookie + nonce → 1 extra round-trip; must HttpOnly + SameSite=Lax

**Mitigation**
- All three flows funnel into the same `upsertUserAndCreateSession()` helper — single source of truth
- State cookie 5min TTL + single-use (cleared on callback)
- Redirect URI strict match enforced by LINE
- Document fallback: if cookie state missing/mismatch → redirect to /auth/line/error?code=csrf

## Three auth paths summary

| Path | Trigger | Token source | Use case |
|------|---------|--------------|----------|
| Account linking ([[0008-line-only-auth-account-linking\|ADR-0008]]) | bot chat → tap link | one-time hash (our DB) | Cross-device, OA-first users |
| LIFF ([[0010-liff-auth\|ADR-0010]]) | LINE in-app | `liff.getIDToken()` | Inside LINE app (Rich Menu, Flex CTA) |
| Web OAuth (this ADR) | landing CTA on browser | OAuth code → id_token | Desktop browser, marketing funnel |

All three end at the same place: Supabase session cookie → `/dashboard`.

## Open questions / TODO

- [ ] LINE Console: add callback URL to allow-list
- [ ] Implement `/api/auth/line/oauth/{start,callback}` routes
- [ ] Add "เริ่มเลย" CTA on landing → `/api/auth/line/oauth/start`
- [ ] Refactor LIFF + OAuth + account-linking → shared `upsertLineUserAndSetSession()` helper
- [ ] Update Glossary: OAuth code flow, state nonce
- [ ] Rate-limit `/oauth/start` per IP (defer until traffic)

## See Also

- [[0008-line-only-auth-account-linking]]
- [[0010-liff-auth]]
- [[0006-supabase-db-auth]]
- LINE Login docs: https://developers.line.biz/en/docs/line-login/integrate-line-login/
- OAuth 2.0 RFC 6749: https://datatracker.ietf.org/doc/html/rfc6749
