# Admin GraphQL cookbook

Copy-paste queries and mutations for the operations this skill covers. Run through whatever authenticated Admin GraphQL tool you have. IDs are GIDs (`gid://shopify/<Type>/<number>`).

## Themes

```graphql
# List themes — find the working draft vs the published (MAIN) one.
{ themes(first: 10) { nodes { id name role } } }

# Read specific files (small ones inline; bundle big ones — see reading-large-theme-files.md).
{ theme(id: "gid://shopify/OnlineStoreTheme/ID") {
    files(first: 5, filenames: ["templates/index.json","assets/base.css"]) {
      nodes { filename size checksumMd5 body { ... on OnlineStoreThemeFileBodyText { content } } }
    } } }

# Write a file (from a staged upload — see byte-exact-file-pushes.md).
mutation($themeId: ID!, $files: [OnlineStoreThemeFilesUpsertFileInput!]!) {
  themeFilesUpsert(themeId: $themeId, files: $files) {
    upsertedThemeFiles { filename } userErrors { filename field message code }
  } }
# vars: { "themeId":"gid://…", "files":[{"filename":"assets/x.css","body":{"type":"URL","value":"<resourceUrl>"}}] }
```

## Staged uploads (for byte-exact writes and binary assets)

```graphql
mutation($input: [StagedUploadInput!]!) {
  stagedUploadsCreate(input: $input) {
    stagedTargets { url resourceUrl parameters { name value } } userErrors { field message }
  } }
# vars: { "input":[{"resource":"FILE","filename":"index.json","mimeType":"application/json","httpMethod":"POST"}] }
```

## Collections

```graphql
# Which template does a collection use? (null/"" => templates/collection.json)
{ collectionByHandle(handle: "mukhwas") { id title templateSuffix image { url } } }

# Set a collection image (fixes category tiles that all show the same fallback).
mutation { collectionUpdate(input: {
    id: "gid://shopify/Collection/ID",
    image: { src: "https://cdn.shopify.com/…/clean-shot.jpg", altText: "…" }
  }) { collection { id image { url } } userErrors { field message } } }

# Replace a wall-of-text description with chip HTML (classes survive sanitization).
mutation($html: String!) { collectionUpdate(input: {
    id: "gid://shopify/Collection/ID", descriptionHtml: $html
  }) { collection { id } userErrors { field message } } }
```

`collection.featured_image` returns `collection.image` if set, else the first product's featured image — which is why several collections can end up showing the same combo/collage image until you set each collection's own `image`.

## Products & media

```graphql
# Inspect a product's media (dimensions reveal banner-shaped images that overflow cards).
{ product(id: "gid://shopify/Product/ID") {
    featuredMedia { preview { image { url } } }
    media(first: 20) { edges { node { ... on MediaImage { id image { url width height } } } } }
  } }

# Reorder media so a clean image leads (fixes a bad featured/hover image). Non-destructive.
mutation($id: ID!, $moves: [MoveInput!]!) {
  productReorderMedia(id: $id, moves: $moves) { job { id done } mediaUserErrors { field message } }
}
# vars: { "id":"gid://…", "moves":[{"id":"gid://shopify/MediaImage/A","newPosition":"0"},
#                                    {"id":"gid://shopify/MediaImage/B","newPosition":"1"}] }
# It's async — re-query media to confirm the new order.
```

Prefer **reorder** over delete. `productDeleteMedia` is destructive — surface it and get explicit confirmation before removing images (e.g. wrong-brand shots).

## Files (ingest an external image → Shopify CDN)

```graphql
mutation { fileCreate(files: [
  { originalSource: "https://raw.githubusercontent.com/…/logo.png", alt: "…", contentType: IMAGE }
]) { files { id fileStatus } userErrors { field message } } }

# fileStatus is UPLOADED immediately but image.url is null until processed — poll:
{ node(id: "gid://shopify/MediaImage/ID") { ... on MediaImage { fileStatus image { url } } } }
# wait for fileStatus: READY. In theme settings reference as shopify://shop_images/<basename>.
```

## Product status & inventory

```graphql
# Activate a DRAFT product (invisible on storefront + previews until ACTIVE + published).
mutation { productUpdate(product: {id: "gid://…/Product/ID", status: ACTIVE}) {
  product { id status } userErrors { field message } } }

# Fix "Sold out" on drop-ship / 0-stock: let variants sell when out of stock.
mutation { productVariantsBulkUpdate(productId: "gid://…/Product/ID",
    variants: [{id: "gid://…/ProductVariant/VID", inventoryPolicy: CONTINUE}]) {
  userErrors { field message } } }
```

## Smart collection + publish

```graphql
mutation { collectionCreate(input: {
    title: "Gifts Under ₹5,000", handle: "gifts-under-5k",
    ruleSet: { appliedDisjunctively: false,
      rules: [{ column: VARIANT_PRICE, relation: LESS_THAN, condition: "5000" }] }
  }) { collection { id handle } userErrors { field message } } }

# Then publish to Online Store (create does NOT publish). Find the id first:
{ publications(first: 5) { nodes { id name } } }          # "Online Store" gid (NOT /1)
mutation { publishablePublish(id: "gid://shopify/Collection/ID",
    input: [{ publicationId: "gid://shopify/Publication/ONLINE_STORE_ID" }]) {
  userErrors { field message } } }
```

## Menus & pages

```graphql
# Replace a menu's entire item list. type: HTTP + relative url = always-works internal link.
mutation { menuUpdate(id: "gid://shopify/Menu/ID", title: "Main menu",
    items: [{ title: "Home", type: HTTP, url: "/" },
            { title: "Necklaces", type: HTTP, url: "/collections/necklaces" }]) {
  menu { id items { title } } userErrors { field message } } }
mutation { menuDelete(id: "gid://shopify/Menu/ID") { deletedMenuId userErrors { field message } } }

# Pages (new Page API): the content field is `body`, NOT bodyHtml.
{ pages(first: 25) { nodes { id title handle bodySummary } } }
mutation { pageUpdate(id: "gid://shopify/Page/ID", page: {title: "Our Story", body: "<p>…</p>"}) {
  page { id title } userErrors { field message } } }
```

## Aliased batch (one round-trip, many writes)

```graphql
mutation Batch($d0: String!, $d1: String!) {
  p0: productUpdate(product: {id: "gid://…/1", descriptionHtml: $d0}) { userErrors { field message } }
  p1: productUpdate(product: {id: "gid://…/2", descriptionHtml: $d1}) { userErrors { field message } }
}
# vars: { "d0": "<p>…</p>", "d1": "<p>…</p>" }  — generate with a script; keeps big HTML out of the query.
```

## Shop

```graphql
{ shop { name primaryDomain { url } myshopifyDomain } }
# shop.name is read-only via API — store name change is manual (Settings → Store details).
```

## Verification habits

- After **any** write, re-fetch and check: `checksumMd5`/`size` for verbatim files; parse + inspect changed fields for re-serialized JSON (`settings_data.json`, templates); re-fetch `descriptionHtml` for collection copy.
- `themeFilesUpsert` returning `{ upsertedThemeFiles: [], userErrors: [] }` is common on success — it is **not** proof; the re-fetch is.
- Media reorder and collection image updates are **async** — the mutation returns before the CDN reflects it; re-query.
