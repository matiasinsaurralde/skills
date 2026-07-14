# go-review

Go security code review plugin. Bug-class coverage is grounded in Go's specific concurrency model (goroutines, channels, `sync`), the `context.Context` cancellation idiom, `unsafe`/cgo boundary rules, and common `net/http`/`encoding/json` misuse — not a port of C/Rust memory-safety bug classes, which mostly don't apply to a garbage-collected, bounds-checked language. Orchestration matches `c-review`/`rust-review`.

## Usage

Invoke with `/go-review:go-review`. The skill will prompt for:

- **Threat model** (`REMOTE` / `LOCAL_UNPRIVILEGED` / `BOTH`)
- **Worker model** (`haiku` / `sonnet` / `opus`)
- **Severity filter** (`all` / `medium` / `high`)
- **Scope subpath** (optional — defaults to whole repo)

Findings + SARIF are written to `$(pwd)/.go-review-results/<iso-timestamp>/`.

## Overview

Inputs (`AskUserQuestion`): threat model, scope subpath (optional), worker model, severity filter.

From these inputs the orchestrator detects Go capability flags (`has_unsafe`, `has_cgo`, `has_concurrency`, `has_context`, `has_reflection`, `has_net_http`, `has_serialization`, `has_crypto`) over the scope and selects clusters from `prompts/clusters/manifest.json`. Each cluster groups related bug classes anchored on a shared mental model and runs as one parallel worker.

The planner caps each **non-consolidated** worker at four passes, splitting larger
clusters into `-1`/`-2`/… chunks. **Consolidated clusters (`unsafe-boundary`,
`concurrency-goroutines`) are never chunked** — one worker owns the whole cluster
so its shared Phase-A inventory is built once and grounds every phase.

Always-on clusters:

- **error-handling** — discarded/ignored errors (ERRIGN), sentinel-error comparison via `==`/`!=` instead of `errors.Is`/`errors.As` (ERRSENTINEL), error wrapping that loses type information (`%v` instead of `%w`) (ERRWRAP), `recover()` misuse — missing `defer` or swallowing a corrupted-state panic (PANICRECOVER).
- **nil-interface-pitfalls** — typed nil stored in an interface making `err != nil` true (NILINTERFACE), nil map write panics (NILMAPWRITE), nil pointer dereference on an unchecked optional field from JSON/proto unmarshal (NILDEREF).
- **defer-closure-hazards** — pre-1.22 loop-variable capture in a closure/goroutine (LOOPCAPTURE), `defer` inside a loop accumulating unbounded resource holds (DEFERLOOP), `defer` scheduled before a required nil/error check (DEFERORDER), deferred call's error silently discarded (DEFERERR).
- **static-hygiene** — stale `go.mod` version (GOMODVER), `go vet`/`staticcheck` absent from CI (LINTCONFIG), deprecated stdlib API usage (DEPRECAPI).

Concurrency clusters (gated on `has_concurrency`):

- **concurrency-goroutines** (consolidated) — goroutine leak with no reachable cancellation path (GOROUTINELEAK), mismatched channel send/receive deadlock (CHANDEADLOCK), `sync.WaitGroup.Add` called inside the spawned goroutine (WGRACE), double channel close or send-on-closed (CHANCLOSE), unbounded blocking channel op with no select/timeout (CHANBLOCK).
- **concurrency-data-race** — everything `go vet`'s `copylocks` doesn't catch: `sync.Mutex`/`RWMutex` copied by value after first use (LOCKCOPY), a field read/written outside the lock guarding its siblings (RACEFIELD), lazy-init race without `sync.Once` (LAZYINITRACE), concurrent map read/write (MAPRACE), a `sync/atomic` value also accessed non-atomically elsewhere (ATOMICMIX).

Other conditional clusters:

- **unsafe-boundary** (`has_unsafe`, consolidated) — invalid `unsafe.Pointer` conversion outside the documented valid patterns (UNSAFECONV), attacker-influenced length in `unsafe.Slice`/`unsafe.String` (UNSAFELEN), struct-layout assumption broken by field changes (UNSAFELAYOUT), a `uintptr` retained across a GC safepoint — Go's memory-safety hazard with no direct C/Rust analogue (UINTPTRUNSAFE).
- **context-lifecycle** (`has_context`) — `context.WithCancel`/`WithTimeout` cancel function never called on all paths (CTXLEAK), `context.Value` key using a built-in type instead of an unexported type (CTXKEYTYPE), `context.Context` stored in a struct field instead of threaded as a parameter (CTXSTORED), long-running loop that never checks `ctx.Done()`/`ctx.Err()` (CTXIGNORED).
- **web-input-handling** (`has_net_http`) — SSRF via unvalidated outbound URL (SSRF), path traversal via `filepath.Join` (PATHTRAVERSAL), open redirect (OPENREDIRECT), `text/template` used for HTML output losing auto-escaping (TEMPLATEXSS), `http.Client`/`http.Server` missing timeouts (HTTPNOTIMEOUT).
- **deserialization-safety** (`has_serialization`) — JSON mass assignment via exported struct fields (JSONMASSASSIGN), custom XML entity map reintroducing entity expansion (XMLENTITY), `encoding/gob` decoding attacker bytes into an interface type (GOBTYPE), unbounded recursion in a custom `UnmarshalJSON`/`UnmarshalYAML` (RECURSEDECODE).

Same orchestration as `c-review`/`rust-review`: workers spawn foreground (one message per wave of ≤16 workers, after an optional cache primer), write markdown-with-YAML-frontmatter finding files, then a dedup-judge merges duplicates, then an fp-judge assigns `fp_verdict` / `severity` / `attack_vector` / `exploitability`. A report safety net then runs: SARIF is regenerated unconditionally, and the orchestrator writes `REPORT.md` itself if the fp-judge failed to.

## Architecture

```
coordinator: write context.md → build_run_plan.py → TaskCreate × M
          → spawn primer (foreground) → spawn M workers (parallel)
          → classify Phase-7 outcomes + write findings-index.txt
          → dedup-judge → fp-judge → report safety net (SARIF + REPORT.md) → return REPORT.md
```

| Subagent type | Purpose | Tool set |
|---|---|---|
| `go-review:go-review-worker` | Run assigned cluster, write findings | Read, Write, Edit, Bash |
| `go-review:go-review-dedup-judge` | Merge duplicates (runs **first**) | Read, Write, Edit, Glob |
| `go-review:go-review-fp-judge` | FP + severity + final reports (runs **second**) | Read, Write, Edit, Bash |

In current Claude Code an agent granted `Bash` is not also granted the dedicated `Glob`/`Grep` tools (the harness expects `find`/`grep`/`rg` via `Bash`). So the worker and fp-judge search and resolve paths with `Read`/`Bash`, running the ripgrep-syntax prompt seeds through `rg`; only the dedup-judge — which holds no `Bash` — uses `Glob`.

## Output directory layout

Default: `$(pwd)/.go-review-results/<iso-timestamp>/`. Contains:

- `context.md` — resolved threat model, severity filter, scope, capability flags, `go.mod` status
- `plan.json` — selected clusters + rendered worker spawn prompts (one per parallel worker)
- `worker-prompts/` — verbatim spawn prompts, one per worker (+ optional `cache-primer.txt`)
- `findings/` — one markdown file per finding (`<PREFIX>-NNN.md` with YAML frontmatter)
- `findings-index.d/` — per-worker shards listing finding paths (survive an orchestrator crash)
- `findings-index.txt` — canonical sorted list of every finding file on disk (reconciled against the shards)
- `run-summary.md` — worker outcome table, retry/abort state, judge status
- `dedup-summary.md` — Tier 1–3 merge + Tier 4 related summary
- `fp-summary.md` — verdict counts and per-primary verdict table
- `REPORT.md` — human-readable final report grouped by severity, filtered per `severity_filter`
- `REPORT.sarif` — SARIF 2.1.0 export, idempotent (full overwrite), always written

## Not for

- Pure C/C++ codebases — use `c-review` instead.
- Rust codebases — use `rust-review` instead.
- Confirming an actual data race with certainty — this plugin flags *suspicious* unsynchronized access statically; run `go test -race` to confirm.
- Kubernetes/cloud-native YAML/RBAC misconfiguration in a Go tool's deployment — out of scope; this plugin covers the Go source only.

## Not yet covered (candidates for future clusters)

- `cgo`-boundary specific bug classes (pointer-passing rule violations, C-allocated memory leaks) — `has_cgo` is detected but no cluster yet consumes it.
- `reflect`-specific pitfalls (panics on unexported field access, unaddressable `Set` calls) — `has_reflection` is detected but no cluster yet consumes it.
- Weak-crypto-primitive / `math/rand`-for-security-tokens findings — `has_crypto` is detected but no cluster yet consumes it.
- SQL injection as its own cluster (currently out of scope; `database/sql` with parameterized queries is the safe default in Go, so prevalence is lower than in other ecosystems — revisit if evidence suggests otherwise).

## References

- [Go Vulnerability Database](https://vuln.go.dev/) — real advisories against stdlib and modules
- [Effective Go](https://go.dev/doc/effective_go) / [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Go Memory Model](https://go.dev/ref/mem)
- [`unsafe` package godoc](https://pkg.go.dev/unsafe) — valid `unsafe.Pointer` conversion patterns
- [cgo documentation](https://pkg.go.dev/cmd/cgo) — pointer-passing rules

## Authors
- [Trail of Bits](https://github.com/trailofbits)
