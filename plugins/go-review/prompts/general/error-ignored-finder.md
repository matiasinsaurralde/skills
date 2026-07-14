---
name: error-ignored-finder
description: Detects security-relevant error returns silently discarded via `_ =` or an unchecked bare call
---

**Finding ID Prefix:** `ERRIGN`.

**Bug shape:** `_ = f()` or a bare `f()` statement where `f` returns `error` (or `(T, error)`) as one of its results, and the error is never inspected on any path.

**Gates:**

1. The call's signature includes an `error` return (confirm via the function/method definition or stdlib doc — do not assume from name alone).
2. The error is discarded: assigned to `_`, or the call appears in statement position with its error result unused, with no surrounding `if err != nil`, `errors.Is`/`errors.As` check, or propagation to the caller.
3. Record the call's apparent purpose (write, close, validation, auth check) as context, but file **every** confirmed silent discard regardless of guessed impact — the fp+severity judge ranks it. Do not drop a confirmed discard because it looks low-impact.

**FPs (reject):**

- The discard is explicitly documented as intentional (a comment naming why the error is safe to ignore, e.g. `_ = w.Close() // best-effort flush, already logged above`).
- The call is a best-effort write to a purely cosmetic/diagnostic sink (a `fmt.Println` to stdout) with no correctness or security requirement. Do **not** wave away a discarded error from a persisted, audit, integrity, or network-facing sink.
- The discard is inside a `defer` — that shape is `DEFERERR` (defer-closure-hazards cluster), not this pass.

**Patch:** check the error and either propagate it (`return fmt.Errorf("...: %w", err)`) or handle it explicitly with a documented rationale.

**Search patterns:**

```
_\s*=\s*\w[\w.]*\(
^\s*\w[\w.]*\([^)]*\)\s*$
```
