# Shopify Ultimate

Hard-won, field-tested techniques for editing Shopify themes and store data programmatically through the **Admin GraphQL API** — especially the newer **Horizon** theme (theme blocks, `color_palette`, custom-liquid sections).

These are packaged as an [Agent Skill](https://agentskills.io/specification.md) so AI agents (Claude Code, Cursor, etc.) can load them on demand.

## What's inside

- **`skills/shopify-theme-ops/`** — the skill. Start with [`SKILL.md`](skills/shopify-theme-ops/SKILL.md).

### The headline techniques

| Problem | Technique |
|---|---|
| Writing a large theme file without corrupting it | **Byte-exact push via staged uploads** (`stagedUploadsCreate` → upload bytes → `themeFilesUpsert` with `body: {type: URL}`) — no transcription |
| Reading a theme file bigger than the tool budget | Bundle it with a large file to force persist-to-disk, then extract |
| A font that isn't in Shopify's font picker | Override Horizon's CSS font variables in `theme.liquid`'s `<head>` + load the font yourself |
| `settings_data.json` "won't save" | It validates atomically and **silently reverts** on a bad handle — verify by re-fetching |
| Collection grid pushed to the right | `main-collection` `product_grid_width: full-width` |
| Customers checking out a ₹0 free-gift-only cart | Sticky-reconcile gift logic + gifts-only checkout guard |

## Install (as a Claude Code plugin marketplace)

```bash
/plugin marketplace add ca-who-codes/shopifyultimate
```

Or copy `skills/shopify-theme-ops/` into your agent's `.agents/skills/` (or `.claude/skills/`).

## Scope

Everything here assumes you have an **authenticated Shopify Admin GraphQL** connection (an MCP server, a private app token, or the Shopify CLI). The skill is about *what to call and in what order* — not about auth.

License: MIT.
