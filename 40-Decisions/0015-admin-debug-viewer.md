---
title: "ADR-0015: Admin debug viewer at admin.lungnote.com (magic-link email auth)"
tags: [adr, admin, observability, auth]
status: Accepted
date: 2026-05-12
---

# ADR-0015 — Admin debug viewer: `admin.lungnote.com` + magic-link email auth

## Status

Accepted (backfilled — initial in [LungNote-webapp#26](https://github.com/PASAKON/LungNote-webapp/pull/26), magic-link auth refactor in [#50](https://github.com/PASAKON/LungNote-webapp/pull/50))

## Context

After [[0014-observability-chat-traces|ADR-0014]] introduced `lungnote_chat_traces`, we needed a UI to browse them — filter by user, drill into one turn, see tool calls + cost. Production needs:

- Operator-only access (not exposed to LINE users)
- Independent auth from LINE LIFF flow ([[0010-liff-auth|ADR-0010]]) — admin should log in from any desktop browser without LINE
- No password (avoids storage + rotation complexity)
- Host-isolated from main app to avoid cookie collision with `@supabase/ssr` user session

## Decision

Mount the admin viewer at **`admin.lungnote.com`** (subdomain) with the following architecture:

### Routes

```
/admin/login                    — magic-link request form
/admin/auth/callback            — exchanges magic-link token → Supabase session
/admin/                         — 24h summary dashboard
/admin/traces                   — paginated trace list w/ filters
/admin/traces/[id]              — single-trace detail (tool calls, meta, error)
```

Listed in the build manifest as `ƒ /admin/...` (server-rendered on demand).

### Auth — Supabase magic-link, no password

- User submits email at `/admin/login` → server action calls `supabase.auth.signInWithOtp({ email, options: { emailRedirectTo: <callback> } })`
- Supabase emails a one-time link → user taps → `/admin/auth/callback` exchanges code → Supabase cookie set
- Magic link TTL = 1 hour (Supabase default)
- **Allowlist gate**: `ADMIN_EMAILS` env (comma-separated). Every admin page calls `getAdminProfile()` (`src/lib/admin/auth.ts`) which loads the Supabase user and **rejects emails not in the allowlist**. Empty allowlist = no access at all.

### Cookie isolation

- Cookies are **host-scoped** to `admin.lungnote.com` (no `Domain=.lungnote.com` setting) — see [`af8cf2b`](https://github.com/PASAKON/LungNote-webapp/commit/af8cf2b) which removed the apex domain sharing after it caused stale session bleed between admin and LIFF
- Admin session and LIFF session are **independent**: signing in to admin does not affect LIFF auth and vice versa
- Same Next.js codebase serves both; Vercel handles the subdomain split

### Theme

Visually marked with a red accent (`/admin/login` `red theme`) so operators never mistake an admin page for a user page.

## Consequences

**Positive**
- Zero password to manage — Supabase handles email delivery + token TTL
- Allowlist via env var = bootstrappable from Vercel UI, no DB seed
- Subdomain isolation means a session bug on one side can't compromise the other (proven empirically by recent cookie issues, which surfaced at apex but never touched admin)
- Reuses existing Supabase project — no new auth vendor

**Negative**
- One more subdomain to DNS + Vercel config
- Magic-link delivery depends on Supabase SMTP (free tier limit applies)
- `ADMIN_EMAILS` env change requires redeploy to take effect (acceptable for low-churn allowlist)
- Magic-link UX has a one-tab redirect — confusing on mobile if user opens link in a different browser than they started the flow in

**Mitigation**
- Document allowlist update in [[../50-Workflows/Multi-Repo-Workflow]] (env var add → redeploy)
- Magic-link "sent" screen explicitly says "open the link on this device"
- Production redirect base honored via `NEXT_PUBLIC_SITE_URL` — preview deploys still work

## Alternatives ที่ไม่เลือก

- **Password auth** — extra storage + rotation
- **LINE Login OAuth for admin** — couples admin tooling to LINE; awkward when LINE OA is the system under debug
- **HTTP Basic auth at Vercel edge** — no audit trail, hard to revoke per-user
- **Path-mounted `/admin` on apex** — cookie collision risk with LIFF / @supabase/ssr (already burned us via [`af8cf2b`](https://github.com/PASAKON/LungNote-webapp/commit/af8cf2b))

## Open questions / TODO

- [ ] Add per-row "replay" action (re-run a turn against current agent) — deferred
- [ ] Rate-limit `/admin/login` per IP — defer until traffic
- [ ] CSV export of `lungnote_chat_traces` for offline analysis

## See Also

- [[0014-observability-chat-traces]] — data source for the viewer
- [[0006-supabase-db-auth]] — Supabase Auth provides magic-link
- [[0008-line-only-auth-account-linking]] — contrast: user auth via LINE, admin auth via email
- Code: [`src/app/admin/`](https://github.com/PASAKON/LungNote-webapp/tree/main/src/app/admin), [`src/lib/admin/`](https://github.com/PASAKON/LungNote-webapp/tree/main/src/lib/admin)
