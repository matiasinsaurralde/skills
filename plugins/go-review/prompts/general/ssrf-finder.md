---
name: ssrf-finder
description: Detects an attacker-controlled URL reaching an outbound HTTP request without an allowlist or scheme/host validation
---

**Finding ID Prefix:** `SSRF`.

**Bug shape:** A URL (or a URL component: host, scheme, path) derived from user-controlled input (request parameter, form field, webhook config, uploaded file referencing a URL) reaches `http.Get`, `http.Post`, `http.NewRequest`, or an underlying `http.Client.Do` call with no validation against an allowlist of permitted hosts/schemes — letting an attacker make the server issue requests to internal services, cloud metadata endpoints (`169.254.169.254`), or arbitrary external hosts on the server's behalf.

**Gates:**

1. A value flowing into `http.Get(url)`, `http.Post(url, ...)`, `http.NewRequest(method, url, ...)`, or an equivalent client call's URL argument traces back to user-controlled input.
2. No validation of the resolved host/scheme against an allowlist (or a denylist of internal/link-local ranges, which is weaker but still a mitigating control worth noting) exists on the path between the input and the request.

**FPs (reject):**

- The URL's host component is validated against a fixed allowlist before the request (confirm the allowlist is actually enforced, not just declared and unused).
- The "attacker-controlled" part of the URL is only a path/query parameter appended to an otherwise-fixed, trusted base host, and the code correctly uses `url.URL` construction (not string concatenation) such that the attacker cannot override the host.

**Patch:** validate the parsed URL's scheme and host against an explicit allowlist before making the request; for internal service calls, avoid taking a full URL from user input at all — prefer selecting from a fixed set of internal endpoints by key.

**Search patterns:**

```
http\.(Get|Post|PostForm|Head)\(
http\.NewRequest\(
url\.Parse\(
```
