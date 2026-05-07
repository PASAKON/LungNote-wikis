---
title: Dev Workflow
tags: [workflow]
---

# Dev Workflow

## ก่อนแตะโค้ดใดๆ

1. อ่าน [[../../CLAUDE|CLAUDE.md root]]
2. อ่าน [[../00-Index/README|wiki index]]
3. ถ้าแก้เรื่อง architecture → อ่าน [[../10-Architecture/Overview]] + ADR ที่เกี่ยว
4. ถ้าแก้ใหม่ที่ส่งผลต่อ architecture → เพิ่ม ADR ใหม่ (ดู [[../20-Conventions/Wiki-Style#7-ADR-Rules|Wiki-Style §7]] + [[Wiki-Workflow#scenario-a]])
5. ถ้าจะแก้ wiki → อ่าน [[../20-Conventions/Wiki-Style|Wiki-Style]] + [[Wiki-Workflow]] ก่อน

## Branch

- `main` — **protected** (linear history, no force-push, no deletion). ห้าม commit ตรง
- `feat/<scope>-<short>` — feature
- `fix/<scope>-<short>` — bug
- `wiki/<topic>` — wiki only (ใน wiki repo)
- `design/<topic>` — design only (ใน design repo)

ทุก branch → PR → rebase merge เข้า main. ดู [[Multi-Repo-Workflow#3-daily-loop|Multi-Repo §3]].

## ก่อน commit

```bash
cd webapp
pnpm lint
pnpm build      # ตรวจ type + build
```

## ก่อน PR

- [[../20-Conventions/Commit-Convention|commit ตาม convention]]
- ถ้าเพิ่ม decision → เพิ่ม ADR ใน `wikis/40-Decisions/`
- ถ้าเพิ่มศัพท์ใหม่ → เพิ่ม [[../30-Domain/Glossary]]

## Local dev

```bash
cd webapp
pnpm dev
```
