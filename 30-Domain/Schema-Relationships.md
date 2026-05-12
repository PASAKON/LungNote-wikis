---
title: Schema Relationships & Access Patterns
tags: [domain, database, supabase, patterns]
---

# Schema Relationships & Access Patterns

How tables in [[../40-Decisions/0009-lungnote-hierarchy-schema|ADR-0009]] connect, and the common queries that turn the schema into product features.

Naming convention: [[../20-Conventions/Database-Naming]]. Migration workflow: [[../50-Workflows/Database-Migration]].

---

## 1. ER Diagram

```
                ┌──────────────┐
                │  auth.users  │ (Supabase managed)
                └──────┬───────┘
                       │ 1:1
              ┌────────▼─────────┐
              │ lungnote_profiles│ (line_user_id, display_name, picture)
              └──────────────────┘

                       │ 1:N         (every entity carries user_id)
       ┌───────────────┼────────────────┬──────────────┐
       ▼               ▼                ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌────────────┐ ┌────────────┐
│lungnote_     │ │lungnote_     │ │lungnote_   │ │lungnote_   │
│  folders     │ │  notebooks   │ │  notes     │ │  tags      │
│ (parent_     │ │ (folder_id   │ │ (notebook_ │ │ (color,    │
│  folder_id)  │◄┤  nullable)   │◄┤  id null)  │ │  unique    │
│  self-FK     │ └──────────────┘ └─────┬──────┘ │  per user) │
└──────────────┘                        │        └─────┬──────┘
                                        │ 1:N          │
                                  ┌─────▼──────┐       │
                                  │lungnote_   │       │
                                  │  todos     │       │
                                  │ (note_id,  │       │
                                  │  position) │       │
                                  └────────────┘       │
                                        │              │
                                        │ M:N          │
                                  ┌─────▼──────────────▼─┐
                                  │ lungnote_notes_tags  │
                                  │ (note_id, tag_id)    │
                                  │ composite PK         │
                                  └──────────────────────┘
```

Cardinality summary:

| From | → | To | Type | Cascade |
|------|---|----|------|---------|
| auth.users | → | lungnote_profiles | 1:1 | cascade |
| auth.users | → | lungnote_folders | 1:N | cascade |
| auth.users | → | lungnote_notebooks | 1:N | cascade |
| auth.users | → | lungnote_notes | 1:N | cascade |
| auth.users | → | lungnote_todos | 1:N | cascade |
| auth.users | → | lungnote_tags | 1:N | cascade |
| lungnote_folders | → | lungnote_folders.parent | 1:N (self) | cascade |
| lungnote_folders | → | lungnote_notebooks.folder_id | 1:N | set null |
| lungnote_notebooks | → | lungnote_notes.notebook_id | 1:N | set null |
| lungnote_notes | → | lungnote_todos.note_id | 1:N | cascade |
| lungnote_notes ↔ lungnote_tags | via | lungnote_notes_tags | M:N | cascade both sides |

---

## 2. ใช้ Relationship ให้คุ้ม — Access Patterns

### 2.1 List notes ของ user (Dashboard home)

ใช้บ่อยสุด. Index `lungnote_notes(user_id, updated_at desc)` รองรับ.

```sql
select id, title, body, notebook_id, updated_at
from lungnote_notes
order by updated_at desc
limit 50;
```

(RLS auto-scope ด้วย `auth.uid() = user_id` — ไม่ต้องใส่ where)

```ts
const { data } = await supabase
  .from("lungnote_notes")
  .select("id, title, body, notebook_id, updated_at")
  .order("updated_at", { ascending: false })
  .limit(50);
```

### 2.2 List notes พร้อม notebook + tag ในรอบเดียว (PostgREST embed)

```ts
const { data } = await supabase
  .from("lungnote_notes")
  .select(`
    id, title, body, updated_at,
    notebook:lungnote_notebooks(id, name, cover_color),
    tags:lungnote_notes_tags(
      tag:lungnote_tags(id, name, color)
    )
  `)
  .order("updated_at", { ascending: false });
```

ผลลัพธ์ shape:
```json
[{
  "id": "...",
  "title": "...",
  "notebook": { "id": "...", "name": "...", "cover_color": "..." } | null,
  "tags": [
    { "tag": { "id": "...", "name": "...", "color": "..." } },
    ...
  ]
}]
```

### 2.3 Notes ใน notebook + todo summary (count + done count)

ใช้สำหรับ Dashboard notebook view.

```ts
const { data } = await supabase
  .from("lungnote_notes")
  .select(`
    id, title, body, updated_at,
    todos:lungnote_todos(id, text, done, position)
  `)
  .eq("notebook_id", notebookId)
  .order("updated_at", { ascending: false });

// derive in app:
data.forEach((n) => {
  n.todoCount = n.todos.length;
  n.todoDone = n.todos.filter((t) => t.done).length;
});
```

ทำใน DB ก็ได้แต่ใช้ view (ดู §6).

### 2.4 Folder tree (recursive)

App caps depth ≤ 3. Fetch flat แล้ว build tree ที่ client.

```ts
const { data } = await supabase
  .from("lungnote_folders")
  .select("id, parent_folder_id, name, updated_at")
  .order("name");
```

```ts
function buildTree(rows) {
  const map = new Map(rows.map((r) => [r.id, { ...r, children: [] }]));
  const roots = [];
  map.forEach((node) => {
    if (node.parent_folder_id) {
      map.get(node.parent_folder_id)?.children.push(node);
    } else {
      roots.push(node);
    }
  });
  return roots;
}
```

ห้าม recursive CTE ใน hot path — flat fetch + client build เร็วกว่าและ RLS-friendly.

### 2.5 Notebooks ใน folder (รวม count notes)

```ts
const { data } = await supabase
  .from("lungnote_notebooks")
  .select(`
    id, name, cover_color, updated_at,
    notes:lungnote_notes(count)
  `)
  .eq("folder_id", folderId)
  .order("updated_at", { ascending: false });

// data[i].notes = [{ count: 12 }]
```

### 2.6 Filter notes ตาม tag

ใช้ junction inner join ผ่าน embed:

```ts
const { data } = await supabase
  .from("lungnote_notes_tags")
  .select(`
    note:lungnote_notes!inner(id, title, body, updated_at)
  `)
  .eq("tag_id", tagId);
// flatten:
const notes = data.map((row) => row.note);
```

หรือใช้ `in` ที่ note level:

```ts
const { data: ids } = await supabase
  .from("lungnote_notes_tags")
  .select("note_id")
  .eq("tag_id", tagId);

const { data: notes } = await supabase
  .from("lungnote_notes")
  .select("id, title, body, updated_at")
  .in("id", ids.map((r) => r.note_id))
  .order("updated_at", { ascending: false });
```

### 2.7 Add tag to note (idempotent)

```ts
await supabase
  .from("lungnote_notes_tags")
  .upsert({ note_id, tag_id }, { onConflict: "note_id,tag_id" });
```

(Composite PK ป้องกัน duplicate auto)

### 2.8 Reorder todos (drag-drop)

Approach 1 — naive (rewrite all): O(n) writes ต่อ reorder

```ts
const updates = reorderedTodos.map((t, i) => ({
  id: t.id,
  position: i,
}));
await supabase.from("lungnote_todos").upsert(updates);
```

Approach 2 — fractional indexing (recommend ภายหลัง): O(1) write ต่อ move

ใช้ float `position` (`5.0`, `10.0`, `15.0`...). ย้าย item ระหว่าง 5-10 → ใส่ `7.5`. Rebalance เมื่อระยะห่าง < epsilon.

ตอนนี้ schema ใช้ integer position — เริ่ม approach 1 ก่อน, สลับ float ภายหลังถ้าคนใช้เยอะ (`alter column position type numeric`).

### 2.9 Toggle todo

```ts
await supabase
  .from("lungnote_todos")
  .update({ done: !current })
  .eq("id", todoId);
```

(RLS check user_id auto)

### 2.10 Delete cascade safe

| Action | Cascade ทำอะไร |
|---|---|
| Delete folder | child folders → child notebooks → child notes → child todos. Notes' notebook_id set null ถ้าเป็น orphan (ไม่อยู่ใน chain) |
| Delete notebook | child notes → child todos (notebook_id ของ note โดน set null ภายใน notebook ที่ลบ — แต่ note ใน notebook นั้นโดนลบ cascade) |
| Delete note | child todos + junction rows |
| Delete tag | junction rows. Notes ไม่กระทบ |
| Delete user | ทุก row ของ user (cascade ทั้งต้นไม้) |

UX implication: ก่อน destructive delete แสดง summary count ที่จะหาย:

```ts
// folder delete preview
const { data: { count: nbCount } } = await supabase
  .from("lungnote_notebooks")
  .select("*", { count: "exact", head: true })
  .eq("folder_id", folderId);
```

---

## 3. RLS Patterns

### 3.1 Owner-scoped (ตารางหลัก)

```sql
create policy <table>_select_own on <table> for select using (auth.uid() = user_id);
create policy <table>_insert_own on <table> for insert with check (auth.uid() = user_id);
create policy <table>_update_own on <table> for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy <table>_delete_own on <table> for delete using (auth.uid() = user_id);
```

### 3.2 Junction table (ตรวจสองด้าน)

```sql
create policy lungnote_notes_tags_all_own on lungnote_notes_tags for all
  using (
    exists (select 1 from lungnote_notes where id = note_id and user_id = auth.uid())
    and exists (select 1 from lungnote_tags where id = tag_id and user_id = auth.uid())
  )
  with check ( /* same */ );
```

ทำไม subquery — junction ไม่มี user_id ตรง. Subquery บังคับว่า user เป็นเจ้าของทั้ง note + tag → กัน user A เอา note ของตัวเองไป tag ด้วย tag ของ user B.

### 3.3 Server-only (no policy)

```sql
alter table <table> enable row level security;
-- ไม่มี policy = ห้าม anon/authenticated เข้า. ใช้ผ่าน service_role เท่านั้น.
```

Tables ที่ใช้ pattern นี้:

| Table | Purpose | ADR |
|-------|---------|-----|
| `lungnote_auth_link_tokens` | One-time auth tokens (mint + redeem server-side) | [[../40-Decisions/0008-line-only-auth-account-linking\|ADR-0008]] |
| `lungnote_conversation_memory` | Rolling 5+5 chat history per LINE userId | [[../40-Decisions/0012-unified-todo-memory-model\|ADR-0012]] / [[../40-Decisions/0016-ai-agent-v2\|ADR-0016]] |
| `lungnote_chat_traces` | Per-turn observability rows; read via admin viewer | [[../40-Decisions/0014-observability-chat-traces\|ADR-0014]] |
| `lungnote_user_memory` | Persistent JSONB facts per LINE userId | [[../40-Decisions/0018-user-memory\|ADR-0018]] |
| `lungnote_agent_settings` | Singleton row — hot-swap system prompt | [[../40-Decisions/0017-claudeflow-patterns\|ADR-0017]] |

Access ผ่าน `createAdminClient()` (`src/lib/supabase/admin.ts`) ที่ใช้ `SUPABASE_SECRET_KEY`. Client code (browser) ห้ามเข้าถึง.

### 3.4 Read-only public view (ไม่ได้ใช้ตอนนี้, แต่ pattern)

```sql
create view lungnote_public_<x> as select ... from lungnote_<x> where is_public = true;
grant select on lungnote_public_<x> to anon;
```

เปลี่ยน RLS แทน + add `is_public` column → policy `select using (is_public = true or auth.uid() = user_id)`.

---

## 4. Performance Hints

### 4.1 Index ทุก FK

ทำแล้วใน migration. Postgres ไม่ index FK auto.

### 4.2 Composite index สำหรับ list-by-user-by-date

```sql
create index lungnote_notes_user_id_updated_at_idx on lungnote_notes(user_id, updated_at desc);
```

→ Dashboard query (§2.1) ใช้ index นี้ตรง.

### 4.3 PostgREST embed = JOIN จริง, ไม่ใช่ N+1

`lungnote_notes?select=*,notebook:lungnote_notebooks(*)` แปลงเป็น single SQL JOIN. ไม่ต้องห่วง N+1.

### 4.4 Count ที่ถูก

```ts
// ❌ ผิด — ดึง row มาทั้งหมด
const { data } = await supabase.from("...").select("*");
const count = data.length;

// ✅ ถูก — ใช้ HEAD + count exact
const { count } = await supabase
  .from("...")
  .select("*", { count: "exact", head: true });
```

### 4.5 Pagination — keyset > offset

```ts
// ❌ offset ช้าเมื่อหน้าลึก
.range(1000, 1050)

// ✅ keyset (cursor by updated_at + id)
.lt("updated_at", lastSeenUpdatedAt)
.order("updated_at", { ascending: false })
.limit(50)
```

---

## 5. Common Anti-Patterns

| ❌ Don't | ✅ Do |
|---|---|
| `select *` | ระบุ column |
| Loop query ใน loop (N+1) | Embed (`select=...,rel(*)`) |
| ดึง row แล้ว count ฝั่ง app | `count: 'exact', head: true` |
| Recursive CTE สำหรับ folder tree (RLS แล้วช้า) | Flat fetch + client tree |
| ใช้ service_role ใน client | RLS-aware client (`createClient` from `lib/supabase/client.ts`) |
| Trigger ลบ cascade ลึก ๆ ผ่าน app code | DB cascade (เร็วกว่า + atomic) |
| Insert duplicate junction row | Upsert with composite onConflict |
| Update position ทีละ row → 100 round trips | Single upsert array |
| Foreign key ที่ไม่ index | Always index FK |

---

## 6. Future: Helper Views (เมื่อ query ซ้ำ)

ถ้า query ซ้ำเยอะใน app, สร้าง view + RLS:

```sql
create view lungnote_notes_with_counts as
select
  n.id, n.user_id, n.title, n.body, n.notebook_id, n.updated_at,
  (select count(*) from lungnote_todos where note_id = n.id) as todo_count,
  (select count(*) from lungnote_todos where note_id = n.id and done = true) as todo_done_count,
  (select count(*) from lungnote_notes_tags where note_id = n.id) as tag_count
from lungnote_notes n;

-- RLS inherits via SECURITY INVOKER on the view
alter view lungnote_notes_with_counts set (security_invoker = true);
```

ทำเมื่อ Dashboard เห็น todo count บ่อย. ตอนนี้ยังไม่ต้อง.

---

## 7. Migration Discipline (recap)

- 1 migration = 1 commit ใน feat branch + PR
- Test local (`supabase db reset`) ก่อน push
- ห้ามแก้ migration ที่ apply แล้ว — สร้าง new migration revert
- ดู [[../50-Workflows/Database-Migration]] ฉบับเต็ม

---

## See Also

- [[../40-Decisions/0006-supabase-db-auth]]
- [[../40-Decisions/0009-lungnote-hierarchy-schema]]
- [[../20-Conventions/Database-Naming]]
- [[../50-Workflows/Database-Migration]]
- [[Glossary]]
- Supabase PostgREST embedding: https://supabase.com/docs/guides/api/joins-and-nesting
