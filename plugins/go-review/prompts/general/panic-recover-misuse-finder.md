---
name: panic-recover-misuse-finder
description: Detects recover() calls that never actually catch a panic (missing defer) or that swallow a panic past a corrupted invariant
---

**Finding ID Prefix:** `PANICRECOVER`.

**Bug shape:** Two distinct sub-shapes under one bug class:
1. `recover()` called outside a `defer`'d function — this **never** catches anything; `recover()` only has an effect when called directly by a deferred function during a panic unwind. Code that expects this to catch a panic is simply wrong, and the goroutine still crashes the process.
2. `recover()` correctly placed in a `defer`, but the handler swallows the panic and lets execution continue as if nothing happened, without resetting whatever invariant the panic indicated was broken (e.g. continuing to serve requests from a handler pool whose shared state may be mid-mutation).

**Gates:**

1. Sub-shape 1: a `recover()` call site not lexically inside a function literal passed to `defer` (or not inside a named function that is itself only ever called via `defer`).
2. Sub-shape 2: a `defer func() { if r := recover(); r != nil { ... } }()` whose handler body neither re-panics, returns an error to the caller, nor performs any visible state-recovery/cleanup — it just logs and falls through.

**FPs (reject):**

- Sub-shape 2 where the handler explicitly converts the panic into a returned `error` (a well-known Go idiom for "panic-to-error at a goroutine or API boundary") and the surrounding state is provably not shared/mutable (a per-request handler with no shared mutable state beyond what's already request-scoped).
- A top-level `main()` or server-loop recover whose sole job is to log-and-continue serving *other* requests, where the panicking request's own goroutine correctly terminates that request's response instead of continuing to use its state.

**Patch:** for sub-shape 1, move the `recover()` call inside a deferred function. For sub-shape 2, either re-panic after cleanup, or ensure the handler fully resets/discards any state the panic may have left inconsistent before continuing.

**Search patterns:**

```
\brecover\(\)
defer\s+func\s*\(\)\s*\{[^}]*recover\(
```
