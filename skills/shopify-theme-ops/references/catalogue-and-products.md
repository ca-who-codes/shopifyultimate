# Catalogue & product ops

Building/fixing the actual products, not the theme. All Admin GraphQL — no staged uploads needed except where noted.

## Getting images onto products (there is no "upload image" call)

`create-product` / `productUpdate` accept `images: [{ url, altText }]` where **`url` must be a public HTTPS URL Shopify can fetch**. Shopify pulls each onto its own CDN. Two reliable ways to get that public URL:

1. **Bulk build (many products at once) → a dedicated PUBLIC repo of images.** Put the image files in a *separate public* GitHub repo (keep your strategy/private repo private), then pass `https://raw.githubusercontent.com/<owner>/<assets-repo>/main/<path>.png` as the image url. One clean product call per product; Shopify ingests each. This scales far better than staged uploads for ~dozens–hundreds of images and has no signed-policy fragility.
2. **Ingest an external URL into Files** (for logos, banners, anything you'll reference as a theme/CDN asset):

```graphql
mutation { fileCreate(files: [
  { originalSource: "https://raw.githubusercontent.com/…/logo.png", alt: "…", contentType: IMAGE }
]) { files { id fileStatus } userErrors { field message } } }
```

`fileCreate` returns `fileStatus: UPLOADED` immediately but the CDN url is **null until it finishes processing**. Poll:

```graphql
{ node(id: "gid://shopify/MediaImage/ID") { ... on MediaImage { fileStatus image { url } } } }
```

Wait for `fileStatus: READY`, then the `image.url` is a real `cdn.shopify.com` URL. In theme settings, reference an uploaded file as `shopify://shop_images/<filename>` (the basename you uploaded).

Manual `stagedUploadsCreate` → GCS → `productCreateMedia` also works but is fragile to hand-drive (signed-policy transcription errors); use the public-URL path instead.

## The "Sold out" trap (drop-ship / no real stock)

Products created with `inventoryItem.tracked: true` and **0 quantity** render **"Sold out"** and are **not purchasable**. For drop-ship (vendor holds stock) or made-to-order, that's wrong. Fix: set every variant to keep selling when out of stock:

```graphql
mutation { productVariantsBulkUpdate(productId: "gid://…/Product/ID",
    variants: [{ id: "gid://…/ProductVariant/VID", inventoryPolicy: CONTINUE }]) {
  userErrors { field message } } }
```

`CONTINUE` = allow purchase at 0 stock (badge flips from "Sold out" to normal/"Sale"). `DENY` = block. Do this for all variants; batch with aliases (below).

## DRAFT products are invisible everywhere

A `DRAFT` product does **not** appear on the storefront **or in theme previews** — a common "why is my new catalogue blank?" It must be `ACTIVE` **and** published to the Online Store channel. Activate many at once with `productBulkUpdate`/the MCP `bulk-update-product-status` tool, or per-product `productUpdate(product: {id, status: ACTIVE})`. To confirm channel publication, use `publishablePublish` (see `collections-menus-pages.md` — same call, products included).

## Aliased batch mutations — the efficiency multiplier

Do NOT fire one mutation per product. Alias many into **one** call with GraphQL **variables** (variables keep big HTML/strings out of the query and dodge escaping):

```graphql
mutation Batch($d0: String!, $d1: String!) {
  p0: productUpdate(product: {id: "gid://…/1", descriptionHtml: $d0}) { userErrors { field message } }
  p1: productUpdate(product: {id: "gid://…/2", descriptionHtml: $d1}) { userErrors { field message } }
  # …up to ~17+ in one call worked fine in practice
}
```

Generate the variables with a script (fetch originals → transform → emit `{d0: "...", d1: "..."}`), then pass query + variables to one `graphql_mutation`. 17 products updated in a single round-trip.

## API arg-name gotchas (they differ per mutation)

- `productUpdate(product: ProductUpdateInput!)` — **`product:`** on recent (2024-10+) API versions. The old `input:` was removed; a modern store (Horizon theme, new Page API) is recent, so use `product:`.
- `collectionUpdate(input: CollectionInput!)` — still **`input:`**.
- If unsure, probe ONE item first; if the arg name is wrong every alias fails identically and you fix once.

## descriptionHtml / body are wholesale-replace

There is no append. `productUpdate` replaces `descriptionHtml` entirely. To add a spec block (e.g. a Weight line) to existing copy: **fetch current `descriptionHtml` → transform in a script → resend the whole thing.** Insert at a stable anchor (e.g. before `<p><em>Care:`), and fold in any find/replace (brand-name fixes) in the same pass since you're rewriting anyway. Verify by re-fetching a couple of edge cases.

Vendor `vendor` field, `title`, `status`, `tags` update the same way and batch the same way.
