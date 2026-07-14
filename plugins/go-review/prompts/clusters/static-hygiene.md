---
name: cluster-static-hygiene
kind: cluster
consolidated: false
covers:
  - gomod-version-stale  # GOMODVER
  - lint-config-missing  # LINTCONFIG
  - deprecated-api       # DEPRECAPI
---

# Cluster: Static hygiene

These are defense-in-depth / hardening-gap findings, not exploitable vulnerabilities in themselves — the same role `rust-review`'s `static-hygiene` cluster plays for Cargo lint config and MSRV. Always `TRUE_POSITIVE` at `LOW` severity when genuinely present (see the FP-judge's hardening-gap rule); the value is surfacing configuration debt an auditor would otherwise have to hunt for manually.

ID prefixes: `GOMODVER`, `LINTCONFIG`, `DEPRECAPI`.

## Phase A

```
Read: go.mod                                    # `go` directive version
rg seed: "\.golangci\.ya?ml|\.golangci\.toml"    # golangci-lint config presence (path, not content match)
Read: any CI workflow file (.github/workflows/*.yml, .gitlab-ci.yml) for `go vet`/`golangci-lint`/`staticcheck` invocation
rg seed: "\"io/ioutil\"" # deprecated post-1.16 package
rg seed: "\bioutil\.(ReadFile|WriteFile|ReadAll|ReadDir|TempFile|TempDir|NopCloser)\b"
rg seed: "\bmath/rand\"" # bare (non-crypto, non-v2) rand import — flag only if used for a security-relevant value elsewhere (that's WEAKRAND territory in a future crypto cluster; here just note staleness if v2 is available and unused)
```

Run finders in declared order.

## Deconfliction

- These three bug classes are independent and file-level / manifest-level by nature — set `function: (file-level)` per the worker protocol's file-level-finding rule.
- `DEPRECAPI` here is scoped to **deprecated stdlib APIs with no direct attacker-controlled data flow** (e.g. `io/ioutil` post-Go-1.16). A deprecated API that **is** reachable from untrusted input with a concrete impact belongs in whichever bug-shaped cluster actually matches the impact (e.g. a deprecated crypto primitive is a crypto-misuse concern, not static-hygiene) — don't use this pass as a catch-all for every deprecated call.
