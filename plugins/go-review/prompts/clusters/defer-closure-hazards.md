---
name: cluster-defer-closure-hazards
kind: cluster
consolidated: false
covers:
  - loop-var-capture       # LOOPCAPTURE
  - defer-in-loop          # DEFERLOOP
  - defer-order-hazard     # DEFERORDER
  - defer-error-discarded  # DEFERERR
---

# Cluster: defer and closure hazards

`defer` and closures are two of Go's most idiomatic constructs and two of its sharpest footguns. Go 1.22 changed loop-variable semantics (each iteration now gets its own variable), so `LOOPCAPTURE` is version-gated — check the module's `go` directive in `go.mod` before filing. `defer` runs at function-return time regardless of where it's written, which is exactly what makes it dangerous inside a loop (accumulates) or ahead of a required check (runs before you wanted it to).

ID prefixes: `LOOPCAPTURE`, `DEFERLOOP`, `DEFERORDER`, `DEFERERR`.

## Phase A

```
rg seed: "for\s+\w+\s*(,\s*\w+\s*)?:?=\s*range\b"      # range loops (candidate loop-var capture sites)
rg seed: "\bgo\s+func\s*\(\s*\)"                        # closures spawned as goroutines with no parameter capture
rg seed: "for\s+[^{]*\{[^}]*\bdefer\b"                  # defer textually inside a for-loop body (heuristic — confirm loop scope, not function scope)
rg seed: "\bdefer\s+\w[\w.]*\.Close\(\)"                # deferred Close() — check ordering vs. preceding err/nil checks
rg seed: "\bdefer\s+\w[\w.]*\("                          # any defer call site (candidate for discarded return value)
```

Run finders in declared order.

## Deconfliction

- `LOOPCAPTURE` applies only to modules whose `go.mod` `go` directive is **older than 1.22** (or to code that captures the loop variable by address/pointer, which is still a bug under 1.22's per-iteration semantics only if the *address* is compared across iterations — read the `go.mod` first). Do not file this against a 1.22+ module unless the capture is by explicit `&`-of-loop-variable pointer identity, which remains meaningful even under the new semantics.
- `DEFERLOOP` is a resource-accumulation concern (unbounded FD/lock holding), distinct from `DEFERORDER` (the defer runs at the *wrong point relative to a check*, not merely *late*). The same `defer f.Close()` site can be flagged as `DEFERLOOP` if it's inside a loop, and separately as `DEFERORDER` if it's placed before an error check on `f`.
- `DEFERERR` is specifically about an error return from the deferred call being dropped (`defer f.Close()` where `Close()` returns an `error` nobody captures) — distinct from `ERRIGN` (error-handling cluster) which covers non-deferred discards. File `DEFERERR` for any discard where the call site is a `defer`.
