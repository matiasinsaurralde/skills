---
name: defer-order-hazard-finder
description: Detects a defer scheduled before a required nil/error check, or cleanup stages deferred in the wrong relative order
---

**Finding ID Prefix:** `DEFERORDER`.

**Bug shape:** `defer resp.Body.Close()` written immediately after an HTTP call but before checking whether `err != nil` (in which case `resp` may be nil, and the deferred close panics), or two related `defer` cleanup calls scheduled in an order that unwinds incorrectly (defers run LIFO, so a `defer conn.Close()` scheduled before `defer tx.Rollback()` runs the rollback *after* the connection is already closed).

```go
resp, err := http.Get(url)
defer resp.Body.Close() // DEFERORDER: resp may be nil if err != nil
if err != nil {
    return err
}
```

**Gates:**

1. A `defer` call is scheduled on a value returned alongside an `error`, before that error has been checked on the same path.
2. Or: two or more `defer` statements whose LIFO unwind order contradicts the resources' actual dependency order (a later-acquired resource's cleanup must run before an earlier-acquired one's, per LIFO — check the acquisition order against the defer order).

**FPs (reject):**

- The `defer` target is provably non-nil regardless of the error (some APIs guarantee a non-nil return even on error — verify against the actual function's documented contract before excusing).
- The defer order already matches correct LIFO unwind for the resources' true dependency order.

**Patch:** check the error first, then `defer` the cleanup: `resp, err := http.Get(url); if err != nil { return err }; defer resp.Body.Close()`. For multi-resource cleanup order, schedule defers in the reverse of the desired unwind order (last acquired, first deferred-to-close).

**Search patterns:**

```
\bdefer\s+\w[\w.]*\.Close\(\)
\bdefer\s+\w[\w.]*\(
```
