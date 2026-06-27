# Stone Rock Studios — Claude Code Context

## Project Overview
Video portfolio and production site for Stone Rock Studios (stonerockstudios.xyz), a creative brand run by Jes and Jackson (de Melo).

- **Live site:** https://stonerockstudios.xyz
- **Hosting:** GitHub Pages
- **Domain:** Custom via GoDaddy

## Tech Stack
- **Frontend:** HTML/CSS/JS — GitHub Pages static site
- **Layout:** Masonry.js (mixed aspect ratio video grid)
- **Video data:** YouTube Data API — pulls from YouTube playlists dynamically
- **Admin backend:** Firebase (in progress — Level 4 build)

## Current Development Phase: Firebase Admin Dashboard
Building a full admin dashboard backed by Firebase that enables:
- Manual control over video ordering
- Video visibility toggling (show/hide per video)
- Category/tag management
- All changes reflected live on the public site — no code edits required

A Claude Code agent instruction document exists for this build. Reference it before making admin dashboard changes.

## Repository Structure (expected)
- `index.html` — main portfolio page
- `admin/` — Firebase admin dashboard (in progress)
- `.github/` — GitHub Actions workflows (if present)
- `CLAUDE.md` — this file

## Key Rules
- Do not break the existing YouTube API playlist integration
- Masonry layout must remain functional across aspect ratios
- Firebase rules should be locked down — admin access only
- Keep the public site fast and lightweight (no heavy dependencies)
- Prefer free/open source solutions before paid alternatives

## Git Workflow
- Main branch: `main`
- Always write clear, descriptive commit messages
- Create feature branches for major changes (e.g. `feature/admin-dashboard`)
- Push to origin when work is stable

---

## Change Log

After every edit, append an entry here so future Claude instances understand what was built, why, and what to watch out for.

---

### 2026-06-26 — Cart item editing (shop.html)

**What changed:** Added three in-cart editing capabilities to the cart drawer in `shop.html`.

**Features added:**
- **Remove item** — ✕ button per cart row. Calls Shopify `cartLinesRemove` mutation.
- **Change quantity** — − / + buttons per cart row. Calls `cartLinesUpdate`. Decrementing to 0 auto-removes the item.
- **Change variant (size/color)** — `<select>` dropdown per item that lists all variants from the already-loaded `products` array (matched by product title). Out-of-stock variants are shown disabled. Calls `cartLinesUpdate` with a new `merchandiseId` to swap variants without removing and re-adding.

**Why these choices:**
- Variant changing uses `cartLinesUpdate` with `merchandiseId` (a single mutation) rather than remove + add, which avoids a brief missing-item state in the UI.
- The variant options are sourced from the in-memory `products` array (already fetched on page load) rather than a second API call, keeping it fast and free.
- Products with a single "Default Title" variant skip the dropdown and show plain text — no unnecessary UI for items with no real choices.
- All three actions disable their control while the async Shopify call is in flight, then `refreshCartUI()` re-renders the full drawer from the updated cart object.

**New functions added (shop.html JS):**
- `cartRemoveLine(cartId, lineId)` — wraps `cartLinesRemove` mutation
- `cartUpdateLines(cartId, lines)` — wraps `cartLinesUpdate` mutation; used for both qty changes and variant swaps

**New CSS classes (shop.html):**
- `.cart-item-head` — flex row for name + remove button
- `.cart-remove` — the ✕ button
- `.cart-variant-sel` — the variant dropdown select
- `.cart-item-controls` — flex row for − qty + and price
- `.cart-qty-btn` / `.cart-qty-num` — the qty control buttons and display

---

### 2026-06-26 — Cart UX polish + product sold-out states (shop.html)

**What changed:** Six new features added across cart drawer, product grid, and product modal.

**Features added:**
- **Discount code field** — input + Apply button in cart footer. Calls `cartDiscountCodesUpdate`. Applied code shows a green confirmation row with ✕ to remove. Invalid codes show inline error. `CART_F` now includes `discountCodes{code applicable}`.
- **Order note** — textarea in cart footer (above discount field). Auto-saves 800ms after the user stops typing via `cartNoteUpdate`. Does not override if the textarea is currently focused (avoids clobbering active typing). Restored from `localStorage` on page load.
- **Subtotal / tax / total breakdown** — `CART_F` updated to request `subtotalAmount`, `totalTaxAmount`, `totalAmount` (previously only `totalAmount` was fetched and mislabeled as Subtotal). Tax row only renders if `taxAmt > 0`; Total row only renders if it differs from Subtotal (i.e., discount or tax is applied).
- **Sold-out state on ATC button** — `updateAtcState()` sets button to disabled "Sold Out" text when selected variant's `availableForSale` is false; "Select options" when no variant matches. Called from both `openModal` and `pickOpt`. Previously the check only fired at click time.
- **"Only X left" low-stock label** — `quantityAvailable` added to the variants query. `updateAtcState()` shows "Only N left" in `atcNote` when `quantityAvailable <= 4` and not null.
- **Sold-out tiles in grid** — If all variants are sold out, the product tile gets class `tile-sold-out` (image dims to 35% opacity) and a "Sold out" sub-label.
- **Cart image hover zoom** — Cart items now use `.cart-item-img-wrap` (68×68, `overflow:hidden`) wrapping `.cart-item-img`. Hover on `.cart-item` scales the image to 1.08×. Requires the wrapper div in the JS template; bare `<div class="cart-item-img">` is no longer used.
- **Badge pop animation** — `refreshCartUI(animateBadge)` now accepts a boolean. Passing `true` triggers a CSS `badge-pop` keyframe when count increases. Called with `true` on ATC add and qty + button; `false`/default on remove, restore from storage.

**Why these choices:**
- `cartNoteUpdate` debounces at 800ms rather than on blur so users get a save without having to click away; the `document.activeElement` guard prevents overwriting what they're typing during re-renders.
- Discount `applicable` flag from Shopify is the only reliable signal for validity — the API always stores the code even if invalid, so you must check `applicable`, not just presence of `discountCodes`.
- Tax/total rows are conditionally shown to avoid redundant "Subtotal $X / Total $X" when no discount or tax applies.
- Image zoom uses a wrapper `overflow:hidden` because `overflow:hidden` on the img itself doesn't clip CSS `transform: scale()` — the clip must be on the parent.

**New functions added (shop.html JS):**
- `cartApplyDiscount(cartId, codes)` — wraps `cartDiscountCodesUpdate`; pass `[]` to remove all codes
- `cartUpdateNote(cartId, note)` — wraps `cartNoteUpdate`
- `updateAtcState()` — sets ATC button text/disabled + low-stock note based on selected variant

**New CSS classes (shop.html):**
- `.cart-item-img-wrap` — replaces bare `.cart-item-img` sizing; holds the zoom clip
- `.cart-note-wrap` / `.cart-note-area` — order note textarea
- `.cart-discount-row` / `.cart-discount-input` / `.cart-discount-btn` — discount code row
- `.cart-discount-applied` / `.cart-discount-code` / `.cart-discount-remove` — applied code UI
- `.cart-discount-err` — error text for invalid codes
- `.cart-tax-row` / `.cart-tax-label` / `.cart-tax-amt` — tax row
- `.cart-total-row` / `.cart-total-label` / `.cart-total-amt` — total row
- `.tile-sold-out` — dims tile image when all variants OOS
- `.tile-oos-label` — "Sold out" text under tile title
- `.cart-badge.pop` + `@keyframes badge-pop` — badge scale animation on count increase

**Watch out for:**
- `CART_F` is now multi-line; any future mutation that adds/modifies `CART_F` fields must account for `note`, `discountCodes`, and the expanded `cost` block.
- The cart footer elements (`#cartNote`, `#discountApply`, etc.) exist in static HTML and are always present — their event listeners are wired once at script load time, not rebuilt on re-render.
- `quantityAvailable` returns `null` when Shopify inventory tracking is disabled for a product; `updateAtcState` checks `qty != null` before showing the low-stock label.

---

### 2026-06-26 — Department filter expanded to all 71 products (shop.html)

**What changed:** `TITLE_DEPT` updated to map all 71 Shopify products to departments. `DEPT_ORDER` cleaned up. Products query bumped from `first: 50` to `first: 100`.

**New departments added:** Writing, Construction, Hair & Makeup, Special Effects (On-Set), VFX (On-Set), Locations, Casting, Craft Services & Catering — plus expanded Production, Direction, G&E, Art Department, Talent, Medical & Safety, Publicity entries.

**Key decisions:**
- Costume Tee → Art Department (user preference)
- Dialogue Coach Tee → Direction (user preference)
- Executive Tee + Executive Producer Tee → both Production
- Girp Tee → G&E (intentional typo product)
- `first: 50` was silently dropping the 21 newest products from the grid — fixed to `first: 100`
- Removed 'Costume' and 'Transport' from DEPT_ORDER (no standalone Costume dept; no Transport products)

---

### 2026-06-26 — Grouped layout with sidebar nav + daily featured rotation (shop.html)

**What changed:** Full layout redesign of the shop page. Replaced flat grid + dropdown filters with department-sectioned layout and sticky sidebar jump nav.

**Features:**
- **Featured strip** — 4 tiles at the top above all departments. Rotates daily from a hardcoded pool of 5 (`FEATURED_POOL`). Uses deterministic day-seeded shuffle (`Math.sin`) so all visitors see the same 4 on a given day. Note: user requested "group of 6" but only specified 5 — add a 6th to `FEATURED_POOL` when ready.
- **Grouped by department** — each department gets a header (`dept-header-name` + count) and its own grid. Ordered by `DEPT_ORDER`.
- **Sticky sidebar** — 158px wide on desktop, `position: sticky; top: 52px; height: calc(100vh - 52px)`. Links smooth-scroll to sections. Active link tracks scroll via `updateActiveSidebarLink()` on the scroll event (checks `getBoundingClientRect().top <= 80`).
- **Mobile sidebar → horizontal strip** — at ≤768px, sidebar becomes a `display: flex; overflow-x: auto` sticky horizontal bar below the nav, with `border-bottom` active indicator instead of `border-left`.

**Removed:** Sort dropdowns (A→Z, Z→A), department dropdown filter, `setSort`, `toggleSort`, `sortedProducts`, `setDept`, `toggleDept`, `buildDeptMenu`, `filteredProducts`, `renderGrid`. The `document.addEventListener('click')` dropdown-close listener was also removed.

**New functions:** `getDailyFeatured()`, `tileHTML(p)`, `renderShop()`, `updateActiveSidebarLink()`

**Watch out for:**
- `tileHTML()` is now the single source of truth for product tile markup — used for both featured and department grids.
- Sidebar links are rebuilt every time `renderShop()` runs. The click listener is re-added each time via `sidebar.innerHTML = ...` + `sidebar.addEventListener`. This is safe because `renderShop` only runs once on init.
- `FEATURED_POOL` titles must match Shopify product titles exactly (case-sensitive). If a title isn't found in `products`, it's silently skipped by `.filter(Boolean)`.
