---
title: "ADR-0009: Schema Hierarchy — Folder → Notebook → Note → Todo + Tag"
tags: [adr, db, schema, supabase]
status: Accepted
date: 2026-05-08
---

# ADR-0009 — Schema Hierarchy

## Status

Accepted

## Context

[[0006-supabase-db-auth|ADR-0006]] established Supabase as DB. Initial migration (20260508000000_lungnote_initial.sql) shipped flat schema:
- `lungnote_profiles` (1:1 auth.users)
- `lungnote_notes` (user_id → auth.users — flat, no grouping)
- `lungnote_auth_link_tokens` (server-only)

[[../30-Domain/Glossary]] declared more entities — Notebook, Folder, Tag, Todo — but they were not yet modeled. As Dashboard MVP grows, deferring grouping creates UI dead-ends + data migration debt later.

## Decision

Add **hierarchical schema** in one migration:

```
auth.users
   │
   ├─1:1─ lungnote_profiles
   │
   ├─1:N─ lungnote_folders                ← optional grouping (nullable parent)
   │         │
   │         └─1:N─ lungnote_notebooks    ← container with cover_color
   │                  │
   │                  └─1:N─ lungnote_notes   ← title + body
   │                            │
   │                            └─1:N─ lungnote_todos   ← checkbox items
   │
   └─1:N─ lungnote_tags
            └─M:N─ lungnote_notes_tags     ← junction
```

### Rules

- **All FKs to `auth.users(id) on delete cascade`** — uniform ownership tree
- **Notes/Todos/Tags carry `user_id` directly** (denormalized) for cheap RLS via `auth.uid() = user_id`
- **Notebook is optional for notes** (`notebook_id nullable`) — backward compatible with existing flat notes
- **Folder is optional for notebooks** (`folder_id nullable`) — flat by default
- **Folder self-referential parent** (`parent_folder_id nullable`) — limit nesting depth in app code (e.g. ≤ 3 levels)
- **Todo `position`** integer for ordering (renumber on insert; client requests with `order by position asc`)
- **Tag `name` UNIQUE per user** (composite unique on `(user_id, name)`)
- **Junction `lungnote_notes_tags`** PK = `(note_id, tag_id)`, no surrogate id
- **Cascade delete throughout** — delete notebook → delete its notes → delete their todos
- **`updated_at` trigger** via `moddatetime` on every table

### RLS

Every table:
```sql
alter table lungnote_<x> enable row level security;
create policy <x>_select_own on <x> for select using (auth.uid() = user_id);
create policy <x>_insert_own on <x> for insert with check (auth.uid() = user_id);
create policy <x>_update_own on <x> for update using (auth.uid() = user_id);
create policy <x>_delete_own on <x> for delete using (auth.uid() = user_id);
```

Junction `lungnote_notes_tags` — RLS via subquery:
```sql
create policy notes_tags_own on lungnote_notes_tags for all
  using (
    exists (select 1 from lungnote_notes where id = note_id and user_id = auth.uid())
    and exists (select 1 from lungnote_tags where id = tag_id and user_id = auth.uid())
  );
```

### Indexes

- Every FK indexed
- `lungnote_notes(user_id, updated_at desc)` for dashboard list
- `lungnote_todos(note_id, position)` for ordered fetch
- `lungnote_tags(user_id, name)` unique

## Consequences

**Positive**
- Full hierarchy ready — no second migration when UI catches up
- Backward-compatible (nullable notebook_id) — existing flat notes still work
- Tag M:N opens label/filter UX
- Cascade chain = simple delete semantics

**Negative**
- More tables = more RLS to maintain
- Folder self-FK risks cycles — must guard in app (or add `with recursive` check trigger later)
- Cascade delete depth = need careful UX confirms ("ลบ folder นี้จะลบ N notebook + M notes + K todos")
- Schema drift possible — drift detection needed during dev

**Mitigation**
- App guards folder depth (≤ 3 levels) at the form layer
- Cascade-delete UX shows count before confirm
- Generate Database type from `supabase gen types` after migration applied

## Alternatives ที่ไม่เลือก

- **Flat (status quo)** — won't scale, repeated work later
- **Notebook only, no Folder/Tag** — half-step, will add Folder/Tag soon anyway
- **Single `parent_id` polymorphic column** (note belongs to either notebook OR folder) — too clever, RLS hard
- **EAV / generic kv** — premature, throws away SQL benefits

## Migration plan

File: `webapp/supabase/migrations/20260508120000_lungnote_hierarchy.sql`

Steps inside the migration:
1. `lungnote_folders` (with self-FK `parent_folder_id`)
2. `lungnote_notebooks` (FK `folder_id`)
3. `alter lungnote_notes add column notebook_id uuid null references lungnote_notebooks(id) on delete set null;`
4. `lungnote_todos`
5. `lungnote_tags`
6. `lungnote_notes_tags` (junction)
7. RLS + policies for all new tables
8. updated_at triggers
9. indexes

No data migration needed — all new tables empty, existing notes keep flat (notebook_id NULL).

## Open questions / TODO

- [x] Apply migration to remote Supabase (`webapp/supabase/migrations/20260508120000_lungnote_hierarchy.sql` applied)
- [x] Regen `webapp/src/lib/supabase/database.types.ts` — hand-updated for all hierarchy tables
- [ ] Update Dashboard to:
  - List notebooks (sidebar / filter) — **not built** (folders/notebooks/tags UI ยังว่าง, ดู [[0012-unified-todo-memory-model|ADR-0012]])
  - "+ New notebook" + cover color picker — **not built**
  - Notes in current notebook (or "All" / "Inbox") — Inbox done, notebook filter not built
  - Inline checklist (todos) inside note edit — done via `/dashboard/todo`
  - Tag chip + filter — **not built**
- [ ] Update `lungnote_notes_tags` Glossary entry — pending UI
- [ ] Decide: nesting depth limit for `lungnote_folders` (recommend ≤ 3, enforce in app) — pending UI

## See Also

- [[0006-supabase-db-auth]]
- [[0008-line-only-auth-account-linking]]
- [[../20-Conventions/Database-Naming]]
- [[../30-Domain/Glossary]]
- [[../50-Workflows/Database-Migration]]
