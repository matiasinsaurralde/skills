---
name: context-cancel-leak-finder
description: Detects a context.WithCancel/WithTimeout/WithDeadline cancel function never called on some path, leaking the context's internal goroutine
---

**Finding ID Prefix:** `CTXLEAK`.

**Bug shape:** `context.WithCancel`/`WithTimeout`/`WithDeadline` return a `cancel` function that **must** be called on every path out of the enclosing scope, even the success path, or the context's internal resources (and any goroutine selecting on its `Done()` channel) are held until the parent context is itself cancelled — which for a long-lived parent may be never. `go vet`'s `lostcancel` catches the trivial case (`cancel` never referenced at all); this pass looks for the subtler case where `cancel` is deferred on the happy path but an early return bypasses the defer, or `cancel` is called conditionally.

**Gates:**

1. A `context.With(Cancel|Timeout|Deadline)(...)` call whose returned `cancel` function is not unconditionally `defer`red immediately after the call.
2. At least one return path from the enclosing function does not call `cancel` before returning.

**FPs (reject):**

- `cancel` is `defer`red immediately after the `With*` call — this covers every return path uniformly, including panics.
- `cancel` is intentionally passed to another goroutine/struct that takes ownership of calling it later, and that ownership transfer is clearly reachable and correct (verify the receiving code actually calls it on every path, not just assume).

**Patch:** `defer cancel()` immediately after the `context.With*` call, before any other logic that could return early.

**Search patterns:**

```
context\.With(Cancel|Timeout|Deadline)\(
\bcancel\s*:?=|\b_, cancel\s*:?=
defer\s+cancel\(\)
```
