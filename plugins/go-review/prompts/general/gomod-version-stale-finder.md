---
name: gomod-version-stale-finder
description: Detects a go.mod `go` directive far behind the toolchain or missing entirely
---

**Finding ID Prefix:** `GOMODVER`.

**Bug shape:** `go.mod`'s `go` directive pins the language version/semantics the module builds against (notably: pre-1.22 loop-variable capture semantics, generics availability at 1.18+, `min`/`max`/`clear` builtins at 1.21+). A stale directive means the module misses both security fixes in newer toolchains and semantic improvements that eliminate whole bug classes (like `LOOPCAPTURE`).

**Gates:**

1. `go.mod`'s `go` line is absent, or declares a version more than ~2 major Go releases behind the current stable release at time of review.
2. Cross-reference: if `LOOPCAPTURE` findings exist elsewhere in this run, note here that upgrading to 1.22+ would eliminate that entire bug class.

**FPs (reject):**

- The module intentionally targets an older Go version for a documented compatibility reason (e.g., a library required to support consumers on an older toolchain) — still worth a LOW note, but the rationale should be reflected in the finding.

**Patch:** bump the `go` directive in `go.mod` to a current, supported Go release and run the test suite to confirm no breakage.

**Search patterns:** `Read: go.mod` directly (no regex needed — this is a single-file, single-field check).
