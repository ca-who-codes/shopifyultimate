# Live verification & browser-automation gotchas

"No userErrors" and "the push succeeded" tell you nothing about how the page **looks**. For anything visual, load the live URL in a real browser and inspect it. This file is the playbook for doing that, plus the traps that make automated verification lie to you.

## WebFetch is text-only — never trust it for design

WebFetch/curl can't render CSS, load webfonts, or run JS. It cannot verify fonts, colors, layout, spacing, or whether an image displays. Use a **browser** (Claude-in-Chrome / a headless browser MCP) for any visual claim. Add a cache-buster (`?v=1`, `?v=2`) on each reload — theme CSS/JS and the CDN cache aggressively, and a client's "it still looks old" is almost always stale cache (tell them **Cmd/Ctrl+Shift+R**).

## The DOM is ground truth — inspect BEFORE writing CSS

Never write CSS from assumed class names. Horizon's classes change across versions and section settings. Open the live page and dump the real structure (`getComputedStyle`, `outerHTML`, class lists) for the element you're about to style, then target exactly that. Selectors observed on one 2025 Horizon build (yours may differ — **verify**):

- Product card: `<product-card class="product-card size-style">` → `a.product-card__link` (invisible full-card overlay, `visually-hidden` label) + `.product-card__content` + `.card-gallery` (**holds the badge**, `card-gallery--badge-top-right`) → `.product-media` (**`aspect-ratio: 4/5`, `object-fit: cover`**, base `img.product-media__image`).
- Title: an `<a class="contents">` (`display:contents`, so set `font-family` on it — it inherits to the text). Price: `<product-price>` → `.price` / `.price__sale .price-item--sale` / `s.price-item--regular`.
- Sale badge element: **`.product-badges__badge`** (inside `.product-badges--top-right`) — NOT `.badge`. Targeting the wrong class is why a "badge restyle" silently no-ops.
- Product grid container: `.resource-list.resource-list--grid` (desktop); a **`hidden--mobile`** sibling is the mobile carousel.

### The grid-collapse gotcha (Horizon resource-list)

`.resource-list--grid` computed `grid-template-columns: 1120px` — a **single full-width column** — at a ~1200px viewport, even with the section's `columns: 4` setting. It only shows 4 columns on very wide screens; between breakpoints it collapses to one giant column per row (page height balloons; each card ~full width). Confirm it's not your CSS (toggle your stylesheet's `disabled` in devtools and re-measure). Fix by forcing a responsive grid, **scoped to desktop widths so you don't un-hide the mobile carousel**:

```css
@media (min-width: 750px)  { .resource-list--grid { grid-template-columns: repeat(3, minmax(0,1fr)) !important; row-gap: clamp(36px,4vw,72px) !important; } }
@media (min-width: 1280px) { .resource-list--grid { grid-template-columns: repeat(4, minmax(0,1fr)) !important; } }
```

(Never set `display` on the `hidden--mobile` grid — you'd reveal it on mobile alongside the carousel.)

## Automation traps that make verification lie

- **Lenis / smooth-scroll libraries break programmatic scroll.** `window.scrollTo` gets clamped/reset while a momentum-scroll lib is active. It also *feels* heavy to users. Remove it for native scroll (faster + scriptable), or drive scroll via the lib's own `.scrollTo(y, {immediate:true})`.
- **Screenshots catch lazy-loaded images mid-render.** A tile/banner can look empty in a screenshot while the image is fine. **Verify in JS before concluding an image is broken:** `img.complete && img.naturalWidth > 0`, and check `getAttribute('src')`. Don't "fix" a non-bug.
- **Automation tabs report `document.hidden = true`.** Any visibility-gated JS (auto-rotators, autoplay, `requestAnimationFrame` loops guarded by `!document.hidden`) will sit paused in the automation tab and look broken — it runs fine for a real focused visitor. Confirm the mechanism another way (manually toggle a class) rather than assuming it's dead.
- **`javascript_tool` blocks returning URL/query-string data** (a safety filter). If a DOM probe returns `[BLOCKED: query string data]`, strip URLs/`srcset` from the returned object — return class names, tag names, computed styles only.

## Performance levers (what actually made it faster)

- **Drop render-blocking CDN libraries.** Three `<script>` tags (GSAP + ScrollTrigger + Lenis, ~100 KB) in `<head>` block first paint. Removing them + going native-scroll is the single biggest win; re-implement the few effects you need in a few lines of vanilla JS.
- **Progressive image loading beats eager.** An auto-rotating gallery that eager-loads 4 extra images × every card downloads 60–100 images up front. Instead hold each layer's src in `data-src` and set `.src` only when it's about to show (warm just the *first* alt image per visible card). Cuts initial bytes by ~4×. See `auto-rotating-gallery.md`.
- Measure with the browser (doc height, network count), not vibes. Homepage doc height dropping 15,000px → 5,500px after fixing a 1-column grid is a real, checkable signal.

## Replicating a reference site (do this, don't design from taste)

When the client says "make it look like `<url>`", **extract that site's actual design tokens** — don't invent your own interpretation:

- Open the reference live, `getComputedStyle` the real elements: `fontFamily`, `fontSize`, `fontWeight`, `letterSpacing`, `color`, `backgroundColor`, `borderRadius`, button padding, badge/pill styling.
- Read its signature color off a real element (e.g. announcement bar bg → `rgb(68,28,55)` = `#441C37`).
- Licensed fonts (Gotham, and site-custom faces) can't be copied — map each to the closest **Google Font** (e.g. Gotham→Montserrat, a flared-roman display→Marcellus) and say so; side-by-side it lands ~95%.
- Rebuild your CSS around those extracted tokens. This turns a subjective "make it nicer" into an objective spec you can verify element-by-element.
