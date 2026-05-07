---
title: Collaboration Workflow
tags: [workflow, collaboration]
---

# Collaboration Workflow

How a small team (2+ devs) ships changes to LungNote without stepping on each other or merging stale branches.

This page **extends** [[Dev-Workflow]] (which covers branch types and pre-commit checks) and [[Multi-Repo-Workflow]] (3-repo daily loop). Read those first.

---

## 1. Daily Start

Run at the beginning of every coding session, in every repo you'll touch:

```bash
git switch main
git pull --rebase
./scripts/pull-env.sh           # webapp only — pulls latest .env from Vercel
git switch -c <type>/<scope>-<short>
```

`./scripts/sync-all.sh` does step 1+2 across all 3 repos in one go.

Branch type/scope conventions live in [[Dev-Workflow#Branch]].

---

## 2. Pre-Push Checklist

**MUST run every time before `git push`.** This is the single rule that prevents the "merged a stale branch" failure mode.

```bash
# 1. Fetch latest
git fetch origin

# 2. Compare. If the right-hand count > 0, your base is stale.
git rev-list --left-right --count HEAD...origin/main
#       ^ahead    ^behind

# 3. If behind, rebase onto latest main:
git rebase origin/main
# resolve conflicts → git add <files> → git rebase --continue

# 4. Verify build still works after rebase (webapp only)
cd webapp && pnpm lint && pnpm build

# 5. Push with --force-with-lease (NEVER --force, NEVER push to main)
git push --force-with-lease
```

**Why `--force-with-lease`:** plain `--force` overwrites the remote unconditionally and silently destroys any commits the other dev added to *your* branch. `--force-with-lease` refuses if the remote moved unexpectedly.

---

## 3. PR Rules

| Rule | Why |
|------|-----|
| Always open a PR — never push to `main` | [[../../CLAUDE\|CLAUDE.md]] hard rule #10 |
| Reviewer = the *other* team member | 2-person team → mandatory peer review |
| Use `gh pr create --fill` (or with explicit body) | Auto-uses commit messages as title/body |
| Squash-merge from GitHub UI | Keeps `main` history linear and readable |
| Delete branch after merge | Repo setting "Automatically delete head branches" |
| Behavior change → update wiki in same or linked wiki PR | [[../../CLAUDE\|CLAUDE.md]] hard rules #6/#7, [[../20-Conventions/Wiki-Style#10-Cross-Repo-Discipline\|Wiki-Style §10]] |

Cross-repo PRs (webapp + wikis together) — see [[Multi-Repo-Workflow#4.3]].

---

## 4. Avoiding Conflicts

- **Pick non-overlapping work upfront.** A 30-second message ("I'm taking the audit_log table, you take LINE Login") beats a 30-minute merge conflict.
- **Push branches early, even WIP.** Use draft PRs for visibility:
  ```bash
  gh pr create --draft --fill
  ```
- **See active branches the other dev pushed:**
  ```bash
  git fetch && git branch -r --no-merged origin/main
  ```
- **Long-lived branches (>2 days) → rebase onto main daily.** Drift compounds. The longer a branch lives, the more painful the eventual rebase.
- **Daily status check across repos:**
  ```bash
  ./scripts/status-all.sh
  ```

---

## 5. GitHub Repo Settings (one-time)

These should already be set on `PASAKON/LungNote-webapp` and `PASAKON/LungNote-wikis`. If not, an admin should enable them:

- **Require pull request reviews before merging** (1 approval minimum)
- **Require branches to be up to date before merging** ← this enforces the rebase rule at the platform level. GitHub blocks the merge button until your branch sits on the latest `main`.
- **Require status checks to pass before merging** (CI: lint + build for webapp)
- **Restrict pushes that create matching branches** to `main` (no direct pushes)
- **Automatically delete head branches** after merge

The "up to date before merging" setting is the most important — it's the safety net for any developer who forgets the pre-push checklist in §2.

---

## 6. Quick Reference

```bash
# Am I behind main right now?
git fetch && git rev-list --left-right --count HEAD...origin/main

# Update my branch with latest main mid-work
git fetch && git rebase origin/main

# Small cleanup after review feedback (instead of new commit)
git commit --amend --no-edit && git push --force-with-lease

# See whose branches are open
git fetch && git branch -r --no-merged origin/main

# Sync all 3 repos (webapp + wikis + design)
./scripts/sync-all.sh
```

---

## See Also

- [[Dev-Workflow]] — branch types, pre-commit checks
- [[Multi-Repo-Workflow]] — 3-repo daily loop, cross-repo PR coordination
- [[Wiki-Workflow]] — workflow specific to editing this vault
- [[../20-Conventions/Commit-Convention]] — conventional commit format
- [[../20-Conventions/Wiki-Style#10-Cross-Repo-Discipline]] — wiki + webapp PR pairing
