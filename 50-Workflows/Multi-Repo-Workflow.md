---
title: Multi-Repo Workflow
tags: [workflow, multi-repo, multi-dev]
---

# Multi-Repo Workflow

LungNote = 3 GitHub repos แยก ([[../40-Decisions/0002-split-repos-webapp-wikis-design|ADR-0002]]):

| Repo | Local dir | Purpose |
|------|-----------|---------|
| `PASAKON/LungNote-webapp` | `webapp/` | Next.js app (deploys to Vercel) |
| `PASAKON/LungNote-wikis` | `wikis/` | Obsidian vault — docs + ADR |
| `PASAKON/LungNote-design` | `design/` | HTML design mockups |

หน้านี้ = workflow สำหรับ dev คนเดียว/หลายคนทำงานข้าม repo ลื่นไหล + กัน conflict.

---

## 1. Workspace Layout (recommended)

```
~/code/lungnote/             # parent dir (no git, just folder)
├── webapp/                  ← clone PASAKON/LungNote-webapp
├── wikis/                   ← clone PASAKON/LungNote-wikis
└── design/                  ← clone PASAKON/LungNote-design
```

แต่ละ subfolder = git repo แยก. `cd webapp && git status` ไม่เห็นไฟล์ของ wikis/design.

---

## 2. First-time Setup (new dev)

### 2.1 Tools

```bash
# pnpm (package manager)
brew install pnpm   # macOS, หรือ npm i -g pnpm

# Vercel CLI (สำหรับ env sync + deploy)
pnpm add -g vercel

# GitHub CLI (recommended)
brew install gh && gh auth login
```

### 2.2 Get access

ขอ org admin (PASAKON) ให้:
- Add เป็น collaborator ทั้ง 3 repo
- Add เป็น Vercel team member (`passgob1-8454s-projects`) — Member role พอ
- Add เป็น Supabase project member ([[../40-Decisions/0006-supabase-db-auth|ADR-0006]] project)

### 2.3 Bootstrap

ใช้ helper script:

```bash
mkdir -p ~/code/lungnote && cd ~/code/lungnote
git clone https://github.com/PASAKON/LungNote-webapp.git webapp
cd webapp
./scripts/setup-workspace.sh ~/code/lungnote
```

Script จะ:
- Clone อีก 2 repo (wikis + design)
- `pnpm install` ใน webapp
- `vercel link` + `vercel env pull .env.local`

### 2.4 Verify

```bash
cd ~/code/lungnote/webapp
pnpm dev          # http://localhost:3000
pnpm build        # ผ่าน lint + typecheck
./scripts/status-all.sh   # ดู status ของ 3 repo
```

---

## 3. Daily Loop (ทำงานปกติ)

### 3.1 Start of day — pull all

```bash
cd ~/code/lungnote/webapp
./scripts/sync-all.sh
```

(Rebase main ของทั้ง 3 repo. Auto-stash ถ้ามี dirty work.)

### 3.2 Branch (NEVER commit ตรง main)

```bash
cd <repo>
git checkout -b <type>/<scope>-<short>
# เช่น
git checkout -b feat/notebook-create
git checkout -b wiki/glossary-supabase
git checkout -b fix/proxy-locale-fallback
```

ดู [[../20-Conventions/Commit-Convention]] สำหรับ type list.

### 3.3 Commit (Conventional Commits)

```bash
git add <files>
git commit -m "feat(notebook): allow color picker"
```

แก้ doc คู่กัน → commit ที่ wiki repo แยก (ดู section 5).

### 3.4 Push + PR

```bash
git push -u origin HEAD
gh pr create --fill --base main
```

ทุก PR target → `main`. Branch protection บังคับ **linear history** + ห้าม force-push.

### 3.5 Merge

ถ้า PR สด สะอาด → **rebase merge** (or squash). **ห้าม merge commit** (branch protection จะ reject).

```bash
gh pr merge --rebase --delete-branch
```

### 3.6 After merge — pull main back

```bash
git checkout main
git pull --rebase --autostash
```

---

## 4. Conflict Prevention

### 4.1 Repo-level rules (enforced via GitHub branch protection)

ทั้ง 3 repos:
- `required_linear_history: true` — ห้าม merge commit (force rebase)
- `allow_force_pushes: false` — ห้าม `git push --force` ที่ main
- `allow_deletions: false` — ห้ามลบ main branch

### 4.2 Dev-level discipline

- **อย่า commit ตรง main** — เปิด branch + PR เสมอ
- **ก่อน push → rebase บน latest main** :
  ```bash
  git fetch origin main && git rebase origin/main
  ```
- **ถ้า rebase ติด conflict** → แก้, `git add`, `git rebase --continue`. ห้าม `git rebase --skip` ถ้าไม่แน่ใจ
- **PR เปิดนาน → rebase ทุก 1-2 วัน** กัน drift
- **Long-running branch** — ห้ามเกิน 1 สัปดาห์ ถ้าเกิน split เป็น multiple PR
- **ห้าม `git push --force`** — ใช้ `git push --force-with-lease` เท่านั้น (กัน overwrite ของคนอื่น)

### 4.3 Cross-repo coordination

ถ้า change กระทบ ≥ 2 repo (เช่น เพิ่ม feature ใน webapp ต้องอัพ wiki):

1. เปิด PR ใน wiki repo **ก่อน** code merge
2. ใส่ link ใน PR description ของ webapp PR ไป wiki PR และกลับ
3. Merge wiki ก่อน → code merge ทีหลัง (กัน doc lag)

ตัวอย่าง PR description:

```markdown
## Related
- Wiki: [LungNote-wikis#42](https://github.com/PASAKON/LungNote-wikis/pull/42)
```

### 4.4 Lock files

`pnpm-lock.yaml` (webapp) = critical file ที่ conflict บ่อยที่สุด.

- ห้ามแก้มือ
- ถ้า conflict → ลบ `pnpm-lock.yaml`, รัน `pnpm install` ใหม่, commit ผลลัพธ์
- ห้ามใช้ `npm` หรือ `yarn` ใน repo นี้ (จะสร้าง lock file ขัดแย้ง)

---

## 5. Sharing .env across Devs

### Single source of truth = **Vercel project env vars**

ทุก secret + config (Supabase keys, DB password, etc.) เก็บที่:
- **Vercel** Project Settings → Environment Variables (Production / Preview / Development)

แต่ละ dev sync ลง local:

```bash
cd webapp
./scripts/pull-env.sh development   # ดึง env "Development" scope ลง .env.local
# หรือ
./scripts/pull-env.sh preview
./scripts/pull-env.sh production    # ระวัง — โหลด prod secrets ลง local
```

### กฎ:

- **Add new env var** → เพิ่มที่ Vercel **ก่อน** commit code ที่ใช้มัน
  ```bash
  vercel env add MY_VAR development preview production
  ```
- **บอกทีม** (Slack/PR description) ให้รัน `./scripts/pull-env.sh` ใหม่
- **`.env.local` ห้าม commit** — อยู่ใน `.gitignore` แล้ว
- **`.env.example`** — keep updated เป็น checklist ของ env vars ที่ต้องมี (ไม่มี value)
- **Rotate secret** → rotate ที่ Vercel (`vercel env rm` + `vercel env add`) → แจ้งทีม pull ใหม่
- **ห้ามแชร์ secret ทาง Slack/email/chat** — ถ้าจำเป็นใช้ 1Password/Bitwarden secure share

### Per-environment scope:

| Vercel env | ใช้ตอน |
|-----------|--------|
| **Development** | `./scripts/pull-env.sh` ของแต่ละ dev (อาจชี้ Supabase project ใหม่/local DB) |
| **Preview** | Vercel preview deploy ของ PR branch |
| **Production** | Vercel deploy ของ `main` (live lungnote.com) |

ตอนนี้ทุก env scope = ค่าเดียวกัน (Supabase prod). พอเริ่ม serious → แยก dev project + dev keys.

---

## 6. Deploy

### 6.1 Auto-deploy (recommended)

`PASAKON/LungNote-webapp:main` → push ทุกครั้ง → Vercel webhook → build + deploy → live `lungnote.com`.

PR branch → push → Vercel preview deploy → URL ใส่ใน PR comment auto.

**ไม่ต้อง deploy ด้วยตัวเอง.** แค่ merge PR.

### 6.2 Manual deploy (fallback)

```bash
cd webapp
vercel --prod   # production
vercel          # preview
```

ใช้เมื่อ: GitHub webhook ตาย, หรือ deploy hot-fix urgent ที่ยังไม่ commit.

### 6.3 Rollback

```bash
vercel rollback <previous-deployment-url>
```

หรือ Vercel dashboard → Deployments → คลิก ⋯ → Promote.

---

## 7. Wiki-only / Design-only changes

### Wiki

```bash
cd ~/code/lungnote/wikis
git checkout -b wiki/<topic>
# แก้ใน Obsidian
git add . && git commit -m "wiki(<scope>): <subject>"
git push -u origin HEAD && gh pr create --fill
```

ไม่ trigger Vercel redeploy ✓.

### Design

```bash
cd ~/code/lungnote/design
git checkout -b design/<topic>
# แก้ HTML mockup
git add . && git commit -m "design(<scope>): <subject>"
git push -u origin HEAD && gh pr create --fill
```

ไม่ trigger Vercel redeploy ✓.

ถ้า design change → webapp ต้อง implement → cross-repo PR (section 4.3).

---

## 8. Troubleshooting

| ปัญหา | ทำยังไง |
|-------|---------|
| `vercel env pull` failed: not authorized | `vercel login` แล้วลองใหม่. ขอ admin add เป็น team member |
| Branch protection reject merge ("not linear") | rebase แล้ว force-push-with-lease บน feature branch ก่อน merge |
| `pnpm-lock.yaml` conflict | ลบ lock + `pnpm install` + commit ใหม่ |
| Vercel build failed: missing env | ตรวจ Vercel dashboard → Settings → Environment Variables → เพิ่มที่ขาด |
| Wiki refer ไป path ใน webapp ที่ลบไปแล้ว | ตอน rename/delete file ใน webapp → grep หา reference ใน wikis แก้ทันที |
| 3 repo ไม่ sync (เห็น `behind: 5`) | `./scripts/sync-all.sh` |

---

## See Also

- [[Dev-Workflow]] — code workflow ทั่วไป
- [[Wiki-Workflow]] — wiki edit workflow
- [[../20-Conventions/Commit-Convention]]
- [[../40-Decisions/0002-split-repos-webapp-wikis-design]]
- [[../40-Decisions/0005-deploy-vercel]]
- [[../40-Decisions/0006-supabase-db-auth]]
