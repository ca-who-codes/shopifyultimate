---
name: shopify-theme-ops
description: Field-tested techniques for building and running a Shopify store programmatically — theme edits, catalogue/product ops, collections, menus, pages, and branding — especially the Horizon theme. Use when the user wants to edit theme files (templates/index.json, sections, assets, layout/theme.liquid, config/settings_data.json), change fonts/colors/logo/favicon or header layout, fix collection or product layout, build custom-liquid sections, push CSS/JS assets, reorder product media, set collection images, build cart free-gift / progress-bar or auto-rotating-gallery logic, bulk-create/update products, get images onto products, fix "Sold out" drop-ship variants, create smart collections, rewrite menus/pages, or replicate a reference site's design — working through the Shopify CLI or an authenticated Admin GraphQL connection (MCP, private-app token) rather than the theme editor UI. Covers CLI theme push, byte-exact writes via staged uploads, reading oversized theme files, the settings_data.json silent-revert trap, forcing unsupported fonts, Horizon's grid/section model, aliased batch mutations, the DRAFT/publish and inventory-policy traps, live browser verification, and cart-gift race conditions.
license: MIT
metadata:
  version: 1.1.0
  author: ca-who-codes
---

# Shopify Theme Ops

You edit Shopify themes and store data **programmatically** through the Admin GraphQL API — not the theme editor UI. This skill is the playbook for doing that reliably, distilled from real production work on the **Horizon** theme.

Assumes either the **Shopify CLI is authenticated for the store** (simplest) or an authenticated Admin GraphQL connection is available (an MCP server, a private-app token + `curl`). Whenever this skill shows `graphql_query` / `graphql_mutation`, use whatever tool your environment exposes for authenticated Admin GraphQL.

## The one rule that saves you: never hand-type file contents into a tool call

Theme files (a `templates/*.json`, `layout/theme.liquid`, a 25 KB CSS asset) are too big and too fragile to retype or paste into a mutation. A single dropped `!important` or brace silently breaks the storefront. **Move bytes, not text.**

**Two ways to move bytes. Prefer the CLI.**

- **Shopify CLI (simplest — use it if it's authed):** keep a local mirror, edit files on disk, `shopify theme push --only <file> --theme <id> --nodelete [--allow-live]`. It does the staged-upload dance for you, fails *loudly* on bad JSON, and needs no policy juggling. Full flags, the pull-fresh-before-edit rule, and draft-vs-live in `references/cli-theme-workflow.md`.
- **Manual staged upload (Admin GraphQL only, no CLI):** `references/byte-exact-file-pushes.md`. The loop for **every** theme-file write this way:
  1. Build the new file **on disk** with a script (Python `json.dumps` guarantees valid JSON; a string-replace on the fetched original guarantees byte-fidelity of everything you didn't touch).
  2. `stagedUploadsCreate` → get a Google Cloud Storage upload target.
  3. `curl` the file **bytes** to that target (validate the returned policy in Python first — hand-copied base64 causes 400/403).
  4. `themeFilesUpsert` with `body: { type: URL, value: <resourceUrl> }` — Shopify fetches the bytes.
  5. **Verify**: re-fetch the file and compare `checksumMd5` (or size, or a landmark substring). `themeFilesUpsert` returning no `userErrors` does **not** guarantee the change applied.

Either way: build JSON with a script, never by hand, and **verify by looking at the live page** (`references/live-verification-and-gotchas.md`), not just at the tool's return value.

## Reading theme files

- Small files (< ~6 KB) come back inline from a `theme.files(filenames:[...])` query.
- Large files (`assets/base.css` can be 100 KB+) blow the tool output budget. **Bundle** the file you need with a known-large file (e.g. `assets/base.css`) in one query so the whole response persists to disk, then extract the one you want with a script. See `references/reading-large-theme-files.md`.
- Always request `checksumMd5` and `size` alongside `body` when you plan to verify a write.

## The silent-revert trap (config/settings_data.json)

`settings_data.json` is **validated atomically and re-serialized** on save. If any value is invalid — most commonly a **font handle Shopify's library doesn't have** — Shopify keeps the previous valid version and `themeFilesUpsert` still returns **no `userErrors`**. It looks like it worked; it didn't.

- Always **re-fetch and diff** after writing `settings_data.json`.
- Isolate a suspected-bad field by changing only that one field and re-fetching.
- Preserve `current.blocks` (theme app-embed blocks) — never drop them when rewriting the file.

Details + the font list reality in `references/fonts-and-settings-data.md`.

## Forcing a font Shopify doesn't offer

Shopify's font picker is a **curated subset** of Google Fonts. Baloo 2, Marcellus, and Mulish (among others) are **not** in it — setting their handle silently reverts (see above). To use *any* font anyway, **override the theme's CSS font variables** instead of the font setting:

1. Load the font yourself (Google Fonts `<link>`, or self-hosted `@font-face`).
2. In `layout/theme.liquid`, right **after** the theme's own font/variable renders (in Horizon: after `{%- render 'theme-styles-variables' -%}` and `{%- render 'color-palette' -%}`), inject a `<style>` that overrides the family variables with `!important`:

```css
:root, body {
  --font-heading--family: 'YourFont', Georgia, serif !important;
  --font-h1--family: 'YourFont', Georgia, serif !important; /* ...h2..h6 */
  --font-body--family: 'YourBody', system-ui, sans-serif !important;
  --font-paragraph--family: 'YourBody', system-ui, sans-serif !important;
}
```

Put it in `<head>` (not the footer) to avoid a flash of the old font. `references/fonts-and-settings-data.md` lists every variable Horizon actually reads.

## Horizon theme architecture (what's different from Dawn)

Horizon is Shopify's theme-blocks flagship. Key facts you need before editing it:

- **Colors**: a single `color_palette` object (`background`, `foreground`, `color1`…), *not* Dawn's `color_schemes`. Section/settings reference `{{ settings.color_palette.foreground }}`.
- **Templates** (`templates/*.json`) list **sections**; sections contain **blocks**; many blocks live in `blocks/*.liquid` as web components.
- **Bespoke HTML**: the reliable escape hatch is a **`custom-liquid` section** (its `custom_liquid` setting). Use it for hero banners, trust rows, category grids, FAQ accordions, chip rows — anything the block system won't do cleanly.
- **Global CSS/JS**: load site-wide assets by adding `<link>`/`<script src="{{ 'file' | asset_url }}">` to a `custom-liquid` section inside `sections/footer-group.json` (renders on every page), or to `layout/theme.liquid`.
- **Fonts**: read from CSS vars `--font-h1--family … --font-h6--family`, `--font-body--family`, `--font-paragraph--family` (plus `--font-heading--family` etc. referenced by block `font` settings).
- **Collection grid**: `sections/main-collection.liquid` + `snippets/product-grid.liquid`. The grid sits in named grid-line columns (`--centered` = `column-1 / span 12`, `--full-width` = `column-0 / span 14`). If a collection grid renders **right-shifted with an empty left gutter**, set the `main-collection` block's `product_grid_width` to `full-width` (was `centered`).
- **Product cards**: `<product-card class="product-card">` → `a.product-card__link` → `.product-card__content` → blocks; title at `[ref="productTitleLink"] [role="heading"]`; price in `<product-price>`.
- **Cart drawer**: `<cart-drawer>` / `.cart-drawer__content`; AJAX cart via `/cart/*.js`; re-renders its lines on change.

Full map in `references/horizon-architecture.md`.

## Store data (not theme files)

These go through normal Admin GraphQL mutations — no staged uploads needed:

- **Collection image** (fixes category tiles that all show the same fallback): `collectionUpdate(input: {id, image: {src: "<existing CDN url>", altText}})`. `collection.featured_image` returns `collection.image` if set, else the first product's image.
- **Collection description as chips**: `collectionUpdate(input: {id, descriptionHtml})`. Classes and inline `style` attributes **survive** Shopify's sanitizer; put shared CSS in a global stylesheet and use classes. Verify by re-fetching `descriptionHtml`.
- **Reorder product media** (fixes a bad image showing on a card / on hover): `productReorderMedia(id, moves: [{id, newPosition}])` — async, returns a job; re-query `media` to confirm. Non-destructive and reversible (unlike `productDeleteMedia`, which is destructive — get explicit confirmation first).

Copy-paste mutations in `references/admin-graphql-cookbook.md`.

## Catalogue & products (building the store, not the theme)

Building/fixing the actual products — `references/catalogue-and-products.md`. The traps that bite every time:

- **Getting images onto products**: there is no "upload image" call. `create-product`/`productUpdate` take `images: [{url}]` where `url` is a **public HTTPS URL Shopify fetches**. For bulk, host the images in a dedicated **public** repo and pass `raw.githubusercontent.com/...` urls. To ingest one external image into Files: `fileCreate(files:[{originalSource, contentType: IMAGE}])`, then **poll `fileStatus` until `READY`** (the CDN url is null before that).
- **"Sold out" on drop-ship**: tracked inventory + 0 qty renders "Sold out" and blocks purchase. Set every variant `inventoryPolicy: CONTINUE`.
- **DRAFT products are invisible** on the storefront *and* theme previews — must be `ACTIVE` + published to Online Store.
- **Batch with aliases**: one `graphql_mutation` with `p0:`, `p1:`, … + variables updates 17 products in a round-trip. Don't loop one call per product.
- **`descriptionHtml` is wholesale-replace** (no append): fetch → transform in a script → resend.
- Arg names differ: `productUpdate(product: …)` (new API), `collectionUpdate(input: …)`.

## Collections, menus, pages

`references/collections-menus-pages.md`:

- **Smart collections** (auto-populating, e.g. "Gifts Under ₹5K"): `collectionCreate(input:{ruleSet:{appliedDisjunctively, rules:[{column: VARIANT_PRICE, relation: LESS_THAN, condition:"5000"}]}})`. `condition` is always a string.
- **Publish, or it stays invisible**: `collectionCreate` doesn't publish to Online Store. `publishablePublish(id, input:[{publicationId}])` with the store's Online Store publication id (query `publications` — it is **not** `/1`).
- **Menus**: `menuUpdate(id, title, items:[{title, type: HTTP, url:"/collections/x"}])` replaces the whole item list; `menuDelete` for leftover duplicate menus. `type: HTTP` + relative url is the always-works internal link.
- **Pages**: new Page API — `pageUpdate(id, page:{title, body})`, field is **`body`** not `bodyHtml`. Watch for stale demo/old-brand copy in About/FAQ/Care/Shipping pages when rebranding.

## Branding: logo, favicon, header — and the manual-only list

`references/branding-logo-and-header.md`:

- **Logo/favicon** are `settings_data.json` keys (`logo`, `favicon`, `logo_height`) set to `shopify://shop_images/<file>`. Get the image into Files first (`fileCreate`). No logo asset? **Render one from the display font with PIL** (supersampled mark + wordmark).
- **Header layout** is `header-group.json` settings (`logo_position`, `menu_position`, `menu_row`, `search_position`, `enable_transparent_header_home`) — turning transparent-on-home **off** fixes washed-out nav over light sections.
- **No API for**: store name (Settings → Store details), custom domain (registrar DNS: A `@`→`23.227.38.65`, CNAME `www`→`shops.myshopify.com`), the password/"Opening soon" gate (a Preferences system page; `templates/password.json` auto-regenerates — don't theme it). Surface these as manual steps. A plain "Opening soon" page the client calls "bad design" is the **password gate, not the store**.

## Verify by looking at the live page

`references/live-verification-and-gotchas.md` — the most important habit for anything visual. **WebFetch can't render CSS/fonts/JS; use a browser.** Inspect the live DOM to get exact class names **before** writing CSS (Horizon's classes vary by version — includes the `.resource-list--grid` collapse-to-1-column trap and its fix). Know the automation lies: Lenis breaks `scrollTo`, screenshots catch lazy-loaded images mid-render (`img.complete && naturalWidth>0` before calling an image broken), automation tabs report `document.hidden` (pauses rotators/autoplay). To **replicate a reference site**, extract its computed styles (fonts, colors, radii) and rebuild to spec — don't design from taste.

## Recurring front-end builds

Two builds show up repeatedly and each has a right way:

- **Auto-rotating product galleries** — cross-fade every product image on a card, dependency-free, images from `/products/{handle}.js`, **progressively loaded** (data-src, load-on-show) so a grid doesn't download 80 images. `references/auto-rotating-gallery.md`.
- **Cart free-gift / threshold progress bars** — below.

## Cart free-gift / threshold progress bars

A recurring build — and a recurring source of a serious bug: **customers checking out a $0 gift-only cart**. If you build or touch one, read `references/cart-free-gift-bar.md`. The three non-negotiables:

1. **Sticky reconcile** — never let a drawer re-render (which fires a "just repaint" event) clobber the pending "remove the now-unqualified gift" job. Coalesced debounce calls must OR the reconcile intent.
2. **Always-remove** unqualified gifts on every cart refresh (adding a gift can be gated to intentional changes; removing an unqualified one cannot).
3. **Checkout guard** — disable the checkout **and** express-checkout buttons whenever the cart is gifts-only (qualifying subtotal ≤ 0), as a belt-and-suspenders against races and JS failures.

## Working style

- **Verify every write** by re-fetching. "No `userErrors`" is necessary, not sufficient.
- **Draft themes only** unless told otherwise — never write the live/published theme without explicit permission.
- Build JSON with a script, never by hand. Prefer **single-quoted HTML attributes** inside custom-liquid values to dodge escaping.
- Mirror what you push into a repo folder so there's a source of truth off the theme.
- No preview? Say so, and design defensively (contained/clipped layouts, fallbacks) so a blind change can't look broken.

## Reference index

**Moving bytes / theme files**
- `references/cli-theme-workflow.md` — the Shopify CLI push/pull loop (the simple byte-mover); pull-fresh-before-edit; draft vs live.
- `references/byte-exact-file-pushes.md` — the manual staged-upload write loop (Admin GraphQL only), with a policy-validating uploader.
- `references/reading-large-theme-files.md` — reading files past the tool budget.
- `references/fonts-and-settings-data.md` — the font-library reality + the CSS-variable override + settings_data pitfalls.
- `references/horizon-architecture.md` — Horizon's sections/blocks/colors/grid/cart map.

**Store data & catalogue**
- `references/catalogue-and-products.md` — images onto products, `fileCreate`, the Sold-out/inventory-policy fix, DRAFT visibility, aliased batch mutations.
- `references/collections-menus-pages.md` — smart collections + publishing, menus, pages.
- `references/admin-graphql-cookbook.md` — copy-paste queries & mutations.

**Branding, verification, builds**
- `references/branding-logo-and-header.md` — logo/favicon, header layout, rendering a logo from a font, the manual-only list (store name, domain, password gate).
- `references/live-verification-and-gotchas.md` — browser verification, live-DOM-first CSS, the grid-collapse trap, automation gotchas, replicating a reference site.
- `references/auto-rotating-gallery.md` — progressive, dependency-free product-card image rotation.
- `references/cart-free-gift-bar.md` — a correct, race-free gift-bar pattern.
