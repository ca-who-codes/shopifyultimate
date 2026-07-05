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

## Shop

```graphql
{ shop { primaryDomain { url } myshopifyDomain } }
```

## Verification habits

- After **any** write, re-fetch and check: `checksumMd5`/`size` for verbatim files; parse + inspect changed fields for re-serialized JSON (`settings_data.json`, templates); re-fetch `descriptionHtml` for collection copy.
- `themeFilesUpsert` returning `{ upsertedThemeFiles: [], userErrors: [] }` is common on success — it is **not** proof; the re-fetch is.
- Media reorder and collection image updates are **async** — the mutation returns before the CDN reflects it; re-query.
