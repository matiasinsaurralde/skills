---
name: mutex-copy-finder
description: Detects a sync.Mutex/RWMutex (or a struct embedding one) copied by value after first use, breaking mutual exclusion
---

**Finding ID Prefix:** `LOCKCOPY`.

**Bug shape:** A struct containing a `sync.Mutex`/`sync.RWMutex` field is passed by value (function parameter, return value, or assignment) after the lock has been used — the copy has its own independent lock state, so code operating on the copy no longer excludes the original's critical section, or vice versa.

**Gates:**

1. A struct type has a `sync.Mutex`/`sync.RWMutex` field (directly or via an embedded type).
2. A function/method has a **value** (non-pointer) receiver or parameter of that type, or the type is assigned/returned by value, and the lock is actually used (`.Lock()`/`.Unlock()`) somewhere reachable from that copy or its original.
3. The copy happens **after** the original could plausibly already be shared across goroutines (not merely a fresh, never-shared local value being copied once before any concurrent use begins).

**FPs (reject):**

- `go vet`'s `copylocks` check already flags the bare syntactic pattern (value receiver on a lock-holding type) — don't re-file a pure syntax-only instance with no plausible concurrent-use path; this pass's value-add is confirming *reachability* (the copy actually happens on a value that's concurrently shared), not re-discovering what vet already reports.
- The value is copied before the struct is ever shared across goroutines or before `.Lock()` is ever called (e.g., building up a zero-value struct during single-threaded initialization).

**Patch:** use a pointer receiver/parameter for any type embedding a lock, and never assign such a type by value once it's shared.

**Search patterns:**

```
sync\.(Mutex|RWMutex)\b
func\s*\([^)]*\s+\w+\s+\)
```
