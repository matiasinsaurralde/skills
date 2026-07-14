---
name: lazy-init-race-finder
description: Detects a check-then-initialize lazy-init pattern racing across goroutines without sync.Once
---

**Finding ID Prefix:** `LAZYINITRACE`.

**Bug shape:** `if x == nil { x = expensiveInit() }` (or an equivalent "if not yet set, set it now" check) reachable from more than one goroutine with no `sync.Once`, mutex, or atomic CAS guarding it. Two goroutines can both observe `x == nil`, both run `expensiveInit()`, and both write — at best wasted work, at worst a data race on `x` itself (`go test -race` will flag the race; this pass flags the pattern statically).

**Gates:**

1. A conditional check on a shared (package-level, struct-field, or captured-by-multiple-closures) variable being nil/zero, followed by an assignment to that same variable inside the `if` body.
2. No `sync.Once.Do(...)`, no mutex held across the whole check-and-set, and no `sync/atomic` compare-and-swap.
3. The variable is reachable from more than one goroutine (package-level vars and struct fields shared across a request-handling goroutine pool are the common case).

**FPs (reject):**

- The check-and-set is provably confined to program startup before any goroutine spawn (`init()` or the first lines of `main()` before any `go` statement).
- `sync.Once` (or an equivalent CAS loop) already guards the pattern.

**Patch:** replace with `sync.Once.Do(func() { x = expensiveInit() })`, or use `sync/atomic`'s `CompareAndSwap` for a lock-free variant.

**Search patterns:**

```
if\s+\w+\s*==\s*nil\s*\{
sync\.Once\b
\.Do\(
```
