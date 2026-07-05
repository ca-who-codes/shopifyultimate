# The Shopify CLI is the byte-mover — prefer it over staged uploads

If `shopify` CLI is **authenticated for the store**, it does the entire staged-upload dance for you in one command. It moves bytes from disk, verifies, and never routes file contents through a tool call. Reach for `byte-exact-file-pushes.md` (manual `stagedUploadsCreate` → curl → `themeFilesUpsert`) **only** when you have Admin GraphQL but no CLI auth.

Check auth once: `shopify theme list` — it prints the store's themes with roles, or prompts to log in.

## The core loop

Keep a **local mirror** of the theme (a `shopify/` folder in a repo = source of truth off the theme). Edit files there, then push only what changed:

```bash
# Pull fresh copies BEFORE editing (see the settings_data trap below)
shopify theme pull --only config/settings_data.json --only config/settings_schema.json --theme <THEME_ID> --path .

# Edit files locally (scripts for JSON, string-replace for byte-fidelity — same discipline as the GraphQL path)

# Push ONLY the files you touched
shopify theme push \
  --only templates/index.json \
  --only assets/brand.css \
  --theme <THEME_ID> \
  --allow-live --nodelete
```

Flags that matter:

- **`--only <path>`** (repeatable) — push just these files. Without it, push syncs the whole theme and **deletes remote files not in your local mirror** — dangerous when you mirror only a subset.
- **`--nodelete`** — never delete remote files. Use it always when mirroring a subset. Belt-and-suspenders with `--only`.
- **`--allow-live`** — **required** to push to the *published* theme. Its absence is a guardrail; its presence is you taking responsibility. Only pass it with explicit permission to touch live.
- **`--path .`** — the theme root of your local mirror (defaults to cwd).
- **`--theme <id>`** — target a specific theme by numeric id (from `shopify theme list`). Pin it; don't rely on "development" defaults.

`shopify theme push` reformats/validates JSON the same way the API does, so a valid local JSON is a valid push. A malformed local JSON fails **loudly** at the CLI (unlike `themeFilesUpsert`, which can no-op silently) — a real advantage.

## Draft vs live, and publishing

- Theme **id** is the numeric id from `shopify theme list` (the `[live]` marker shows which is published).
- Publishing a theme (making a draft live) is **not** exposed to write via the Admin GraphQL API and the CLI's publish is a deliberate manual step — treat "make it live" as a human action in Admin → Online Store → Themes. You can *push to* a live theme with `--allow-live`, but flipping which theme is live stays with the user.
- If a "draft" theme unexpectedly needs `--allow-live`, it's because the user **published it** — re-check `shopify theme list` before assuming.

## The settings_data.json pull-fresh rule

`config/settings_data.json` is written by the **theme editor** every time the user touches a color, font, logo, or section setting in Admin. Your local mirror goes stale the moment they do. **Always `theme pull --only config/settings_data.json` immediately before editing it**, edit the fresh copy, push it back. Skipping this silently reverts whatever the user changed in the editor since your last pull.

Pull `config/settings_schema.json` too when you need to discover exact setting **ids** (e.g. `logo`, `favicon`, `logo_height`) — grep it for the keys under the group you care about.

## Reading files via CLI

`shopify theme pull --only <path>` writes the file to disk where you can `Read`/`grep`/script it — sidesteps the "file bigger than the tool budget" problem entirely (no need for the bundle-with-a-large-file trick in `reading-large-theme-files.md` when the CLI is available).

## Verify

Same rule as everywhere: a clean push is necessary, not sufficient. For anything user-visible, **load the live URL in a browser and look** (`live-verification-and-gotchas.md`). Add a cache-buster query (`?v=1`, `?v=2`) each reload — Shopify's CDN caches theme assets hard, and so does the browser.
