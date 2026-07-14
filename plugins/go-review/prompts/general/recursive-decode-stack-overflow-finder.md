---
name: recursive-decode-stack-overflow-finder
description: Detects unbounded recursion in a custom UnmarshalJSON/UnmarshalYAML on nested attacker-controlled input, causing a stack-overflow crash
---

**Finding ID Prefix:** `RECURSEDECODE`.

**Bug shape:** A custom `UnmarshalJSON`/`UnmarshalYAML` method that recurses into nested values (a recursive data structure like a tree, a nested config format, or a self-referential type) with no depth limit. Deeply nested attacker-supplied input (thousands of nested arrays/objects) drives the recursion to a stack overflow, which in Go is an unrecoverable **fatal error** — `recover()` cannot catch it, unlike a normal panic — making it a reliable, unrecoverable crash of the entire process, not just the handling goroutine.

**Gates:**

1. A type defines a custom `UnmarshalJSON`/`UnmarshalYAML` method that calls itself (directly or through a chain: `A.UnmarshalJSON` decodes a field of type `A`, or type `B` whose own unmarshal calls back into `A`).
2. No explicit depth counter/limit is threaded through the recursive calls.
3. The input driving the recursion is attacker-controlled (a request body, an uploaded config file) with no upstream size/depth cap (e.g., a max-nesting-depth check before or during parsing).

**FPs (reject):**

- The recursive structure has a hardcoded, compile-time-bounded maximum depth that the type system enforces (rare, but e.g. a fixed-depth nested struct rather than a truly recursive/self-referential type).
- An explicit depth parameter is threaded through the recursive unmarshal calls and enforced with an error return once exceeded.
- The underlying decoder library (not custom code) already enforces a nesting-depth limit before invoking any custom `UnmarshalJSON` — verify this is actually true for the library in use, don't assume.

**Patch:** thread an explicit depth counter through the recursive unmarshal calls (e.g., via a context value or an accumulator parameter) and return an error once a configured maximum depth is exceeded, rather than recursing unboundedly.

**Search patterns:**

```
func\s*\([^)]*\)\s*UnmarshalJSON\(
func\s*\([^)]*\)\s*UnmarshalYAML\(
```
