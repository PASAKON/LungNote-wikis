---
title: Commit Convention
tags: [conventions, git]
---

# Commit Convention

ใช้ **Conventional Commits**.

## Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

## Types

| Type | Use |
|------|-----|
| `feat` | new feature |
| `fix` | bug fix |
| `docs` | docs only (รวม wikis) |
| `style` | format, no code change |
| `refactor` | restructure, no behavior change |
| `perf` | performance |
| `test` | tests |
| `chore` | tooling, deps, config |
| `wiki` | wiki-only update (custom) |

## Scope examples

`webapp`, `wikis`, `auth`, `patient`, `ui`, `i18n`, `pwa`

## Examples

```
feat(patient): add intake form
fix(auth): refresh token race condition
wiki(architecture): document data flow
chore: bump next to 16.2.6
```

## Rules

- Subject ภาษาอังกฤษ, lowercase, ไม่มี period
- Body อธิบาย "why" ไม่ใช่ "what"
- Breaking change → `feat!:` หรือ footer `BREAKING CHANGE:`

## See Also

- [[Code-Style]]
- [[../50-Workflows/Dev-Workflow]]
