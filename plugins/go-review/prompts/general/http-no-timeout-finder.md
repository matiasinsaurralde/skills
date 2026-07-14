---
name: http-no-timeout-finder
description: Detects http.Client or http.Server construction with no timeouts set, enabling slowloris-style resource exhaustion
---

**Finding ID Prefix:** `HTTPNOTIMEOUT`.

**Bug shape:** Two related but distinct shapes under one bug class:
1. **Client side:** `http.Client{}` (zero value) or `&http.Client{Transport: ...}` with no `Timeout` field set — a request to a slow or malicious server can hang the calling goroutine indefinitely (`http.DefaultClient` has the same problem and is a common accidental default).
2. **Server side:** `http.Server{}` missing `ReadTimeout`/`ReadHeaderTimeout`/`WriteTimeout`/`IdleTimeout`, or bare `http.ListenAndServe(addr, handler)` (which offers no way to set any server timeout at all) — a slowloris-style client that trickles bytes can hold a connection (and its goroutine) open indefinitely, exhausting the server's connection/goroutine budget.

**Gates:**

1. Client: `http.Client{...}` construction (or use of `http.DefaultClient`/`http.Get`/`http.Post` package-level helpers, which use `DefaultClient` under the hood) with no `Timeout` field and no per-request `context.WithTimeout` wrapping the call.
2. Server: `http.Server{...}` construction missing one or more of `ReadTimeout`, `ReadHeaderTimeout`, `WriteTimeout`, `IdleTimeout`; or a bare `http.ListenAndServe`/`http.ListenAndServeTLS` call (structurally cannot set timeouts).

**FPs (reject):**

- Client: every call site using this client wraps its `context.Context` with `context.WithTimeout`/`WithDeadline`, providing an equivalent bound even without the `Client.Timeout` field.
- Server: the process sits behind a reverse proxy/load balancer that already enforces its own timeouts on client connections before they reach this server (still worth a LOW note as defense-in-depth, since the Go process itself remains exposed to any direct-connection path).

**Patch:** set `http.Client{Timeout: N}` for outbound clients, or thread a `context.WithTimeout` through every request; construct `http.Server{ReadHeaderTimeout: N, ReadTimeout: N, WriteTimeout: N, IdleTimeout: N}` explicitly instead of `http.ListenAndServe`.

**Search patterns:**

```
http\.Client\{|&http\.Client\{
http\.Server\{|&http\.Server\{
http\.ListenAndServe\(
http\.DefaultClient\b
```
