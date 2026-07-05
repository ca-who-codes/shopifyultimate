# Shopify Ultimate

Hard-won, field-tested techniques for editing Shopify themes and store data programmatically through the **Admin GraphQL API** — especially the newer **Horizon** theme (theme blocks, `color_palette`, custom-liquid sections).

These are packaged as an [Agent Skill](https://agentskills.io/specification.md) so AI agents (Claude Code, Cursor, etc.) can load them on demand.

## What's inside

- **`skills/shopify-theme-ops/`** — the skill. Start with [`SKILL.md`](skills/shopify-theme-ops/SKILL.md).

### The headline techniques

| Problem | Technique |
|---|---|
| Writing a large theme file without corrupting it | **Prefer the Shopify CLI** (`theme push --only … --nodelete`) — it moves bytes and fails loudly. No CLI? **Byte-exact push via staged uploads** (`stagedUploadsCreate` → upload bytes → `themeFilesUpsert`) |
| Reading a theme file bigger than the tool budget | `theme pull --only <file>` (CLI), or bundle with a large file to force persist-to-disk |
| A font that isn't in Shopify's font picker | Override Horizon's CSS font variables in `theme.liquid`'s `<head>` + load the font yourself |
| `settings_data.json` "won't save" / reverts | Validates atomically and **silently reverts** on a bad handle; also the **theme editor overwrites it** — pull fresh before editing, verify by re-fetching |
| Getting images onto products at scale | `images:[{url}]` with **public** `raw.githubusercontent.com` urls; `fileCreate` + poll `READY` for Files |
| New product / collection is invisible | DRAFT & unpublished are hidden — `ACTIVE` + `publishablePublish` to Online Store |
| Everything shows "Sold out" (drop-ship) | Set variants `inventoryPolicy: CONTINUE` |
| Updating many products fast | **Aliased batch mutation** (`p0:`, `p1:`, … + variables) — one round-trip |
| Client hates the design of the "Opening soon" page | That's the **password gate**, not the store — a Preferences system page, not a theme file |
| "Make it look like `<competitor>`" | Extract the reference's computed styles (fonts/colors/radii) and rebuild to spec |
| Collection grid pushed right / collapsed to 1 column | `product_grid_width: full-width`; force responsive `.resource-list--grid` columns |
| Customers checking out a ₹0 free-gift-only cart | Sticky-reconcile gift logic + gifts-only checkout guard |

## Install (as a Claude Code plugin marketplace)

```bash
/plugin marketplace add ca-who-codes/shopifyultimate
```

Or copy `skills/shopify-theme-ops/` into your agent's `.agents/skills/` (or `.claude/skills/`).

## Scope

Everything here assumes you have an **authenticated Shopify Admin GraphQL** connection (an MCP server, a private app token, or the Shopify CLI). The skill is about *what to call and in what order* — not about auth.

License: MIT.
