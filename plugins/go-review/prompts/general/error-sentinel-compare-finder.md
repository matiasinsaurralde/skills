---
name: error-sentinel-compare-finder
description: Detects sentinel-error comparison via == or != instead of errors.Is/errors.As, which breaks once the error is wrapped
---

**Finding ID Prefix:** `ERRSENTINEL`.

**Bug shape:** `if err == sql.ErrNoRows` or `if err != io.EOF` — direct equality against a sentinel error value. This works only as long as nobody ever wraps the error with `fmt.Errorf("...: %w", err)` upstream; once wrapped, the comparison silently and permanently fails, changing program behavior (e.g. a "not found" case falls through to a generic error path).

**Gates:**

1. `err == <sentinel>` or `err != <sentinel>` where `<sentinel>` is a package-level `var Err... = errors.New(...)` / stdlib sentinel (`sql.ErrNoRows`, `io.EOF`, `os.ErrNotExist`, etc.) or a project-defined sentinel.
2. No prior `errors.Is`/`errors.As` guarding the same check on the same variable in the same function.

**FPs (reject):**

- The comparison is against a genuinely un-wrappable sentinel by design contract (rare — confirm the codebase never wraps this specific error anywhere before excusing it).
- `io.EOF` comparisons in the specific, idiomatic `Read`-loop shape (`n, err := r.Read(buf); if err == io.EOF { break }`) immediately after the read call, matching the stdlib's own documented `io.Reader` contract — this exact idiom is what the stdlib itself uses and is not a security-relevant bug, though still worth a LOW note if the codebase wraps errors elsewhere inconsistently.

**Patch:** replace `err == sentinel` with `errors.Is(err, sentinel)`, and any type assertion `err.(*MyError)` with `errors.As(err, &target)`.

**Search patterns:**

```
err\s*==\s*\w+\.Err\w+
err\s*!=\s*\w+\.Err\w+
err\s*==\s*io\.EOF
\w+\.\([*]?\w+Error\)
```
