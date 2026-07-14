---
name: deprecated-api-finder
description: Detects deprecated stdlib API usage with no direct attacker-controlled data flow (e.g. io/ioutil post-1.16)
---

**Finding ID Prefix:** `DEPRECAPI`.

**Bug shape:** Continued use of a stdlib package/function the Go project itself has deprecated in favor of a replacement — most commonly `io/ioutil` (deprecated since Go 1.16 in favor of `io`/`os` equivalents). Not usually an exploitable bug on its own, but signals a codebase that hasn't kept pace with stdlib guidance, and some deprecations exist specifically because the old API had a footgun (e.g., older crypto/tls defaults).

**Gates:**

1. Import of a package/function stdlib has marked `// Deprecated:` in its godoc.
2. No direct attacker-controlled data flow into the deprecated call that would make it a different, higher-severity bug class (in which case file under that class instead, per this cluster's Deconfliction rule).

**FPs (reject):**

- The deprecated API is used in a vendored/generated file the project doesn't control (check for `// Code generated ... DO NOT EDIT` headers).
- A newer, security-relevant deprecation (e.g. a crypto primitive) that has an attacker-reachable data flow — file that under the appropriate bug class instead of here.

**Patch:** replace with the documented stdlib successor (e.g. `io/ioutil.ReadFile` → `os.ReadFile`, `ioutil.ReadAll` → `io.ReadAll`).

**Search patterns:**

```
"io/ioutil"
\bioutil\.(ReadFile|WriteFile|ReadAll|ReadDir|TempFile|TempDir|NopCloser)\b
```
