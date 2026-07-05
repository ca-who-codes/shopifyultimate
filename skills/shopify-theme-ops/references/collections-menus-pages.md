# Collections, menus, and pages

Store structure via Admin GraphQL. All batch well with aliases (`catalogue-and-products.md`).

## Smart (auto-populating) collections

A "Gifts Under ₹5K" type collection should be a **smart collection** — define a rule, and products flow in/out automatically as prices/tags change. Zero maintenance.

```graphql
mutation { collectionCreate(input: {
    title: "Gifts Under ₹5,000",
    handle: "gifts-under-5k",
    descriptionHtml: "<p>…</p>",
    ruleSet: { appliedDisjunctively: false, rules: [
      { column: VARIANT_PRICE, relation: LESS_THAN, condition: "5000" }
    ] }
  }) { collection { id handle } userErrors { field message } } }
```

- `appliedDisjunctively: false` = **AND** all rules; `true` = **OR**.
- Common columns: `VARIANT_PRICE`, `TAG`, `TYPE`, `VENDOR`, `TITLE`, `VARIANT_INVENTORY`. Relations: `LESS_THAN`, `GREATER_THAN`, `EQUALS`, `CONTAINS`, `STARTS_WITH`.
- `condition` is always a **string**, even for numbers (`"5000"`).

## Collections are NOT published on create

`collectionCreate` (and `create-collection`) make the collection but do **not** publish it to the Online Store — it won't appear on the storefront until you publish it to that channel. Same trap as DRAFT products.

```graphql
# 1. Find the Online Store publication id (do this once, cache it).
{ publications(first: 5) { nodes { id name } } }   # → "Online Store" gid

# 2. Publish the collection (or product) to it.
mutation { publishablePublish(
    id: "gid://shopify/Collection/ID",
    input: [{ publicationId: "gid://shopify/Publication/ONLINE_STORE_ID" }]
  ) { userErrors { field message } } }
```

Note: the Online Store `Publication` id is store-specific and **not** `gid://shopify/Publication/1` — query `publications` for the real one (guessing `/1` errors with "Invalid publication id").

Set a collection's tile image the same way as in the cookbook: `collectionUpdate(input: {id, image: {src, altText}})` — `featured_image` falls back to the first product's image until you do.

## Navigation menus

Menus are separate objects (not theme files). The theme's header/footer reference them by **handle** (`main-menu`, `footer`). `menuUpdate` **replaces the entire item list** — pass the full desired set:

```graphql
mutation { menuUpdate(
    id: "gid://shopify/Menu/ID", title: "Main menu",
    items: [
      { title: "Home",      type: HTTP, url: "/" },
      { title: "Necklaces", type: HTTP, url: "/collections/necklaces" },
      { title: "Contact",   type: HTTP, url: "/pages/contact" }
    ]) { menu { id items { title } } userErrors { field message } } }

mutation { menuDelete(id: "gid://shopify/Menu/ID") { deletedMenuId userErrors { field message } } }
```

- `type: HTTP` with a **relative** url (`/collections/x`, `/pages/y`, `/`) is the universal, always-works path for internal links — no need to resolve `resourceId`s. Other types (`COLLECTION`, `PAGE`, `FRONTPAGE`, `SEARCH`) exist but need a resourceId; skip them unless you have a reason.
- Cleaning up duplicate/leftover menus (`main-menu-1`, etc. from theme installs) is legit housekeeping — they're invisible admin clutter; delete the unused ones and populate the ones the theme actually reads.

## Pages

The **new** Page API (2024-04+, what a Horizon store uses) uses `pageUpdate(id, page: {...})` and the content field is **`body`**, not `bodyHtml`:

```graphql
mutation Pages($about: String!, $faq: String!) {
  about: pageUpdate(id: "gid://shopify/Page/ID1", page: {title: "Our Story", body: $about}) {
    page { id title } userErrors { field message } }
  faq:   pageUpdate(id: "gid://shopify/Page/ID2", page: {title: "FAQ", body: $faq}) {
    page { id title } userErrors { field message } }
}
```

Query pages with `pages(first: 25) { nodes { id title handle bodySummary } }`. Watch for **stale content from the theme demo / a prior brand** living in About/FAQ/Care/Shipping/Size pages (and even the literal old brand name) — rewrite them when you rebrand; they're easy to miss because they're not in the theme files.
