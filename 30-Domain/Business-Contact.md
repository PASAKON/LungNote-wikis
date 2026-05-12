---
title: Business Contact & Operator Identity
tags: [domain, business, contact, accounts]
---

# Business Contact & Operator Identity

Reference page เก็บ contact + account identity ของ **LungNote** project. ไว้สำหรับ future-self / new team member จะได้รู้ว่า service ไหนใช้ email อะไร log in.

> ⚠️ **ไม่ใช่ end-user auth.** LungNote ใช้ LINE-only auth ([[../40-Decisions/0008-line-only-auth-account-linking|ADR-0008]]). Email + phone บนหน้านี้ = **operator/owner identity** สำหรับ 3rd-party services เท่านั้น.

---

## Primary Identity

| Channel | Value |
|---------|-------|
| **Email** | `lungnote.official@gmail.com` |
| **Phone** | `+66 83 775 8831` |

---

## ใช้ที่ไหนบ้าง (planned)

### Operator sign-in (via Google OAuth หรือ email/password)

| Service | Purpose | สถานะ |
|---------|---------|------|
| Vercel | Deploy webapp, env vars, billing — see [[../40-Decisions/0005-deploy-vercel|ADR-0005]] | TBD |
| Supabase | DB + Auth project `qkaxvockysyazmtormvf` — see [[../40-Decisions/0006-supabase-db-auth|ADR-0006]] | TBD |
| GitHub `PASAKON` org | 3 repos (webapp / wikis / design) — see [[../40-Decisions/0002-split-repos-webapp-wikis-design|ADR-0002]] | TBD |
| LINE Developer Console | Messaging channel `2010007749` + LIFF — see [[../40-Decisions/0007-line-oa-messaging|ADR-0007]] | TBD |
| OpenRouter | LLM API key (`OPENROUTER_API_KEY`) billing | TBD |
| Namecheap | Domain `lungnote.com` registrar | TBD |
| Cloudflare | DNS / WAF (ถ้าใช้ในอนาคต) | n/a |
| Stripe | Payment (ถ้ามี paid plan ในอนาคต) | n/a |
| Sentry / PostHog | Error / analytics (ถ้าเพิ่มภายหลัง) | n/a |

### Outbound business communication

- **Sales / partnership** — รับ business inquiry
- **Education plan** — Glossary ระบุ `school@lungnote.app` (ไม่ตรงโดเมน production `.com` — flag for cleanup)
- **Press / legal / DMCA** — official channel
- **User support** — manual reply หาก user ติดต่อนอก LINE OA

### Inbound notifications (รับเตือน)

- Vercel billing alerts + deploy failures
- Supabase project alerts (DB pause, free-tier warning)
- LINE Developer Console policy changes
- GitHub security alerts ของ 3 repos
- Namecheap domain renewal — **ห้ามให้หมดอายุ**

### Recovery / 2FA

- Email = account recovery anchor ของทุก service ข้างบน
- Phone = SMS 2FA **backup** (Authenticator app เป็นหลัก)

---

## Optional Upgrade — Google Workspace ($6/mo)

ถ้าซื้อ Workspace ภายหลัง สามารถสร้าง `*@lungnote.com` aliases:

| Alias | Use |
|-------|-----|
| `hello@lungnote.com` | General contact |
| `support@lungnote.com` | User support |
| `school@lungnote.com` | Education plan (แก้ `school@lungnote.app` ที่ Glossary ระบุผิด) |
| `legal@lungnote.com` | Legal / DMCA |
| `noreply@lungnote.com` | Transactional sender |

Forward ทุกอันมา `lungnote.official@gmail.com`.

---

## See Also

- [[Glossary]] — Business section
- [[../40-Decisions/0002-split-repos-webapp-wikis-design]]
- [[../40-Decisions/0005-deploy-vercel]]
- [[../40-Decisions/0006-supabase-db-auth]]
- [[../40-Decisions/0007-line-oa-messaging]]
- [[../40-Decisions/0008-line-only-auth-account-linking]]
