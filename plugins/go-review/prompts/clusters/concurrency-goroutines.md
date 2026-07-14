---
name: cluster-concurrency-goroutines
kind: cluster
consolidated: true
covers:
  - goroutine-leak          # GOROUTINELEAK
  - channel-deadlock        # CHANDEADLOCK
  - waitgroup-race          # WGRACE
  - channel-close-misuse    # CHANCLOSE
  - channel-unbounded-block # CHANBLOCK
---

# Cluster: Goroutine and channel lifecycle

Go has no borrow checker and no automatic goroutine cleanup: a `go func(){...}()` with no reachable exit condition runs (and leaks its stack + captured state) for the life of the process. This cluster maps every goroutine spawn and channel operation in scope and audits the lifecycle each implies — this is Go's closest analogue to `rust-review`'s panic-DoS cluster: an unrecoverable resource leak instead of an unrecoverable panic, same DoS impact class.

ID prefixes: `GOROUTINELEAK`, `CHANDEADLOCK`, `WGRACE`, `CHANCLOSE`, `CHANBLOCK`.

---

## Phase A — Build the goroutine/channel inventory (ONCE per run)

Run these scans and keep results as `goroutine_map` for all five passes:

```
rg seed: "\bgo\s+func\s*\("                          # anonymous goroutine spawns
rg seed: "\bgo\s+[A-Za-z_][A-Za-z0-9_.]*\("           # named-function/method goroutine spawns
rg seed: "\bchan\s+[A-Za-z_\[\]*]"                    # channel type declarations
rg seed: "make\(\s*chan\b"                            # channel construction (capacity reveals buffering)
rg seed: "<-\s*\w|\w\s*<-"                            # channel send/receive operators
rg seed: "\bclose\(\s*\w+\s*\)"                       # explicit channel close
rg seed: "sync\.WaitGroup\b"                          # WaitGroup declarations
rg seed: "\.Add\(|\.Done\(\)|\.Wait\(\)"              # WaitGroup method calls
rg seed: "\bselect\s*\{"                              # select statements (may or may not have a default/timeout)
rg seed: "context\.With(Cancel|Timeout|Deadline)\("   # cancellation sources reachable from a spawn site
rg seed: "time\.After\(|time\.NewTimer\(|context\.Done\(\)" # timeout/cancellation signals available to a select
```

For each `go` statement, identify its **spawn site** (the enclosing function) and walk forward through the goroutine body: does it terminate on its own (loop with a break condition, single-shot task), or does it block on a channel/lock indefinitely? Record `goroutine_map[spawn_site] = { body_summary, blocking_ops[], has_cancellation_path, is_reachable_from_untrusted_input }`. For each channel, record every send site and every receive site, whether `make(chan T)` (unbuffered) or `make(chan T, N)` (buffered), and whether/where it is `close`d.

Do NOT file findings during Phase A.

---

## Phase B — Run these passes in order, reusing `goroutine_map`

### 1. `GOROUTINELEAK` — Goroutine leak

A goroutine leaks when it blocks forever (on a channel send/receive, a `sync.Mutex`, an unbounded `for` loop with no break) and nothing in the program can unblock it once its caller has moved on — most commonly: a worker goroutine sends a result on an unbuffered channel whose only reader gave up (timed out, the caller function returned) and never reads. `time.After` inside a hot loop (creates a new timer each iteration; the old ones aren't GC'd until they fire) is a related, lower-severity leak worth noting in the same pass.

Bar for filing: the spawn site's caller has a path (early return, timeout, error branch) that stops waiting on the goroutine's channel/completion signal, **and** the goroutine has no `select` with a cancellation case (`ctx.Done()`, a dedicated `stop chan struct{}`) that would let it exit instead of blocking forever.

**FPs to reject:**
- The goroutine writes to a buffered channel with capacity >= the number of possible writers, and nothing needs to read it before the program can safely discard it (fire-and-forget metrics/logging with an explicitly documented drop policy).
- A `select` in the goroutine body already includes a case on `ctx.Done()` or an explicit stop channel that is provably closed/cancelled on every caller exit path.
- The goroutine is genuinely meant to run for the life of the process (a long-running worker registered once at startup, not spawned per-request).

**Patch:** thread a `context.Context` (or a dedicated `done chan struct{}`) into the goroutine and `select` on it alongside the blocking channel op; size result channels so a giving-up caller can't block the producer (buffer of 1 for a single result), or have the goroutine `select` on the same cancellation signal the caller uses to give up.

### 2. `CHANDEADLOCK` — Mismatched send/receive deadlock

An unbuffered (or under-buffered) channel deadlocks when the number of sends exceeds the number of concurrent receivers ready to receive (or vice versa) with no other goroutine able to make progress. Common shapes: sending on an unbuffered channel from the main goroutine with no reader spawned yet; a `sync.WaitGroup.Wait()` that can never return because a spawned goroutine panics before calling `Done()`; two goroutines each waiting on a channel the other is supposed to fill first.

Trace each channel in `goroutine_map`: count static send sites vs. receive sites, and check whether every send site's goroutine could plausibly outlive its receivers (or vice versa) under a realistic interleaving.

**FPs to reject:**
- The channel is buffered with capacity matching the maximum possible in-flight sends before a receiver drains it.
- A `select` with a `default` case makes the send/receive non-blocking.
- The mismatch is guarded by a `sync.WaitGroup` or explicit count that provably keeps sends and receives balanced.

**Patch:** buffer the channel to the known maximum in-flight count, add a `select`/`default` or timeout, or restructure so the number of senders and receivers is statically balanced (e.g., a single fan-in goroutine that closes the channel once all producers finish).

### 3. `WGRACE` — WaitGroup.Add called inside the goroutine

`wg.Add(1)` must happen in the **spawning** goroutine, before the `go` statement — calling it from inside the spawned goroutine races with a concurrent `wg.Wait()` in the parent, which may observe the counter as already zero (all previously-added goroutines finished) and return before this one was ever counted.

```go
var wg sync.WaitGroup
for _, item := range items {
    go func(item Item) {
        wg.Add(1)         // WGRACE: too late — Wait() may already have returned
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()
```

**FPs to reject:**
- `wg.Add(1)` correctly appears before the `go` statement (in the loop body, not inside the closure).
- The code doesn't use a `WaitGroup` at all (different cluster's territory).

**Patch:** move `wg.Add(1)` immediately before (or as part of) the `go` statement, in the parent goroutine.

### 4. `CHANCLOSE` — Double close or send-on-closed-channel

Closing a channel twice, or sending on a channel after it's been closed, panics (`close of closed channel` / `send on closed channel`) — always a bug, and a DoS if reachable from untrusted input driving concurrent request handling. Common shapes: two goroutines racing to `close()` the same completion channel; a fan-out/fan-in pattern where the producer closes a shared results channel that other still-running producers also write to.

**FPs to reject:**
- `close()` is guarded by a `sync.Once` so only one goroutine can ever execute it.
- Only a single, well-identified owner goroutine ever calls `close()`, and no other goroutine sends after that owner's close point (verified via the inventory's send/receive site list).

**Patch:** designate exactly one owner for `close()` (typically the sole producer, or a coordinator using `sync.Once`); have consumers signal completion via a separate `done` channel instead of relying on close-triggered range termination when there are multiple producers.

### 5. `CHANBLOCK` — Unbounded blocking channel op with no escape hatch

A channel send or receive with no `select`/`default`/timeout blocks the calling goroutine indefinitely if the other side never shows up — on a request-handling goroutine, this is a DoS: one slow/malicious caller can pin a worker forever. Distinct from `GOROUTINELEAK` (which is about background goroutines never exiting) — this pass is about a **foreground** blocking call with attacker-influenced timing (e.g., waiting on a channel fed by a downstream RPC/DB call with no context deadline).

**FPs to reject:**
- The blocking op is provably bounded by an upstream `context.Context` deadline that the goroutine's caller enforces (verify the context is actually threaded to this call, not just declared upstream).
- The channel is purely internal, fed by code in the same request whose worst-case latency is bounded and small (e.g., a same-process fan-out with a bounded, already-audited worker pool).

**Patch:** wrap the blocking channel op in a `select` with a `case <-ctx.Done():` or `case <-time.After(timeout):` branch that returns an error instead of blocking forever.

---

## Deconfliction

Report only one finding per `(path, line)`. Priority (higher wins):

1. `CHANDEADLOCK` > `CHANBLOCK` when the mismatch is provably deterministic (always deadlocks on the observed send/receive counts) rather than merely possible under adversarial timing — `CHANDEADLOCK` names the certain bug more precisely.
2. `WGRACE` is independent of `GOROUTINELEAK` — report both when a goroutine both races its `Add()` and never exits.
3. `CHANCLOSE` outranks `CHANBLOCK` when the panic-causing close is the root cause of a subsequent block.

---

## Token-economy reminder

All five passes operate on the same `goroutine_map`. Build it ONCE; do not re-search `go func`, `chan`, `select`, or `WaitGroup` patterns per pass.
