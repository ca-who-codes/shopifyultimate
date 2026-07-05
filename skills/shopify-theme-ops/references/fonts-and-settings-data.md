# Fonts, the font-library reality, and settings_data.json

## Shopify's font picker is a curated subset — not all Google Fonts

`config/settings_data.json` stores fonts by **handle**: `{family}_n{weight}` for normal, `{family}_i{weight}` for italic (e.g. `nunito_n4`, `fraunces_n7`, `montserrat_n7`). Only fonts in Shopify's curated library have valid handles. Confirmed-common examples that **are** available: Assistant, DM Sans, DM Serif Display, EB Garamond, Fraunces, Inter, Jost, Lora, Montserrat, Noto Sans/Serif, Nunito, Playfair Display, Poppins, Work Sans.

Fonts that are popular but were **not** in the library when this was written (their handle silently fails): **Baloo 2, Marcellus, Mulish** (and many others). Don't assume a Google Font is available.

## The silent-revert trap

When you save `settings_data.json` with an invalid font handle (or any invalid value), Shopify:

1. Validates the whole file **atomically**.
2. On failure, **keeps the previous valid version**.
3. Still returns **`themeFilesUpsert { userErrors: [] }`** — no error surfaced.

So a font change can look successful (clean 201, no errors) yet the stored file still shows the old handle. **Always re-fetch `settings_data.json` and confirm the field actually changed.** To find which field is rejected, change one field at a time and re-fetch.

Also: **preserve `current.blocks`** when rewriting `settings_data.json`. Those are theme app-embed blocks (chat widgets, review apps, etc.). Dropping them disables the apps. When you edit via `json.loads` → change one field → `json.dumps`, they're preserved automatically; just don't hand-rebuild the file.

## Forcing any font via a CSS-variable override

When the font you want isn't in the library, don't fight the setting — **override the theme's CSS font variables** and load the font yourself.

**Horizon reads these family variables** (grep `base.css` to confirm for your theme version):
`--font-h1--family` … `--font-h6--family`, `--font-body--family`, `--font-paragraph--family`. Block `font` settings also reference `--font-heading--family`, `--font-primary--family`, `--font-subheading--family`, `--font-accent--family`.

Inject into `layout/theme.liquid`, in `<head>`, **after** the theme sets its own variables (in Horizon that's after `{%- render 'theme-styles-variables' -%}` and `{%- render 'color-palette' -%}`):

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Marcellus&family=Mulish:wght@400;500;600;700&display=swap">
<style>
  :root, body {
    --font-heading--family: 'Marcellus', Georgia, serif !important;
    --font-h1--family: 'Marcellus', Georgia, serif !important;
    --font-h2--family: 'Marcellus', Georgia, serif !important;
    --font-h3--family: 'Marcellus', Georgia, serif !important;
    --font-h4--family: 'Marcellus', Georgia, serif !important;
    --font-h5--family: 'Marcellus', Georgia, serif !important;
    --font-h6--family: 'Marcellus', Georgia, serif !important;
    --font-primary--family: 'Marcellus', Georgia, serif !important;
    --font-accent--family: 'Marcellus', Georgia, serif !important;
    --font-subheading--family: 'Mulish', system-ui, sans-serif !important;
    --font-body--family: 'Mulish', system-ui, -apple-system, sans-serif !important;
    --font-paragraph--family: 'Mulish', system-ui, -apple-system, sans-serif !important;
  }
</style>
```

Why it works: `!important` on a custom-property declaration wins the cascade over the theme's non-important `:root` value regardless of source order; and because your fallback stack **replaces** the theme's (no old font name in it), there's no flash of the previous font — just fallback → your font.

### Placement matters

- **`<head>` (theme.liquid)**: no flash. Correct for fonts.
- **Footer custom-liquid `<style>`**: works, but the override parses late, so headings render in the old font first and then swap — a visible flash. Avoid for fonts.

### Downsides to accept

- The theme editor's font picker still shows the *setting* (old font), because you overrode in CSS, not in `settings_data.json`. Cosmetic.
- The theme still loads the set font's `@font-face` (unused). Negligible.
- If you want zero external requests / no CSP worries, self-host `.woff2` via the staged-upload flow and `@font-face` them instead of Google Fonts.

## Bonus: reusable component CSS in the head

The same `<head>` `<style>` block is a great home for small global component CSS you reference from many places — e.g. chip styles used by rewritten collection descriptions (`.bcp-cd`, `.bcp-cd__chip`), so each description stays tiny (classes only) and consistent.
