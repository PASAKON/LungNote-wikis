---
title: Database Naming Conventions
tags: [conventions, database, supabase, schema]
---

# Database Naming Conventions

กฎสำหรับ DB schema ของ LungNote (Supabase Postgres).
ใช้กับทุก table / view / function / migration.

---

## 1. Table Prefix (กฎเหล็ก)

**ทุก table ใน schema `public` ต้องขึ้นต้นด้วย `lungnote_`**

```sql
-- ✅ correct
create table lungnote_profiles (...);
create table lungnote_notebooks (...);
create table lungnote_notes (...);
create table lungnote_todos (...);
create table lungnote_folders (...);
create table lungnote_tags (...);
create table lungnote_auth_link_tokens (...);

-- ❌ wrong
create table profiles (...);
create table notebooks (...);
```

**ทำไม:**
- กัน collision ถ้า Supabase project shared (e.g., มี mooniex tables ใน DB เดียว)
- Filter ง่ายตอน query `pg_tables where tablename like 'lungnote_%'`
- Backup / migration export กรองได้ตรง prefix
- Onboarding LLM/dev เห็น table ปุ๊บรู้ทันทีว่าของ LungNote

**ข้อยกเว้น:**
- Schema `auth.*`, `storage.*`, `realtime.*` = Supabase managed, ห้ามแตะ + ไม่ต้อง prefix
- Schema อื่น (เช่น `public_archive.*`) ที่เราสร้างเอง — prefix ไม่บังคับ แต่ recommend ทำเหมือนกัน

---

## 2. Table Name

| Rule | Example |
|------|---------|
| `lungnote_<entity_plural>` snake_case | `lungnote_notebooks`, `lungnote_notes` |
| Plural เสมอ | `lungnote_users` ❌ (managed by auth.users) → ใช้ `lungnote_profiles` |
| Junction table = `<a>_<b>` หลัง prefix | `lungnote_notebooks_tags` |
| ห้ามภาษาไทย | `lungnote_บันทึก` ❌ → `lungnote_notes` ✅ |

---

## 3. Column Name

- **snake_case** เสมอ (`created_at`, `line_user_id`, `display_name`)
- Foreign key = `<table_singular>_id` (`notebook_id` ใน `lungnote_notes`)
- Boolean = `is_<adjective>` หรือ `has_<noun>` (`is_archived`, `has_attachment`)
- Timestamp = `<verb>_at` (`created_at`, `updated_at`, `deleted_at`)
- Soft delete = `deleted_at timestamptz null`
- Enum-like text = ระบุ `check` constraint หรือใช้ Postgres ENUM

---

## 4. Required Columns ทุก table

```sql
id uuid primary key default gen_random_uuid(),
created_at timestamptz default now() not null,
updated_at timestamptz default now() not null
```

ยกเว้น junction table / lookup table ที่ไม่ต้องการ.

`updated_at` auto-update ผ่าน trigger:

```sql
create trigger lungnote_<table>_updated_at
  before update on lungnote_<table>
  for each row execute function moddatetime(updated_at);
```

(หรือเขียน function `set_updated_at()` ของ project เอง)

---

## 5. Index

- Foreign key ทุกอันต้องมี index: `create index lungnote_<table>_<column>_idx on lungnote_<table>(<column>);`
- Unique constraint = `create unique index ...` หรือ `unique` ในตัว table definition
- Composite index ลำดับ column ตาม cardinality (สูง → ต่ำ)

---

## 6. RLS (Row-Level Security) — กฎเหล็ก

ทุก table ใน `public` schema ต้อง:

```sql
alter table lungnote_<table> enable row level security;
```

แล้วเขียน policy อย่างน้อย 1 อัน. **ไม่มี policy = no access** (default deny).

Policy ทั่วไป:

```sql
-- read own data
create policy lungnote_<table>_select_own
  on lungnote_<table> for select
  using (auth.uid() = user_id);

-- insert own
create policy lungnote_<table>_insert_own
  on lungnote_<table> for insert
  with check (auth.uid() = user_id);

-- update own
create policy lungnote_<table>_update_own
  on lungnote_<table> for update
  using (auth.uid() = user_id);

-- delete own
create policy lungnote_<table>_delete_own
  on lungnote_<table> for delete
  using (auth.uid() = user_id);
```

Server-only table (เก็บ secret/log): RLS on, **ไม่มี policy** → access เฉพาะ service_role.

---

## 7. Migration

- ใช้ Supabase CLI: `supabase migration new <name>`
- ไฟล์อยู่ใน `webapp/supabase/migrations/<timestamp>_<name>.sql`
- ห้ามแก้ migration ที่ apply แล้ว — สร้างใหม่
- ทุก migration **idempotent ถ้าเป็นไปได้** (`create table if not exists`, `create index if not exists`)
- Down migration: ไม่บังคับ, แต่ถ้ามี เขียนใน comment ของ migration เดียวกัน

---

## 8. Function / Trigger / Type Naming

```sql
-- function
create or replace function lungnote_<verb>_<noun>(...) returns ...

-- trigger
create trigger lungnote_<table>_<event>
  before|after insert|update|delete on lungnote_<table>
  for each row execute function lungnote_<verb>_<noun>();

-- enum
create type lungnote_<noun>_status as enum ('todo', 'doing', 'done');
```

---

## 9. Audit Log (recommended)

ทุก mutation บน `lungnote_*` table ที่สำคัญ → log ใน `lungnote_audit_log`:

```sql
create table lungnote_audit_log (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id),
  table_name text not null,
  row_id uuid not null,
  action text not null check (action in ('insert','update','delete')),
  before jsonb,
  after jsonb,
  created_at timestamptz default now()
);
```

(ใช้ trigger generic หรือ application-level log แล้วแต่ scope)

---

## 10. Forbidden

- ❌ ห้ามใช้ `select * from <table>` ใน production code (ระบุ column เสมอ)
- ❌ ห้าม run DDL ผ่าน Supabase dashboard ตรงๆ — ผ่าน migration เท่านั้น
- ❌ ห้ามใช้ `service_role` key ใน client-side code
- ❌ ห้ามตั้ง column ชื่อ reserved word (`user`, `order`, `group`, `type`) — ใช้ `user_id`, `display_order`, `group_name`, `kind` แทน
- ❌ ห้าม cascade delete จาก `auth.users` ลึกเกิน 2 ระดับ (อ่านยาก, อันตราย)

---

## See Also

- [[Code-Style]]
- [[../40-Decisions/0006-supabase-db-auth]]
- [[../30-Domain/Glossary]] — RLS, PostgREST, profiles
- Supabase migration docs: https://supabase.com/docs/guides/cli/local-development#database-migrations
