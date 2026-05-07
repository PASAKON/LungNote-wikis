---
title: LungNote Wiki — Index
tags: [index, root]
---

# LungNote Wiki

Source of truth สำหรับ context, decisions, conventions ของโปรเจค **LungNote**.

> **กฎเหล็ก:** Dev / LLM ทุกคนต้องอ่าน wiki ก่อน build / create / edit / deploy ใน `webapp/`.
> ดู [[../../CLAUDE|CLAUDE.md root]] เพื่อรายละเอียด.

## Map

- [[../10-Architecture/Overview|Architecture Overview]] — system shape, data flow
- [[../20-Conventions/Code-Style|Code Conventions]] — naming, lint, formatting
- [[../20-Conventions/Commit-Convention|Commit Convention]]
- [[../20-Conventions/Wiki-Style|Wiki Style & Authoring Rules]] — กฎเขียน-แก้-ลบ wiki
- [[../30-Domain/Glossary|Domain Glossary]] — product + business + technical terms
- [[../40-Decisions/README|Decision Log (ADR)]] — บันทึกการตัดสินใจ
- [[../50-Workflows/Dev-Workflow|Dev Workflow]] — branch, PR, review (code)
- [[../50-Workflows/Wiki-Workflow|Wiki Workflow]] — workflow แก้ vault
- [[../50-Workflows/Multi-Repo-Workflow|Multi-Repo Workflow]] — 3-repo setup, .env sharing, conflict prevention
- [[../90-Templates/ADR-Template|ADR Template]]
- [[../90-Templates/Page-Template|General Page Template]]

## Conventions ใน vault นี้

สรุปสั้น (ดูเต็มที่ [[../20-Conventions/Wiki-Style|Wiki-Style]]):

- Folder prefix ตัวเลข = ลำดับอ่าน, อย่าเปลี่ยน
- ใช้ `[[wikilink|alias]]` ไม่ใช่ markdown link ภายใน vault
- Cross-repo (ไป webapp/design) → ใช้ GitHub URL เต็ม
- ทุกหน้ามี frontmatter: `title`, `tags`
- File name = ASCII kebab/Title-Case (ห้ามไทย / space / underscore)
- ภาษาไทย/อังกฤษผสมได้ — technical term ใช้อังกฤษเสมอ
- Accepted ADR: ห้ามแก้ Decision/Consequences ตรงๆ — supersede ด้วย ADR ใหม่
