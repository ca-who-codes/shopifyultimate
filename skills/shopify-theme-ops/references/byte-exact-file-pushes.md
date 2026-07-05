# Byte-exact theme file writes via staged uploads

**Problem.** You need to write a theme file — a 20 KB `templates/index.json`, a 25 KB CSS asset, `layout/theme.liquid`, `config/settings_data.json`. The obvious path — inline the content into a `themeFilesUpsert` mutation (`body: {type: TEXT, value: "..."}`) — forces the file's bytes through your context and a tool call. For anything non-trivial that means retyping/reproducing it, and a single dropped character silently breaks the storefront.

**Solution.** Never inline. Upload the **bytes** from disk to Shopify's staged storage, then point `themeFilesUpsert` at the resulting URL. Zero transcription.

## The five steps

### 1. Build the file on disk with a script

- New/edited JSON template → build the dict in Python and `json.dumps` it (guaranteed valid). To edit an existing file, fetch it, `json.loads`, change the one field, `json.dumps` back — everything you didn't touch is byte-identical.
- Preserve the leading auto-generated comment block that Shopify prepends to JSON templates (`/* ... auto-generated ... */`) — split it off before parsing, prepend it after.

### 2. `stagedUploadsCreate` → get an upload target

```graphql
mutation Stage($input: [StagedUploadInput!]!) {
  stagedUploadsCreate(input: $input) {
    stagedTargets { url resourceUrl parameters { name value } }
    userErrors { field message }
  }
}
```
```json
{ "input": [{ "resource": "FILE", "filename": "index.json",
              "mimeType": "application/json", "httpMethod": "POST" }] }
```

`resource: FILE` works for any text asset. Use the matching `mimeType`: `application/json`, `text/css`, `text/javascript`, `text/x-liquid`. You get back a Google Cloud Storage `url`, a set of signed form `parameters` (including a long `policy` and `x-goog-signature`), and a `resourceUrl`.

### 3. `curl` the bytes to the target — but validate the policy first

The `policy` is base64(JSON) and the `x-goog-signature` is a 512-char hex string. **Hand-copying them into a `curl -F ...` command is the #1 cause of `HTTP 400`/`403` here.** Validate the policy decodes and matches your key/date in Python *before* uploading, and build the multipart POST in Python so there's no shell-quoting either:

```python
import base64, json, subprocess
SIG    = "<x-goog-signature>"
POLICY = "<policy>"
pol = json.loads(base64.b64decode(POLICY))                 # throws if you mistyped it
key  = next(c['key']         for c in pol['conditions'] if isinstance(c, dict) and 'key' in c)
date = next(c['x-goog-date'] for c in pol['conditions'] if isinstance(c, dict) and 'x-goog-date' in c)
assert key.endswith("index.json"), key                      # sanity-check the target
args = ["curl","-s","-o","/dev/null","-w","HTTP %{http_code}\n","-X","POST",
        "https://shopify-staged-uploads.storage.googleapis.com/",
        "-F","Content-Type=application/json","-F","success_action_status=201","-F","acl=private",
        "-F","key="+key,"-F","x-goog-date="+date,
        "-F","x-goog-credential=<x-goog-credential>",
        "-F","x-goog-algorithm=GOOG4-RSA-SHA256","-F","x-goog-signature="+SIG,"-F","policy="+POLICY,
        "-F","file=@index.json;type=application/json"]      # <-- the bytes, from disk
print(subprocess.run(args, capture_output=True, text=True).stdout)
```

A successful upload returns **`HTTP 201`**. (Order of `-F` fields doesn't matter for GCS POST, but the `file` field must be present and last is conventional.)

### 4. `themeFilesUpsert` from the staged URL

```graphql
mutation Upsert($themeId: ID!, $files: [OnlineStoreThemeFilesUpsertFileInput!]!) {
  themeFilesUpsert(themeId: $themeId, files: $files) {
    upsertedThemeFiles { filename }
    userErrors { filename field message code }
  }
}
```
```json
{ "themeId": "gid://shopify/OnlineStoreTheme/<ID>",
  "files": [{ "filename": "templates/index.json",
              "body": { "type": "URL", "value": "<resourceUrl from step 2>" } }] }
}
```

Shopify fetches the bytes from the staged `resourceUrl` and writes them into the theme.

### 5. Verify — always

`themeFilesUpsert` frequently returns `upsertedThemeFiles: []` **and** `userErrors: []` even on success, so the response tells you little. Re-fetch and check:

```graphql
{ theme(id: "gid://shopify/OnlineStoreTheme/<ID>") {
    files(first: 1, filenames: ["assets/your-file.css"]) {
      nodes { filename size checksumMd5 }
    } } }
```

For a verbatim-stored file (assets, `theme.liquid`), `checksumMd5` should equal your local file's MD5 and `size` should match to the byte. For JSON that Shopify re-serializes (`settings_data.json`, and it lightly reformats templates), the MD5 **won't** match — verify by parsing the fetched JSON and inspecting the fields you changed instead.

## Gotchas

- **`settings_data.json` can silently revert** even after a clean 201 + no `userErrors`, if a value fails validation (bad font handle is the classic). Always re-fetch and diff the specific fields. See `fonts-and-settings-data.md`.
- **Staged targets expire** (~24 h) and are single-use per successful upload. If an upload 403s, re-check your policy copy; the target itself is usually fine to retry.
- **Verbatim vs re-serialized**: assets and `.liquid` are stored byte-for-byte (MD5 verifiable). JSON config/templates are validated and may be reformatted — don't rely on MD5 for those.
- This same staged-upload flow is how you'd push **binary** assets (self-hosted `.woff2` fonts, images) too — same steps, different `mimeType`.
