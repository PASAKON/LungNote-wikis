---
title: "ADR-0002: Monorepo — webapp/ + wikis/"
tags: [adr, structure]
status: Accepted
date: 2026-05-07
---

# ADR-0002 — Monorepo: webapp/ + wikis/

## Status

Accepted

## Context

ต้องการให้ context (รวมถึง LLM context) มาก่อน code. Dev/LLM คนใหม่ควรอ่าน "หลักการ" ก่อนแก้ไฟล์.

## Decision

โครง root ระดับเดียว, 2 folder หลัก:

```
LungNote Projects/
├── webapp/   # code (Next.js)
├── wikis/    # docs (Obsidian vault)
└── CLAUDE.md # บังคับให้อ่าน wikis ก่อนแก้
```

- Single git repo (`PASAKON/LungNote`)
- `wikis/` เป็น Obsidian vault → wikilinks `[[ ]]` + graph view
- `CLAUDE.md` เป็น entry point สำหรับ LLM agent

## Consequences

**Positive**
- 1 repo → atomic commit (code + doc พร้อมกัน)
- Obsidian wikilinks → navigate ง่าย, graph view ช่วย discover
- LLM อ่าน CLAUDE.md ก่อน → context ครบโดยไม่ต้องสั่ง

**Negative**
- `wikis/` ไม่ deploy แต่ติดมาใน clone — เพิ่มขนาด repo (เล็กน้อย)
- ต้อง maintain wiki ให้ตรงโค้ด — ใช้ ADR ช่วย

## Alternatives

- Wiki แยก repo → แยกแต่ sync ยาก
- GitHub Wiki built-in → ไม่ support wikilinks Obsidian-style
