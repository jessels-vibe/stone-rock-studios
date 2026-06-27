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

### (Pre-existing) — Admin Dashboard (admin.html)

**What it is:** A full Firebase-backed admin panel at `/admin.html` for managing the Stone Rock Studios video portfolio. Not built in this Claude session — documented here retroactively for future context.

**Features:**
- **Firebase auth** — email/password login (`admin@stonerockstudios.xyz`). Auth state gates the entire admin UI.
- **Firebase Realtime Database** — all video data lives at `db.ref('videos')`. Saved as a flat object keyed by `videoId`. Index.html reads this same ref to render the public site.
- **Firebase Storage** — custom thumbnail uploads stored at `thumbnails/{videoId}`.
- **YouTube sync** — pulls from 16 hardcoded playlist IDs (2 per category × 8 categories) via YouTube Data API v3. Only adds new videos not already in the DB. Fetches title, thumbnail, duration, and aspect ratio.
- **Drag-and-drop reordering** — supports multi-select group drag. Drop target uses mouse Y midpoint to determine insert before/after. Order is written as `v.order = i` on save.
- **Per-video fields:** `videoId`, `title`, `customTitle` (override display name), `role`, `client`, `externalLink`, `thumbnail`, `aspectRatio`, `portrait`, `category`, `tags[]`, `featured`, `visible`, `order`, `duration`, `addedAt`.
- **Category sidebar** — Moto, Food & Drink, Lifestyle, Documentary, Event Coverage, Real Estate, Social Reels, Music Videos. Each maps to one or more YouTube playlist IDs in `PLAYLISTS`.
- **Filter views** — Featured, Hidden, Portrait/Vertical, Duplicates (sidebar).
- **Stats bar** — Total / Featured / Visible / Hidden / Vertical counts.
- **Search** — filters by title or videoId in real time.
- **Bulk actions** — select multiple rows; bulk feature/unfeature, show/hide, add tag, delete.
- **Hover thumbnail preview** — 300ms delay popup follows cursor, shows title, videoId, ratio, tags.
- **Thumbnail tools** — ↺ refresh from YouTube API, ↑ upload custom image to Firebase Storage.
- **Aspect ratio selector** — 16:9, 9:16, 1:1, 4:3, 4:5, 21:9. Updates `portrait` bool used by index.html for Masonry layout.
- **Tag chips** — multi-category per video via checkbox dropdown. `tags[0]` becomes `category`.
- **Featured / Visible toggles** — per-row pill buttons. Hidden rows dim to 40% opacity in the list.
- **Duplicate detection** — flags videos with the same `videoId` appearing more than once. Yellow left-border on row. One-click "Remove Dupes" keeps first occurrence.
- **Add video modal** — paste any YouTube URL or bare video ID, fetches preview via API, set category/ratio/featured before adding.
- **External link field** — auto-detects platform (YouTube, Instagram, TikTok) and shows badge.
- **Save & Publish** — writes full `videos` object to Firebase. Unsaved changes tracked by `dirty` flag with yellow save bar. Discard reloads from DB.
- **Cmd+S** keyboard shortcut to save.

**Firebase services (v10.12.0 compat SDK):** `firebase-auth`, `firebase-database`, `firebase-storage`.

**Watch out for:**
- The `PLAYLISTS` array and `YT_API_KEY` are hardcoded in the script block. If playlists change or the API key rotates, update them there.
- `isPortrait()` heuristic: uses thumbnail dimensions if available, falls back to duration ≤ 180s as a proxy for Shorts. Can misclassify long-form vertical videos.
- The public site (`index.html`) reads `db.ref('videos')` and filters by `v.visible === true`, orders by `v.order`. Any field the public site uses must be saved to Firebase — it does not use `customTitle`, `role`, `client`, or `externalLink` unless index.html was updated to do so.
- Firebase Realtime Database rules are not documented here — verify they restrict write access to authenticated users only.
- `admin.html` is publicly accessible at the URL — it just requires login. There is no `.htaccess` or route guard hiding it.

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

### 2026-06-27 — Promo announcement bar (shop.html)

**What changed:** Added a white announcement bar above the nav advertising two active Shopify automatic discounts.

**Deals shown:** "Buy 3, Get 1 Free" and "Free Shipping on Orders Over $100" — these are Shopify automatic discounts that apply at checkout and can't be surfaced dynamically from the Storefront API, so they're hardcoded in the bar.

**Layout changes to accommodate the bar:**
- Wrapped `<nav>` in `<div class="site-header">` — the wrapper is `position: fixed; top: 0` instead of the nav itself
- `.dept-sidebar` sticky top updated from `52px` → `88px`; height from `calc(100vh - 52px)` → `calc(100vh - 88px)`
- Hero `padding-top` updated from `120px` → `156px` (desktop) and `90px` → `126px` (mobile)
- Promo bar is `36px` tall; nav is `52px` tall; total fixed header = `88px`

**To update the deals:** Edit the two `.promo-deal` spans inside `<div class="promo-banner">` in shop.html. To remove entirely, delete the `.promo-banner` div — no offset adjustments needed since it's inline content, not in the fixed header.

---

### 2026-06-27 — Coming Soon page + nav cleanup (index.html, shop.html)

**What changed:** Replaced the full portfolio page with a minimal "Coming Soon" page. Removed the "Work" nav link from shop.html.

**index.html** is now a single-screen coming soon page: Stone Rock Studios wordmark, "Coming Soon" headline, "Something is in the works." sub, and a subtle "Visit the Shop" link to shop.html. All portfolio JS, Masonry, YouTube API, and Firebase code has been removed from index.html.

**shop.html nav:** Removed `<a class="nav-link" href="index.html">Work</a>` since the portfolio is not public. The nav now only shows Shop, Bag, and Get in Touch.

**Why:** Portfolio is private while in production. Shop is the public-facing presence.

**Watch out for:** The full portfolio code (Masonry, YouTube playlists, Firebase integration) is gone from index.html — it lives in git history if needed. When the portfolio goes live, it will need to be rebuilt or restored from a prior commit.

---

### 2026-06-27 — Search bar (shop.html)

**What changed:** Added a real-time product search bar at the top of the shop main content area, above the Best Selling strip.

**How it works:**
- Input sits in `.shop-search-wrap` above the normal layout. The normal layout (`#shopNormal`) and search results (`#searchResults`) are toggled via `display` — one is always hidden.
- `searchShop(q)` filters the in-memory `products` array by `title.toLowerCase().includes(q)`. Matched products render into `#searchGrid` using the existing `tileHTML()` function (no `mostPopular` flag).
- Results label shows count: "N results for 'query'" or "No results for 'query'".
- Clearing the input (or erasing to empty string) restores the normal Best Selling + department layout.

**New elements:** `.shop-search-wrap`, `.shop-search`, `#shopSearch`, `#searchResults`, `#searchResultsLabel`, `#searchGrid`, `#shopNormal` (wrapper div around the normal layout).

**Watch out for:** `#featuredWrap` and `#shopSections` are now children of `#shopNormal`. Any code that references them by ID still works, but the toggle hides/shows the parent `#shopNormal`, not each child individually.

---

### 2026-06-26 — Fix "Most Popular" label appearing on all non-first dept tiles (shop.html)

**What changed:** One-line fix in `renderShop()` — changed `byDept[dept].map(tileHTML)` to `byDept[dept].map(p => tileHTML(p))`.

**Why:** `Array.prototype.map` passes `(element, index, array)` to its callback. When `tileHTML` was passed directly as the callback, the array index became the `mostPopular` argument — index 0 is falsy (no label), but index 1, 2, 3… are truthy, so every non-first shirt in every department incorrectly showed "Most Popular". Wrapping with an arrow function ensures only the explicit `i === 0` check in the featured strip produces the label.

**Watch out for:** Any future call to `.map(tileHTML)` will reproduce this bug. Always use `.map(p => tileHTML(p))` or `.map((p, i) => tileHTML(p, i === 0))` depending on intent.

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
