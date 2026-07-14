---
name: nil-map-write-finder
description: Detects a write to a nil map on an attacker-reachable path, panicking with "assignment to entry in nil map"
---

**Finding ID Prefix:** `NILMAPWRITE`.

**Bug shape:** A `map[K]V`-typed variable/field that may be nil (zero value, or a struct field left unset by `json.Unmarshal`/a constructor that doesn't initialize it) is written to (`m[k] = v`) without a preceding `make(map[K]V)` or nil check. Reading a nil map is safe in Go (returns the zero value); **writing** to one panics.

**Gates:**

1. A map-typed variable/field is written via index assignment (`m[k] = v`) or `delete(m, k)` is fine but assignment is not.
2. No `make(map[...])` initialization is guaranteed on every path reaching the write (check constructors, `init()`, and every call site that might pass an uninitialized struct).
3. The map's nil-ness is plausibly attacker-influenced (e.g., the containing struct came from `json.Unmarshal` of a request body where the map field was omitted, leaving it nil).

**FPs (reject):**

- The map is always initialized via `make()` in every constructor/path that can reach the write (verify all constructors, not just the common one).
- The write is guarded by a nil check that initializes the map first (`if m == nil { m = make(map[K]V) }`).

**Patch:** initialize the map with `make(map[K]V)` in the constructor/zero-value path, or add a nil-check-and-initialize guard immediately before the first write.

**Search patterns:**

```
map\[[^\]]+\][\w*\[\]]
\w+\[[^\]]+\]\s*=
make\(map\[
```
