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
- **After every commit, immediately push to `origin main` without waiting to be asked.** Changes that aren't pushed don't appear on the live GitHub Pages site (stonerockstudios.xyz).
- After every code change, append an entry to the Change Log below covering what changed, why, and anything to watch out for.

---

## Change Log

---

### 2026-08-05 — Shareable product links (shop.html)

**What changed:** Products in the shop are now shareable via URL.

**How it works:**
- When a product modal opens, the browser URL silently updates to `?product=Director+Tee` (using `history.replaceState`). No page reload — just the address bar changes.
- When the modal closes, the param is removed from the URL.
- A "Copy shareable link" button at the bottom of the modal copies the current URL to clipboard and shows "Link copied!" for 2 seconds.
- On page load, if `?product=` is in the URL, the matching product modal opens automatically. If `?q=` is in the URL (search query), it pre-fills the search bar and runs the search.

**Share flow:** Open any shirt → hit "Copy shareable link" → send the URL → recipient lands with that exact shirt's modal already open.

**Watch out for:**
- Product lookup on load uses an exact title match (`p.title === productParam`). If a product title changes in Shopify, existing shared links for that product will silently fail (no modal opens, no error shown).
- `?q=` fallback does a substring search — useful for constructing search links manually but less precise than `?product=`.

---

### 2026-07-28 — Traffic Analytics panel (admin.html)

**What changed:** Added a "Traffic" section to the admin sidebar that shows a full GA4 analytics dashboard — no separate tab needed.

**Features:**
- Sessions by day bar chart (spikes auto-highlighted white at >2.5× daily average)
- Traffic sources table: channel group + source/medium breakdown with percentage share bars
- Landing pages table: top 25 landing pages by session count, with page name and user counts
- Summary stats: Sessions, Users, New Users, Returning, Engaged Sessions
- Date range toggle: 7d / 30d / 90d
- Refresh button + timestamp

**Setup (one-time):** Requires a GA4 Property ID (numeric, from GA4 Admin → Property details) and an OAuth 2.0 Client ID from Google Cloud Console (APIs & Services → Credentials, Web Application type, with `https://stonerockstudios.xyz` as an authorized JavaScript origin). Credentials are saved to `localStorage` — only needs to be entered once per browser. Uses Google Identity Services for the OAuth popup; the GA4 Data API token lasts ~1 hour (reconnect button appears when expired).

**Watch out for:**
- The OAuth client must have the Google Analytics Data API enabled in Google Cloud Console (APIs & Services → Library → "Google Analytics Data API").
- Token is in-memory only (no persistence). Refreshing the admin page requires re-clicking "Connect" (the credentials auto-fill but the token must be re-fetched). This is by design — storing OAuth tokens in localStorage is a security anti-pattern.
- `sessionDefaultChannelGroup` dimension in GA4 uses Google's channel grouping rules (Organic Search, Direct, Referral, etc.) — not custom channel groups.

---

### 2026-07-22 — Sync blocklist + automatic hidden-video ordering (admin.html)

**Sync blocklist:**
When a synced YouTube video is deleted from the admin, its videoId is written to `db.ref('syncBlocklist')` in Firebase. On load, the blocklist is read and merged into the `existing` set that sync checks — so deleted (or manually replaced) videos never get re-added by a future sync. Applies to single delete and bulk delete. Manual entries (`isManual: true` / `ext_*` IDs) are excluded from the blocklist since they don't live in YouTube playlists.

**Automatic hidden-to-bottom ordering:**
Hidden videos now automatically sink below all visible videos whenever visibility changes — single toggle, bulk hide/show, or drag-and-drop. `sinkHiddenToBottom()` does a stable sort: all visible videos first (preserving their relative order), then all hidden (preserving their relative order), then rewrites `order` indices. This means you can never accidentally leave a hidden video sitting above a visible one.

**Watch out for:**
- Blocklist entries in Firebase (`syncBlocklist/`) are permanent — if you ever want a deleted video to be re-synced, you'd need to remove it from that node manually in Firebase console.
- Showing a previously hidden video places it at the end of the visible section (bottom of visible, top of hidden). Drag it up to the desired position after showing.

---

### 2026-07-20 — Unify browser pixel to Shopify CAPI pixel (all pages)

**What changed:** Replaced pixel ID `1706510330484603` ("Stone Rock Website") with `1275243804483057` ("Stone Rock Studios's pixel") in all 5 HTML files: shop.html, index.html, about.html, seed.html, admin.html.

**Why:** The ad campaign was updated to optimize against pixel `1275243804483057` (the Shopify-connected pixel that receives CAPI Purchase events). The old browser pixel (`1706510330484603`) was a separate dataset — PageView and ViewContent events from product browsing were invisible to the campaign. Now all browser events (PageView, ViewContent, Lead) flow into the same pixel as Shopify's CAPI events, giving the campaign full funnel visibility.

**Watch out for:** The old pixel `1706510330484603` ("Stone Rock Website") still exists in Meta Business Manager but is now effectively unused. Do not reference it for any new ad campaigns. All pixel work going forward uses `1275243804483057` only.

---

### 2026-07-18 — Meta Pixel deduplication (shop.html)

**What changed:** Removed `fbq('track', 'AddToCart')` and `fbq('track', 'InitiateCheckout')` browser pixel calls from `shop.html`. Shopify's Facebook & Instagram channel (now set to Maximum data sharing) fires both events server-side via CAPI with hashed customer identity data. The browser-side versions fired without any identity context, causing double-counting and 0.0/10 event match quality on those events. GA4 `add_to_cart` and `begin_checkout` events are unchanged.

**What the browser pixel still fires:** `PageView` (shop page load) and `ViewContent` (product modal open) — Shopify doesn't fire these on the custom domain.

**Watch out for:** If Shopify's Facebook & Instagram channel is ever disconnected or downgraded from Maximum, AddToCart and Purchase would go untracked on the Meta side entirely — there's no browser fallback for those events anymore.

---

### 2026-07-18 — Mobile nav fix; favicon and OG image (index.html, about.html, shop.html, favicon.png, og-image.png)

**What changed:** Three fixes in one commit.

**Mobile nav wrapping** — Nav logo changed from plain text "Stone Rock Studios" to `Stone Rock<br>Studios` with `line-height: 1.25`, giving the preferred 2-line layout. Previously the logo was wrapping to 3 lines because `.nav-links` gap was 40px and "Get in Touch" was present, leaving no horizontal room for the logo text. Mobile override now sets `.nav-links { gap: 16px }` and `.nav-cta { display: none }`.

**Favicon + OG image** — Brand logo PNG copied to `favicon.png` and `og-image.png` in the project root. Both filenames were already referenced in `<head>` across all pages — they just didn't exist. `<link rel="apple-touch-icon">` added to all three pages for iOS home screen icon support.

**Watch out for:**
- "Get in Touch" is hidden on mobile; the contact form is still reachable by navigating to About and scrolling.
- If more nav links are added in future, re-check mobile spacing — at 16px gap with 4 items it fits comfortably, but 5+ might squeeze the logo again.

---

### 2026-07-18 — Fix "Don't see your role?" nudge left padding on mobile (shop.html)

**What changed:** `.suggest-nudge` left padding changed from `var(--gap)` (3px) to `20px`. Same root cause as the featured-header fix — `var(--gap)` is the tile grid gap, not a content margin, so it left the nudge text nearly flush with the shop-main edge on mobile.

**Watch out for:** Any element using `padding: 0 var(--gap)` as content left-padding will have the same misalignment on mobile. Content margins should use `20px` px values, not `var(--gap)`.

---

### 2026-07-14 — Brand guidelines applied (index.html, about.html, shop.html)

**What changed:** Implemented official Stone Rock Studios brand guidelines across all three public-facing pages.

**Typography:**
- Font family changed from `"Helvetica Neue", Helvetica, 'Inter'...` to `'Inter', sans-serif` — Inter is now first priority
- Google Fonts link updated to load only weight 600 (SemiBold); removed 300, 400, 500, 700
- All `font-weight: 700` (bold) and `font-weight: 300` (light) changed to 600 throughout
- Letter-spacing tightened: large positive values (0.1em–0.25em) replaced with -0.01em; headlines use -0.03em to -0.04em

**Colors:**
- Brand palette is strictly black and white (#000000 and #FFFFFF)
- `--black` changed from `#0a0a0a` → `#000000`; `--text` from `#f0f0f0` → `#ffffff`
- All `#fefefe` occurrences replaced with `#ffffff`
- Muted/secondary states use `rgba(255,255,255,0.45)` (white opacity) rather than gray codes
- Borders use `rgba(255,255,255,0.08)`; hover states use proportionally higher opacity

**Casing:**
- Removed all `text-transform: uppercase` from CSS (nav, labels, buttons, categories, sidebar, drawers)
- Fixed hardcoded uppercase HTML nav text: "WORK" → "Work", "ABOUT" → "About", "SHOP" → "Shop"

**Corners:**
- `border-radius: 0` applied to: play button, ext-link badge, cart badge, color swatches, modal dots, suggest inputs/buttons

**Watch out for:**
- If new UI elements are added, do not use `text-transform: uppercase`, rounded corners, or positive letter-spacing — brand says no all caps and all sharp edges
- Font weight should stay at 600 (SemiBold) for all text, including body copy
- Only use #000000 and #ffffff; avoid grays (use white/black opacity for hierarchy)

---

### 2026-07-07 — Manual video entry + Hulu support (admin.html, index.html)

**What changed:** Added support for portfolio entries that live on external platforms (Hulu, Vimeo, etc.) rather than YouTube.

**Admin (`admin.html`):**
- Add Video modal now has a YouTube / Manual Entry tab toggle.
- Manual Entry form: title, thumbnail upload (to Firebase Storage at `thumbnails/ext_<timestamp>`), external link, category, aspect ratio, featured toggle.
- Manual entries stored with `isManual: true` and `videoId: ext_<timestamp>`.
- Hulu added to `detectPlatformLabel()`.
- `refreshThumbnail()` guards against calling YouTube API on manual entries (`v.isManual` early return).

**Index (`index.html`):**
- `openModal()` refactored to accept a full video object instead of individual params `(ytId, cat, title, ratio)`. All callers updated.
- Manual entries open a modal showing the thumbnail fullscreen with a darkening overlay and a centered "Watch on [Platform]" CTA button. YouTube entries unchanged.
- `closeModal()` now also resets ext content visibility and iframe display style.
- Hulu added to `platformIcon()` (green rounded-square H icon) and new `platformName()` helper added.
- Modal HTML: `#modalExtContent`, `#modalExtThumb`, `#modalExtOverlay`, `#modalExtBtn` added inside `.modal-video`.

**Watch out for:**
- The Hulu badge click goes directly to the external URL (existing `stopPropagation` on `.tile-ext-link` handles this). The thumbnail click opens the in-site modal — badge is the only exit point.
- `isManual` must be `true` (not just truthy) for the ext modal to show — entries synced from YouTube will never have this field set.
- `ext_<timestamp>` IDs are unique by construction but will appear as `ext_1234567890` in the admin video ID column — expected.

---

### 2026-07-06 — Remove Buy 3 Get 1 Free from promo banner (shop.html)

**What changed:** Removed "Buy 3, Get 1 Free" pill and its divider from the promo banner. Updated sub-copy from "Buy 3 get 1 free applied automatically at checkout" to "Free shipping applied automatically at checkout." Discount deactivated in Shopify.

---

### 2026-07-06 — 100% cotton label in product modal (shop.html)

**What changed:** Added a static "100% cotton" line below the price in the product modal. Styled as small uppercase muted text (`#888`, 10px, 0.12em tracking). Same for all products — hardcoded in HTML, not pulled from Shopify.

---

### 2026-07-06 — Sort size options in modal (shop.html)

**What changed:** Size pills in the product modal now render in XS→S→M→L→XL→2XL→3XL→4XL→5XL→6XL order. Previously Shopify returned variants in creation order, which had S appearing last.

**How:** Added `SIZE_ORDER` constant. In `renderOpts()`, when the option name matches `/size/i`, values are sorted against `SIZE_ORDER` before mapping to pills. Any size not in the list falls to the end.

---

### 2026-07-04 — Email field added to shirt suggestion forms (shop.html)

**What changed:** Both suggestion forms now include an optional email input so visitors can be notified when their requested shirt is added.

- **Bottom `#suggestSection` form** — email input stacked below the role input, above the submit button. Sub-copy updated: "Drop your role below — leave your email and we'll notify you when it's added." Success message updated to "Got it — we'll notify you when it's added."
- **Zero-results `#zeroSuggestForm`** — same treatment. Sub-copy: "Drop your role and email below — we'll notify you when it's added." Success message updated to match.

**Why:** Email was optional (not `required`) to avoid blocking users who just want to suggest without getting notified.

**How email arrives:** Formspree sends both `role` and `email` fields in the submission email to the inbox tied to endpoint `mwvdoqdb`. No JS changes needed — both handlers use `new FormData(e.target)` which automatically picks up any `name` attributes.

**Watch out for:** Formspree free tier has a 50 submissions/month limit. If volume grows, upgrade the plan or switch to a paid endpoint.

---

### 2026-07-01 — Cart error handling, stale cart revalidation, tags filter fix

**What changed:** Three silent bugs patched.

- **Cart buttons frozen on network error** — remove, qty +/−, and variant change handlers in the cart drawer had no try/catch. If a Shopify API call failed mid-flight, the button would stay `disabled` forever. All three now catch errors and re-enable the button so the user can retry.
- **Stale cart from localStorage** — `loadCartFromStorage` now calls `cartFetch()` (new function) to revalidate the stored cart against Shopify immediately on page load. Shows cached data instantly, then overwrites with fresh data. If the cart is expired on Shopify's end, localStorage is cleared.
- **Category filter dropping videos on index.html** — `v.tags || [v.category]` treated an empty array `[]` as truthy, so videos with `tags: []` were silently excluded from department filters. Fixed to `v.tags?.length ? v.tags : [v.category]`.

**Watch out for:** `loadCartFromStorage` is now `async`. The init IIFE doesn't await it (fire-and-forget) — the initial render happens immediately from cache, the Shopify refresh happens in background. This is intentional.

---

### 2026-07-01 — Meta Pixel added to seed.html and admin.html

**What changed:** Meta Pixel snippet (ID `1706510330484603`) added to `seed.html` and `admin.html` inside `<head>`, matching the placement on the other three pages.

**Why:** Diagnostic check revealed the Pixel was missing from both internal pages. All 5 pages now have GA4 + Meta Pixel firing in `<head>` on every page load.

**Watch out for:** If any new page is ever added, it needs both the GA4 snippet and the Meta Pixel snippet manually — there is no shared layout to inject from.

After every edit, append an entry here so future Claude instances understand what was built, why, and what to watch out for.

---

### 2026-07-01 — Fan Favorites alignment + search fix (shop.html)

**What changed:** Two bug fixes to reduce friction for visitors.

- **Fan Favorites alignment** — `.featured-header` left padding changed from `var(--gap)` (3px) to `20px`. On mobile, the sidebar collapses to a horizontal strip and the shop-main takes full width, so 3px padding made "Fan Favorites" / "The ones your crew keeps reaching for." start 17px to the left of the nav logo and hero text (both at 20px). Added a mobile override to also reduce top padding slightly. On desktop, this also aligns the featured header with the promo banner (which already used 20px internal padding).
- **Search bar returning no results** — `searchShop()` was setting `resultsEl.style.display = ''` to "show" the results container, but `#searchResults` has `display: none` in CSS. Clearing the inline style caused it to revert to the CSS rule, hiding all results. Fixed by setting to `'block'` explicitly.

**Watch out for:**
- If `var(--gap)` is ever used for featured-header padding again, the mobile alignment will break — horizontal padding on featured-header should stay in px, not use the tile-gap variable.
- The search bug pattern (setting `display = ''` to show an element that has `display: none` in CSS) could affect any similar toggle elsewhere. Always set to an explicit value ('block', 'flex', etc.) rather than clearing.

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

### 2026-06-28 — Hero size, mobile shop, about page content, redundant label removed

**shop.html:**
- Hero title shrunk from `clamp(48px,7vw,88px)` to `clamp(32px,5vw,56px)` — gets users to products faster
- Hero padding reduced: `120px 36px 48px` → `80px 36px 32px`
- Removed redundant "BEST SELLING" label above "Fan Favorites" heading
- Mobile: tighter hero padding, smaller promo/crew callout text, `font-size:16px` on search input (prevents iOS auto-zoom on focus)

**about.html:**
- Added "Shop the Collection →" CTA button below the bio
- Added Formspree contact form (same endpoint as index.html: `mwvdoqdb`) — Name, Email, Message fields
- Updated `--muted` to `#aaa` to match shop page
- Mobile: `font-size:16px` on inputs to prevent iOS zoom

---

### 2026-06-28 — Best Selling section + crew-buy callout + tile prices (shop.html)

**What changed:** Three conversion-focused updates to push the shop harder.

- **Best Selling section** — upgraded from a small label to a full header: "Fan Favorites" bold heading + "The ones your crew keeps reaching for." sub. Section now has `background: var(--surface)` and a bottom border to visually separate it from departments.
- **Crew-buy callout** — white strip between Best Selling and the department sections: "Outfitting your whole crew? Buy 3 tees, get 1 free — applied automatically at checkout."
- **Prices on tiles** — `.tile-price` was `display:none`, now shows the product's `minVariantPrice` on every tile. Removes a click of friction for price-sensitive browsers.

**Why:** 94 sessions, 0 add-to-carts. Diagnosis: decision paralysis from 71 options + no clear price anchor + crew-buy deal buried in the banner.

---

### 2026-06-28 — About page copy update (about.html)

**What changed:** Replaced the two-paragraph studio bio with a single line: "The funniest production studio in the world."

---

### 2026-06-30 — Department order, pricing, promo banner, checkout friction (shop.html)

**What changed:** Four updates in one session.

**Department order rewritten** — `DEPT_ORDER` reordered by on-set hierarchy (most to least bossy): Direction → Production → Talent → Camera → G&E → Sound → Art Department → Stunts → Construction → Hair & Makeup → Writing → Special Effects (On-Set) → VFX (On-Set) → Locations → Medical & Safety → Casting → Craft Services & Catering → Post (On-Set Liaisons) → Publicity. User moved Talent above Camera specifically.

**Price raised to $24.99** — was $19.99. Reason: margin at $19.99 with Tag Stitch fulfillment ($8.99 shirt + ~$6 shipping = $14.99 cost) left only ~$5/sale, making ad spend unprofitable at realistic conversion rates. At $24.99 margin is ~$10/sale.

**Free shipping on all orders** — changed from "Free Shipping on Orders Over $100" to "Free Shipping on All Orders". Shipping cost was baked into the new price. Removes checkout surprise that was likely causing abandonment.

**Promo banner sub-copy updated** — "Applied automatically at checkout — no code needed" → "Buy 3 get 1 free applied automatically at checkout — no code needed" (clarifies what auto-applies).

**Next up — checkout subdomain (not yet done):** Checkout currently sends customers to `stonerockstudios.myshopify.com` (new tab), which is a trust break. Plan is to add `shop.stonerockstudios.xyz` CNAME → `shops.myshopify.com` in GoDaddy, add it as custom domain in Shopify, then replace myshopify.com hostname in `checkoutUrl` before redirecting. Waiting on user to add DNS record and Shopify domain first.

**Watch out for:**
- Tag Stitch fulfillment cost is $8.99 + shipping — do not price below $22 or ad spend becomes unprofitable.
- Free shipping is now promised on the site — make sure Shopify shipping settings reflect free shipping or the checkout will contradict the banner.

---

### 2026-06-28 — Promo banner copy updates (shop.html)

**What changed:** Two copy iterations on the promo banner deals text.
- Added "Tees Starting at $19.99" as a middle deal between Buy 3 Get 1 Free and Free Shipping
- Updated to "All Tees $19.99!" for a cleaner, more direct read

Final banner reads: **Buy 3, Get 1 Free · All Tees $19.99! · Free Shipping on Orders Over $100**

---

### 2026-06-28 — Restore portfolio (index.html, shop.html)

**What changed:** Reverted index.html back to the full portfolio (Firebase + YouTube playlist integration). Restored Work nav link in shop.html alongside Dispatch's About link.

**Why:** Portfolio was temporarily hidden behind a coming soon page — now live again.

---

### 2026-06-30 — GA4 tracking tag added to all pages

**What changed:** Added Google Analytics 4 tag (`G-F0F5ZDTLY3`) immediately after `<head>` on all five HTML pages: `index.html`, `about.html`, `shop.html`, `seed.html`, `admin.html`.

**Why:** Connected GA4 to Meta Ads for Shops performance reporting. Tag fires `PageView` on every page load and feeds data into the `G-F0F5ZDTLY3` property. Required for Meta to track the full journey from ad click → site action.

**Watch out for:** If any new pages are added, the GA4 tag must be added manually — there is no shared layout to inject it from.

---

### 2026-06-29 — Meta Pixel added to index.html and about.html

**What changed:** Added the Meta Pixel snippet (ID `1706510330484603`) to `index.html` and `about.html`.

**Why:** Pixel was only present on `shop.html` — all homepage and about page visits were invisible to Meta Ads Manager. Meta was reporting 0 website events because traffic landing on the homepage never triggered the Pixel.

**Watch out for:** All three pages now fire `PageView` on load. `shop.html` additionally fires `AddToCart` at line ~970. If a 4th page is ever added to the site, it needs the Pixel snippet too — there's no shared layout/template to inject it from.

---

### 2026-06-28 — Klaviyo email capture popup (shop.html)

**What changed:** Added one `<script>` tag to shop.html that loads Klaviyo's onsite JS. This activates the existing "Email & SMS Popup" form (form ID `U9c6JM`) on the shop page.

**Public API Key:** `YvXr4f` (Stone Rock Studios Klaviyo account)

**How it works:** Klaviyo's script handles all popup logic — timing, display rules, form submission. The form is managed entirely in Klaviyo's dashboard (Sign-up forms → Email & SMS Popup → Edit form). No code changes needed to update the popup's design, offer, or targeting rules.

**Watch out for:** The popup is currently set to "All Domains" permissions, so it will fire on stonerockstudios.xyz. If Klaviyo's domain targeting is ever restricted, the popup will stop showing — check Manage Permissions on the API Keys page.

---

### 2026-06-27 — About page + nav links (about.html, index.html, shop.html) — Dispatch

**What changed:** Added a standalone About page and wired it into navigation across the site.

**New file — `about.html`:** Matches the site's nav/typography exactly. Fixed nav with Stone Rock Studios logo, About (active) + Shop links + Get in Touch CTA. Content section: "About" label, "Stone Rock Studios" headline, two short paragraphs ("We're a two-person creative studio…" and "Based in Los Angeles. Available worldwide."). Responsive — collapses padding and headline size at ≤768px.

**`index.html`:** Single "Visit the Shop" link replaced with a `.links-row` flex row containing two links side by side: "About" (→ about.html) and "Visit the Shop" (→ shop.html).

**`shop.html` nav:** Added `<a class="nav-link" href="about.html">About</a>` before the Shop link so About appears in the shop nav.

**Watch out for:** `about.html` has self-contained CSS (no shared stylesheet). If nav styles change elsewhere, about.html needs to be updated manually to stay consistent.

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

### 2026-07-02 — "Don't see your role?" nudge link + zero-results inline form (shop.html)

**What changed:** Two new entry points to the shirt suggestion form.

- **Suggest nudge below search bar** — A small "Don't see your role? Suggest it →" text link always visible below the search input. Clicking smooth-scrolls to `#suggestSection` (the existing bottom form). Fires `suggest_nudge_click` GA4 event. Implemented as a `<span id="suggestNudge">` with a click listener to avoid anchor-scroll jank on mobile.
- **Zero-results inline form** — When search returns 0 matches, `searchShop()` shows `#searchZeroState` below the results grid. Shows "We don't have that one yet." + "Drop your role below and we'll put it on the list." + an inline suggest form (same Formspree endpoint `mwvdoqdb`). The input is pre-filled with the search query (capitalised) so the user sees "gaffer" already in the field. Fires `shirt_suggestion` (with `source: 'search_zero_state'`) + `generate_lead` GA4 on success. Zero state is hidden when query is cleared or results are found.

**Why:** The bottom form was invisible — users had to scroll past all products to find it. The nudge anchors it near discovery and the zero-results state catches high-intent users exactly when they hit a gap.

**Watch out for:**
- `#zeroSuggestForm` and the bottom `#suggestForm` post to the same Formspree endpoint — both land in the same inbox. The hidden `_subject` field value ("Shirt suggestion") is the same; the GA4 `source` param distinguishes where it came from.
- Pre-fill uses `q.charAt(0).toUpperCase() + q.slice(1)` (capitalises first letter only). If the query is multi-word, only the first character is capitalised.
- Zero-state success hides the form but keeps `#zeroSuggestSuccess` showing. If the user clears search and searches something else, the zero state re-appears but `show` is checked so the pre-fill is skipped — success message persists for the session.

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
