---
title: Designer Handoff — LINE Dashboard MVP
tags: [workflow, design, handoff, line, dashboard]
---

# Designer Handoff — LINE Dashboard MVP

Spec ที่ Designer ต้อง deliver สำหรับ feature [[../40-Decisions/0008-line-only-auth-account-linking|LINE-only Auth]] + Dashboard MVP (notes CRUD).

Code/wiring จะทำหลัง assets ลง.

---

## Brand Tokens (reference)

มาจาก landing page ([`webapp/src/components/landing/landing.css`](https://github.com/PASAKON/LungNote-webapp/blob/main/src/components/landing/landing.css)):

| Token | Light | Dark |
|---|---|---|
| `--bg` | `#faf8f4` | `#1e1e1c` |
| `--surface` | `#fffef9` | `#2a2a28` |
| `--border` | `#e0ddd4` | `#3a3835` |
| `--muted` | `#8a8578` | `#8a8578` |
| `--fg` | `#2c2a25` | `#e8e6e0` |
| `--accent` | `#6aab8e` | `#6aab8e` |
| `--accent-light` | `#e8f4ed` | `#1a3328` |
| `--yellow` | `#f0d87a` | `#b8a550` |
| `--yellow-light` | `#fdf6dc` | `#3d3a2a` |

Fonts: **Sarabun** (body, TH+EN), **Caveat** (display, EN-only headlines), **JetBrains Mono** (code/monospace).

Style cue: hand-drawn / sketchy — SVG turbulence filter (`#sketchy`) บน borders + circles.

---

## Deliverable 1 — LINE Rich Menu

**Spec:** [LINE Rich Menu reference](https://developers.line.biz/en/docs/messaging-api/using-rich-menus/)

| Property | Value |
|---|---|
| Image | PNG, **2500×1686 px** (large size) |
| Format | sRGB, < 1 MB |
| Layout | 6-tile grid (3×2) — แต่ละ tile = button area |
| Tile size | ~833×843 px |
| Default state | Show เมื่อ user เปิด chat ของ OA |

**Tiles (left→right, top→bottom):**

1. 📊 **Dashboard** — postback `action=open_dashboard`
2. 📓 **โน้ตของฉัน** — postback `action=list_notes` (ส่ง quick summary แล้ว)
3. ➕ **เพิ่มโน้ต** — postback `action=quick_add_note` (bot prompt next message)
4. 🔍 **ค้นหา** — postback `action=search_notes`
5. ⚙️ **ตั้งค่า** — postback `action=settings`
6. ❓ **ช่วยเหลือ** — postback `action=help`

**Style guideline:**
- ใช้ brand tokens ข้างบน
- Hand-drawn icons (style สอดคล้องกับ landing)
- Dark text บน light bg (default)
- Caveat สำหรับ icon label EN, Sarabun สำหรับ TH

**Output for Claude:**
- 1 PNG file: `lungnote-rich-menu-default-2500x1686.png`
- 1 JSON file: rich menu config (areas + actions) — โครงตัวอย่าง:
  ```json
  {
    "size": { "width": 2500, "height": 1686 },
    "selected": false,
    "name": "LungNote default",
    "chatBarText": "เมนู",
    "areas": [
      { "bounds": { "x": 0, "y": 0, "width": 833, "height": 843 },
        "action": { "type": "postback", "data": "action=open_dashboard" } },
      ...
    ]
  }
  ```

---

## Deliverable 2 — Flex Messages (JSON)

ทั้งหมด JSON ตาม [Flex Message Simulator](https://developers.line.biz/flex-simulator/).

### 2.1 — Welcome Card (on follow event)

ใช้เมื่อ user add OA เป็นเพื่อนครั้งแรก.

Content:
- Logo (LungNote icon)
- Headline "ยินดีต้อนรับสู่ LungNote 📓"
- Subtitle "จดโน้ต เช็คลิสต์ จัดระเบียบชีวิต"
- Primary button: "📊 เปิด Dashboard" → postback `action=open_dashboard`
- Secondary button: "❓ วิธีใช้" → postback `action=help`

### 2.2 — Dashboard Link Card (on text "dashboard")

ส่งหลัง user request link.

Content:
- Headline "🔗 ลิงก์ Dashboard ของคุณ"
- Body "ลิงก์นี้ใช้ได้ภายใน 5 นาที, 1 ครั้งเท่านั้น"
- Primary button (URI action): "เปิด Dashboard" → URL `https://lungnote.com/auth/line?t=<token>` (Claude inject token at runtime)
- Footer: "หากลิงก์หมดอายุ พิมพ์ 'dashboard' อีกครั้ง"

### 2.3 — Note Saved (after quick add)

Confirmation หลัง user สร้างโน้ตผ่าน OA.

Content:
- Icon ✅
- Headline "บันทึกแล้ว"
- Body แสดง preview ของโน้ต (truncate 80 chars)
- Footer button: "ดูทั้งหมด" → postback `action=list_notes`, "เปิด Dashboard" → postback `action=open_dashboard`

### 2.4 — Error Card

Generic error.

Content:
- Icon ⚠️
- Headline "เกิดข้อผิดพลาด"
- Body (parameterized text)
- Button: "ลองอีกครั้ง" → postback `action=retry`

**Output for Claude:**
- 4 JSON files in `design/line-flex/`:
  - `welcome.json`
  - `dashboard-link.json` (use placeholder `{{TOKEN_URL}}`)
  - `note-saved.json` (placeholder `{{NOTE_PREVIEW}}`)
  - `error.json` (placeholder `{{MESSAGE}}`)

---

## Deliverable 3 — Dashboard UI (Web)

**Route:** `/[locale]/dashboard` (TH default)
**Viewport:** mobile-first (375px), responsive up to desktop (1080px max)
**Audience:** opened mostly inside LINE in-app browser (Safari/Blink WebView)

### 3.1 — Pages required

| Page | Path | Content |
|---|---|---|
| Notes list | `/dashboard` | Header + "+" FAB + list of notes (card per note, click → edit) |
| Edit note | `/dashboard/notes/[id]` | Title input + body textarea + Save / Delete buttons |
| New note | `/dashboard/notes/new` | Same form, empty |

### 3.2 — Components needed

| Component | Notes |
|---|---|
| `<DashboardHeader>` | Logo + user avatar (line_picture_url) + display name + logout |
| `<NoteCard>` | Title + preview (first 80 chars) + relative date + click area |
| `<NoteList>` | Grid (1 col mobile, 2 desktop), empty state |
| `<NoteForm>` | Title (required), body (markdown-ish for now, plain), Save/Delete CTAs |
| `<EmptyState>` | "ยังไม่มีโน้ต — สร้างเล่มแรกเลย" + button |
| `<LoadingSkeleton>` | Per-card shimmer |
| `<ErrorBoundary>` | Friendly message + retry |
| FAB "+ Add note" | bottom-right, always visible |

### 3.3 — Style direction

- Reuse landing's sketchy style (border + filter url(#sketchy))
- Notebook-paper feel (lined background subtle)
- Caveat for note titles (display feel), Sarabun for body
- Min-touch-target 44×44px (LINE WebView is mobile)
- Use `--accent` (#6aab8e) for primary CTA
- Dark mode auto (prefers-color-scheme)

### 3.4 — States to mock

- ✅ list with 0 notes (empty state)
- ✅ list with 3-5 notes (typical)
- ✅ list with 50+ notes (scroll, performance)
- ✅ edit form blank (new)
- ✅ edit form with content
- ✅ saving (button loading)
- ✅ delete confirmation (modal/sheet)
- ✅ session loading (cookie pending)
- ✅ session error (no cookie / expired)

### 3.5 — Output

- Figma file shared link OR HTML mocks ใน `LungNote-design` repo:
  - `dashboard/list.html`
  - `dashboard/edit.html`
  - `dashboard/empty.html`
  - `dashboard/error.html`
- Mock interactions noted in comments (click X → goto Y)

---

## Coordination

1. Designer พร้อม → push to [LungNote-design](https://github.com/PASAKON/LungNote-design) repo (own branch + PR)
2. Notify Claude (via Discord/PR comment): "Rich Menu ready" / "Flex JSON ready" / "Dashboard mocks ready"
3. Claude: implement → wire assets → push PR
4. Designer review PR preview deploy → feedback → iterate

---

## Spec Constraints (don't break)

- **No raw email collected** — synthetic email only ([[../40-Decisions/0008-line-only-auth-account-linking|ADR-0008]])
- **No web Login button** — entry only via LINE OA link
- **All copy in TH primarily** (EN secondary, but landing already i18n)
- **DB tables prefix** `lungnote_*` ([[../20-Conventions/Database-Naming]])
- **No client-side service_role key** — server-only via `lib/supabase/admin.ts`

---

## See Also

- [[../40-Decisions/0008-line-only-auth-account-linking]]
- [[LINE-Auth-Flow]]
- [[Database-Migration]]
- [[../20-Conventions/Database-Naming]]
- [LungNote-design repo](https://github.com/PASAKON/LungNote-design)
