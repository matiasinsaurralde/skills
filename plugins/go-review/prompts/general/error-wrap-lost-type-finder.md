---
name: error-wrap-lost-type-finder
description: Detects error wrapping that loses the type/chain information downstream errors.Is/errors.As needs
---

**Finding ID Prefix:** `ERRWRAP`.

**Bug shape:** `fmt.Errorf("context: %v", err)` — the `%v` verb formats the error into a plain string, discarding the underlying error value entirely. Any downstream `errors.Is`/`errors.As` on the returned error can never see through it to the original, even though the code looks like proper error wrapping.

**Gates:**

1. `fmt.Errorf(fmtString, ..., err)` where the format string uses `%v` or `%s` for the error argument instead of `%w`.
2. The resulting error is returned (not just logged) — a wrap used purely for a log line has no downstream `errors.Is`/`errors.As` consumer and is lower-severity, but still worth noting if the codebase's error-handling convention elsewhere expects wrapping to be inspectable.
3. Somewhere downstream (same package or an importer), an `errors.Is`/`errors.As`/type-assertion call exists that would need to see through this wrap to function correctly — if you can find such a call, this raises confidence; if you can't, still file at lower confidence (the absence of a current consumer doesn't make future wrapping-breakage safe).

**FPs (reject):**

- The error is deliberately **not** meant to be unwrapped further (a public API boundary that intentionally returns an opaque error, documented as such).
- `%w` is already used correctly.

**Patch:** use `%w` in place of `%v`/`%s` for the error verb: `fmt.Errorf("context: %w", err)`.

**Search patterns:**

```
fmt\.Errorf\([^)]*%v[^)]*,\s*err\)
fmt\.Errorf\([^)]*%s[^)]*,\s*err\)
fmt\.Errorf\([^)]*%w
```
