---
name: map-concurrent-access-finder
description: Detects a plain Go map read/written from multiple goroutines without synchronization
---

**Finding ID Prefix:** `MAPRACE`.

**Bug shape:** A plain `map[K]V` (not `sync.Map`) is written from more than one goroutine, or written from one goroutine while read from another, with no mutex/lock guarding the accesses. Go's map implementation detects concurrent writes at runtime and crashes the process with `fatal error: concurrent map writes` (or `concurrent map read and map write`) — this is a hard crash, not a recoverable panic, making it a reliable DoS if an attacker can trigger concurrent access on a request-handling path.

**Gates:**

1. A `map[K]V`-typed variable, field, or global is written (`m[k] = v`, `delete(m, k)`) or read (`m[k]`, `for range m`) from more than one goroutine's reachable code path.
2. No `sync.Mutex`/`sync.RWMutex` visibly guards every access site to that map.
3. The map is not `sync.Map` (which is designed for exactly this and needs no additional locking).

**FPs (reject):**

- Every access site is provably guarded by the same mutex.
- The map is only ever populated once at startup before any goroutine spawn, then only read (never written) afterward — concurrent reads of an unmodified map are safe.
- The type is `sync.Map`.

**Patch:** guard every access with a `sync.Mutex`/`sync.RWMutex`, or switch to `sync.Map` if the access pattern fits its documented use case (write-once/read-many, or disjoint key sets per goroutine).

**Search patterns:**

```
map\[[^\]]+\][\w*\[\]]
sync\.Map\b
\bgo\s+func\b|\bgo\s+\w+\(
```
