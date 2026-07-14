---
name: defer-error-discarded-finder
description: Detects a deferred call's error return silently discarded, e.g. `defer f.Close()` swallowing a flush failure
---

**Finding ID Prefix:** `DEFERERR`.

**Bug shape:** `defer f.Close()` where `Close()` returns an `error` — a `defer` statement cannot itself check the returned error inline, so unless the code wraps it in a closure, the error is unconditionally dropped. This matters most for buffered writers (`bufio.Writer`, `os.File` on some platforms) where `Close()` failing means data was never actually flushed to disk/network, and the caller has no idea.

**Gates:**

1. A `defer` call site whose target function/method returns an `error` (verify via signature, not name).
2. No wrapping closure captures and handles that error (`defer func() { if cerr := f.Close(); cerr != nil { ... } }()`).
3. The call is on a sink where a silent close/flush failure has a real consequence (file writes, network connections, database transactions) — not a resource where close failure is inconsequential (an already-fully-drained reader).

**FPs (reject):**

- The deferred call is wrapped in a closure that captures and handles (logs, returns, or joins into) the error.
- The resource's `Close()`/equivalent is documented as always returning nil in this codebase's usage (rare — verify, don't assume).

**Patch:** wrap the deferred call in a closure that captures the error: `defer func() { if cerr := f.Close(); cerr != nil && err == nil { err = cerr } }()` (using named returns), or use a helper like `errors.Join` to combine it with the function's primary error.

**Search patterns:**

```
\bdefer\s+\w[\w.]*\.Close\(\)
\bdefer\s+\w[\w.]*\.Flush\(\)
defer\s+func\s*\(\)\s*\{[^}]*Close\(
```
