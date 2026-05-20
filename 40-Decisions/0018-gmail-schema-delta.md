---
title: "ADR-0018: Gmail Schema Delta — tables + source enum extension"
tags: [adr, db, schema, supabase, gmail]
status: Proposed
date: 2026-05-20
---

# ADR-0018 — Gmail Schema Delta

## Status

Proposed

## Context

[[0017-gmail-integration-readonly-v1|ADR-0017]] ต้องการ persistence layer
สำหรับ:

1. **Gmail OAuth connection** ต่อ user — เก็บ encrypted refresh_token, scope,
   status, sync cursor
2. **Synced message index** — ป้องกัน double-extract เมลเดียวกันสองครั้ง +
   เก็บผลตัดสินของ AI (`is_todo`)
3. **Origin tag** บน to-do ที่มาจาก email — ขยาย enum `lungnote_todos.source`
   ที่ [[0012-unified-todo-memory-model|ADR-0012]] กำหนด `('chat','web','liff')`

Existing schema (per [[0009-lungnote-hierarchy-schema|ADR-0009]] +
[[0012-unified-todo-memory-model|ADR-0012]]):

```sql
-- lungnote_todos (relevant columns)
source text not null default 'web'
  check (source in ('chat','web','liff')),
due_at timestamptz,
due_text text,
note_id uuid not null references lungnote_notes(id) on delete cascade,
user_id uuid not null references auth.users(id) on delete cascade,
position int not null,
text text not null check (length(text) between 1 and 2000),
done boolean not null default false,
created_at timestamptz, updated_at timestamptz
```

ปัจจุบัน `lungnote_todos` ยังไม่มีคอลัมน์เก็บ external origin id / URL.
ผ่านมาทุก source อยู่ใน LungNote เอง.

## Decision

Migration ใหม่ `webapp/supabase/migrations/<YYYYMMDDHHMMSS>_lungnote_gmail.sql`
ทำใน transaction เดียว:

### 1. Extend `lungnote_todos.source` enum

```sql
alter table lungnote_todos drop constraint if exists lungnote_todos_source_check;
alter table lungnote_todos
  add constraint lungnote_todos_source_check
  check (source in ('chat','web','liff','email'));
```

### 2. Add origin tracking columns ที่ `lungnote_todos`

```sql
alter table lungnote_todos
  add column source_external_id text,           -- e.g. Gmail message_id, future: LINE message_id, etc.
  add column source_url text;                   -- e.g. Gmail permalink, deep link
```

ทั้งสอง nullable — รองรับ source='chat'/'web'/'liff' ที่ไม่มี external ref.
Constraint: ถ้า `source='email'` → ต้องมี `source_external_id` (enforce ใน
app layer ก่อน, ภายหลังเพิ่ม partial check ถ้าจำเป็น).

Index:

```sql
create unique index lungnote_todos_user_source_external_idx
  on lungnote_todos(user_id, source, source_external_id)
  where source_external_id is not null;
```

= กัน double-insert todo จาก message_id เดียวกัน.

### 3. New table `lungnote_gmail_connections`

```sql
create table lungnote_gmail_connections (
  id                          uuid primary key default gen_random_uuid(),
  user_id                     uuid not null references auth.users(id) on delete cascade,
  google_user_id              text not null,
  email                       text not null,
  refresh_token_enc           bytea not null,          -- AES-256-GCM (IV || ciphertext || tag)
  access_token_enc            bytea,                   -- cached, can be null
  access_token_expires_at     timestamptz,
  scope                       text not null,           -- space-delimited Google scopes
  status                      text not null default 'active'
                                check (status in ('active','revoked','error','expired')),
  last_error                  text,                    -- last sync error message, nullable
  last_history_id             text,                    -- Gmail historyId watermark, nullable on first
  last_synced_at              timestamptz,
  watch_expires_at            timestamptz,             -- Gmail users.watch() expiration (max 7d, null pre-watch)
  watch_resource_state        text,                    -- e.g. 'active', 'expired' — debug field
  created_at                  timestamptz not null default now(),
  updated_at                  timestamptz not null default now(),
  constraint lungnote_gmail_connections_user_unique unique (user_id)
);
```

v1: `unique(user_id)` = 1 Gmail / user. multi-account v2 = drop constraint +
add `is_default` flag.

### 4. New table `lungnote_gmail_synced_messages`

```sql
create table lungnote_gmail_synced_messages (
  id                uuid primary key default gen_random_uuid(),
  user_id           uuid not null references auth.users(id) on delete cascade,
  connection_id    uuid not null references lungnote_gmail_connections(id) on delete cascade,
  message_id        text not null,                              -- Gmail messages.id
  thread_id         text,
  internal_date     timestamptz,                                -- Gmail internalDate
  from_truncated    text,                                       -- "name <domain>" (truncated, debug only)
  subject_truncated text,                                       -- first 80 chars (debug + dedup heuristic)
  scanned_at        timestamptz not null default now(),
  is_todo           boolean not null default false,             -- AI judgment
  todo_id           uuid references lungnote_todos(id) on delete set null,
  ai_reason         text,                                       -- short AI reasoning (debug, <= 200 chars)
  constraint lungnote_gmail_synced_messages_unique unique (user_id, message_id)
);
```

`from_truncated` + `subject_truncated` = debug only (trace viewer), ไม่ใช่
full email content storage. cap subject 80 chars + from "Name <domain.com>"
(strip local part email — privacy minimal).

### 5. updated_at triggers

```sql
create trigger lungnote_gmail_connections_updated_at
  before update on lungnote_gmail_connections
  for each row execute function moddatetime(updated_at);
```

`moddatetime` extension มีอยู่แล้วจาก [[0009-lungnote-hierarchy-schema|ADR-0009]]
migrations.

### 6. Indexes

```sql
create index lungnote_gmail_connections_user_id_idx     on lungnote_gmail_connections(user_id);
create index lungnote_gmail_connections_status_idx      on lungnote_gmail_connections(status)
  where status = 'active';                                -- cron filter
create index lungnote_gmail_synced_messages_user_idx    on lungnote_gmail_synced_messages(user_id);
create index lungnote_gmail_synced_messages_conn_idx    on lungnote_gmail_synced_messages(connection_id);
create index lungnote_gmail_synced_messages_todo_idx    on lungnote_gmail_synced_messages(todo_id)
  where todo_id is not null;
create index lungnote_gmail_synced_messages_scanned_idx on lungnote_gmail_synced_messages(user_id, scanned_at desc);
```

### 7. RLS policies (deny-by-default per [[0006-supabase-db-auth|ADR-0006]])

```sql
alter table lungnote_gmail_connections     enable row level security;
alter table lungnote_gmail_synced_messages enable row level security;

-- lungnote_gmail_connections: owner only
create policy gmail_conn_select_own on lungnote_gmail_connections
  for select using (auth.uid() = user_id);
create policy gmail_conn_insert_own on lungnote_gmail_connections
  for insert with check (auth.uid() = user_id);
create policy gmail_conn_update_own on lungnote_gmail_connections
  for update using (auth.uid() = user_id);
create policy gmail_conn_delete_own on lungnote_gmail_connections
  for delete using (auth.uid() = user_id);

-- lungnote_gmail_synced_messages: owner only
create policy gmail_msg_select_own on lungnote_gmail_synced_messages
  for select using (auth.uid() = user_id);
create policy gmail_msg_insert_own on lungnote_gmail_synced_messages
  for insert with check (auth.uid() = user_id);
create policy gmail_msg_update_own on lungnote_gmail_synced_messages
  for update using (auth.uid() = user_id);
create policy gmail_msg_delete_own on lungnote_gmail_synced_messages
  for delete using (auth.uid() = user_id);
```

Server-side cron / token refresh ใช้ `service_role` key bypass RLS (default
Supabase behavior). Caller ต้อง pass `user_id` explicit ใน query เสมอ.

### 8. Type regen

```bash
pnpm supabase gen types typescript --linked > webapp/src/lib/supabase/database.types.ts
```

หรือ hand-update ถ้าโปรเจคยังไม่ใช้ auto-gen (per [[0009-lungnote-hierarchy-schema|ADR-0009]]
TODO).

## Consequences

**Positive**

- Schema extension แบบ additive — ไม่ break existing migrations / data
- RLS pattern เหมือนตารางเดิมทุกใบ → audit consistent
- Unique constraint ป้องกัน duplicate todo จาก message_id เดียวกัน
- `from_truncated` + `subject_truncated` debug ได้โดยไม่เก็บ PII เต็ม
- ทุกความสัมพันธ์ cascade delete จาก `auth.users` → ลบ user = ลบ Gmail data ครบ

**Negative**

- `lungnote_todos.source` check constraint = drop + recreate ทุกครั้งที่
  เพิ่ม source ใหม่ (LIFF, email, ในอนาคต SMS, calendar, ฯลฯ). Constraint
  rebuild lock table สั้นๆ — OK ในขนาดปัจจุบัน, อาจต้อง `NOT VALID` +
  `VALIDATE CONSTRAINT` แบบ async ถ้า table โต > 100k rows
- `refresh_token_enc` ใน `bytea` = ดึงผ่าน Supabase REST จะถูก base64 — OK
  แต่ต้อง decode ฝั่ง server
- `lungnote_gmail_synced_messages` จะโต linearly ตามจำนวน email scanned —
  ต้อง pruning policy ภายหลัง (เช่น เก็บ 90 วัน) = defer

**Mitigation**

- Source enum migration ทดสอบใน staging ก่อน push prod
- Pruning cron = ADR เพิ่มเติมเมื่อตารางถึง 1M rows
- Bytea field = unit test ทดสอบ encrypt → store → decrypt → match

## Alternatives ที่ไม่เลือก

- **เก็บ refresh_token plaintext** — ขัด security policy ทันที, audit fail
- **ใช้ Supabase Vault** สำหรับ encrypt — Vault feature ยังเป็น beta + lock-in
  เพิ่ม → ใช้ pg_crypto / Node crypto แทน
- **Single big `external_integrations` table** (polymorphic) — generic แต่
  schema flexibility ต่ำ, RLS หยาบ. ทำ table แยกต่อ provider clearer
- **Store full message body** — ละเมิด privacy minimum, scope creep
- **ไม่มี `lungnote_gmail_synced_messages`** (re-fetch + AI judge ทุกครั้ง) —
  เปลือง Gemini Flash + Gmail quota, ไม่มี audit trail

## Open questions / TODO

- [ ] Confirm `moddatetime` extension มีใน production project (ดูจาก
  migration `20260508120000_lungnote_hierarchy.sql`)
- [ ] เลือก timestamp ของ migration file (`YYYYMMDDHHMMSS_lungnote_gmail.sql`)
- [ ] ทดสอบ encrypt round-trip ใน Vitest + integration test (RLS scoping)
- [ ] Decide pruning policy ภายหลัง (เก็บ scanned messages กี่วัน)
- [ ] Run `supabase gen types` หลัง migration applied
- [ ] Update [[../30-Domain/Schema-Relationships]] เพิ่ม diagram ใหม่
- [ ] Update [[../20-Conventions/Database-Naming]] ถ้ามี pattern ใหม่ (เช่น
  `*_enc` suffix สำหรับ encrypted column)

## See Also

- [[0006-supabase-db-auth]]
- [[0009-lungnote-hierarchy-schema]]
- [[0012-unified-todo-memory-model]]
- [[0017-gmail-integration-readonly-v1]]
- [[0019-gmail-extract-agent-tool]]
- [[../20-Conventions/Database-Naming]]
- [[../50-Workflows/Database-Migration]]
