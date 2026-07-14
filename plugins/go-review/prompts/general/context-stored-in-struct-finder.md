---
name: context-stored-in-struct-finder
description: Detects context.Context stored as a struct field instead of threaded as each method's first parameter
---

**Finding ID Prefix:** `CTXSTORED`.

**Bug shape:** Go's convention (and the `context` package's own documentation) is that a `Context` should be the first parameter of every function that needs it, never stored in a struct — storing it in a struct field means every method sharing that struct sees the *same* context regardless of which call it's actually serving, breaking per-call cancellation/deadline/value scoping. A struct built once and reused across many requests (a long-lived service object) that stores a `Context` field will hand every caller the context from whenever the struct was constructed, not the context of the current call.

**Gates:**

1. A struct type has a field of type `context.Context`.
2. That struct is not itself a short-lived, single-call-scoped value that's constructed fresh per request/operation and discarded immediately after (a "request-scoped struct" storing its own context can be a defensible, narrow exception, but should still be flagged at low severity for the convention violation and to prompt review of its actual scope).

**FPs (reject):**

- The struct is constructed fresh for exactly one call and never reused (a genuinely request-scoped struct) — still note as a style/convention issue at low severity, not FP, since the pattern remains fragile to future refactors.

**Patch:** remove the `Context` field and add `ctx context.Context` as the first parameter of every method that needs it, per the standard Go convention.

**Search patterns:**

```
context\.Context\s*$
type\s+\w+\s+struct\s*\{
```
