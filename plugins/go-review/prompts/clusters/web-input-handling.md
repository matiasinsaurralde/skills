---
name: cluster-web-input-handling
kind: cluster
consolidated: false
covers:
  - ssrf                        # SSRF
  - path-traversal               # PATHTRAVERSAL
  - open-redirect                # OPENREDIRECT
  - template-autoescape-bypass   # TEMPLATEXSS
  - http-no-timeout               # HTTPNOTIMEOUT
---

# Cluster: Web input handling

Go's `net/http` and templating stdlib make the safe path easy (`html/template` auto-escapes; `http.Client{}`'s zero value has no timeout by default, which is the trap) — this cluster covers the classic web-input bug shapes reframed for Go's specific APIs and idioms (`net/http`, `net/url`, `path/filepath`, `html/template` vs `text/template`), plus common router frameworks (`gin`, `echo`, `chi`) where the same shapes recur with different handler signatures.

ID prefixes: `SSRF`, `PATHTRAVERSAL`, `OPENREDIRECT`, `TEMPLATEXSS`, `HTTPNOTIMEOUT`.

## Phase A

```
rg seed: "http\.(Get|Post|PostForm|Head)\(|http\.NewRequest\("   # outbound HTTP call sites
rg seed: "filepath\.Join\(|path\.Join\("                          # path construction (candidate traversal sinks)
rg seed: "http\.Redirect\("                                        # redirect sinks
rg seed: "\"text/template\""                                       # text/template import (candidate autoescape bypass if used for HTML)
rg seed: "\"html/template\""                                       # html/template import (safe default — confirm it's actually the one used for HTML output)
rg seed: "http\.Client\{|&http\.Client\{"                          # http.Client construction — check for Timeout field
rg seed: "http\.Server\{|&http\.Server\{"                          # http.Server construction — check for ReadTimeout/WriteTimeout/IdleTimeout
rg seed: "http\.ListenAndServe\("                                   # bare ListenAndServe (no way to set server timeouts through this call)
```

Run finders in declared order.

## Deconfliction

- `SSRF` and `OPENREDIRECT` both start from "attacker-controlled URL/string" but differ in the sink: `SSRF` is the app itself making the request server-side; `OPENREDIRECT` is the app telling the *client's browser* to navigate there via a `Location` header. Same source, different sink — file separately if both sinks are reachable from the same tainted value.
- `PATHTRAVERSAL` is Go's analogue of rust-review's `PATHJOIN` — same bug shape (attacker segment reaches `filepath.Join`/`path.Join` and escapes the intended root), Go-specific because `filepath.Clean`'s `..`-collapsing semantics and OS-specific separator handling differ from Rust's `Path` API.
- `TEMPLATEXSS` fires when `text/template` (no auto-escaping) is used to render a value that ends up served as `Content-Type: text/html` — not merely because `text/template` appears in the file (it's the correct choice for non-HTML output like email bodies or config files).
- `HTTPNOTIMEOUT` covers both the client side (`http.Client{}` zero-value / no `Timeout` field set, or a per-request context with no deadline) and the server side (`http.Server{}` missing `ReadHeaderTimeout`/`ReadTimeout`/`WriteTimeout`, or bare `http.ListenAndServe` which offers no way to set them) — treat as one bug class, note which side in the Description.
