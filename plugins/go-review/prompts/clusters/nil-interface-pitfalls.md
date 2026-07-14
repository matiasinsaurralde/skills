---
name: cluster-nil-interface-pitfalls
kind: cluster
consolidated: false
covers:
  - typed-nil-interface  # NILINTERFACE
  - nil-map-write        # NILMAPWRITE
  - nil-pointer-deref    # NILDEREF
---

# Cluster: Nil pitfalls

Go's zero-value semantics are usually forgiving (a nil slice can be ranged over, a nil map can be read from) but three shapes reliably surprise even experienced Go engineers: a typed nil stored in an interface making `!= nil` lie, a write to a nil map panicking where a read would have silently returned the zero value, and a nil pointer dereference on a struct field that JSON/proto unmarshal left unset.

ID prefixes: `NILINTERFACE`, `NILMAPWRITE`, `NILDEREF`.

## Phase A

```
rg seed: "var\s+\w+\s+\*\w+"                       # typed pointer variable declarations (candidates for later interface assignment)
rg seed: "func\s+\w*\([^)]*\)\s*(\([^)]*\)\s*)?error\s*\{"  # functions returning a bare `error` (candidates for typed-nil-return bugs)
rg seed: "return\s+\w+\s*$"                        # bare-variable returns inside an error-returning function — check if the variable is a typed pointer
rg seed: "map\[[^\]]+\][\w*\[\]]"                   # map declarations (cross with nil-check presence before writes)
rg seed: "\w+\[[^\]]+\]\s*="                        # map/slice index-assignment sites (candidate nil-map writes)
rg seed: "json\.Unmarshal\(|proto\.Unmarshal\(|yaml\.Unmarshal\("  # deserialization sites (fields may be left nil/zero)
rg seed: "\.\w+\.\w+"                               # chained field access (candidate nil-deref on an optional nested struct)
```

Run finders in declared order.

## Deconfliction

- `NILINTERFACE` is specifically about a **typed nil pointer assigned to an interface variable** (most commonly a named error type returned as the `error` interface) — not any nil pointer in general. A plain `*T` that is nil and dereferenced directly is `NILDEREF`.
- `NILMAPWRITE` vs `NILDEREF`: a nil **map** write panics with `assignment to entry in nil map`; a nil **pointer** dereference panics with `invalid memory address or nil pointer dereference`. Different runtime panic, different bug_class — don't conflate a nil map bug with a nil pointer bug just because both are "nil something."
- `NILDEREF` on a field populated by `encoding/json`/`encoding/proto`/`gopkg.in/yaml` unmarshal is the highest-value shape: any optional/pointer field the schema allows to be absent is a candidate if the code accesses it without a nil check.
