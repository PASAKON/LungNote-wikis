---
title: "ADR-0010: LIFF for In-LINE Auth (hybrid with Account Linking)"
tags: [adr, auth, line, liff]
status: Accepted
date: 2026-05-08
---

# ADR-0010 — LIFF for In-LINE Auth

## Status

Accepted

## Context

[[0008-line-only-auth-account-linking|ADR-0008]] picked **Account Linking** (one-time token in chat link) as the only auth path. It works in any browser tab, but in real LINE-app usage:

- User taps a Flex button inside chat → opens in-LINE WebView
- Account-linking link still issues a fresh token round-trip
- Each tap consumes one token (single-use, 5 min TTL) — wasteful if user re-taps the menu / Flex card multiple times
- Designer Flex template carries placeholder `https://liff.line.me/YOUR_LIFF_ID/dashboard` — built for LIFF originally
- Rich Menu (designer-supplied) targets `liff.line.me` URLs

LIFF lets the in-app WebView know the LINE user identity with zero round-trips:

- `liff.init({liffId})` — reads bot link credential
- `liff.getIDToken()` — JWT signed by LINE, server can verify and extract `sub` (LINE userId)
- Works only inside LINE WebView (or after `liff.login()` redirects)

## Decision

**Hybrid:** keep Account Linking, ADD LIFF as the preferred path inside LINE app.

```
User in LINE app
   ├── chat OA → "dashboard" → bot reply with one-time link
   │      → tap → /auth/line?t=<token> → cookie set → /dashboard
   │   (account linking, ADR-0008 — fallback / external browsers)
   │
   └── Rich Menu / Flex button → liff.line.me/<LIFF_ID>/dashboard
          → in-app WebView opens our /liff page
          → liff.init + liff.getIDToken
          → POST /api/auth/liff { idToken }
          → server verifies via LINE OIDC + mints Supabase session
          → cookie set → /dashboard
       (LIFF, this ADR)
```

Both paths converge on the same `auth.users` + `lungnote_profiles` rows (keyed by `line_user_id`).

### Concrete

- **LIFF channel**: add **LINE Login** product to the existing Messaging API channel (no separate channel)
- **LIFF endpoint URL**: `https://lungnote.com/liff`
- **Scope**: `profile openid` (no email — synthetic email pattern from ADR-0008 still applies)
- **Bot link**: ON (so the LIFF app and the OA share `userId` namespace)
- **Env var**: `NEXT_PUBLIC_LINE_LIFF_ID` — public (loaded by client SDK)
- **id_token verification**: POST `https://api.line.me/oauth2/v2.1/verify` with `id_token` + `client_id` (= LINE Login channel ID, also added as env `LINE_LOGIN_CHANNEL_ID`). Server-side, no SDK needed for verify.
- **Client SDK**: `@line/liff` (tiny, ~30 KB)

### Code shape

| Path | Type | Purpose |
|------|------|---------|
| `src/app/liff/page.tsx` | client RSC | init LIFF, get id_token, POST to api, redirect |
| `src/app/api/auth/liff/route.ts` | route handler | verify id_token, upsert user, mint Supabase session |
| `src/lib/auth/liff-verify.ts` | server lib | call LINE verify endpoint, return claims |
| `src/lib/line/flex.ts` | server lib | URL rewrite — point Flex buttons at `liff.line.me/<LIFF_ID>/dashboard` (instead of one-time auth-link URL) |
| `src/proxy.ts` | middleware | add `/liff` to `LOCALE_FREE_PREFIXES` (no locale rewrite) |

### Session reuse

Once cookie is set on `lungnote.com`, all subsequent requests authenticate via `@supabase/ssr` middleware. LIFF is only needed at first entry; no per-request id_token check.

## Consequences

**Positive**
- 0 token round-trip — Rich Menu and Flex buttons work straight to dashboard
- Per-tap idempotent (no token waste)
- Designer Flex template URLs work as-is (`liff.line.me/<LIFF_ID>/...`)
- Fallback path (account linking) remains for desktop browsers / shared links

**Negative**
- 2 auth paths to maintain (LIFF + account linking)
- LIFF pages only work inside LINE WebView OR via `liff.login()` redirect (extra OAuth bounce)
- Client-side flash before redirect (init + token fetch + POST takes ~500 ms)
- One more env var (`NEXT_PUBLIC_LINE_LIFF_ID`) + LINE Login channel ID
- Adds `@line/liff` dep (small, tree-shaken)

**Mitigation**
- Skeleton/loading UI on `/liff` page (no flash of empty)
- Retry button if `liff.init` fails
- Both flows share the same upsert + magic-link logic — DRY in `lib/auth/`

## Alternatives ที่ไม่เลือก

- **LIFF only** — breaks desktop browser users + shared links
- **Account linking only (status quo)** — wastes tokens on every Rich Menu tap; designer Flex templates need full rewrite
- **OAuth Authorization Code flow without LIFF** — LIFF gives free `getProfile` and bot-link unification; no reason to roll our own

## Open questions / TODO

- [x] User: LINE Login product enabled on Messaging API channel `2010007749`
- [x] User: LIFF app created — `LIFF_ID=2010007749-Ms996Bfr`, endpoint `https://lungnote.com/liff`, scope `profile openid`, Bot Link ON
- [x] User: env vars set in Vercel — `NEXT_PUBLIC_LINE_LIFF_ID`, `LINE_LOGIN_CHANNEL_ID=2010007749`, `LINE_LOGIN_CHANNEL_SECRET`
- [x] Install `@line/liff`
- [x] Implement `/liff` page + `/api/auth/liff` + `lib/auth/liff-verify`
- [x] Update Flex URL rewrite to use LIFF URL if env set, fallback to account-linking URL
- [x] Designer login states wired (spinner / success / error) at `/liff` (designer's `preview-all-states.html`)
- [x] Wire designer Rich Menu config — installed via [`scripts/install-rich-menu.sh`](https://github.com/PASAKON/LungNote-webapp/blob/main/scripts/install-rich-menu.sh), substitutes `YOUR_LIFF_ID` → `2010007749-Ms996Bfr`
- [x] Migrate auth paths to shared `lib/auth/line-session.ts#upsertLineUserAndSetSession` (LIFF + OAuth use it; account-linking migration tracked in next refactor)
- [ ] Telemetry: log LIFF vs account-linking vs OAuth ratio (Vercel Analytics)

## See Also

- [[0007-line-oa-messaging]]
- [[0008-line-only-auth-account-linking]]
- LINE LIFF docs: https://developers.line.biz/en/docs/liff/overview/
- LINE id_token verify: https://developers.line.biz/en/reference/line-login/#verify-id-token
