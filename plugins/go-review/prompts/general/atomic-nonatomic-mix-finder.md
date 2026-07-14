---
name: atomic-nonatomic-mix-finder
description: Detects a sync/atomic value also read or written through a plain, non-atomic access on another path
---

**Finding ID Prefix:** `ATOMICMIX`.

**Bug shape:** A field is sometimes accessed via `sync/atomic` (`atomic.AddInt64`, `atomic.LoadInt32`, or the typed `atomic.Int64`/`atomic.Bool`/etc. wrappers) and sometimes accessed as a plain field read/write elsewhere in the codebase. The atomic accessor provides no protection against a concurrent plain access to the same memory — this is a real data race even though half the accesses look correctly synchronized.

**Gates:**

1. A field/variable has at least one atomic accessor call (`atomic.Load*`, `atomic.Store*`, `atomic.Add*`, `atomic.CompareAndSwap*`, or a method on an `atomic.Int32`/`Int64`/`Uint32`/`Uint64`/`Bool`/`Value`/`Pointer` typed field).
2. The **same** field/variable is also read or written via a plain (non-atomic) expression somewhere else reachable from a different goroutine.

**FPs (reject):**

- The plain access is provably confined to single-threaded initialization before any goroutine spawns, or to code that runs strictly after all goroutines touching the atomic value have joined (e.g. after a `sync.WaitGroup.Wait()`).
- The "plain access" is actually reading the typed `atomic.Int64` wrapper's zero value in a context where atomicity doesn't matter (e.g., printing the struct's address, not its value).

**Patch:** make every access to the field go through the atomic accessor (or the typed `atomic.*` wrapper's methods) — never mix atomic and plain access to the same memory location.

**Search patterns:**

```
sync/atomic|atomic\.(Load|Store|Add|CompareAndSwap)\w*\(
atomic\.(Int32|Int64|Uint32|Uint64|Bool|Value|Pointer)\b
```
