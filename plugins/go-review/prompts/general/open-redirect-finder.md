---
name: open-redirect-finder
description: Detects an unchecked user-controlled value used as an HTTP redirect target
---

**Finding ID Prefix:** `OPENREDIRECT`.

**Bug shape:** A user-controlled value (a `redirect_to`/`next`/`returnUrl` query parameter, or a `Referer` header) is passed directly to `http.Redirect(w, r, target, ...)` with no validation that `target` is a same-site relative path or an allowlisted host — letting an attacker craft a link through the trusted domain that redirects victims to an attacker-controlled phishing site.

**Gates:**

1. User-controlled input flows into `http.Redirect`'s `url` argument (or a framework equivalent: `c.Redirect(...)` in gin/echo).
2. No validation that the target is a relative path (starts with a single `/`, not `//` which browsers treat as protocol-relative to an arbitrary host) or matches an allowlisted host.

**FPs (reject):**

- The target is validated to be a relative path only (rejecting absolute URLs and protocol-relative `//host` values) before the redirect.
- The target is validated against an explicit allowlist of permitted redirect hosts.

**Patch:** reject any redirect target that isn't a same-site relative path (starts with `/` but not `//`), or validate the host against an allowlist using `url.Parse` before redirecting.

**Search patterns:**

```
http\.Redirect\(
r\.(URL\.Query\(\)\.Get|FormValue)\("
```
