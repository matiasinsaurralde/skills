---
name: defer-in-loop-finder
description: Detects a defer statement inside a loop, accumulating unbounded resource holds until the enclosing function returns
---

**Finding ID Prefix:** `DEFERLOOP`.

**Bug shape:** `defer` runs at **function** return, not at the end of the enclosing block. A `defer f.Close()` (or `.Unlock()`, or any cleanup call) written inside a `for` loop does not release the resource at the end of each iteration — it accumulates, holding every iteration's resource open until the whole function returns. On an attacker-controlled iteration count (processing N uploaded files, N request items), this is an unbounded resource leak (file descriptors, held locks) for the duration of the function.

**Gates:**

1. A `defer` statement appears lexically inside a `for` loop's body (not merely inside the enclosing function).
2. The deferred call releases a resource acquired earlier in the same loop iteration (file handle, lock, connection) — not a value that's cheap/bounded regardless of iteration count.
3. The loop's iteration count is plausibly attacker-influenced or otherwise unbounded (processing a request-supplied list, paginated results, etc.) rather than a small, fixed, internally-known count.

**FPs (reject):**

- The loop is provably bounded to a small, fixed number of iterations known at compile time (e.g., iterating over a hardcoded slice of 3 config keys).
- The "resource" is trivial to hold for the function's lifetime (e.g., deferring a no-op cleanup function with no real resource).

**Patch:** wrap the loop body in an anonymous function (or extract a helper function) so `defer` inside it triggers at the end of each iteration: `for _, item := range items { func() { f, _ := open(item); defer f.Close(); process(f) }() }`.

**Search patterns:**

```
for\s+[^{]*\{[^}]*\bdefer\b
```
