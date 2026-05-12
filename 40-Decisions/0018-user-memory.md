---
title: "ADR-0018: Persistent user memory — facts the agent learns"
tags: [adr, ai, agent, memory, personalization]
status: Accepted
date: 2026-05-12
---

# ADR-0018 — Persistent user memory

## Status

Accepted (backfilled — feature shipped alongside ClaudeFlow bundle in [LungNote-webapp#41](https://github.com/PASAKON/LungNote-webapp/pull/41), migration `20260509200000_user_memory.sql`)

## Context

[[0016-ai-agent-v2|ADR-0016]]'s `TurnContext` resets every turn. Conversation memory (rolling 5+5 user/assistant pairs in `lungnote_conversation_memory`) keeps short-term coherence. Neither persists long-term facts about a user.

Examples that should survive across turns indefinitely:

- "ฉันชื่อ ภาคิน" → bot should call them by name later
- "ฉันเรียนคณะวิศวกรรมศาสตร์ มหาวิทยาลัย X" → context for future schedule reminders
- "วิชาที่ฉันเรียน: ฟิสิกส์, แคลคูลัส, เคมี" → grow-able list

Conversation memory is the wrong shape (capped at 10 turns; old facts age out). Putting it into todos pollutes the todo list. Need a separate, structured KV per user.

## Decision

New table `lungnote_user_memory`:

```sql
create table lungnote_user_memory (
  line_user_id text primary key,
  memory       jsonb not null default '{}',
  updated_at   timestamptz default now()
);
-- RLS on, no policies — server-only via service-role admin client.
```

### Agent integration

- **Load at turn start** — `runAgent` fetches `loadUserMemory(lineUserId)` in parallel with conversation memory; injects as **dynamic** system prompt suffix:
  ```
  # USER MEMORY (persistent facts about this user — use freely; never expose JSON)
  {"name":"ภาคิน","subjects":["ฟิสิกส์","แคลคูลัส"]}
  ```
  This block is in the non-cached portion ([[0017-claudeflow-patterns|ADR-0017]]) — keeps cache key stable across users.
- **Update via `update_memory` tool** — `update_memory(action: "set"|"delete", key, value)`. Set-with-array merges union (so subject lists grow incrementally); set-with-scalar overwrites. Delete removes the key.
- **Prompt instruction** — agent told to update memory **freely on its own judgment**; never ask permission. Examples in prompt show common cases (name capture, role, ongoing list).

### Why JSONB single column (not separate columns / EAV)

- Keys evolve as agent learns what's worth remembering — no migration treadmill
- Postgres jsonb has full index/query support if we ever need to filter on a known key
- Easy to dump + inspect via admin viewer

### Privacy

- Server-only (no RLS policies = no end-user read access)
- Agent system prompt instructs: never expose raw JSON to user; reference facts in natural language
- User can request deletion via chat ("ลบข้อมูลของฉัน") — agent runs `update_memory("delete", ...)` for each key (TBD — see TODO)

## Consequences

**Positive**
- One row per user = trivial cleanup on user delete (cascade not wired; line_user_id is the PK, not FK)
- Agent can personalize over time without engineering work per fact type
- Lives outside the todo list — no UI pollution
- Inspectable via admin viewer

**Negative**
- JSONB schemaless = agent can write anything; no validation. A buggy prompt could fill memory with junk
- "Update freely" prompt instruction relies on Claude's judgment — Gemini path may behave differently
- No automatic forget mechanism — facts pile up forever unless explicitly deleted
- Personally identifiable info goes in unencrypted (mitigated by server-only access)

**Mitigation**
- Memory size: cap at ~10KB per user via app-layer check (TODO)
- Add admin viewer page to inspect + edit individual user memory (TODO)
- Document "what NOT to remember" in the system prompt (passwords, full addresses, etc.)
- `line_user_id` is the only PII; no email, no synthetic email reference

## Alternatives ที่ไม่เลือก

- **Stuff into `lungnote_profiles`** — that's identity, not learned facts; mixing them complicates RLS
- **Vector DB embeddings** — overkill for KV-shaped facts; query cost not justified
- **Per-fact rows (EAV)** — every set/get becomes N round trips; query patterns don't need it
- **Append-only event log** — clean but adds compaction work; jsonb upsert is simpler

## Open questions / TODO

- [ ] User-facing "view / clear my memory" command in chat
- [ ] Admin viewer page: `/admin/users/[id]/memory`
- [ ] Memory size cap + LRU eviction policy
- [ ] Cascade-on-user-delete: `line_user_id` is text not FK to auth.users — orphan rows if user is deleted via Supabase admin

## See Also

- [[0016-ai-agent-v2]] — runtime + tool framework
- [[0017-claudeflow-patterns]] — dynamic prompt block construction
- [[0012-unified-todo-memory-model]] — disambiguation: this is NOT the "memory item" (todo) concept; this is **persistent USER facts**
- [[0015-admin-debug-viewer]] — future surface for inspecting user memory
- Code: [`src/lib/agent/user_memory.ts`](https://github.com/PASAKON/LungNote-webapp/blob/main/src/lib/agent/user_memory.ts), [`src/lib/agent/tools/profile/update_memory.ts`](https://github.com/PASAKON/LungNote-webapp/blob/main/src/lib/agent/tools/profile/update_memory.ts)
