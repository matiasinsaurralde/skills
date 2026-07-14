---
name: inconsistent-lock-field-finder
description: Detects a struct field read or written without holding the mutex that guards its sibling fields
---

**Finding ID Prefix:** `RACEFIELD`.

**Bug shape:** A struct has a `sync.Mutex`/`sync.RWMutex` field and a documented (or clearly-intended) convention that it guards certain other fields, but at least one access site reads or writes one of those fields without holding the lock — usually because that one access site was added later, or is on a "fast path" someone assumed didn't need it.

**Gates:**

1. The struct has at least one field consistently accessed under `.Lock()`/`.RLock()` at most call sites (establishing the convention).
2. At least one other access site to the **same field** (or a field with an equivalent invariant) has no surrounding lock/unlock in its call path.
3. The struct/field is reachable from more than one goroutine (check the concurrency-goroutines cluster's inventory if available, or confirm via `go func`/method calls from multiple spawn sites).

**FPs (reject):**

- The unlocked access happens only during single-threaded construction/initialization before the value is ever shared.
- The field is documented as intentionally lock-free (e.g., it's itself a `sync/atomic` value, or immutable after construction) — that's a different, non-buggy pattern.
- The access is a `sync.RWMutex.RLock()`'d read where only reads (never writes) need protection and this really is a read-only path — but confirm no writer exists anywhere without the corresponding `Lock()`.

**Patch:** wrap the unlocked access in the same `Lock()`/`Unlock()` (or `RLock()`/`RUnlock()` for read-only) pattern used at the other access sites, or move the field into a small helper type whose methods enforce the lock internally.

**Search patterns:**

```
sync\.(Mutex|RWMutex)\b
\.Lock\(\)|\.Unlock\(\)|\.RLock\(\)|\.RUnlock\(\)
```
