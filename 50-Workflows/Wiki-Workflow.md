---
title: Wiki Workflow
tags: [workflow, wiki]
---

# Wiki Workflow

ขั้นตอนปฏิบัติเวลาทำงานกับ vault. กฎเนื้อหาดู [[../20-Conventions/Wiki-Style]].

## Local setup

1. Clone wiki repo:
   ```bash
   git clone https://github.com/PASAKON/LungNote-wikis.git wikis
   ```
2. เปิดด้วย Obsidian → Open folder as vault → เลือก `wikis/`
3. Plugins ที่แนะนำ (ทำเอง):
   - **Backlinks** (built-in) → เปิด
   - **Outline** (built-in) → เปิด
   - **Templater** (community) → ผูก `90-Templates/` เป็น templates folder

## เริ่ม page ใหม่

```bash
cp wikis/90-Templates/Page-Template.md wikis/<NN-folder>/<Slug>.md
# หรือ ADR
cp wikis/90-Templates/ADR-Template.md wikis/40-Decisions/NNNN-<slug>.md
```

ในไฟล์ใหม่:
1. แก้ frontmatter (`title`, `tags`)
2. เขียนเนื้อหา
3. update folder index (ถ้ามี) + `00-Index/README.md` ถ้า cross-folder

## Workflow ตาม scenario

### Scenario A: เปลี่ยน architecture (auth, db, deploy, ฯลฯ)

1. เขียน ADR ใหม่ใน `40-Decisions/` (`status: Proposed`)
2. open PR ที่ wiki repo → ทีม review
3. merge → flip `status: Accepted`
4. ถ้ากระทบโค้ด → open PR ที่ webapp repo, link ADR ใน PR description
5. update `10-Architecture/Overview.md` ถ้า system shape เปลี่ยน

### Scenario B: เพิ่ม feature ที่มี term ใหม่

1. update `30-Domain/Glossary.md` (เพิ่ม term + definition)
2. ถ้า term มี deep behavior → สร้าง page ใน `30-Domain/` (เช่น `Notebook-Model.md`)
3. open PR คู่ webapp + wiki

### Scenario C: เปลี่ยน convention (code style, naming, etc.)

1. update `20-Conventions/<page>.md`
2. commit message: `wiki(convention): <what changed>`
3. ถ้ามี code เก่าผิด convention ใหม่ → migrate ใน webapp (PR แยก)

### Scenario D: Wiki ผิด แต่ code ถูก

1. แก้ wiki ให้ตรง code
2. commit: `docs(wiki): align <page> with current implementation`

### Scenario E: Code เปลี่ยนโดย wiki ตามไม่ทัน

1. ใน webapp PR ใส่ `// TODO: wiki` + open issue ที่ wiki repo
2. ภายใน 1 working day → merge wiki update + ปิด issue

## Cross-repo PR linking

ใน PR description ที่ webapp repo:

```markdown
## Wiki updates
- [LungNote-wikis#42](https://github.com/PASAKON/LungNote-wikis/pull/42) — ADR-0007 + Glossary
```

ใน PR description ที่ wiki repo:

```markdown
## Related code
- [LungNote-webapp#15](https://github.com/PASAKON/LungNote-webapp/pull/15) — Supabase Auth integration
```

## Review checklist

ดู [[../20-Conventions/Wiki-Style#11-Review-Checklist-สำหรับ-wiki-PR|Wiki-Style §11]].

## Common mistakes

- ลืม update `40-Decisions/README.md` หลังเพิ่ม ADR
- แก้ Accepted ADR ตรงๆ แทนที่จะ supersede
- ใช้ markdown link `[x](y.md)` แทน wikilink `[[y]]`
- ตั้งชื่อไฟล์ภาษาไทย / มี space
- ลืม update glossary หลังเจอ term ใหม่
- Commit รวม wiki + code ไว้ด้วยกัน (ตอนนี้ split repo แล้ว — แยก commit แยก repo)

## See Also

- [[../20-Conventions/Wiki-Style]]
- [[Dev-Workflow]]
- [[../20-Conventions/Commit-Convention]]
