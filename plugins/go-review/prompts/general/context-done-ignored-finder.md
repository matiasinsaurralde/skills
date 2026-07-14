---
name: context-done-ignored-finder
description: Detects a long-running loop that receives a valid context but never checks ctx.Done()/ctx.Err(), so cancellation never takes effect
---

**Finding ID Prefix:** `CTXIGNORED`.

**Bug shape:** A function accepts a `context.Context` parameter (implying it supports cancellation/timeout) but its main work loop never selects on `ctx.Done()` or checks `ctx.Err()` between iterations — the context is threaded through as a parameter (satisfying the type signature/convention) but functionally ignored. Callers who cancel or time out the context see no effect: the loop runs to completion regardless, which under `REMOTE`/`LOCAL_UNPRIVILEGED` threat models with attacker-influenced loop bounds is a resource-exhaustion DoS the caller believed it had already mitigated via timeout.

**Gates:**

1. A function receives a `ctx context.Context` parameter.
2. The function contains a loop (`for`) or a chain of blocking operations whose total duration is unbounded or attacker-influenced.
3. Neither `ctx.Done()` (in a `select`) nor `ctx.Err()` (checked between iterations) appears anywhere in that loop/chain.

**FPs (reject):**

- The function immediately delegates all its work to a single call that itself correctly respects the passed context (e.g., `return db.QueryContext(ctx, ...)`), rather than running its own loop.
- The loop is provably short and bounded (a small, fixed number of iterations with no per-iteration blocking) where responsiveness to cancellation mid-loop isn't meaningful.

**Patch:** add a `select { case <-ctx.Done(): return ctx.Err(); default: }` check (or a `case <-ctx.Done()` alongside a channel receive) at each loop iteration boundary, or use context-aware stdlib variants (`*Context` suffixed methods) for each blocking call inside the loop.

**Search patterns:**

```
context\.Context
for\s*\{|for\s+\w
ctx\.Done\(\)|ctx\.Err\(\)
```
