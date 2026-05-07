---
title: Dev Workflow
tags: [workflow]
---

# Dev Workflow

## ก่อนแตะโค้ดใดๆ

1. อ่าน [[../../CLAUDE|CLAUDE.md root]]
2. อ่าน [[../00-Index/README|wiki index]]
3. ถ้าแก้เรื่อง architecture → อ่าน [[../10-Architecture/Overview]] + ADR ที่เกี่ยว
4. ถ้าแก้ใหม่ที่ส่งผลต่อ architecture → เพิ่ม ADR ใหม่

## Branch

- `main` — protected, deployable
- `feat/<scope>-<short>` — feature
- `fix/<scope>-<short>` — bug
- `wiki/<topic>` — wiki only

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
