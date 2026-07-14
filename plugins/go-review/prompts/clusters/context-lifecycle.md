---
name: cluster-context-lifecycle
kind: cluster
consolidated: false
covers:
  - context-cancel-leak       # CTXLEAK
  - context-key-collision     # CTXKEYTYPE
  - context-stored-in-struct  # CTXSTORED
  - context-done-ignored      # CTXIGNORED
---

# Cluster: context.Context lifecycle

`context.Context` is Go's idiomatic cancellation/deadline-propagation mechanism, and `go vet`'s `lostcancel` analyzer already catches the most trivial case (a `cancel` func from `context.WithCancel` never called on any path). This cluster covers what `lostcancel` misses: conditional/error-path cancel leaks, the `context.Value` key-collision footgun, `Context` stored where it shouldn't be, and cancellation signals nobody actually checks.

ID prefixes: `CTXLEAK`, `CTXKEYTYPE`, `CTXSTORED`, `CTXIGNORED`.

## Phase A

```
rg seed: "context\.With(Cancel|Timeout|Deadline)\("        # cancel-func-producing constructors
rg seed: "\bcancel\s*:?=|\b_, cancel\s*:?="                 # cancel variable bindings (trace to a defer/call site)
rg seed: "context\.WithValue\("                              # context.Value writes — check key type
rg seed: "\.Value\(\s*\""                                    # context.Value reads keyed by a string literal (immediate red flag)
rg seed: "type\s+\w+Key\s+(struct|int|string)"               # candidate unexported key types (good pattern) vs exported/builtin (bad)
rg seed: "context\.Context\s*$|context\.Context\s*//|context\.Context\s*\`"  # struct field of type context.Context (heuristic: field declarations)
rg seed: "for\s*\{|for\s+\w"                                 # loop bodies — check whether ctx.Done()/ctx.Err() appears inside
rg seed: "ctx\.Done\(\)|ctx\.Err\(\)"                        # cancellation-check sites present
```

Run finders in declared order.

## Deconfliction

- `CTXLEAK` is about the `cancel` **function** never being called on some path (resource/goroutine leak) — distinct from `CTXIGNORED`, which is about a *long-running loop* never consulting `ctx.Done()`/`ctx.Err()` even though a valid `ctx` was passed in and its cancel function elsewhere is called correctly.
- `CTXKEYTYPE` fires only on the **write** side (`context.WithValue` with a built-in-typed key) or a **read** using a raw string/int literal directly — a read using a properly-typed unexported key constant is not a finding even if a sibling package also defines its own unrelated key type.
- `CTXSTORED` is a design smell (a struct holding a `context.Context` field instead of receiving it as the first parameter of each method) — file it once per struct definition, not once per field access.
