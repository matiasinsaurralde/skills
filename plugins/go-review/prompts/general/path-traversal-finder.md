---
name: path-traversal-finder
description: Detects a request-derived path segment reaching filepath.Join/path.Join and escaping the intended base directory
---

**Finding ID Prefix:** `PATHTRAVERSAL`.

**Bug shape:** A path segment from user input (filename parameter, uploaded file's declared name, URL path component) reaches `filepath.Join(baseDir, userInput)` and is then used to open/read/write/serve a file, with no validation that the joined result stays within `baseDir`. `filepath.Join` calls `Clean` internally, which collapses `..` segments **syntactically** but does not prevent the result from resolving outside `baseDir` if `userInput` itself starts with enough `../` to walk up past it — `Join`'s cleaning is not a security boundary by itself.

**Gates:**

1. User-controlled input flows into `filepath.Join(...)` or `path.Join(...)` as one of the path segments.
2. The joined result is used in a filesystem operation (`os.Open`, `os.ReadFile`, `http.ServeFile`, `os.Create`) without a subsequent check that the resolved absolute path is still within the intended base directory.

**FPs (reject):**

- The code validates the joined result via `filepath.Rel(baseDir, joined)` (rejecting any result starting with `..`) or an equivalent containment check after joining, before the filesystem operation.
- The user input is validated upstream to reject any `/` or `..` before it ever reaches `Join` (a strict filename-only allowlist, e.g. matching `^[\w.-]+$`).
- `http.FileServer`/`http.Dir` is used, which already does its own traversal-safe path resolution internally (verify it's actually `http.Dir`-based serving, not a hand-rolled `os.Open` off a joined path).

**Patch:** after joining, verify containment with `filepath.Rel(baseDir, joined)` and reject if the result starts with `..` or is absolute; or validate the untrusted segment against a strict allowlist pattern before joining at all.

**Search patterns:**

```
filepath\.Join\(|path\.Join\(
os\.(Open|Create|ReadFile|WriteFile)\(
http\.ServeFile\(
filepath\.Rel\(
```
