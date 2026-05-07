---
title: "ADR-0002: Split repos — webapp / wikis / design"
tags: [adr, structure]
status: Accepted
date: 2026-05-07
---

# ADR-0002 — Split repos: webapp / wikis / design

## Status

Accepted (revised 2026-05-07 — reversed earlier monorepo decision before first push)

## Context

ตอนแรกตัดสินใจเป็น monorepo (`PASAKON/LungNote`) เพื่อ atomic commit ของ code + doc. แต่ก่อน push พบว่ามี requirement จริง:

- `webapp/` deploy ขึ้น Vercel — Vercel ทำงานง่ายสุดเมื่อ root = repo root
- `wikis/` เป็น Obsidian vault — clone แยก, ไม่ต้องดึง webapp มาด้วยถ้าจะ index/search
- `design/` เป็น HTML mockup — design rev ไม่ควร trigger redeploy webapp
- ไม่มี CI cross-folder ที่ต้องการ atomic commit จริง

## Decision

แยก 3 repo บน GitHub (owner: `PASAKON`):

| Repo | Content | Visibility |
|------|---------|-----------|
| `PASAKON/LungNote-webapp` | Next.js app (root = `webapp/` content) | private (TBD) |
| `PASAKON/LungNote-wikis` | Obsidian vault (root = `wikis/` content) | private (TBD) |
| `PASAKON/LungNote-design` | HTML design mockups (root = `design/` content) | private (TBD) |

Local working layout ยังเป็น 1 directory tree:

```
LungNote Projects/
├── webapp/   # ↔ PASAKON/LungNote-webapp
├── wikis/    # ↔ PASAKON/LungNote-wikis
├── design/   # ↔ PASAKON/LungNote-design
├── CLAUDE.md
└── README.md
```

แต่ละ subfolder = แยก git remote (ของตัวเอง). Root-level `CLAUDE.md` + `README.md` เป็น dev-only context (mirror ใน `webapp/` repo เป็น primary).

LLM/dev workflow: เปิด `LungNote Projects/` parent dir, ทำงานข้าม subfolder ได้, แต่ commit แยกต่อ repo.

## Consequences

**Positive**
- Vercel deploy ตรง webapp repo, ไม่ต้องตั้ง root subdir
- Update wiki ไม่ trigger Vercel redeploy
- Design/wiki contributors ไม่ต้อง clone โค้ด
- แต่ละ repo มี changelog/issue board ของตัวเอง

**Negative**
- เสีย atomic "code + doc" commit — ต้อง discipline เปิด PR คู่กันเอง
- LLM ต้อง 2 step: read wiki จาก local sibling dir, แต่ link refer คน clone เดี่ยวอาจหาไม่เจอ
- 3 repo = 3 README + 3 settings ต้องดูแล

**Mitigation**
- Root-level `CLAUDE.md` + `README.md` (อยู่ใน webapp repo เป็น primary) อธิบาย sibling layout
- ทุก ADR commit ขึ้น `LungNote-wikis` ภายใน 1 working day หลัง code merge ที่กระทบ
- ใช้ wikilinks `[[...]]` ใน vault ปกติ; cross-repo link ใช้ GitHub URL

## Alternatives

- **Monorepo (`PASAKON/LungNote`)** — ตัดสินใจเดิม. ดีตรง atomic commit. แพ้ตรง Vercel rootDir + redeploy noise.
- **Wiki แยก, code+design รวม** — design ไม่ค่อย change บ่อย, ไม่จำเป็น
- **3 repo + meta repo (submodule)** — submodule overhead สูงเกิน, LLM ใช้ยาก

## See Also

- [[0005-deploy-vercel]] — webapp deploy รับประโยชน์จาก split
- [[../50-Workflows/Dev-Workflow]] — workflow ต้อง update รับ split layout
