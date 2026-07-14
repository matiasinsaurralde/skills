---
name: cluster-unsafe-boundary
kind: cluster
consolidated: true
covers:
  - unsafe-pointer-conversion  # UNSAFECONV
  - unsafe-len-cap             # UNSAFELEN
  - unsafe-layout-assumption   # UNSAFELAYOUT
  - uintptr-retained           # UINTPTRUNSAFE
---

# Cluster: unsafe boundary

Go's type and memory safety end at the `unsafe` package. Unlike Rust's `unsafe` (which is common and has an established idiom of `// SAFETY:` comments), Go code that imports `unsafe` is rare, and every conversion must match one of the small number of patterns the [`unsafe.Pointer` godoc](https://pkg.go.dev/unsafe#Pointer) documents as valid — anything else is undefined behavior even if it happens to work today, because the Go runtime's garbage collector and stack-copying mechanism are free to invalidate assumptions the compiler can't see through `unsafe`.

ID prefixes: `UNSAFECONV`, `UNSAFELEN`, `UNSAFELAYOUT`, `UINTPTRUNSAFE`.

---

## Phase A — Build the unsafe-usage map (ONCE per run)

Run these scans and keep results as `unsafe_map` for all four passes:

```
rg seed: "\bunsafe\.Pointer\b"                          # any unsafe.Pointer conversion
rg seed: "\bunsafe\.(Sizeof|Alignof|Offsetof)\("        # layout introspection
rg seed: "\bunsafe\.(Slice|SliceData)\("                # slice construction/deconstruction from a raw pointer
rg seed: "\bunsafe\.(String|StringData)\("              # string construction/deconstruction from a raw pointer
rg seed: "\buintptr\b"                                  # uintptr declarations/conversions
rg seed: "reflect\.(SliceHeader|StringHeader)\b"        # pre-1.20 header-struct pattern (superseded by unsafe.Slice/String)
rg seed: "\*\(\*\w+\)\(unsafe\.Pointer\("               # pointer-cast-then-dereference idiom
```

For each `unsafe.Pointer` conversion, classify it against the [documented valid patterns](https://pkg.go.dev/unsafe#Pointer):
1. Conversion of a `*T1` to `unsafe.Pointer` to `*T2`, given `T1` and `T2` have compatible memory layouts.
2. Conversion of `unsafe.Pointer` to `uintptr` (not retained — see pattern 4) for arithmetic, then back to `unsafe.Pointer`, in the same expression, no intervening function call or GC-visible operation.
3. Conversion of a `reflect.Value.Pointer()`/`.UnsafeAddr()` result — same expression-only restriction as pattern 2.
4. `syscall.Syscall`'s `uintptr` arguments, which are special-cased by the compiler to keep the referent alive for the call's duration only.

Any conversion that doesn't match one of these four patterns — or matches pattern 2/3 but the `uintptr` is stored in a variable, passed to another function, or otherwise crosses a GC safepoint before being converted back — is a candidate finding. Record `unsafe_map[site] = { pattern_matched (or none), source_type, dest_type, crosses_safepoint }`.

Do NOT file findings during Phase A.

---

## Phase B — Run these passes in order, reusing `unsafe_map`

### 1. `UNSAFECONV` — Invalid unsafe.Pointer conversion

File when a conversion doesn't match any of the four documented valid patterns above. The most common real-world violation: converting between struct pointer types whose fields don't actually agree in layout (size, alignment, field order) — the compiler will not catch this, and it silently reads/writes garbage or adjacent memory.

**FPs to reject:**
- The conversion matches pattern 1 and `T1`/`T2` are verified identical or compatible layouts (e.g., both are `struct { x, y int32 }` with matching field order, or one is `[N]byte` sized exactly to the other).
- The conversion is inside a well-known, audited zero-copy helper (e.g., a vetted `unsafe`-based string/byte-slice conversion following the exact `unsafe.String(unsafe.SliceData(b), len(b))` idiom — Go 1.20+'s sanctioned replacement for the old header-struct trick).

**Patch:** prefer `unsafe.Slice`/`unsafe.String` (Go 1.20+) for the byte-slice/string zero-copy conversion idiom instead of hand-built `reflect.SliceHeader`/`StringHeader` manipulation; for struct reinterpretation, verify layouts with `unsafe.Sizeof`/`Offsetof` assertions or switch to explicit field-by-field copying.

### 2. `UNSAFELEN` — Attacker-influenced length/capacity in unsafe.Slice/String

`unsafe.Slice(ptr, len)` and `unsafe.String(ptr, len)` trust `len` completely — there is no bounds check against the underlying allocation. If `len` is derived from attacker-controlled input (a length field parsed from network/file/IPC data) without validation against the actual allocation size, the resulting slice/string permits out-of-bounds reads (and, for `unsafe.Slice`, out-of-bounds writes through the slice) that Go's normal bounds-checked slicing would have prevented.

**FPs to reject:**
- `len` is validated against a known-good bound (the source allocation's actual size) before the `unsafe.Slice`/`unsafe.String` call, on every path reaching it.
- `len` is a compile-time constant or derived only from trusted, internally-generated values.

**Patch:** validate the length against the source buffer's real size before constructing the unsafe slice/string; if the length comes from a wire format, bounds-check it against `len(sourceBuffer)` first.

### 3. `UNSAFELAYOUT` — Struct layout assumption broken by field changes

Code using `unsafe.Offsetof`, `unsafe.Sizeof`, or a hardcoded byte offset to reach into a struct's memory bypasses Go's field-name-based access — any later edit that adds, removes, reorders, or resizes a field (including compiler-inserted padding changes across Go versions or GOARCH) silently breaks the offset without a compile error. This is especially dangerous when the offset constant is hand-written rather than derived from `unsafe.Offsetof` at compile time.

**FPs to reject:**
- The offset is computed via `unsafe.Offsetof(s.Field)` at the point of use (self-correcting if the struct changes) rather than a hardcoded numeric literal.
- The struct is `//go:notinheap`/cgo-exported with an explicitly pinned, tested, and documented layout contract shared with the C side (still worth a LOW note if undocumented, but not the same severity as a silent internal assumption).

**Patch:** replace hardcoded offsets with `unsafe.Offsetof` expressions computed against the current struct definition, or better, avoid raw offset arithmetic entirely in favor of normal field access.

### 4. `UINTPTRUNSAFE` — uintptr retained across a GC safepoint

This is Go's memory-safety hazard with **no direct C or Rust analogue**: Go's garbage collector can move stack-allocated objects during a stack growth/shrink, and (in some GC implementations/phases) can relocate heap objects. A `uintptr` derived from a pointer is just an integer to the GC — it does not keep the referent alive and does not get updated if the referent moves. Storing a pointer-derived `uintptr` in a variable, struct field, or global, and converting it back to a pointer **after** any intervening function call, channel operation, or allocation, risks dereferencing a stale or moved address.

```go
p := unsafe.Pointer(&someStruct)
addr := uintptr(p)     // UINTPTRUNSAFE: addr is now just a number to the GC
doSomethingThatMayAllocate() // a GC cycle / stack growth can happen here
q := unsafe.Pointer(addr)   // stale — someStruct may have moved
```

**FPs to reject:**
- The `uintptr` round-trip happens in a single expression with no intervening call (matches valid pattern 2/3 above).
- The `uintptr` is a `syscall.Syscall` argument (pattern 4 — compiler keeps the referent alive for the call only).
- The `uintptr` is never converted back to a pointer — it's used purely as an opaque integer (address logging, hashing) and never dereferenced.

**Patch:** never store a pointer-derived `uintptr` across a call boundary; if you need a stable handle to a Go object for longer than one expression, use `runtime.Pinner` (Go 1.21+) to pin it, or restructure to avoid the round-trip entirely (e.g., pass the typed pointer, not a `uintptr`).

---

## Deconfliction

Report only one finding per `(path, line)`. Priority (higher wins):

1. `UINTPTRUNSAFE` > `UNSAFECONV` when the unsound conversion is specifically the safepoint-crossing round-trip — name the more specific bug.
2. `UNSAFELEN` > `UNSAFECONV` when the site is an `unsafe.Slice`/`unsafe.String` call with an unvalidated length — file the length-trust issue, not a generic conversion note.
3. `UNSAFELAYOUT` is orthogonal — report alongside any of the above if both apply to the same site (a bad offset feeding an unvalidated `unsafe.Slice` length is both `UNSAFELAYOUT` and `UNSAFELEN`).

---

## Token-economy reminder

All four passes operate on the same `unsafe_map`. Build it ONCE; do not re-search `unsafe.Pointer`, `uintptr`, or offset patterns per pass.
