---
title: Code Conventions
tags: [conventions, code-style]
---

# Code Conventions

## TypeScript

- `strict: true` เสมอ
- ห้าม `any` — ใช้ `unknown` + narrow
- Prefer `type` สำหรับ unions, `interface` สำหรับ object shape ที่ extend ได้

## Naming

| Item | Convention | Example |
|------|------------|---------|
| File component | PascalCase | `PatientCard.tsx` |
| File util/hook | kebab-case | `use-patient.ts`, `format-date.ts` |
| Component | PascalCase | `PatientCard` |
| Hook | camelCase + `use` prefix | `usePatient` |
| Constant | SCREAMING_SNAKE | `MAX_RETRIES` |
| Type / Interface | PascalCase | `Patient`, `PatientDTO` |

## Imports

- Absolute via `@/*` (alias ใน `tsconfig.json`)
- Order: external → `@/` → relative

## Tailwind

- Mobile-first (default = mobile, ใช้ `sm:` `md:` ขยายขึ้น)
- จัด class ด้วย `cn()` util (เพิ่มใน `@/lib/utils`)
- ห้าม inline style ยกเว้นค่า dynamic ที่คำนวณ runtime

## ESLint

- รัน `pnpm lint` ก่อน push
- ห้าม disable rule ทั้ง file — disable เฉพาะ line + comment เหตุผล

## See Also

- [[Commit-Convention]]
- [[../10-Architecture/Overview]]
