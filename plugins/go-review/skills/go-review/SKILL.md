---
name: go-review
description: Performs comprehensive Go security review for goroutine/channel concurrency hazards, data races beyond what go vet catches, nil-interface and defer/closure pitfalls, the unsafe/cgo boundary, context.Context lifecycle mistakes, and web-input handling. Use when auditing Go services, CLIs, or libraries — particularly those with goroutines, channels, `unsafe`, or net/http handlers.
allowed-tools: Agent AskUserQuestion SendMessage TaskCreate TaskUpdate TaskList Read Write Bash
---

# Go Security Review

Runs in the main conversation (invoke via `/go-review:go-review`). Orchestrator owns the `Task*` ledger as bookkeeping for retries; workers and judges have no Task tools. Workers and judges are named plugin subagents (`go-review:go-review-worker`, `go-review:go-review-dedup-judge`, `go-review:go-review-fp-judge`); tool sets are declared in `plugins/go-review/agents/*.md`. Findings are exchanged via markdown-with-YAML files in a shared output directory.

## When to Use

Go application/library security review: goroutine and channel lifecycle hazards, data races beyond `go vet`'s mechanical checks, nil-interface and defer/closure footguns, `unsafe`/cgo boundary issues, `context.Context` lifecycle mistakes, and web-input handling (`net/http`, path traversal, SSRF, template auto-escaping).

## When NOT to Use

- Pure C/C++ codebases — use `c-review` instead.
- Rust codebases — use `rust-review` instead.
- Smart contracts (Solana programs / NEAR contracts / Ink!) — use `solana-vulnerability-scanner` or the contract-specific skill.
- Confirming an actual data race — static analysis can only flag *suspicious* unsynchronized access; recommend `go test -race` to confirm any `concurrency-data-race` finding with certainty.
- Kubernetes/cloud-native YAML/RBAC misconfiguration in a Go-authored tool's *deployment* — that's a different skill; go-review covers the Go source, not cluster config.

## Subagents

| Subagent type | Purpose | Tool set |
|---|---|---|
| `go-review:go-review-worker` | Run assigned cluster, write findings | Read, Write, Edit, Bash |
| `go-review:go-review-dedup-judge` | Merge duplicates (runs **first**) | Read, Write, Edit, Glob |
| `go-review:go-review-fp-judge` | FP + severity + final reports (runs **second**) | Read, Write, Edit, Bash |

Tools come from each agent's frontmatter at spawn time. The orchestrator's `Task*`/`Agent`/`Bash`/etc. come from this skill's `allowed-tools`. **Search-tool / `Bash` interaction:** in current Claude Code, an agent granted `Bash` is **not** also granted the dedicated `Glob` **or `Grep`** tools (the calls return `No such tool available`; the harness expects `find`/`grep`/`rg` via `Bash` instead). So only the dedup-judge — the one agent that holds **no** `Bash` — uses `Glob`; the worker, fp-judge, and the orchestrator resolve and search paths with `Read` / `Bash` `find` / `rg` / `grep` / `test -f` instead. Because the cluster/finder prompt seeds are written in ripgrep regex syntax (`\s`, `\d`, `\b`), `Bash`-holding agents must run them with **`rg`**. If `rg` is not installed its call fails *loudly* (`command not found`) — fall back to `grep -E` with POSIX classes (`\s`→`[[:space:]]`, `\d`→`[[:digit:]]`, drop `\b`), never a raw-`\s` `grep` whose *silent* empty becomes a bad `cleared`. Do **not** reintroduce `Glob`/`Grep` into a `Bash`-holding agent's protocol.

---

## Architecture

```
coordinator: write context.md → build_run_plan.py → TaskCreate × M
          → spawn primer (foreground) → spawn M workers (parallel)
          → classify Phase-7 outcomes + write findings-index.txt
          → dedup-judge → fp-judge → report safety net (SARIF + REPORT.md) → return REPORT.md
```

Output directory contains: `context.md`, `plan.json`, `worker-prompts/`, `findings/`, `findings-index.d/` (per-worker shards), `findings-index.txt`, `coverage/` (per-worker coverage-gate files), `run-summary.md`, `dedup-summary.md`, `fp-summary.md`, `REPORT.md`, `REPORT.sarif`.

**Path convention:** every later phase shells out to `${GO_REVIEW_PLUGIN_ROOT}/scripts/*.py`, so resolve that variable first to the plugin directory that contains `prompts/clusters/unsafe-boundary.md` (and `scripts/build_run_plan.py`). Try in order, first hit wins:

1. **Native Claude Code** — `${CLAUDE_PLUGIN_ROOT}`, accepted if `Bash: ls "${CLAUDE_PLUGIN_ROOT}/prompts/clusters/unsafe-boundary.md"` resolves.
2. **Codex** — `${CODEX_PLUGIN_ROOT}` (set it the same way if that var is present and resolves the marker).
3. **Fallback search** — covers Codex installs under `~/.codex`, Claude installs under `~/.claude`, and a local checkout / repo run: `Bash: find ~/.claude ~/.codex . -path '*/plugins/go-review/prompts/clusters/unsafe-boundary.md' -print -quit 2>/dev/null`. Take the match and strip the trailing `/prompts/clusters/unsafe-boundary.md` to get the root (the home dirs are searched before `.` so an installed copy wins over any vendored copy in the audited repo).

Set `GO_REVIEW_PLUGIN_ROOT` to the resolved root. If all three fail, **abort** with a message naming the roots searched — do not enter Phase 4 with an empty variable (every `python3 "${GO_REVIEW_PLUGIN_ROOT}/scripts/..."` call would fail with a confusing path error).

**Scope convention:** keep two scopes separate throughout the run:

- `finding_scope_root` — the user-requested audit subtree. Workers may only file findings whose vulnerable location is inside this subtree.
- `context_roots` — read-only repo roots/files workers and judges may inspect to verify reachability, callers, wrappers, build flags, mitigations, and threat-model details. Default to `.` unless the user explicitly forbids broader context. Reading context outside `finding_scope_root` is allowed; filing findings there is not.

---

## Rationalizations to Reject

- **"`go vet`/`staticcheck` already checks this."** They check the *syntactic* pattern locally (a value receiver on a lock-holding type, a lost `cancel` call) — they don't trace attacker reachability or impact. Don't skip a cluster because a linter theoretically covers part of it; the worker's job is to confirm the bug is real and reachable, which is exactly the analysis vet/staticcheck don't attempt. Note in `static-hygiene` findings whether vet/staticcheck is wired into CI — that's informational, not a reason to skip the other clusters.
- **"It's Go, so it's memory-safe, skip the deep dive."** Memory safety doesn't prevent goroutine leaks, data races (Go's own runtime crashes loudly on concurrent map writes), deadlocks, typed-nil-interface bugs, or SSRF/path-traversal logic errors — the majority of this plugin's clusters have nothing to do with memory unsafety at all.
- **"`has_unsafe=false` so skip unsafe-boundary entirely."** Correct — do skip it; that's what the gate is for. But don't generalize this into skipping `has_cgo`/`has_context`/etc. clusters without checking their actual flags first. Each conditional cluster has its own independent gate; verify each one from the Phase 1 probes, don't assume "no unsafe" implies "no cgo" or "no context usage."
- **"Background spawns parallelize the workers."** They do not — `Agent` calls in a single assistant message already run concurrently. `run_in_background=true` defeats the Phase 6a primer cache, so every worker pays full cache-creation (`cache_read_input_tokens=0`) and the ~15 K-token primer is wasted M times. Default: omit `run_in_background` from worker spawns.
- **"I'll re-derive the cluster list / paths / pass prefixes inline instead of running `build_run_plan.py`."** The script is the only authority for selection and rendering. Paraphrasing it drops fields that the worker self-check requires, producing `worker-N abort: spawn prompt malformed`. Always run the script and `Read plan.json`.
- **"The run partially succeeded — I'll just write `REPORT.md` from what completed."** Hiding partial runs behind a successful report is a correctness bug. If any Phase-5 cluster task is not `completed`, surface it prominently in `run-summary.md` and the final response.
- **"Zero findings — skip Phase 8."** Always run both judges and Phase 8b: dedup-judge writes a minimal no-op `dedup-summary.md` on an empty index, fp-judge writes empty `REPORT.md`/`REPORT.sarif`, and Phase 8b's SARIF generator emits `results: []` for the empty case. SARIF consumers depend on a stable artifact set.
- **"`Bash: ls README*` is fine for the preflight."** Under zsh, an unmatched glob aborts the whole compound command before `2>/dev/null` runs. Use `find` (never fails on no-match) — and not `Glob`, which is unavailable to an agent that also holds `Bash`.

---

## Orchestration Workflow

Run these phases **in the main conversation**.

### Phase 0: Parameter Collection

**Entry:** skill invoked. **Exit:** `threat_model`, `worker_model`, `severity_filter` resolved; `scope_subpath` resolved or set to `"."`; `finding_scope_root=scope_subpath`; `context_roots` resolved.

The skill is invoked directly (no command wrapper). Parse any free-text arguments the user passed on the `/go-review:go-review` line (e.g. `internal/api only`, `high severity only`, `use haiku`) and pre-fill the answers they imply — then ask for any missing required parameters with **one** `AskUserQuestion` call. Never silently default the required parameters.

Required parameters:

| Parameter | Values | How to infer from args |
|---|---|---|
| `threat_model` | `REMOTE` / `LOCAL_UNPRIVILEGED` / `BOTH` | Words like "remote", "network", "attacker" → `REMOTE`; "local", "unprivileged" → `LOCAL_UNPRIVILEGED`; otherwise ask. |
| `worker_model` | `haiku` / `sonnet` / `opus` | Explicit model name in args. Otherwise ask (no silent default). |
| `severity_filter` | `all` / `medium` / `high` | "all", "every", "noisy" → `all`; "medium and above" → `medium`; "high only", "criticals only" → `high`. Otherwise ask — **no silent default**. |
| `scope_subpath` | repo-relative directory (optional) | Phrases like "X only", "just audit X/", "review subdirectory X" → `internal/X/` or the matching subdir. Apply fuzzy matching against top-level subdirectories of the repo. If absent, set `"."`; if ambiguous, ask. |

Call `AskUserQuestion` exactly once with only unresolved required parameters (`threat_model`, `worker_model`, `severity_filter`) plus `scope_subpath` only when the user explicitly requested a narrowed scope but it is ambiguous. If the required parameters were all pre-filled and scope is absent or resolved, skip the question.

After resolving `scope_subpath`, set `finding_scope_root="${scope_subpath:-.}"`. Set `context_roots="."` by default so workers can verify callers/build settings outside a narrowed subtree without filing out-of-scope findings. If the user explicitly asks to forbid broader context, set `context_roots="${finding_scope_root}"` and note that reachability confidence may be lower.

### Phase 1: Prerequisites

**Entry:** Phase 0 complete. **Exit:** `has_unsafe`, `has_cgo`, `has_concurrency`, `has_context`, `has_reflection`, `has_net_http`, `has_serialization`, `has_crypto` flags determined. Abort with a clear message if no `*.go` files exist under `${finding_scope_root}`.

Probe within `${finding_scope_root:-.}` with the `Bash` commands below (non-empty output ⇒ flag true). The dedicated `Grep`/`Glob` tools are unavailable to this orchestrator because it holds `Bash` — use `grep`/`rg`/`find` via `Bash`. (The probe regexes use `\s`/`\b`; if your `grep` lacks GNU `\s` support, run them with `rg -uu` — which honors `\s` and still searches ignored files — or, if `rg` is not installed either, replace `\s`→`[[:space:]]` and drop `\b`. Widening is safe here: a false-positive capability flag only adds a harmless extra worker, whereas a missed match would skip a whole pass.)

```bash
# Go source presence (precondition)
find "${finding_scope_root:-.}" -name '*.go' -print -quit

# has_unsafe
grep -rlE '\bunsafe\.(Pointer|Add|Slice|SliceData|String|StringData)\b' --include='*.go' "${finding_scope_root:-.}" | head -1

# has_cgo
grep -rlE 'import\s+"C"|^\s*//\s*#cgo\b' --include='*.go' "${finding_scope_root:-.}" | head -1

# has_concurrency
grep -rlE '\bgo\s+func\b|\bgo\s+[A-Za-z_][A-Za-z0-9_.]*\(|sync\.(Mutex|RWMutex|WaitGroup|Once|Map)\b|\bchan\s|<-\s*chan|sync/atomic' --include='*.go' "${finding_scope_root:-.}" | head -1

# has_context
grep -rlE 'context\.Context\b|context\.With(Cancel|Timeout|Deadline|Value)\(' --include='*.go' "${finding_scope_root:-.}" | head -1

# has_reflection
grep -rlE '\breflect\.(ValueOf|TypeOf|New|Indirect)\(' --include='*.go' "${finding_scope_root:-.}" | head -1

# has_net_http
grep -rlE 'net/http"|http\.Handler\b|http\.HandleFunc\(|\bgin\.|\becho\.|\bchi\.' --include='*.go' "${finding_scope_root:-.}" | head -1

# has_serialization
grep -rlE 'encoding/(json|gob|xml)"|"gopkg\.in/yaml|proto\.(Marshal|Unmarshal)\(' --include='*.go' "${finding_scope_root:-.}" | head -1

# has_crypto
grep -rlE 'crypto/(md5|sha1|des|rc4)"|"math/rand"' --include='*.go' "${finding_scope_root:-.}" | head -1
```

As with the other flags, non-empty output means the flag is true. These detectors are intentionally conservative: when in doubt they set the flag true, because a false-positive flag only costs a harmless extra worker while a false-negative would skip a real pass.

Also probe `go.mod` presence and version (informational — note in `run-summary.md` the module path and `go` directive version):

```bash
# context_roots may be comma-separated (build_run_plan.py treats it as a list),
# so probe each root rather than passing "a,b" as one (nonexistent) path.
echo "${context_roots:-.}" | tr ',' '\n' | while IFS= read -r root; do
  find "${root:-.}" -name 'go.mod' -print -quit
done | head -1
```

If `go.mod` is found, `Read` it once to capture the `go` directive version for `context.md` — the `LOOPCAPTURE` finder needs it to decide whether pre-1.22 loop-variable capture semantics apply.

### Phase 2: Output Directory

**Entry:** Phase 1 flags set. **Exit:** absolute `output_dir` resolved; `${output_dir}/findings/` and `${output_dir}/coverage/` exist.

Resolve an absolute path for `output_dir` (default: `$(pwd)/.go-review-results/$(date -u +%Y%m%dT%H%M%SZ)/`):

```bash
mkdir -p "${output_dir}/findings" "${output_dir}/coverage"
```

The `coverage/` subdirectory holds per-worker coverage-gate audit files (`coverage/worker-{N}.md`). Workers write to it instead of embedding the table in their reply — see `agents/go-review-worker.md` step 5.

### Phase 3: Codebase Context

**Entry:** `${output_dir}` exists. **Exit:** `${output_dir}/context.md` written.

Skim `README.{md,rst,txt}` and any build/manifest file (`go.mod`, `go.sum`, `Makefile`, `.golangci.yml`) — preflight with `find` (via `Bash`) before any `Read` (a `Read` on a missing file aborts the turn; `Glob` is unavailable to this orchestrator because it holds `Bash`). Do **not** use `Bash: ls README*` for the preflight: under zsh, an unmatched glob aborts the whole compound command before `2>/dev/null` runs. Use `find . -maxdepth 2 -name 'README*' -o -name 'go.mod' -o -name '.golangci.yml' -o -name '.golangci.toml'`, which never fails on no-match.

Write `${output_dir}/context.md` with: YAML frontmatter (`threat_model`, `severity_filter`, `scope_subpath`, `finding_scope_root`, `context_roots`, `has_unsafe`, `has_cgo`, `has_concurrency`, `has_context`, `has_reflection`, `has_net_http`, `has_serialization`, `has_crypto`, `output_dir`, `go_mod` as `present`/`absent` plus the `go` directive version when present), then a short markdown body with five sections — **Purpose** (1-3 sentences), **Scope** (what's in `finding_scope_root`, and that findings outside it are out of scope), **Entry points** (where untrusted data enters: HTTP handlers, CLI args, file/config input, gRPC/message-queue consumers), **Trust boundaries** (sandboxed vs trusted peers vs arbitrary remote), **Existing hardening** (`go vet`/`staticcheck`/`golangci-lint` in CI, `go test -race` in CI, `govulncheck` usage, fuzzing harnesses).

### Phase 4: Build Run Plan (deterministic)

**Entry:** capability flags + `threat_model` known; `${output_dir}/findings/` exists. **Exit:** `${output_dir}/plan.json` and `${output_dir}/worker-prompts/*.txt` written; `M = worker_count` known.

Selection, filtering, path resolution, and spawn-prompt rendering are **delegated to the script** to keep spawn prompts complete and consistent:

```bash
python3 "${GO_REVIEW_PLUGIN_ROOT}/scripts/build_run_plan.py" \
  --plugin-root "${GO_REVIEW_PLUGIN_ROOT}" --output-dir "${output_dir}" \
  --threat-model "${threat_model}" --severity-filter "${severity_filter}" \
  --scope-subpath "${finding_scope_root:-.}" --context-roots "${context_roots:-.}" \
  --has-unsafe "${has_unsafe}" --has-cgo "${has_cgo}" \
  --has-concurrency "${has_concurrency}" --has-context "${has_context}" \
  --has-reflection "${has_reflection}" --has-net-http "${has_net_http}" \
  --has-serialization "${has_serialization}" --has-crypto "${has_crypto}" \
  --max-passes-per-worker 4
```

The script writes `plan.json` + `worker-prompts/worker-N.txt` + (if `--cache-primer=true`, the default) `worker-prompts/cache-primer.txt`, and prints a JSON summary on stdout. Exits non-zero on any missing prompt — surface the message and stop. With the default `--max-passes-per-worker 4` the planner selects the 6 always-on clusters (18 bug classes) → **M ≈ 6 workers** for a Go codebase with no concurrency/unsafe/context/net-http/serialization usage (rare in practice), up to all 10 clusters (41 bug classes) → **M ≈ 12–13 workers** for a typical concurrent Go service with HTTP handlers and JSON deserialization. M is the post-chunk worker count (`plan.workers.length`), so it runs above the cluster count when a cluster's pass count exceeds the chunk cap — chunking splits multi-pass **non-consolidated** clusters (e.g. `concurrency-data-race`, `web-input-handling` at 5 passes each), while the two **consolidated** clusters (`unsafe-boundary`, `concurrency-goroutines`) are never chunked: one worker each builds the shared inventory once and runs all its phases. After it returns, `Read plan.json` for the structured selection — never re-derive filtering or paths.

`--max-passes-per-worker N` caps the per-worker pass count. The planner deterministically splits any **non-consolidated** cluster with more than `N` passes into `ceil(K/N)` contiguous chunks; each chunk becomes its own `go-review-worker` spawn with a `-{i}`-suffixed `cluster_id` (e.g. `concurrency-data-race-1`, `concurrency-data-race-2`). **Consolidated clusters (`unsafe-boundary`, `concurrency-goroutines`) are exempt — never chunked, regardless of pass count or override — so one worker builds their shared Phase-A inventory once and runs every phase.** The shared prompt-cache prefix and `Cluster prompt:` path are byte-identical across chunks, so the cache primer still warms every worker. Default 4 is calibrated against the manifest's heavy-tail clusters (`concurrency-data-race` and `web-input-handling` each have 5 passes). Pass `--max-passes-per-worker 0` to disable all chunking.

### Phase 5: Create Bookkeeping Tasks (orchestrator-internal)

**Entry:** `${output_dir}/plan.json` exists; `M = plan.workers.length`. **Exit:** `cluster_task_ids[]` created (1:1 with `plan.workers`), all `pending`.

The task ledger is **orchestrator bookkeeping only** (TUI visibility + Phase-7 retry tracking) — workers never read or write it. One `TaskCreate` per worker, populating `metadata` with `kind="cluster"`, `worker_n`, `cluster_id`, `spawn_prompt_path`, `pass_prefixes`, `attempt=1` — all values copied verbatim from `plan.workers[i]`. Track `cluster_task_ids[]` in `plan.workers` order.

### Phase 6: Spawn workers (optional cache-primer first, then M in parallel)

**Entry:** `cluster_task_ids[]` populated; per-worker spawn prompt files exist at `${output_dir}/worker-prompts/worker-N.txt`. **Exit:** all M `Agent` calls — across every wave — have returned (the parallel spawn block(s) completed).

#### Phase 6a: Cache primer (gated on `plan.run.cache_primer`)

A parallel batch from cold start cannot share cache (all M requests dispatch simultaneously, none has finished writing). To warm the prefix, spawn a tiny primer first — **foreground** (background spawns don't share cache with subsequent foreground spawns).

If `plan.run.cache_primer == true`, `build_run_plan.py` has written `${output_dir}/worker-prompts/cache-primer.txt`. Spawn it in its own assistant message: `Read` the file, pass verbatim as `Agent` `prompt` with `subagent_type=go-review:go-review-worker`, `model=${worker_model}`, `description="Go review cache primer"`, no `run_in_background`. The script wrote the prefix byte-identical to `worker-1.txt` through the `<context>` block — that byte-identity is what gives the parallel workers their cache hit. The primer trailer contains `Cache primer: true`, which the worker system prompt treats as a first-class mode and returns exactly `worker-PRIMER abort: cache primer (no analysis performed)` in one text response with zero tool calls. Discard the abort line — Phase 7 ignores it (no `worker-N` id).

Foreground spawn already serializes — no `sleep` needed before Phase 6b. Skip Phase 6a entirely if `plan.run.cache_primer == false`.

#### Phase 6b: Spawn M real workers in parallel (one message per wave of ≤16)

> **STOP — read this before composing the spawn message.**
>
> Workers MUST be spawned **foreground** (no `run_in_background` field, or `run_in_background=false`).
> "Parallel" here means *one assistant message containing the wave's `Agent` calls* — that already runs them concurrently. (For large `M`, split into consecutive waves of ≤16 calls, one message per wave — see "Required spawn shape" below.) **Background spawns are NOT how you parallelize this skill.**
>
> Background spawns defeat Phase 6a's primer cache: every worker pays full cache-creation on its first turn (`cache_read_input_tokens=0`), and the primer's ~15 K tokens are wasted M times over.
>
> Before sending the spawn message, audit your draft: every `Agent` call must have **no** `run_in_background` key. If you wrote `run_in_background=true`, delete it.

**Required spawn shape:** emit a single assistant message containing the wave's `Agent` tool invocations — that one message is what runs them concurrently. Sequential spawning (one `Agent` call per message) serializes the review and is also wrong, but that failure is loud (timing); the background-spawn failure is silent (cost).

**Waves when `M` exceeds the per-message cap.** The harness caps the number of `Agent` calls it will dispatch from a single assistant message (observed: ~20 in Claude Code). So when `M > 16`, **plan the waves up front**: split the workers into consecutive waves of **≤16 `Agent` calls**, each wave its own single assistant message. Rules:

- **Within a wave:** all `Agent` calls in **one** message, **foreground** (no `run_in_background`) — identical shape to a single-wave run.
- **Across waves:** wave _k+1_ is a **separate** message that can only be sent after wave _k_'s `Agent` calls all return (a tool-use message ends the turn). Waves are therefore serialized with respect to each other — that is correct and loud; accept it. Do **not** try to overlap them.
- **Never** reach for `run_in_background=true` to fit more workers in one message. More *waves*, never background.
- **Cache across waves:** the primer prefix has a ~5-minute cache TTL that refreshes on every hit, so back-to-back waves keep hitting it. If a later wave will start more than ~5 minutes after the previous one, re-spawn the Phase-6a primer in its own message first to re-warm the prefix before that wave.
- **Balance the waves** so no wave hugs the cap and the last wave isn't a tiny straggler.
- After every wave has returned, proceed to Phase 7 with the **full** set of M worker results.

For each worker `N ∈ [1..M]` (in its assigned wave):

1. `Read: ${output_dir}/worker-prompts/worker-N.txt`
2. Pass the file contents **verbatim** as the `Agent` tool's `prompt` argument:

| Parameter | Value |
|-----------|-------|
| `subagent_type` | `go-review:go-review-worker` |
| `model` | `${worker_model}` (haiku / sonnet / opus) |
| `description` | `Go review worker N` |
| `prompt` | the full text of `worker-N.txt` (no edits) |
| `run_in_background` | **field MUST be omitted, OR set to `false`.** Never `true`. See the foreground-spawn warning above. |

The spawn prompt is the single authority. Pass it verbatim — every field is required by the worker's self-check; any deviation triggers `worker-N abort: spawn prompt malformed`.

**Anti-patterns to reject:**

- **Passing `run_in_background=true`** (see warning above).
- **Cramming more than ~16 `Agent` calls into one message** when `M` is large — the harness silently keeps only the first ~20 and drops the rest. Use balanced waves of ≤16, never background spawns, to cover all M.
- Hand-typing the spawn prompt instead of reading `worker-N.txt`.
- Inserting Task-related instructions ("first call TaskList", "Assigned task id: <N>"). Workers have no Task tools.
- Editing the rendered prompt before passing it (trimming "redundant" fields, collapsing pass lists).

### Phase 7: Wait for Workers and Classify Outcomes

**Entry:** all M Phase-6 `Agent` calls have returned. **Exit:** every cluster has either succeeded or been retried up to the cap; `${output_dir}/findings-index.txt` written.

The Phase-6 `Agent` invocations block until each worker returns. Inspect each worker's return text and apply this classifier in order — first match wins:

| # | Match (in return text) | Outcome | Action |
|---|---|---|---|
| 1 | `worker-N complete:` | **provisional success** | Parse the `wrote N finding files` count, then run the artifact validator below before `TaskUpdate` to `completed`. |
| 2 | `abort: spawn prompt malformed`, `abort: pre-work budget exceeded`, or `abort: TaskList unavailable` (legacy) | **non-retryable orchestrator bug** | Stop the run, surface the abort + spawn-prompt path. Re-running the same prompt repeats the failure — pre-work-budget exhaustion always means the worker couldn't pass its self-check, which a retry won't fix. |
| 3 | other `worker-N abort:` | **retryable** | Mark `pending`, set `metadata.abort_reason`, `needs_respawn=true`, increment `attempt`. |
| 4 | `Agent` errored or no `complete:`/`abort:` token | **retryable** | Same as #3 (transient worker crash). |

If any non-retryable, stop. Otherwise, **before re-spawning, clear each retryable worker's prefix-space on disk** — the Phase-7 index is built from disk, so a crashed attempt's higher-id stragglers (files the replacement never re-emits) would otherwise be resurrected into the report. Loop over the worker's actual `pass_prefixes` (from its task `metadata`), substituting each real prefix for `${pfx}` — do **not** run the command with a literal `PREFIX`:

```bash
# zsh-safe: `find … -delete` never aborts on no-match (an `rm PREFIX-*.md` glob would).
# Replace `PREFIX1 PREFIX2` with the worker's actual space-separated pass_prefixes.
for pfx in PREFIX1 PREFIX2; do
  find "${output_dir}/findings" -maxdepth 1 -type f -name "${pfx}-*.md" -delete
done
```

Then re-spawn each `pending` retryable with `attempt <= 2` in one parallel block (cap = 2 attempts per cluster). `attempt` was just incremented to `2` on the first failure, so the guard must admit `2` to allow the single retry — `attempt < 2` would block every retry. A second failure increments to `3`, which fails `<= 2` and ends retries. Replacement workers reuse deterministic finding IDs per prefix, so a cleared prefix-space plus a fresh write yields a consistent shard / coverage / disk set.

#### Sanity-check + write index

For every provisional `complete:` cluster, validate the worker-owned shard, coverage file, coverage rows, filed IDs, and claimed finding count against `plan.json` before marking the task completed. Run one command per completed worker, or validate multiple workers in one command. Both claimed-count forms below are valid; do not pass bare `worker-N=N` values without either grouping them after a `--claimed-count` flag or repeating the flag.

```bash
python3 "${GO_REVIEW_PLUGIN_ROOT}/scripts/validate_artifacts.py" "${output_dir}/plan.json" \
  --worker worker-N --claimed-count worker-N=<claimed_count_from_complete_line>
```

```bash
python3 "${GO_REVIEW_PLUGIN_ROOT}/scripts/validate_artifacts.py" "${output_dir}/plan.json" \
  --worker worker-1 --worker worker-2 \
  --claimed-count worker-1=0 worker-2=3
```

```bash
python3 "${GO_REVIEW_PLUGIN_ROOT}/scripts/validate_artifacts.py" "${output_dir}/plan.json" \
  --worker worker-1 --worker worker-2 \
  --claimed-count worker-1=0 --claimed-count worker-2=3
```

If validation exits non-zero, treat the completion as malformed and retryable (classifier row #4): mark the task `pending`, store the validator output in `metadata.abort_reason`, set `needs_respawn=true`, and increment `attempt`. Missing `findings-index.d/worker-N.txt`, missing `coverage/worker-N.md`, missing coverage rows, invalid `skipped:` rows, filed IDs absent from the shard or disk, and claimed-count mismatches are all malformed completions. After the retry cap, leave the cluster task incomplete and surface the validator output in `run-summary.md` and the final response. Only validation-clean provisional completions may be `TaskUpdate`d to `completed`.

Then build the index. The canonical index is the set of finding files **actually on disk**, not the shard union — building from disk guarantees that a finding written without a matching shard entry (a worker that crashed between its `Write` and its shard-append, or the single-prefix empty-shard trap the worker prompt warns about) is still picked up by dedup → fp-judge → REPORT/SARIF instead of silently vanishing, and that every index entry resolves to a real file:

```bash
# Canonical index = every finding file on disk. `find` never fails on no-match
# (an empty findings/ yields an empty index — the unambiguous "zero findings"
# signal). `sort -u` collapses Phase-7 retry duplicates: replacement workers reuse
# deterministic ids, so the same path appears once.
find "${output_dir}/findings" -maxdepth 1 -type f -name '*.md' 2>/dev/null \
  | sort -u > "${output_dir}/findings-index.txt"

# Reconcile against the per-worker shards: any path on disk but in NO shard is an
# orphan whose worker failed to record it. It is already in the index above (so it
# is NOT dropped) — print it so the bookkeeping gap can be surfaced. Non-fatal.
if [ -d "${output_dir}/findings-index.d" ]; then
  # Reconcile by basename (finding ids are unique), so a path-format difference
  # between the worker `find` and this one (trailing slash, /var↔/private/var)
  # cannot manufacture false orphans. Any basename on disk but in no shard is an
  # orphan whose worker failed to record it.
  comm -13 \
    <(find "${output_dir}/findings-index.d" -maxdepth 1 -type f -name 'worker-*.txt' -exec awk 1 {} + 2>/dev/null | sed 's#.*/##; /^[[:space:]]*$/d' | sort -u) \
    <(find "${output_dir}/findings" -maxdepth 1 -type f -name '*.md' 2>/dev/null | sed 's#.*/##' | sort -u)
fi
```

The shards stay the per-worker audit trail (`validate_artifacts.py` checks them) and the dedup-judge's crash-recovery fallback, but they no longer gate what reaches the pipeline. For each orphan basename the reconcile prints, map its `<PREFIX>` to the owning worker via `plan.json` and note in `run-summary.md` that that worker's shard was incomplete — the finding is already in the index (so it is not lost), but the bookkeeping gap should be visible. Still cross-check the index line count against the sum of `wrote N` worker claims; log mismatches but don't abort.

After task updates and index creation, run `TaskList` and write `${output_dir}/run-summary.md` with:

- resolved parameters (`threat_model`, `severity_filter`, `finding_scope_root`, `context_roots`, capability flags `has_unsafe`/`has_cgo`/`has_concurrency`/`has_context`/`has_reflection`/`has_net_http`/`has_serialization`/`has_crypto`, `go.mod` version)
- worker outcome table (`worker_n`, `cluster_id`, claimed finding count, shard line count, coverage-file path (`coverage/worker-{N}.md`), task status, retry/abort state)
- `findings-index.txt` line count and any mismatch against worker claims
- judge status once Phase 8 finishes, or the reason a judge was skipped/failed

If any Phase-5 cluster task is not `completed` — **or** any worker returned a `complete:` line carrying the `truncated at hard cap` token — include it prominently in `run-summary.md` and the final response. A hard-cap-truncated worker is marked `completed` for ledger purposes but is a **partial** result: do not let that `completed` status hide the incomplete coverage behind a successful report.

**Always run Phase 8 even on zero findings** — both judges short-circuit on an empty index: dedup-judge writes a minimal no-op `dedup-summary.md`, and fp-judge writes empty `REPORT.md`/`REPORT.sarif` so SARIF consumers get a stable artifact set.

### Phase 8: Judge Pipeline (sequential, dedup → fp+severity)

**Entry:** `findings-index.txt` exists. **Exit:** dedup-judge and fp-judge have returned; `dedup-summary.md`, `fp-summary.md`, `REPORT.md`, and ideally `REPORT.sarif` are written.

Each judge's full protocol is its system prompt (`agents/go-review-{dedup,fp}-judge.md`); spawn prompts pass only per-run variables. Do **not** reference `prompts/internal/judges/` — those files don't exist.

> **STOP — these two judges run in SEQUENCE, not in parallel.** Unlike the Phase-6b workers (which you spawn as M `Agent` calls in *one* message precisely because that runs them concurrently), the judges have a hard data dependency: fp-judge must see the `merged_into` / `also_known_as` annotations dedup-judge writes, and it only skips files already carrying `merged_into`. If you emit both `Agent` calls in one message they run concurrently — fp-judge reads findings before any merge annotations exist, judges every duplicate as a separate primary, and (because `dedup-summary.md` doesn't exist yet) trips its "dedup did not run" fallback, producing an inflated, duplicated `REPORT.md`/SARIF.
>
> Spawn dedup-judge in its **own** assistant message, wait for its `dedup-judge complete:` (or `abort:`) return, **then** spawn fp-judge in a **separate** message. Before composing the fp-judge spawn, confirm dedup finished — `Bash: test -f ${output_dir}/dedup-summary.md` must succeed (or you saw the dedup `complete:` token). **Never put both judge `Agent` calls in the same message.**

1. **First message** — `Agent(subagent_type="go-review:go-review-dedup-judge", description="Dedup judge", prompt=f"output_dir: {output_dir}")`. Wait for its return and classify it (below) before continuing.
2. **Then, in a separate message** — `Agent(subagent_type="go-review:go-review-fp-judge", description="FP + severity judge", prompt=f"output_dir: {output_dir}\nsarif_generator_path: {sarif_generator_path}")` — resolve `sarif_generator_path` to `${GO_REVIEW_PLUGIN_ROOT}/scripts/generate_sarif.py`.

**Judge failure handling.** Same shape as Phase 7's classifier, applied to judge return text:

- `… complete:` → **success.**
- `… abort:` → **non-retryable for that judge.** Surface the abort line plus `ls -l ${output_dir}/findings-index.txt`, then **still run Phase 8b** (its SARIF + `REPORT.md` safety net guarantees the artifact set even when a judge aborts), and stop without spawning further judges. "Stop" means do not continue the judge pipeline — it does **not** mean skip Phase 8b.
- No `complete:` (help message / error / question) → **retryable once.** `SendMessage(to=<agentId>, …)` rather than a fresh spawn (the agent already paid the protocol-parse cost). Include the explicit finding paths from `findings-index.txt`. If the second try still fails, surface the transcript and continue to Phase 8b.

### Phase 8b: Report safety net (SARIF + REPORT.md)

**Entry:** fp-judge returned, or the run aborted early. **Exit:** `${output_dir}/REPORT.sarif` and `${output_dir}/REPORT.md` both exist.

```bash
test -d "${output_dir}/findings" && python3 "${GO_REVIEW_PLUGIN_ROOT}/scripts/generate_sarif.py" "${output_dir}"
```

Run the SARIF generator unconditionally whenever `findings/` exists — it is idempotent (full overwrite), emits `results: []` for zero-survivor runs, and handles partial runs (findings without `fp_verdict` are emitted as `LIKELY_TP`, **exempt from the `severity_filter`** since their severity was never judge-validated, and marked `unjudged: true` / `severity_validated: false` with an `[UNVALIDATED SEVERITY — not judged]` message prefix — so an inferred severity guess can never silently drop them under a `medium`/`high` filter). Always overwriting protects against an fp-judge that crashed mid-write and left a corrupt `REPORT.sarif` on disk.

If the generator prints a `WARNING: skipped N …` line on stdout (it also records `invocations[].properties.skipped_findings` in the SARIF and a `warning` notification per dropped file), one or more finding files were unreadable or had no parseable frontmatter and were **excluded from the report**. This is a dropped result — surface it prominently in `run-summary.md` and the final response with the count and paths, the same way a non-`completed` cluster task is surfaced. Do not let the otherwise-clean SARIF hide the loss.

Then guarantee `REPORT.md` exists. Unlike SARIF (mechanical), `REPORT.md` is the fp-judge's **curated** artifact, so do **not** overwrite a judge-written one. (The fp-judge writes `REPORT.md` with a `Bash` heredoc, not the `Write` tool, because the harness blocks the `Write` tool for subagent report files — do not "fix" the judge by re-mandating `Write`. The orchestrator is the main agent and is **not** subject to that block, so its own `Write` below works.) Check for it, and if it is missing (the judge crashed, even its `Bash`-heredoc write failed, or it returned the report as chat text instead of writing the file), **the orchestrator writes `REPORT.md` itself** rather than failing the run:

- If the fp-judge returned the report body in its transcript, `Write` that text verbatim to `${output_dir}/REPORT.md`.
- Otherwise synthesize it from the on-disk findings: take the survivor primaries (`fp_verdict ∈ {TRUE_POSITIVE, LIKELY_TP}`, no `merged_into`; if the judge never ran, treat a finding with no `fp_verdict` as a survivor) listed in `findings-index.txt`, apply `severity_filter` from `context.md` **to judged survivors only** — unjudged findings (no `fp_verdict`) are included regardless of filter and rendered under an `Unvalidated (severity not judged)` section with a `[UNVALIDATED SEVERITY — not judged]` label, mirroring the SARIF behavior so a strict filter never silently drops them — and `Write` a `REPORT.md` mirroring the fp-judge template — YAML frontmatter (`stage: final-report`, `threat_model`, `severity_filter`, `total_primaries`, `reported_findings`), a severity-distribution table, then one section per reported finding grouped by severity (embed the Description / Code / Data flow / Impact / Recommendation body for CRITICAL/HIGH; reference the finding file for MEDIUM/LOW).

Either way, note in `${output_dir}/run-summary.md` that `REPORT.md` was orchestrator-synthesized (not judge-authored). Skip the SARIF generator and this check only if `${output_dir}/findings/` doesn't exist (Phase 2 failed). After this phase, update `${output_dir}/run-summary.md` with judge / SARIF / report status.

### Phase 9: Return Report

**Entry:** Phase 8b complete. **Exit:** every item in [Success Criteria](#success-criteria) verified true; `REPORT.md` returned to the caller.

Before composing the response, walk the [Success Criteria](#success-criteria) checklist below and confirm each bullet against on-disk artifacts (`TaskList` for cluster tasks, `ls`/`Read` for the files). If any criterion fails, surface the failure prominently in the response — do **not** hide a partial run behind a successful report.

Then `Read ${output_dir}/REPORT.md` and return its content to the caller. Append an Artifacts list pointing at `findings/`, `findings-index.txt`, `run-summary.md`, `dedup-summary.md`, `fp-summary.md`, `REPORT.md`, `REPORT.sarif`.

---

## Finding file frontmatter — three stages

Authoritative schema: `agents/go-review-worker.md` ("Finding File Format"). Three-stage write:

1. **Worker** — base fields (`id`, `bug_class`, `title`, `location`, `function`, `confidence`, `worker`) + seven body sections.
2. **Dedup-judge** — adds `merged_into` on duplicates, or `also_known_as` + `locations` on primaries that absorbed.
3. **FP+Severity judge** — adds `fp_verdict` + `fp_rationale` on every primary; on survivors (`TRUE_POSITIVE`/`LIKELY_TP`) also adds `severity`, `attack_vector`, `exploitability`, `severity_rationale`.

## Bug classes / clusters

Authoritative: `prompts/clusters/manifest.json`. 41 bug classes across 10 clusters. 6 clusters are gated `always` (`error-handling`, `nil-interface-pitfalls`, `defer-closure-hazards`, `static-hygiene` — 14 bug classes total) or `has_concurrency` (`concurrency-goroutines`, `concurrency-data-race` — 10 bug classes). 4 clusters are conditionally gated on other flags: `unsafe-boundary` on `has_unsafe`; `context-lifecycle` on `has_context`; `web-input-handling` on `has_net_http`; `deserialization-safety` on `has_serialization`. `unsafe-boundary` and `concurrency-goroutines` are fully consolidated (their sub-prompts are not re-read at runtime).

---

## Success Criteria

The phase exits already cover most of this; the orchestrator-visible end-state is:

- Every Phase-5 cluster task is `completed` (verify via `TaskList`).
- `${output_dir}/run-summary.md` exists and records resolved scope/context, `go.mod` probe result, worker claims vs index count, task status, and judge/SARIF status.
- Every primary finding (no `merged_into`) has `fp_verdict` + `fp_rationale`; every survivor (`TRUE_POSITIVE`/`LIKELY_TP`) also has `severity`, `attack_vector`, `exploitability`, `severity_rationale`.
- `REPORT.md` exists, severity-filtered per `severity_filter` (Phase 8b safety net guarantees this even when the fp-judge fails to write it).
- `REPORT.sarif` exists (Phase 8b safety net guarantees this).
