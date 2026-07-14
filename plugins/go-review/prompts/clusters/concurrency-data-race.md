---
name: cluster-concurrency-data-race
kind: cluster
consolidated: false
covers:
  - mutex-copy               # LOCKCOPY
  - inconsistent-lock-field  # RACEFIELD
  - lazy-init-race           # LAZYINITRACE
  - map-concurrent-access    # MAPRACE
  - atomic-nonatomic-mix     # ATOMICMIX
---

# Cluster: Data races beyond `go vet`

`go vet`'s `copylocks` analyzer already flags the most mechanical shape of `LOCKCOPY` (a function receiver or parameter passed by value that contains a `sync.Mutex`). This cluster's job is everything vet does **not** attempt: whether a copied/uncopied lock is actually **reachable concurrently** with impact, inconsistent locking discipline across a struct's fields, lazy-init races, concurrent map access, and atomics that are only sometimes accessed atomically. Static analysis cannot prove a race the way `go test -race` can — treat every finding here as "suspicious, needs `-race` confirmation" and say so in the finding.

ID prefixes: `LOCKCOPY`, `RACEFIELD`, `LAZYINITRACE`, `MAPRACE`, `ATOMICMIX`.

## Phase A

```
rg seed: "sync\.(Mutex|RWMutex)\b"                     # lock field declarations
rg seed: "func\s+\([^)]*\s+\w+\s+\)"                   # value (non-pointer) receivers — cross-reference against structs holding a lock field
rg seed: "\.Lock\(\)|\.Unlock\(\)|\.RLock\(\)|\.RUnlock\(\)"  # lock/unlock call sites
rg seed: "sync\.Once\b"                                # explicit lazy-init guard present
rg seed: "if\s+\w+\s*==\s*nil\s*\{[^}]*=\s*"           # double-checked-locking-shaped lazy init without sync.Once
rg seed: "map\[[^\]]+\][\w*\[\]]"                      # map type declarations (cross with goroutine spawns from the concurrency-goroutines inventory)
rg seed: "sync\.Map\b"                                 # already-synchronized map type in use
rg seed: "sync/atomic|atomic\.(Int32|Int64|Uint32|Uint64|Bool|Value|Pointer)\b"  # atomic field/package usage
```

Run finders in declared order.

## Deconfliction

- `LOCKCOPY` vs `RACEFIELD`: `LOCKCOPY` is the lock *itself* being copied (breaking mutual exclusion for every field it guards); `RACEFIELD` is a specific field read/written **without** holding a lock that guards its siblings, even though the lock itself is never copied. File `LOCKCOPY` when the copy is the root cause.
- `LAZYINITRACE` is a special case of `RACEFIELD` narrow enough to deserve its own class (the classic "check nil, then initialize" race) — file `LAZYINITRACE` when the shape matches, not the more generic `RACEFIELD`.
- `MAPRACE` is independent of the others — Go's map implementation panics loudly on concurrent write detection (`fatal error: concurrent map writes`) rather than corrupting silently, which makes it a DoS-class finding as much as a correctness one; note both angles.
- `ATOMICMIX` fires only when the **same** memory location is reached through both an atomic accessor and a plain read/write on different paths — an atomic used consistently everywhere is not a finding.
