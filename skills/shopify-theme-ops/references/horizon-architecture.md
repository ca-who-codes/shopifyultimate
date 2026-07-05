# Horizon theme architecture map

Horizon is Shopify's theme-blocks flagship. If you've only worked in Dawn, these are the differences that matter when editing programmatically.

## Colors: `color_palette`, not `color_schemes`

`config/settings_data.json` → `current.color_palette` is a single object: `background`, `foreground`, `color1`, `color5`, etc. Everything references it as `{{ settings.color_palette.foreground }}`. There is no per-section scheme picker like Dawn's `color_schemes`.

## Templates → sections → blocks

- `templates/*.json` list **sections** (each with a `type`, `settings`, and `blocks`) plus an `order` array.
- Section liquid lives in `sections/*.liquid`; block liquid in `blocks/*.liquid` (often web components: `<product-card>`, `<product-price>`, `<cart-drawer>`).
- **`sections/*-group.json`** (`header-group`, `footer-group`) are section groups rendered on every page via `{% sections 'footer-group' %}` in `layout/theme.liquid`.

## The custom-liquid escape hatch

For anything the block system won't build cleanly (hero banners, trust rows, category grids, FAQ accordions, benefit chips), add a section of `type: custom-liquid` and put your HTML/CSS in its `custom_liquid` setting:

```json
"bcp_hero": {
  "type": "custom-liquid",
  "settings": { "custom_liquid": "<style>...</style><div class='...'>...</div>",
                "section_width": "full-width", "padding-block-start": 0, "padding-block-end": 0 }
}
```

Tips:
- Use **single-quoted** HTML attributes inside `custom_liquid` so you don't have to escape `"` when the value is embedded in JSON.
- Reference theme fonts/colors with the CSS vars (`var(--font-heading--family)`, `#122154` etc.) so custom sections stay consistent with the theme.
- Reference images by their CDN URL directly in `<img src>`, or `{{ 'file.png' | asset_url }}` for theme assets.

## Loading global CSS/JS

Add loaders to a `custom-liquid` section inside `sections/footer-group.json` (renders site-wide):

```html
<link rel="stylesheet" href="{{ 'your.css' | asset_url }}">
<script src="{{ 'your.js' | asset_url }}" defer="defer"></script>
```

Or, for head placement (fonts, critical CSS), edit `layout/theme.liquid`.

## Fonts

Read from CSS variables: `--font-h1--family` … `--font-h6--family`, `--font-body--family`, `--font-paragraph--family` (base.css), plus `--font-heading--family` / `--font-primary--family` / `--font-subheading--family` / `--font-accent--family` referenced by block `font` settings. See `fonts-and-settings-data.md`.

## Collection page layout (the right-shift bug)

- `sections/main-collection.liquid` renders `results-list` → `.collection-wrapper` (a CSS grid) → the `filters` block + `snippets/product-grid.liquid`'s `#ResultsList.main-collection-grid`.
- The grid is placed by **named grid lines** defined in `base.css`: `--centered: column-1 / span 12`, `--full-width: column-0 / span 14`, with `--centered-column-number: 12`.
- The `main-collection` section setting `product_grid_width` toggles between `centered` and `full-width`.
- **Symptom**: the product grid renders **right-shifted with a large empty left gutter** (worst on wide screens; a collection description, if present, sits awkwardly to the left). **Fix**: set `product_grid_width` to `full-width` in `templates/collection.json`'s `main` section. That makes the wrapper span `1 / -1` and the grid fill the row.
- All collections that don't set a `templateSuffix` share `templates/collection.json`, so one change fixes them all.

## Product cards

DOM: `<product-card class="product-card" data-product-id>` → `a.product-card__link` (whole-card link) → `.product-card__content.product-grid__card` → blocks. Title: `[ref="productTitleLink"]` (a `display:contents` anchor) → `p[role="heading"]`. Price: `<product-price>` → `.price` (with `<s>`/compare for sale). "Second image on hover" is a global setting — if a product's 2nd media is a wide banner it will overflow the card on hover; fix by reordering media (see cookbook) or clipping the card media.

To restyle cards globally without touching every template, ship a scoped CSS+JS asset (loaded via the footer) that keys off grid contexts (`.product-grid`, `.product-list`, `.product-recommendations`, `.shopify-section`) and **excludes** the cart drawer / `<dialog>`. A `MutationObserver` re-applies to cards Horizon hydrates or re-renders. Keep it defensive: never remove/reparent the card link, gallery, or price DOM.

## Cart drawer

`<cart-drawer>` in a `<dialog>`; content in `.cart-drawer__content`; AJAX cart via `/cart/(add|change|update|clear).js`; reads via `/cart.js`. It re-renders its line items on change. Inject custom cart UI into `.cart-drawer__content` and re-inject on re-render via a `MutationObserver`. See `cart-free-gift-bar.md` for the cart-change detection pattern (fetch + XHR interception + events).

## Discovering the current state

- `themes(first: 10) { nodes { id name role } }` — find the working draft (`role: UNPUBLISHED`) vs `MAIN` (published). **Never write `MAIN` without explicit permission.**
- `theme(id) { files(filenames: [...]) { nodes { filename size checksumMd5 body {...} } } }` — read specific files.
- A collection's template is `collectionByHandle(handle) { templateSuffix }` (`null`/`""` → default `templates/collection.json`).
