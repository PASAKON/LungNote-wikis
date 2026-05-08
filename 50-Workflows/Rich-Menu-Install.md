---
title: Rich Menu Install Runbook
tags: [workflow, line, rich-menu, ops]
---

# Rich Menu Install Runbook

ขั้นตอน install / refresh Rich Menu ของ LINE OA. ใช้ทุกครั้งที่ designer ส่ง PNG ใหม่ หรือเปลี่ยน LIFF route mapping.

Script: [`webapp/scripts/install-rich-menu.sh`](https://github.com/PASAKON/LungNote-webapp/blob/main/scripts/install-rich-menu.sh)

---

## 1. Pre-requisites

| Tool | Install |
|---|---|
| `jq` | `brew install jq` |
| `curl` | preinstalled on macOS / Linux |

Env (export ก่อนรัน):

```bash
export LINE_CHANNEL_ACCESS_TOKEN=$(grep '^LINE_CHANNEL_ACCESS_TOKEN' webapp/.env.local | cut -d= -f2- | tr -d '"')
export NEXT_PUBLIC_LINE_LIFF_ID=$(grep '^NEXT_PUBLIC_LINE_LIFF_ID' webapp/.env.local | cut -d= -f2- | tr -d '"')
```

`LINE_CHANNEL_ACCESS_TOKEN` = Messaging API channel token (NOT LINE Login channel).

---

## 2. Inputs

Designer's [`LungNote-design`](https://github.com/PASAKON/LungNote-design) repo:

| File | Purpose |
|---|---|
| `line-rich-menu/lungnote-rich-menu-default.json` | rich menu config (areas, actions, name) — ships with `YOUR_LIFF_ID` + `lungnote.app` placeholders |
| `line-rich-menu/lungnote-rich-menu-default-2500x1686.png` | dark theme PNG, RGB, < 1MB |
| `line-rich-menu/lungnote-rich-menu-light-2500x1686.png` | light theme PNG (alternative) |

Make sure designer repo is cloned alongside webapp (`~/code/lungnote/design/`).

---

## 3. Run

From webapp root:

```bash
cd webapp
./scripts/install-rich-menu.sh \
  ../design/line-rich-menu/lungnote-rich-menu-default.json \
  ../design/line-rich-menu/lungnote-rich-menu-default-2500x1686.png
```

**Idempotent:** script lists existing rich menus, deletes any with the same name (`LungNote Default Rich Menu`), then creates fresh + uploads PNG + sets default for all users.

**Substitutions performed inline:**
- `YOUR_LIFF_ID` → `$NEXT_PUBLIC_LINE_LIFF_ID` (so `liff.line.me/<liffId>/dashboard` etc. resolve correctly)
- `lungnote.app` → `lungnote.com` (designer JSON ships with `.app`, our domain is `.com`)

**Expected output:**

```
==> Substituting placeholders
    name = LungNote Default Rich Menu
==> Listing existing rich menus
==> Creating rich menu
    richMenuId = richmenu-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
==> Uploading PNG
==> Setting as default for all users
✅ Rich menu installed.
```

---

## 4. Verify

```bash
# Confirm default is set
curl -fsSL -H "Authorization: Bearer $LINE_CHANNEL_ACCESS_TOKEN" \
  https://api.line.me/v2/bot/user/all/richmenu | jq

# Should return: {"richMenuId":"richmenu-..."}
```

หรือเปิด LINE app → chat OA → menu bar ด้านล่างมี Rich Menu ปรากฏ. กด tile → เปิด LIFF / lungnote.com URL ตามที่กำหนด.

---

## 5. Switch theme (light vs dark)

```bash
# Re-run with light PNG (script auto-deletes old + uploads new)
./scripts/install-rich-menu.sh \
  ../design/line-rich-menu/lungnote-rich-menu-default.json \
  ../design/line-rich-menu/lungnote-rich-menu-light-2500x1686.png
```

(การเปลี่ยนรูปใช้ DELETE + CREATE — `richMenuId` ใหม่ทุกครั้ง. นั่นเป็น LINE API constraint, ไม่มี endpoint update PNG ของ rich menu เดิม)

---

## 6. Per-user overrides (advanced)

ถ้าต้องการ rich menu ต่างกันต่อ user (เช่น Admin menu vs User menu):

```bash
curl -fsS -X POST \
  -H "Authorization: Bearer $LINE_CHANNEL_ACCESS_TOKEN" \
  -H "Content-Length: 0" \
  https://api.line.me/v2/bot/user/{userId}/richmenu/{richMenuId}
```

ตอนนี้ใช้แค่ default อันเดียว, อนาคตอาจ split.

---

## 7. Common errors

| HTTP | Cause | Fix |
|---|---|---|
| 401 | bad / expired access token | refresh `LINE_CHANNEL_ACCESS_TOKEN` ใน Vercel + `.env.local` |
| 411 Length Required | POST without body | already handled in script (`Content-Length: 0` header) |
| 400 Invalid JSON | Bad areas/bounds in JSON | check designer JSON has all required fields (`size`, `selected`, `name`, `chatBarText`, `areas[]`) |
| 413 Image too large | PNG > 1MB | designer must re-export with smaller file size |
| Image not RGB | 4-channel RGBA PNG rejected | designer convert to RGB |

---

## 8. CI/CD

ตอนนี้ install ทำมือ. ในอนาคตอาจ:
- GitHub Action: ใน `LungNote-design` repo, on push to main + change in `line-rich-menu/`, trigger workflow ที่ดึง webapp env + รัน install
- Or: webhook hook from design repo merge → webapp action

Defer until rich menu changes บ่อย. ตอนนี้ทำมือก็พอ.

---

## See Also

- [[../40-Decisions/0007-line-oa-messaging]]
- [[../40-Decisions/0010-liff-auth]] — LIFF endpoint URL ที่ rich menu ชี้ไป
- LINE Rich Menu API: https://developers.line.biz/en/reference/messaging-api/#rich-menu
- [[Multi-Repo-Workflow]]
