# Branding: logo, favicon, header layout — and what you CAN'T change via API

## Logo & favicon live in settings_data.json

`config/settings_schema.json` (group `logo_and_favicon`) exposes these ids; you set them in `current` of `config/settings_data.json`:

| id | type | value form |
|---|---|---|
| `logo` | image_picker | `shopify://shop_images/<filename>` |
| `logo_inverse` | image_picker | (for dark/transparent headers) |
| `favicon` | image_picker | `shopify://shop_images/<mark>.png` |
| `logo_height` | range | px (e.g. 42) |
| `logo_height_mobile` | range | px (e.g. 30) |

Workflow: get the image into Files (`fileCreate` → wait `READY`, see `catalogue-and-products.md`), then **pull settings_data fresh**, set `current.logo`/`current.favicon` to `shopify://shop_images/<basename>`, push. Setting a `logo` **replaces the text store-name wordmark** in the header. The favicon updates the browser-tab icon (may need a browser restart to refresh — it's cached hard).

## Header layout is header-group.json settings (not CSS)

`sections/header-group.json` → `header_section.settings` controls the *structure* Elaara-style luxury sites use:

```json
"logo_position": "center",          // center | left
"menu_position": "center",          // where the nav sits
"menu_row": "bottom",               // top (beside logo) | bottom (its own row under logo)
"search_position": "left",
"enable_transparent_header_home": false   // ← see below
```

**`enable_transparent_header_home: false`** is the fix for washed-out nav: a transparent header renders nav in a light/inverse color that's invisible once it overlaps a light section on scroll. Turning it off gives a solid header with dark, readable nav everywhere. (A centered logo + nav-on-second-row + solid white header is the generic "premium jewellery/fashion" header shape.)

## Rendering a logo from a font (no designer, no asset)

When there's no logo file, generate one deterministically with Python + PIL — free, reproducible, editable:

- Download the exact display font's TTF (Google Fonts `css2` returns the `.ttf` url; `curl` it).
- **Supersample** (render at 4×, downscale with `LANCZOS`) for smooth edges on thin serif strokes and outline marks.
- Compose: an outline mark (e.g. a faceted-gem line drawing) + tracked wordmark + a small subline with flanking hairlines, all in the brand color on transparent.
- Save a full lockup PNG **and** a square mark-only PNG for the favicon.
- Host it (public assets repo `/brand/`), `fileCreate` it into Files, set as `logo`/`favicon`.

It's a raster PNG — crisp at 2× retina in a header. Keep the render script; regenerate at any size for print/packaging later. (A true vector SVG logo would need a design tool; PIL raster is the pragmatic "make it real now" path.)

## What you CANNOT do via API — tell the user to do these

These have **no writable API**; surface them as manual steps rather than silently failing:

- **Store name** (the wordmark if no logo image, browser-tab title, transactional-email sender, checkout, password page all read it). Manual: **Settings → Store details → Store name.** One field, fixes all of those.
- **Connecting a custom domain.** Manual: **Settings → Domains → Connect existing domain**, then set registrar DNS: **A record `@` → `23.227.38.65`**, **CNAME `www` → `shops.myshopify.com`**. Propagation can take hours. There is no domain-connect mutation.
- **The password / "Opening soon" gate.** It's a store-**Preferences** system page (Online Store → Preferences → Restrict access), **not** a theme file. `templates/password.json` **auto-regenerates** and will overwrite your edits — don't try to theme it via files. If the client says "the design is bad" and you see a plain "Opening soon" page, they're looking at the **password gate**, not the real store — have them turn off password protection (or preview the theme) to see the actual design.
- **Checkout branding, publishing a theme live, staff/permissions, payment/shipping config.** Manual, in Admin.

Also flag any **brand-name spelling mismatch** between the domain and the on-site name (e.g. domain `kanakiajewels.com` vs theme wordmark "Kanakya") before mass-editing — it's a one-question decision that changes every label you write.
