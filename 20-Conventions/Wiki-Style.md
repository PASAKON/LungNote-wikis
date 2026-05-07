---
title: Wiki Style & Authoring Rules
tags: [conventions, wiki, meta]
---

# Wiki Style & Authoring Rules

กฎเขียน-แก้-ลบ ใน `wikis/` (Obsidian vault, repo `PASAKON/LungNote-wikis`).
ใช้ร่วมกันทั้งทีม + LLM agent.

---

## 1. Folder Structure

| Folder | Purpose | Allowed file types |
|--------|---------|--------------------|
| `00-Index/` | Map / TOC ของ vault | `README.md` เท่านั้น |
| `10-Architecture/` | System shape, data flow, boundary | Overview + sub-pages |
| `20-Conventions/` | Code style, commit, wiki style, naming | Convention pages |
| `30-Domain/` | Glossary + domain model docs | Glossary + per-bounded-context |
| `40-Decisions/` | ADR (Architecture Decision Records) | `NNNN-<slug>.md` + `README.md` index |
| `50-Workflows/` | Dev / wiki / release / on-call workflows | Workflow pages |
| `90-Templates/` | Templates สำหรับสร้าง page ใหม่ | `*-Template.md` |

**กฎ:**
- เลข prefix = ลำดับอ่าน, อย่าเปลี่ยน
- เพิ่ม top-level folder ใหม่ → ต้อง propose ADR ก่อน (เปลี่ยน vault structure)
- เพิ่ม sub-folder ภายใน existing top-level → ทำได้เลย แต่ update `00-Index/README` ด้วย

---

## 2. Frontmatter (mandatory)

ทุกหน้าต้องมี frontmatter YAML:

```markdown
---
title: <Page Title — แสดงใน Obsidian/GitHub>
tags: [<comma-separated>]
---
```

ADR เพิ่ม `status` + `date`:

```markdown
---
title: "ADR-NNNN: <title>"
tags: [adr, <topic>]
status: Proposed | Accepted | Deprecated | Superseded by NNNN
date: YYYY-MM-DD
---
```

**กฎ:**
- `title` = exact human-readable, ใช้ใน graph view + tab title
- `tags` ≥ 1 tag เสมอ; ใช้ kebab-case; reuse tag เดิมถ้ามี
- ห้ามใส่ `aliases` ถ้าไม่จำเป็น (Obsidian feature, ทำให้ refer สับสน)

---

## 3. Naming

| Type | Convention | Example |
|------|------------|---------|
| Folder | `NN-Kebab-Case/` | `40-Decisions/` |
| ADR file | `NNNN-<short-slug>.md` (4-digit, lowercase) | `0005-deploy-vercel.md` |
| Convention/Workflow page | `Title-Case.md` | `Code-Style.md`, `Wiki-Style.md` |
| Domain page | `Title-Case.md` หรือ `Glossary.md` | `Glossary.md` |
| Template | `<Name>-Template.md` | `ADR-Template.md` |
| README (folder index) | `README.md` | `00-Index/README.md` |

**กฎ:**
- File slug ต้อง URL-safe (ASCII, `-` separator)
- ห้ามใช้ space, ห้ามใช้ `_`, ห้ามใช้ Thai ในชื่อไฟล์
- Title ใน frontmatter เป็นอะไรก็ได้ (ไทย/อังกฤษ); ชื่อไฟล์ต้องอังกฤษเสมอ

---

## 4. Links

ใน vault → ใช้ **wikilinks** Obsidian style:

```markdown
ดู [[../40-Decisions/0005-deploy-vercel|ADR-0005]] เพื่อรายละเอียด.
```

**กฎ:**
- `[[ ]]` สำหรับ link ภายใน vault
- ใส่ `|alias` เสมอเมื่ออยากให้ display text ต่างจาก path
- ห้ามใช้ markdown link `[text](path.md)` ภายใน vault (graph view ไม่จับ)
- Cross-repo link (ไป webapp/design) → ใช้ GitHub URL เต็ม:
  ```
  ดูโค้ด [src/proxy.ts](https://github.com/PASAKON/LungNote-webapp/blob/main/src/proxy.ts)
  ```
- Anchor: `[[Page#Heading|alias]]`

---

## 5. ภาษา

- ผสมไทย-อังกฤษได้
- Technical term ใช้อังกฤษเสมอ (`server component`, `RLS`, `middleware`)
- ชื่อทางการ business ใช้ตามสะดวก (จดเป็นไทยได้ ถ้า user-facing)
- ห้ามใช้ภาษา 3 (เช่น ญี่ปุ่น) ถ้าไม่ใช่ quote

---

## 6. Page Lifecycle

### เพิ่ม page ใหม่

1. เลือก folder ตามหัวข้อ (ตาม section 1)
2. copy template จาก `90-Templates/` (ถ้ามี) หรือใช้ `Page-Template.md`
3. ตั้งชื่อ file ตาม section 3
4. กรอก frontmatter (section 2)
5. เพิ่ม link ใน folder index (`README.md`) หรือ `00-Index/README.md` ถ้า cross-folder

### แก้ page เดิม

- **Behavior change** (เปลี่ยน fact / convention) → commit message ต้องบอก why
- **Typo / format / link fix** → commit `style(wiki): ...` หรือ `docs(wiki): typo in ...`
- ถ้า page เป็น ADR ที่ Accepted แล้ว → **ห้ามแก้ Decision/Consequences ตรงๆ** ดู section 7

### ลบ page

- ห้ามลบ ADR (history matter) — mark `status: Deprecated` แทน
- หน้าอื่น: ลบได้ ถ้าไม่มี link refer (Obsidian ตรวจได้: Settings → Backlinks)
- Commit message: `docs(wiki): remove <page> — superseded by <new>`

---

## 7. ADR Rules

ADR = บันทึก architectural decision ใน `40-Decisions/`.

**เมื่อต้องเขียน ADR:**
- เพิ่ม dependency ที่กระทบ architecture (auth, db, state mgmt, deploy host)
- เปลี่ยน folder structure / repo layout
- เลือก protocol/format/standard ที่ผูก yourself ระยะยาว
- Trade-off ที่ดูแล้วต้องอธิบาย "ทำไมไม่อีกตัว"

**ขั้นตอน:**
1. คัด `90-Templates/ADR-Template.md` → `40-Decisions/NNNN-<slug>.md`
2. NNNN = next number (ดู `40-Decisions/README.md` ล่าสุด)
3. กรอก: Status `Proposed`, Context, Decision, Consequences, Alternatives
4. Update `40-Decisions/README.md` index — เพิ่ม row
5. Open PR: ถ้าทีมเห็นด้วย → merge + flip status เป็น `Accepted`
6. ถ้า ADR เก่าโดน supersede:
   - ADR เก่า: `status: Superseded by NNNN`, ใส่ link ไป ADR ใหม่
   - ADR ใหม่: ใน Context อ้าง ADR เก่า + อธิบายเหตุที่เปลี่ยน
   - ห้ามลบไฟล์ ADR เก่า

**ห้ามแก้ Accepted ADR เนื้อใน** ยกเว้น:
- Typo / รูปแบบ
- เพิ่ม See Also link
- Mark `Superseded by` (ไม่ใช่แก้ Decision เดิม)

ถ้าต้องเปลี่ยนแนวคิด → เขียน ADR ใหม่ supersede.

---

## 8. Glossary

`30-Domain/Glossary.md` = สิ่งเดียวที่รวม term.

**กฎ:**
- เจอ term ใหม่ใน code/PR/discussion → เพิ่มทันที
- ถ้าไม่ชัวร์ความหมาย → ใส่ `(?)` ท้าย, เปิด PR ถามคนตรวจ
- แยก section: Product / Business / Technical Alias
- 1 term = 1 row, definition ≤ 2 บรรทัด, link ไป page ที่ deep-dive ถ้ามี

---

## 9. Commit Message สำหรับ wiki

ตาม [[Commit-Convention]]:

```
wiki(<scope>): <subject>
```

| Scope | ใช้เมื่อ |
|-------|--------|
| `architecture` | แก้ใน `10-Architecture/` |
| `convention` | แก้ใน `20-Conventions/` |
| `domain` | แก้ใน `30-Domain/` |
| `adr` | เพิ่ม/แก้ ADR ใน `40-Decisions/` |
| `workflow` | แก้ใน `50-Workflows/` |
| `template` | แก้ใน `90-Templates/` |
| `index` | แก้ index/map |

Examples:

```
wiki(adr): add 0006 — choose Supabase for auth + db
wiki(domain): add Notebook, Note, Folder to glossary
wiki(convention): clarify wikilink alias rule
docs(wiki): typo in Wiki-Style section 3
```

---

## 10. Cross-Repo Discipline

Wiki + webapp + design = 3 repo แยก ([[../40-Decisions/0002-split-repos-webapp-wikis-design|ADR-0002]]).

**กฎ:**
- PR ที่เปลี่ยน webapp behavior กระทบ doc → เปิด PR คู่ที่ wiki repo, link cross-repo (mention URL ใน PR description)
- Wiki update ภายใน 1 working day หลัง webapp merge
- ถ้า wiki update ไม่ทัน → ใส่ `// TODO: wiki` ใน code + GitHub issue ที่ wiki repo

---

## 11. Review Checklist (สำหรับ wiki PR)

- [ ] Frontmatter ครบ (`title`, `tags`)
- [ ] File name ตาม convention (section 3)
- [ ] Link เป็น wikilink (ภายใน vault) หรือ GitHub URL (cross-repo)
- [ ] ถ้าเพิ่ม page ใหม่ → update folder README + 00-Index ถ้าจำเป็น
- [ ] ถ้าแก้ glossary → ตรวจว่าไม่มี duplicate term
- [ ] ถ้าเพิ่ม ADR → update `40-Decisions/README.md` index
- [ ] ภาษาถูก (ไทย/อังกฤษ ตาม section 5)
- [ ] Commit message ตาม section 9

---

## See Also

- [[Code-Style]]
- [[Commit-Convention]]
- [[../50-Workflows/Dev-Workflow]]
- [[../50-Workflows/Wiki-Workflow]]
- [[../90-Templates/ADR-Template]]
- [[../90-Templates/Page-Template]]
