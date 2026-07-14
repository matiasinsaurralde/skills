---
name: typed-nil-interface-finder
description: Detects a typed nil pointer stored in an interface value, making `!= nil` true even though the underlying pointer is nil
---

**Finding ID Prefix:** `NILINTERFACE`.

**Bug shape:** Go's most-cited gotcha. An interface value is a `(type, value)` pair; a nil `*T` assigned to an interface variable produces an interface with a non-nil `type` and a nil `value` — comparing that interface to `nil` returns `false`. Most common real-world shape: a function declares `var p *MyError` (nil by default), conditionally sets `p` only on some paths, and unconditionally `return p` from a function whose declared return type is the `error` interface — every caller's `if err != nil` is now true even when nothing went wrong.

```go
func doThing() error {
    var p *MyError // nil
    if somethingWentWrong {
        p = &MyError{...}
    }
    return p // NILINTERFACE: always non-nil as an `error` interface value, even when p is nil
}
```

**Gates:**

1. A function returns an interface type (commonly `error`, but any interface qualifies).
2. The returned value is a variable declared with a concrete pointer type (`var p *MyError` or `p := (*MyError)(nil)`), not the interface type itself.
3. There exists a path where the pointer variable is never reassigned away from nil before the return.

**FPs (reject):**

- The function's local variable is already declared as the interface type (`var err error`) rather than a concrete pointer type — assigning nil there and returning it is fine.
- The pointer is checked and explicitly converted (`if p == nil { return nil }; return p`) before return.

**Patch:** either declare the local variable as the interface type from the start, or add an explicit nil check before returning: `if p == nil { return nil }`.

**Search patterns:**

```
var\s+\w+\s+\*\w+
func\s+\w*\([^)]*\)\s*(\([^)]*\)\s*)?error\s*\{
return\s+\w+\s*$
```
