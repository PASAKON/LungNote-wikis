---
title: "ADR-0012: Unified note + todo memory model with date awareness"
tags: [adr, schema, todo, memory, ai]
status: Accepted
date: 2026-05-08
---

# ADR-0012 — Unified note + todo memory model

## Status

Accepted

## Context

Earlier schema ([[0009-lungnote-hierarchy-schema|ADR-0009]]) treated `lungnote_notes` and `lungnote_todos` as separate concepts:
- **Notes** — long-form text. Captured via "จด" / "บันทึก" prefix in LINE chat.
- **Todos** — checkbox items. Created only via the `/dashboard/todo` UI.

In practice — and per the product owner's directive — users treat both as a single thing: **เตือนความจำ** (memory aid). A "note" like "พรุ่งนี้ส่งการบ้านฟิสิกส์ครูไพสินทร์" is functionally a todo with a due date. Splitting the model:
- forces two prefixes ("จด" vs "todo") that users won't remember
- hides chat-captured items in the notes list when the user expected them in /todo
- means due dates can't be attached to "notes" at all

We also want the AI to extract dates from natural language ("พรุ่งนี้", "วันพุธหน้า", "อาทิตย์หน้า", "ในอีก 3 วัน") at capture time, not via a follow-up turn.

## Decision

**`lungnote_todos` is the canonical memory item.** `lungnote_notes` stays as a container (folders for grouping); chat-captured items land in a per-user **"📥 Inbox"** note.

### Schema delta

```sql
alter table lungnote_todos
  add column due_at      timestamptz,           -- when (null = no date)
  add column due_text    text,                  -- raw user phrase ("พรุ่งนี้")
  add column source      text not null default 'web'
    check (source in ('chat','web','liff'));

-- raise text cap so chat-captured longer thoughts fit
alter table lungnote_todos drop constraint lungnote_todos_text_check;
alter table lungnote_todos
  add constraint lungnote_todos_text_check
  check (length(text) between 1 and 2000);

create index lungnote_todos_user_id_due_at_idx
  on lungnote_todos(user_id, due_at) where due_at is not null;
create index lungnote_todos_user_id_source_idx
  on lungnote_todos(user_id, source);
```

Migration: `supabase/migrations/20260508140000_lungnote_todos_due_at.sql`.

### Unified capture prefix

LINE webhook recognizes one regex:
```
^(จด|บันทึก|note|save|todo|ทำ|เตือน)\s+([\s\S]+)$
```
All prefixes route through `lib/memory/save.ts → saveMemoryFromLine` → `lungnote_todos` row with `source='chat'`.

### AI extraction with date awareness

`lib/ai/memory-extract.ts` replaces `lib/ai/note-extract.ts`. The system prompt **injects today's Bangkok date + weekday** so the LLM can resolve relative phrases deterministically:

| User input | Extracted |
|------------|-----------|
| "พรุ่งนี้ส่งการบ้านฟิสิกส์" | `text: "ส่งการบ้านฟิสิกส์"`, `due_at: <tomorrow 09:00 BKK>`, `due_text: "พรุ่งนี้"` |
| "วันพุธหน้าประชุม 3 โมง" | `text: "ประชุม"`, `due_at: <next Wed 15:00>`, `due_text: "วันพุธหน้า 3 โมง"` |
| "ซื้อนม" | `text: "ซื้อนม"`, `due_at: null`, `due_text: null` |

Returns `{ text, due_at, due_text }`. Falls back to `{text: trimmed, due_at: null, due_text: null}` on any LLM failure — capture never blocks on AI.

### UX (`/dashboard/todo`)

- query selects `due_at, due_text`
- sort: items with `due_at` first (soonest), then `created_at` desc
- "วันนี้" tab matches `due_at` = today (or no-due + created today)
- pill next to text: `เลย N วัน` (red) / `วันนี้ HH:MM` (orange) / `พรุ่งนี้` / `อีก N วัน` (mint) / `D MMM` (muted)

LINE reply also surfaces the resolved date: `บันทึกแล้ว ✓ "..."\n⏰ พรุ่งนี้ (ส., 9 พ.ค. 09:00)`

### What stays the same

- `lungnote_notes` table — used only as todo container ("📥 Inbox", future user-named groups). The notes list UI in dashboard is unchanged for now.
- Existing todos created from `/dashboard/todo` Web — `source='web'`, `due_at=null`. No backfill needed.
- RLS, indexes on `(note_id, position)` — untouched.

## Consequences

**Positive**
- One mental model for users: **everything is a memory item**.
- Due-date awareness from day one (no follow-up turn needed).
- Chat capture and Web capture write to the same table — `/dashboard/todo` is the single source of truth.
- AI cost stays bounded — one extraction call per capture, same as before.

**Negative**
- `lungnote_notes` becomes a thin container layer. If we don't add notebook UI within ~1 month, we should reconsider whether `note_id` should become nullable on `lungnote_todos`. Tracked as a follow-up — not blocking.
- `due_at` is a single timestamp. Recurring reminders ("ทุกวันจันทร์") need a separate `recurrence` column or join table — out of scope.
- LLM resolution of relative dates can be wrong (e.g. "อาทิตย์หน้า" ambiguity: next Sunday vs. next week?). We surface `due_text` in the LINE reply and the UI tooltip so the user sees what was understood and can correct it.

## Phase 2 (deferred)

- **Tool-calling AI fallback** — let the AI invoke `save_memory(text, due_at)` from free-form chat without requiring a prefix. Needs OpenRouter tool-call format wired into `lib/ai/reply.ts`. Out of scope for MVP.
- **Daily reminder cron** — Vercel Cron at 08:00 BKK pushes a LINE message of items where `due_at < tomorrow_start`. Requires Pro plan budget approval.

## References

- Builds on [[0009-lungnote-hierarchy-schema|ADR-0009]] (table layout)
- Replaces note-creation prefix logic introduced in champ's first PR (kept the regex shape, swapped the destination)
- See [[../30-Domain/Glossary|Glossary]] for **Memory item** / **Inbox note** / **due_text** terms
