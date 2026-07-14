---
name: cluster-error-handling
kind: cluster
consolidated: false
covers:
  - error-ignored           # ERRIGN
  - error-sentinel-compare  # ERRSENTINEL
  - error-wrap-lost-type    # ERRWRAP
  - panic-recover-misuse    # PANICRECOVER
---

# Cluster: Error handling flow

Go forces every fallible call to return an explicit `error` — there is no exception mechanism forcing the caller to notice a failure. The entire cluster is about the many ways that discipline breaks down in practice: silently dropped errors, brittle error comparisons, wrapping that destroys the type information `errors.As` needs, and `recover()` used to paper over a panic instead of restoring a safe state.

ID prefixes: `ERRIGN`, `ERRSENTINEL`, `ERRWRAP`, `PANICRECOVER`.

## Phase A

```
rg seed: "_\s*=\s*\w[\w.]*\("        # `_ = someCall(...)` discarding a return (confirm one of the returns is an error)
rg seed: "if\s+err\s*!=\s*nil\s*\{\s*\}"  # empty error-handling block
rg seed: "==\s*\w+\.Err\w*\b|!=\s*\w+\.Err\w*\b"  # sentinel comparison via == / != instead of errors.Is
rg seed: "fmt\.Errorf\(.*%v.*,\s*err\)"  # %v loses the wrapping chain errors.Unwrap needs (should be %w)
rg seed: "fmt\.Errorf\(.*%w"          # %w wrapping present — confirm the wrapped error's type survives downstream errors.As use
rg seed: "\brecover\(\)"              # recover() call sites
rg seed: "defer\s+func\s*\(\)\s*\{[^}]*recover\("  # the idiomatic deferred-recover shape
```

Run finders in declared order.

## Deconfliction

- `ERRIGN` (a return value silently discarded) vs `DEFERERR` (defer-closure-hazards — an error discarded specifically inside a `defer`'d call): file `DEFERERR` when the discard is inside a `defer`, `ERRIGN` otherwise.
- `ERRSENTINEL` is about the comparison operator (`==`/`!=` instead of `errors.Is`/`errors.As`), independent of whether the error was wrapped — `ERRWRAP` is about the wrapping site itself losing type information. The same code path can have both: a `%v`-wrapped error compared with `==` downstream is `ERRWRAP` at the wrap site and `ERRSENTINEL` at the compare site — file both if both sites are distinct.
- `PANICRECOVER` covers both "recover with no defer" (never actually catches the panic — the goroutine still crashes) and "recover that swallows and continues past a corrupted invariant" — these are different code shapes but the same underlying `bug_class`; describe which sub-shape applies in the finding's Description.
