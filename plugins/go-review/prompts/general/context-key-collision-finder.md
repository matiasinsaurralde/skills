---
name: context-key-collision-finder
description: Detects context.Value keyed by a built-in type (string/int) instead of an unexported type, risking cross-package key collisions
---

**Finding ID Prefix:** `CTXKEYTYPE`.

**Bug shape:** `context.WithValue(ctx, "userID", id)` uses a bare `string` (or `int`) as the key. Because `context.Value` matches keys by `==` comparison across the *entire* interface{} key space, any other package that also happens to use the string `"userID"` as a context key will collide — either silently overwriting the intended value or being read back by the wrong consumer. The documented, safe idiom is an unexported type defined specifically to be a context key, which no other package can construct or collide with.

**Gates:**

1. `context.WithValue(ctx, key, value)` where `key` is a string or int literal, or a `var`/`const` of a built-in type (not a custom unexported type).
2. Or: `ctx.Value("someKey")` reading with a raw string/int literal directly.

**FPs (reject):**

- The key is a value of a custom unexported type defined specifically as a context key (`type contextKey int; const userIDKey contextKey = 0`) — this is the correct, collision-safe idiom.
- The key is an exported type from a well-known, single-owner package specifically designed for this purpose (rare, but some frameworks provide their own safe key types) — still note if the pattern otherwise looks fragile.

**Patch:** define an unexported type for context keys (`type ctxKey struct{}` or `type ctxKey int` with named constants) local to the package that owns the value, and use that type's values as keys instead of raw strings/ints.

**Search patterns:**

```
context\.WithValue\(
\.Value\(\s*"
type\s+\w+Key\s+(struct|int|string)
```
