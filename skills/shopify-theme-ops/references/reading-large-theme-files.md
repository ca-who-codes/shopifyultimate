# Reading theme files that exceed the tool budget

`theme.files(filenames: [...])` returns each file's `body` inline. Small files are fine, but `assets/base.css` (Horizon's is ~100 KB), big JSON templates, and long liquid files overflow the tool-output limit and either truncate or get persisted to a temp file you then have to parse.

## Getting a file onto disk so a script can edit it byte-exact

To *edit* a file byte-exact you need its exact bytes on disk (see `byte-exact-file-pushes.md`). But a small file (< ~6 KB) comes back **inline** in the response — which means it's in your context, not on disk, and reproducing it by hand reintroduces the transcription risk you're trying to avoid.

**Trick: force the response to persist to disk by bundling.** Request the file you want **together with a known-large file** in one query. The combined response blows the inline budget, so the harness writes the whole thing to a file. Then extract the one you want with a script:

```graphql
{ theme(id: "gid://shopify/OnlineStoreTheme/<ID>") {
    files(first: 2, filenames: ["config/settings_data.json", "assets/base.css"]) {
      nodes { filename body { ... on OnlineStoreThemeFileBodyText { content } } }
    } } }
```

```python
import json
d = json.load(open("<persisted-response-file>.txt"))
body = next(n['body']['content'] for n in d['data']['theme']['files']['nodes']
            if n['filename'] == 'config/settings_data.json')
# now edit `body` (split off the leading /* auto-generated */ comment, json.loads, change, dump)
open('settings_data.json','w').write(new_body)
```

`assets/base.css` is a reliable "ballast" file because it's always large. Any file over ~50 KB works.

## Grepping a huge stylesheet

To understand Horizon's CSS (grid columns, font variables, component classes) without reading 100 KB into context: persist `base.css` once, save it to disk, then `grep`:

```bash
grep -n "collection-wrapper" base.css          # find the grid layout rules
grep -noE "\-\-font-[a-z0-9]+--family" base.css | sort -u   # enumerate font variables
grep -nE "\-\-centered:|\-\-full-width:" base.css           # named grid lines
```

## Notes

- Always co-request `size` and `checksumMd5` when you'll verify a later write.
- The persisted-file path is returned in the truncation error/notice — read it with a normal file read (or `jq`/Python), in chunks if needed.
- Don't try to read a 100 KB file "just to look" — grep for the selector/variable you actually need.
