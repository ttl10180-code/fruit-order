# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

월림마을 (Wollim Village) — a Korean direct-from-farm fruit ordering site for 파파야메론 (papaya melon) and 참외 (Korean melon). All UI strings, comments, and customer-visible text are in Korean; preserve Korean text verbatim when editing.

The repo is two standalone static HTML files plus a hero image:

- `Index.html` — customer-facing 5-step order flow (product → shipping → confirm → payment → "맛있게 먹기").
- `admin.html` — password-protected admin dashboard (orders list, status changes, settings, SMS templates).
- `IMG_4541.png` — hero image, also referenced via raw.githubusercontent.com for OG/Twitter cards.

There is **no build system, no package.json, no test suite, no lint config**. Each HTML file is fully self-contained: inline `<style>`, inline `<script>`, no modules, no bundler. Edit the HTML directly and reload in a browser to verify.

## Architecture

### Persistence — Google Apps Script + Sheets, not a real backend

All order data lives in a Google Sheet exposed by a Google Apps Script Web App. The endpoint URL is hardcoded in both files:

```
https://script.google.com/macros/s/AKfycbxZ_QKrO9STcUM71pRQi3TB_Trq3LBfTcjxngbpo_btWOyJoU2WU7RWWuq6wVIBThCe/exec
```

Wire protocol (used by both files; the Apps Script source is **not in this repo**):

- `GET  ?action=getOrders` → `{ orders: [...] }`
- `POST` body `{ orderNum, orderTime, name, phone, address, address2, memo, items, totalAmount, paymentStatus, melon5, melon10, chamoe5, chamoe10 }` — new order
- `POST` body `{ action:'updatePayment', orderNum, status, totalAmount }` — admin status change / customer "입금 완료" confirmation

POSTs use `mode: 'no-cors'` (fire-and-forget; responses are not readable). Don't add response-handling logic to POSTs — it will silently fail.

### localStorage as cross-page config store

Both files communicate through `localStorage`. Keys to know:

- `gs_webhook` — Apps Script URL, read by both `Index.html` (for `loadMelonStock` and order POSTs) and `admin.html`. Both pages currently force-overwrite this on load with `FIXED_URL`.
- `admin_gs_webhook`, `admin_prices`, `admin_account`, `admin_kakaopay`, `admin_sms_templates` — admin-only settings.

The admin "💰 가격 설정" tab writes prices to `localStorage.admin_prices`, but **`Index.html` does not read these** — it has its own hardcoded `PRODUCTS` constant. The admin UI explicitly warns about this on screen ("주문 사이트(index.html)도 직접 수정"). When changing prices, edit `PRODUCTS` in `Index.html` directly in addition to using the admin UI.

### Index.html order flow

- `PRODUCTS`, `sel` (selected map), `qty` (quantity map) drive everything; SKUs are `melon5 | melon10 | chamoe5 | chamoe10`.
- `goPage(n)` toggles `.page` divs and the `.step-tab` indicators (steps 1–5). Each transition has inline validation (e.g. step 3 requires name/phone/address).
- `SOLD_OUT` map gates buying — currently `chamoe5` and `chamoe10` are flagged as 마감.
- Melon stock: `MELON_INIT_KG = 455`; `loadMelonStock()` fetches all orders with status in `['입금대기','입금완료','배송중']` and sums `melon5*5 + melon10*10` to compute remaining kg, displayed live.
- **Known bug**: `toggleProd` and `chQty` reference `MELON_MAX_KG`, which is never defined — only `MELON_INIT_KG` exists. Stock-cap alerts will throw a ReferenceError. If you touch this code, decide whether to rename to `MELON_INIT_KG` or introduce `MELON_MAX_KG`.
- Order numbers are client-generated: `'#' + Math.floor(100000 + Math.random()*900000)`.
- Address search uses Daum Postcode (`postcode.v2.js`) loaded from CDN.
- `window.onbeforeunload` is set when entering page-4 to warn users not to navigate away before clicking "입금 완료". `confirmPayment()` clears it.

### admin.html

- Login is a hardcoded password constant `PW = '820523'` checked client-side. There is no real auth — anyone who reads the source has access. Don't pretend it's secure.
- Two tabs: `📋 주문 관리` and `⚙️ 설정`. `switchTab(name, el)` toggles `.tab-content.active`.
- Filters: status filter (`전체 | 입금대기 | 입금완료 | 배송중 | 선물 | 배송완료`), date range (`date-from`/`date-to`), and keyword search across name/phone/orderNum.
- Status set: `입금대기, 입금완료, 배송중, 배송완료, 선물`. Changing to `선물` zeroes `totalAmount` both locally and in the POST body.
- SMS sending is done via `sms:` URI links opened in the user's device (no SMS gateway). Templates support `{이름} {상품} {금액} {날짜} {주소}` placeholders.
- Tracking number entry assumes a `442-210` Lotte prefix; the input takes the trailing 5 digits.
- `getDemoOrders()` returns hardcoded sample data when the fetch to Apps Script fails — useful for offline UI work.

## Working in this repo

### Running / testing

Open the HTML files in a browser. There is no dev server — but because both files hit the live Apps Script, opening from `file://` works. To exercise the full flow without polluting real data, comment out the `fetch(wh, ...)` calls in `submitOrder` / `confirmPayment` / `changeStatus` while iterating.

### Deployment

The site is served directly from this GitHub repo (the OG image and favicon use `raw.githubusercontent.com/ttl10180-code/fruit-order/main/IMG_4541.png`). Pushing to `main` updates production. Commit messages historically have been "Update Index.html" / "Update admin.html" from GitHub's web editor — feel free to write more descriptive ones.

### Conventions when editing

- Keep HTML/CSS/JS inline in a single file per page; don't split into separate `.css` / `.js` assets without a reason — the project deliberately ships as drop-in static files.
- Many lines pack multiple statements on one line (especially in `Index.html`'s `<script>`). Match the surrounding style rather than reformatting unrelated code.
- The two pages share storage but **not** a price source of truth. If you add a setting that should apply to the customer page, also update the corresponding constant in `Index.html`.
- When changing the order schema (fields in the POST payload), remember the Apps Script + Google Sheet columns must match — and that source is not in this repo.
