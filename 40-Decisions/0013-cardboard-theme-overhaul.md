---
title: "ADR-0013: Cardboard theme overhaul (replaces Mint)"
tags: [adr, brand, theme, design]
status: Accepted
date: 2026-05-09
---

# ADR-0013 — Cardboard theme overhaul

## Status

Accepted

## Context

Original brand used a mint-green accent (`#6aab8e`) with off-white surfaces, paired with a sketchy SVG-filtered note aesthetic. After product-owner review the mint felt too generic / clinical for a Thai student-focused notes app. Designer iterated and landed on a **cardboard** palette — warm amber/yellow on tinted off-white surfaces, plus a hand-drawn box mascot — which evokes a paper/cardboard scrapbook feel that matches the app's "บันทึก, จดโน้ต" tone.

The old palette was hard-coded in 4 surfaces:
- `webapp/src/app/[locale]/dashboard/dashboard.css` (token block + dark-mode override)
- `webapp/src/app/liff/liff.css`
- `webapp/src/components/landing/landing.css`
- 5 LINE Flex Message JSONs (`welcome`, `note-saved`, `dashboard-link`, `error`, `notification`)
Plus standalone constants in `opengraph-image.tsx`, `apple-icon.png`, `icon.svg`, `manifest.ts`, `Hero.tsx`, `Features.tsx`, `liff/loading.tsx` spinner.

## Decision

Adopt the **cardboard palette + mascot** as the canonical brand. Source of truth lives at `design/brand/cardboard-palette.html`.

### Color tokens

| Token | Light | Dark |
|---|---|---|
| `--bg` | `#F5EAD4` | `#1A1810` |
| `--surface` | `#FAF5E8` | `#2A2618` |
| `--border` | `#D4C4A0` | `#3D3828` |
| `--muted` | `#A08050` | `#8A7850` |
| `--fg` | `#3A3020` | `#E8E2D0` |
| `--accent` | `#C9A040` | `#C9A040` |
| `--accent-light` | `#F0E4C4` | `#3D3420` |
| `--yellow` (tape) | `#D4A855` | `#B89030` |
| `--red` | `#C45A3A` | `#C45A3A` |
| `--orange` | `#D4A040` | `#D4A040` |

Semantic (both modes):
- success `#7A9A50` on `#E8F0D8`
- warning `#D4A040` on `#F0E4C4`
- error `#C45A3A` on `#F0D8D0`

### WordMark

`Lung` in foreground color + `Note` in `--accent`. Caveat font (existing `--font-display`).

### Mascot

Hand-drawn wobbly box with check-mouth + side arms. Stored at `design/mascot-icon/lungnote-mascot-icon.svg` plus PNG sizes 32 / 64 / 120 / 180 / 256 / 512 / 1024. Replaces the old plain checkbox icon.

The 180px PNG ships as `webapp/src/app/apple-icon.png`. The SVG ships as `webapp/src/app/icon.svg`. OG image (`opengraph-image.tsx`) inlines a scaled version of the same paths.

### Migration

One sweep across webapp:
- CSS token blocks rewritten in 3 files
- 5 Flex JSONs color-by-color (same hex map)
- Standalone hex literals replaced via sed across `Hero.tsx`, `Features.tsx`, `LiffClient.tsx`, `liff/page.tsx`, `liff/loading.tsx`, `manifest.ts`, `[locale]/layout.tsx`, `opengraph-image.tsx`
- Icon assets copied from `design/mascot-icon/`
- Rich Menu PNG (`design/line-rich-menu/lungnote-rich-menu-default-2500x1686.png`) was already overwritten in the design repo's cardboard commit; reinstall on LINE OA via existing `webapp/scripts/install-rich-menu.sh` runbook ([[../50-Workflows/Rich-Menu-Install]])

CSS variable names preserved (`--accent`, `--yellow`, `--orange` etc.) so structural rules don't move; only the hex values change. The `.mint` utility class on `stat-mini-val` still exists but its color is sourced from `var(--accent)` so it auto-updates — left in place for now to keep diffs surgical, will rename to `.accent` in a follow-up.

## Consequences

**Positive**
- Distinct brand voice — warmer, less generic than mint.
- Mascot opens up future product surfaces (loading states, empty states, sticker pack).
- All hex literals now centralized; future theme changes touch ~3 CSS files + 5 JSONs.

**Negative**
- LINE Rich Menu must be reinstalled manually after merge (LINE caches the PNG by ID).
- Users on prefers-color-scheme dark see a fairly different look; QA the dark variant end-to-end.
- The `.mint` class name is now misleading. Renaming is deferred to keep this PR's diff focused.

## References

- Source palette: `design/brand/cardboard-palette.html`
- Mascot assets: `design/mascot-icon/`
- Rich Menu reinstall: [[../50-Workflows/Rich-Menu-Install]]
- Glossary updated with "Cardboard palette" + "Mascot" entries
