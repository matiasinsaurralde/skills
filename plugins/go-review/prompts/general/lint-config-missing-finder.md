---
name: lint-config-missing-finder
description: Detects go vet/staticcheck/golangci-lint absent from CI, forgoing cheap mechanical bug-class coverage
---

**Finding ID Prefix:** `LINTCONFIG`.

**Bug shape:** `go vet` ships with the toolchain and catches `copylocks`, `lostcancel`, `printf` format-string mismatches, and several other mechanical bug shapes for free — this cluster's own `LOCKCOPY`/`CTXLEAK` passes exist specifically to go *beyond* what vet catches, not to replace it. A codebase with no `go vet`/`staticcheck`/`golangci-lint` step in CI is missing that free first line of defense.

**Gates:**

1. No CI workflow file (`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Makefile` test target, etc.) invokes `go vet`, `staticcheck`, or `golangci-lint`.
2. `go vet` runs as part of `go test` automatically for the packages under test, but only if `go test ./...` is actually run in CI — verify that too before concluding vet coverage is entirely absent.

**FPs (reject):**

- `go vet`/`staticcheck`/`golangci-lint` is present but configured with an overly permissive exclude list that disables security-relevant checks (`copylocks`, `printf`) — that's still worth flagging, but describe it as "lint present but weakened," not "lint missing."
- `go test ./...` runs in CI (which implicitly runs `go vet` on tested packages) even without an explicit `go vet` invocation — this counts as partial coverage; note the gap for untested packages specifically.

**Patch:** add `go vet ./...` and `staticcheck ./...` (or `golangci-lint run`) as a required CI step.

**Search patterns:** `Read` CI workflow files directly; `rg seed: "\.golangci\.ya?ml|\.golangci\.toml"` for config presence.
