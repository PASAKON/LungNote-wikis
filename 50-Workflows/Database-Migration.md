---
title: Database Migration Workflow
tags: [workflow, database, supabase, migration]
---

# Database Migration Workflow

ขั้นตอนสร้าง / แก้ / apply migration ของ Supabase.

Naming + structure ดู [[../20-Conventions/Database-Naming]].

---

## Setup (one-time per dev)

```bash
# install Supabase CLI
brew install supabase/tap/supabase
# หรือ
pnpm add -g supabase

# login (browser flow)
supabase login

# link to LungNote project
cd webapp
supabase link --project-ref qkaxvockysyazmtormvf
# จะถาม database password — ดึงจาก SUPABASE_DB_PASSWORD ใน .env.local
```

---

## สร้าง migration ใหม่

```bash
cd webapp
supabase migration new <slug>
# เช่น
supabase migration new lungnote_initial_schema
```

ไฟล์เกิดที่ `webapp/supabase/migrations/<timestamp>_<slug>.sql`. แก้ไฟล์, เขียน SQL ตาม [[../20-Conventions/Database-Naming]].

---

## Test ก่อน push

```bash
# spin up local Postgres + apply migrations ทั้งหมด (รวมอันใหม่)
supabase db reset

# (optional) เปิด local Studio
supabase start
# Studio: http://localhost:54323
```

ถ้า migration พัง → แก้ไฟล์, รัน `supabase db reset` อีกครั้ง.

---

## Apply ไป remote (Production)

⚠️ **ทำกับ remote เฉพาะหลัง PR review + merge**.

```bash
cd webapp

# dry-run: ดู diff ระหว่าง local migration กับ remote schema
supabase db diff --linked

# push migration ไป remote
supabase db push
```

CI ในอนาคต: GitHub Action → on merge to main → `supabase db push --include-all`.

---

## Rollback

ไม่มี auto down-migration. ต้องเขียน new migration ที่ undo:

```bash
supabase migration new lungnote_revert_<original-slug>
# เขียน SQL undo ใน file ใหม่
```

ห้ามแก้ migration ที่ apply แล้วโดยตรง.

---

## Schema Drift

ถ้ามีคนแก้ schema ผ่าน Supabase dashboard ตรงๆ (ผิด convention) → ดู diff:

```bash
supabase db diff --linked --schema public
```

แล้วสร้าง migration ใหม่จาก diff:

```bash
supabase db diff --linked --schema public --file lungnote_capture_drift
# review file ก่อน commit
```

---

## RLS Verify

หลัง apply ทุก migration → ทดสอบ RLS:

```sql
-- ใน Supabase SQL Editor (impersonate user)
set role authenticated;
set request.jwt.claim.sub = '<some-user-uuid>';

select * from lungnote_notes;  -- ควรเห็นเฉพาะของ user นี้
```

หรือ integration test ฝั่ง Next.js:
- sign in user A → query → expect rows ของ A เท่านั้น
- attempt insert ที่ user_id != A → expect denied

---

## Common Mistakes

- ❌ แก้ schema ผ่าน dashboard ตรงๆ (ทำให้ local migration ไม่ตรง remote)
- ❌ ลืม `enable row level security` (default = anon เห็นทุก row)
- ❌ ลืม policy → table มี RLS on แต่ไม่มี policy = default deny ทุก request
- ❌ ตั้ง column foreign key ไม่มี index (slow query)
- ❌ ใช้ `service_role` key ใน client code (bypass RLS = data leak)
- ❌ ลืม prefix `lungnote_` (ดู [[../20-Conventions/Database-Naming]])

---

## See Also

- [[../20-Conventions/Database-Naming]]
- [[../40-Decisions/0006-supabase-db-auth]]
- [[Dev-Workflow]]
- Supabase migration docs: https://supabase.com/docs/guides/cli/local-development#database-migrations
