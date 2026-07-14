---
name: loop-var-capture-finder
description: Detects a pre-Go-1.22 loop variable captured by reference in a closure or goroutine, observing the wrong/final value
---

**Finding ID Prefix:** `LOOPCAPTURE`.

**Bug shape (pre-1.22 semantics only):** `for _, item := range items { go func() { use(item) }() }` — before Go 1.22, `item` is one variable reused across all iterations; every spawned goroutine/closure captures the *same* variable, and by the time they run, the loop may have advanced (or finished), so most/all goroutines observe the final value instead of "their" iteration's value.

**Gates:**

1. **First check `go.mod`'s `go` directive.** If it declares `go 1.22` or later, this bug class does not apply to `for`/`range` loop variables (Go 1.22 gives each iteration its own variable) — do not file unless the capture is of an explicit `&loopVar` pointer whose *address* is compared/stored across iterations (rare; the per-iteration variable still has a distinct address under 1.22, so this sub-case is about code relying on pointer identity in a way that's still broken).
2. A `for`/`range` loop spawns a goroutine (`go func() {...}()`) or stores a closure (appending to a slice of `func()`) that references the loop variable directly, without taking it as a parameter or shadowing it with `:=` inside the loop body.

**FPs (reject):**

- The module's `go.mod` declares Go 1.22+.
- The closure/goroutine takes the loop variable as an explicit parameter (`go func(item Item) { use(item) }(item)`) or the loop body shadows it (`item := item`) before capture — both are the correct pre-1.22 idiom.

**Patch:** upgrade the module's Go version to 1.22+ if feasible, or pass the loop variable as an explicit parameter to the closure/goroutine.

**Search patterns:**

```
for\s+\w+\s*(,\s*\w+\s*)?:?=\s*range\b
\bgo\s+func\s*\(\s*\)
```
