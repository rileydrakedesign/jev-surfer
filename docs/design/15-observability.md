# 15 · Observability

**Status:** draft for review
**Spec sections:** §18 (all), §11.9, §13.3 (failure logging), §17.3 (walk trace stats), §19 (log privacy), §23 Phase 1 (`--explain` flat mode), Phase 5 (decisions log, `surf stats`)
**Depends on:** 00-foundations, 09-router (owns `RouteTrace`), 07-judge (call stats), 10-lease, 12-delivery (CLI), 13-config, 14-security-privacy (salt, redaction), 16-evaluation (trace consumer)
**Code:** `surf/log/decisions.py` (record model, writer, reader, rotation), `surf/log/explain.py` (text renderer for `--explain`), `surf/log/stats.py` (aggregation for `surf stats`)

---

## 1. Purpose and scope

Three things let a person see what `surf` did and why:

1. **Decision records**: one JSON line per route in `.surf/logs/decisions.jsonl`.
2. **`surf route --explain`**: a readable dump of one route's `RouteTrace`.
3. **`surf stats`**: aggregates over the decision records.

| In scope (v1) | Out of scope (v1) |
|---|---|
| Record schema, privacy defaults, append safety with concurrent writers, rotation | Remote telemetry of any kind (14 §4.1) |
| `--explain` text and JSON formats | Dashboards, OpenTelemetry export |
| `surf stats` metrics and output | Stop-hook "files opened" feedback (spec §15.4 optional; v2) |
| Decision-log behavior for eval runs | Per-query eval reports (16) |

The spec adds three new files to the package layout (`log/explain.py`, `log/stats.py`). They're small and keep `decisions.py` focused on I/O.

---

## 2. Interfaces

```python
# surf/log/decisions.py
class DecisionLog:
    def __init__(self, logs_dir: Path, cfg: LogConfig, *, clock: Clock) -> None: ...
    def append(self, rec: DecisionRecord) -> None: ...       # never raises; see §4.2
    def iter_records(self, *, since: datetime | None = None) -> Iterator[DecisionRecord]: ...
    @property
    def corrupt_lines(self) -> int: ...                       # counted during the last iteration

def build_record(req: RouteRequest, res: RouteResult, trace: RouteTrace | None,
                 *, adapter: Adapter, cfg: Config, salt: bytes,
                 redactor: Redactor, clock: Clock) -> DecisionRecord: ...

# surf/log/explain.py
def render_explain(trace: RouteTrace, res: RouteResult, *, show_cards: bool = False,
                   show_requests: bool = False, width: int = 100) -> str: ...

# surf/log/stats.py
def compute_stats(records: Iterable[DecisionRecord], *, top: int = 10,
                  catalog: CatalogReader | None = None) -> StatsReport: ...
def render_stats(rep: StatsReport) -> str: ...
```

Who calls what:

| Caller | Call |
|---|---|
| `route/pipeline.py` (09) | `route()` returns `RouteResult` with `trace` always built internally (cheap) but only attached when `explain=True`. The pipeline hands `(req, res, trace)` to a `on_route_done` callback supplied by the caller |
| Adapters (12): Claude Code hook, MCP server, CLI | Pass `on_route_done = lambda …: log.append(build_record(…))`. Hook: emit stdout payload and flush **first**, then append, then exit |
| `eval/runner.py` (16) | Its own `DecisionLog` in the run directory (§4.6) |
| `cli.py` | `surf route --explain [--json] [--cards] [--show-requests]`, `surf stats …` |

`build_record` needs the trace even when `explain=False` (walk stats, latency breakdown). So the pipeline always passes the trace to the callback; only the public `RouteResult.trace` field is gated by `explain` (00 §3). This is a clarification for 09.

---

## 3. Data structures

### 3.1 `DecisionRecord` (one line)

Spec §18.1 fields, plus additions marked **new**.

```python
class JudgeStats(BaseModel):
    backend: str; model: str
    calls: int; input_tokens: int | None          # None when the backend doesn't report tokens
    errors: list[JudgeError] = []                 # new: {code: "timeout"|"http_5xx"|"http_4xx"|"breaker_open"|"invalid_response", status: int|None, n: int}

class WalkStats(BaseModel):
    levels: int; requests: int
    dead_end_guard: bool; deadline_hit: bool
    candidates: int                               # new: walk candidates before expansion

class LeaseInfo(BaseModel):                       # new
    action: Literal["none", "reuse", "union", "replace", "expired", "disabled"]
    generation: int | None; stale: bool

class DecisionRecord(BaseModel):
    v: Literal[1] = 1                             # new: record schema version
    route_id: str                                 # "r_" + ULID (00 §3)
    ts: str                                       # UTC, RFC 3339, second precision
    surf_version: str                             # new
    adapter: Literal["claude_code", "mcp", "cli", "eval"]   # new
    session_id: str | None
    prompt_hash: str                              # "hmac-sha256:<hex>" (14 §4.8, D-14-1)
    prompt_chars: int                             # new: length of the raw prompt
    prompt_text: str | None = None                # new: redacted text, only if privacy.log_prompt_text
    redactions: dict[str, int] = {}               # new: type -> count (14 §4.2)
    status: RouteStatus
    route_mode: Literal["skip", "lease", "small", "walk", "flat"] | None   # new: which path ran
    continuity: ContinuityOut | None              # {choice, confidence}
    needs_context: float | None
    path_hits: list[SurfaceId]
    walk: WalkStats | None
    pool_size: int | None
    selected: list[SurfaceId]
    capabilities: CapOut                          # {use: [...], skip: [...]}
    low_confidence: bool                          # new
    note_lines: int                               # new: 0 when nothing injected
    lease: LeaseInfo                              # new
    judge: JudgeStats | None
    latency_ms: LatencyMs                         # {total, call1?, walk?, expand?, final?, overhead?, process?}
                                                  # process = hook process start + imports before routing (12 §7); not in total
    index_head: str | None
    error: ErrorInfo | None = None                # new: {type: "KeyError", where: "route/walk.py:212"}
    truncated: bool = False                       # new: lists were cut to fit record_max_bytes
```

Rules:
- Keys are written in model field order; compact separators; `None`-valued optional fields are **omitted** (`exclude_none=True`) to keep lines small. Readers treat a missing key as `None`.
- `error.type` is the exception class name; `error.where` is `module:lineno` of the innermost `surf` frame. **The exception message is not stored**, since it may quote prompt or card text (14 T8). The full traceback goes to stderr only when `SURF_DEBUG=1`.
- Ids in `selected`/`path_hits` are catalog ids (already committed data), not prompt text.

### 3.2 Files

```
.surf/logs/
├── decisions.jsonl        # current
├── decisions.1.jsonl      # newest rotated … decisions.4.jsonl oldest   (log.max_files = 5 total)
└── decisions.lock         # empty file used for flock
```

All created with mode `0600`; the directory `0700`. `.surf/logs/` is gitignored (spec §9.1).

### 3.3 `RouteTrace` fields consumed here

09-router owns `RouteTrace`. `--explain`, `build_record` and eval attribution (16 §4.6) need these fields. This list is the contract 09 must satisfy:

| Field | Type | Used by |
|---|---|---|
| `mode` | `"small" \| "walk" \| "flat"` | record, explain |
| `call1` | `{needs_context: float, continuity: ChoiceA \| None, caps: dict[id, float], content: dict[id, float] (small mode), latency_ms}` | all |
| `path_hits` | `list[{id, strength, raw_token_index}]` (index into the prompt's token list, not the token) | all |
| `walk.nodes` | `list[WalkNode{id, parent, depth, p, cum, action: "expand"\|"flatten"\|"candidate"\|"pruned"\|"guard"\|"skipped_path_hit", chunk_no}]` | explain, attribution |
| `walk.levels`, `walk.requests`, `walk.dead_end_guard`, `walk.deadline_hit`, `walk.depth_cap_hit`, `walk.latency_ms` | scalars | record, stats, attribution |
| `expansion` | `list[{id, score, via_kind, via_anchor, admitted: bool}]` incl. the top 20 **rejected** (score < threshold) | explain, attribution |
| `pool` | `list[{id, source: "path_hit"\|"walk"\|"flatten"\|"expansion"\|"small", rank, pre_score}]` in rank order, **before** truncation | explain, attribution |
| `pool_cut` | `list[id]` removed by `max_candidates` | attribution |
| `final` | `dict[id, float]` final Noul per judged candidate; `final_latency_ms` | all |
| `above_threshold` | `list[id]` passing `tau_final` or the path-hit floor, in priority order | attribution |
| `budget_cut` | `list[id]` above threshold but cut by `max_pointers` | attribution |
| `collapsed` | `dict[dir_id, list[file_id]]` | explain, eval matching |
| `selected` | `Selection` | all |
| `lease` | `LeaseInfo` + `delta: list[id]` (additions on `extends`) | record, eval |
| `judge_requests` | `list[{key, state, questions}]` only when `show_requests` / eval record mode (heavy) | explain `--show-requests`, fixture keys |
| `latency_ms` | `LatencyMs` | record |

---

## 4. Behavior

### 4.1 Building a record

1. `prompt_hash = redact.prompt_hash(raw_prompt, salt=salt)`; salt from `load_or_create_salt(.surf/cache)` (14 §3.4), loaded once per process.
2. If `privacy.log_prompt_text`: `prompt_text = redactor.redact(raw).text[:2000]`.
3. Copy fields from `RouteResult` and trace; `route_mode` from status + `trace.mode`: `skipped → skip`, `lease-reuse → lease`, otherwise `trace.mode`.
4. Serialize. If the encoded line exceeds `log.record_max_bytes`, drop list tails in this order until it fits: `path_hits`, `capabilities.skip`, `capabilities.use`, `selected` (to 12), `prompt_text`; set `truncated = true`.

### 4.2 Append safety with concurrent writers

Writers are separate short-lived processes (each `UserPromptSubmit` hook run, CLI invocations) plus possibly one long-lived MCP server. Requirements: no interleaved bytes within a line, no lost rotation, never block a route noticeably, never raise.

Design: **one `write()` per record on an `O_APPEND` descriptor, opened per append, with an advisory lock that guards rotation.**

```python
def append(self, rec):
    if not self.cfg.enabled: return
    try:
        line = rec.model_dump_json(exclude_none=True).encode() + b"\n"       # ≤ record_max_bytes (16 KiB)
        with self._lock(timeout_ms=self.cfg.lock_timeout_ms) as locked:     # flock(LOCK_EX) on decisions.lock, polled
            fd = os.open(self.path, os.O_WRONLY | os.O_APPEND | os.O_CREAT, 0o600)
            try:
                _write_all(fd, line)                                        # loop only on short writes
                if locked and os.fstat(fd).st_size >= self.cfg.max_bytes:
                    self._rotate(fd)
            finally:
                os.close(fd)
    except Exception:
        self._warn_once()                                                   # stderr, once per process
```

Why each piece:

| Choice | Reason |
|---|---|
| `O_APPEND` | The kernel sets the offset to EOF atomically with each `write()`. Two processes can't overwrite each other's bytes |
| Single `write()` of the whole line | On local POSIX filesystems a regular-file `write()` under `O_APPEND` isn't split by other appenders (Linux holds the inode lock for the write). `PIPE_BUF` (4096 on Linux, 512 on macOS) governs pipes, not regular files, so it's not the relevant bound; we still cap lines at 16 KiB so that a short write is essentially impossible |
| `flock` on a separate lockfile | Makes the rotation check-and-rename atomic with respect to other writers, and makes appends safe on filesystems where `O_APPEND` atomicity is weaker (NFS). The lockfile is never renamed, so the lock identity survives rotation |
| Lock timeout 50 ms, then write **without** the lock | A stuck process holding the lock must not stall routing. Unlocked appends are still `O_APPEND` single writes; only rotation is skipped (it happens on a later append) |
| Open per append | A long-lived MCP server would otherwise keep writing into a file that another process rotated to `.1`. `open`+`close` costs ~20–40 µs |
| Reader tolerance | Readers skip lines that fail JSON parsing and count them (`corrupt_lines`). A torn line (crash mid-write on NFS) loses one record, never the file |

Windows: `flock` is replaced by `msvcrt.locking` on the lockfile (1 byte, non-blocking, polled); `O_APPEND` behaves the same for single writes. Rotation with `os.replace` fails if another process has the file open without `FILE_SHARE_DELETE`; on failure, rotation is skipped and retried on the next append (Q-15-2).

### 4.3 Rotation (10 MB × 5, spec §18.2)

Called under the lock with the open fd.

1. `st_fd = fstat(fd)`, `st_path = stat(self.path)`. If `(st_dev, st_ino)` differ, another writer already rotated: return.
2. If `st_path.st_size < max_bytes`: return (someone rotated and new writes landed).
3. `unlink(decisions.{max_files-1}.jsonl)` (ignore missing).
4. For `k = max_files-2 … 1`: `os.replace(decisions.k.jsonl, decisions.{k+1}.jsonl)` (ignore missing).
5. `os.replace(decisions.jsonl, decisions.1.jsonl)`. The next append creates a fresh file.

A writer that opened the file before step 5 and hasn't written yet will write into `decisions.1.jsonl`. That record is still read by `surf stats` (it reads all files). Total disk use ≤ `max_bytes × max_files` + one record.

### 4.4 `surf route --explain` output

Text format (default). Deterministic ordering; ASCII only; ids, not card text, unless `--cards`.

```
surf route r_01J8Z3K4QW…  status=routed  mode=walk  continuity=new(0.91)  total=1240ms
prompt: 47 chars, hash hmac-sha256:3b1f…  redactions: none
index: a1b2c3d (fresh)  judge: jev/jev-1.13.0  calls=4  input_tokens=14210

path hits: none

call 1  [310 ms]
  needs_context            0.97
  cap mcp:supabase         0.82  USE
  cap skill:db-migrations  0.44  -
  cap mcp:figma            0.03  SKIP

walk  [620 ms, 2 levels, 5 requests, speculative L1 used]
  root
  |-- 0.71  src/                         >  expand   (1204 files)
  |   |-- 0.66  src/fulfillment/         =  flatten  (18 files, cum 0.47 x0.9)
  |   |-- 0.52  src/api/orders/          =  flatten  (9 files, cum 0.37 x0.9)
  |   |-- 0.21  src/billing/             -  pruned
  |   `-- +19 more below 0.35 (max 0.18)
  |-- 0.44  docs/                        >  expand   (96 files)
  |   `-- 0.61  docs/fulfillment/        =  flatten  (4 files)
  |-- 0.30  supabase/                    -  pruned
  `-- +7 more below 0.35 (max 0.12)

expansion  (threshold 0.30)
  + db:orders          0.74  schema_ref  <- code:src/fulfillment/ship.ts
  + db:shipments       0.58  schema_ref  <- code:src/fulfillment/ship.ts
  . code:src/jobs/x.ts 0.22  co_change   <- code:src/api/orders/[id].ts   (rejected)

pool  31 of max 40 (cut 0)

final pass  [290 ms]  threshold 0.60, path-hit floor 0.20
  0.88  SELECT  code:src/fulfillment/ship.ts          walk
  0.81  SELECT  db:orders                             expansion
  0.72  SELECT  doc:docs/fulfillment/shipping-lifecycle.md  flatten
  0.55  below   db:shipments                          expansion
  ... 27 more below threshold (max 0.49)

selection  3 content (max 12), budget cut: none, collapsed: none
lease  action=replace generation=1

note (7 lines):
  [surf] Likely relevant — open as needed, nothing is preloaded:
  ...
```

Symbols: `>` expanded, `=` flattened, `+` candidate/admitted, `-` pruned, `!` dead-end guard, `.` rejected. For each walk node at most `explain.max_children` (8) non-chosen children are listed, then a `+N more below τ (max p)` line. Chosen children are always listed.

Other modes:

| Flag | Output |
|---|---|
| `--json` with `--explain` | `{"result": RouteResult, "trace": RouteTrace}` as JSON; stable key order; for tools and eval |
| `--cards` | each line followed by the indented card text as sent (after query-time redaction) |
| `--show-requests` | appends the exact redacted judge requests (state + questions) as JSON blocks. The privacy audit view: what would leave the machine (14 §4.1) |
| `--dry-run` (with `--explain`) | uses the `null`-answer judge to show request composition without calling the network; walk stops at level 1 |

Phase 1 exit asks for `surf route --explain` in **flat mode** (before the router exists). In Phase 1, `mode=flat` renders the call-1 and final-pass sections only; the walk section reads `walk  (not run: flat mode)`.

`--explain` never writes a decision record with `adapter=cli` unless `--log` is passed: explaining is debugging, and it would skew `surf stats`.

### 4.5 `surf stats`

```
surf stats [--since 7d|2026-09-01] [--session ID] [--adapter claude_code|mcp|cli] [--top 10] [--json]
```

Reads `decisions.4.jsonl … decisions.jsonl` in that order (oldest first), filters, aggregates.

| Section | Metric | Definition |
|---|---|---|
| Volume | routes, sessions, date range | counts |
| Status mix | share per `RouteStatus` | count / total |
| Latency | p50, p95, max of `latency_ms.total` per path type | nearest-rank percentile; path type = `route_mode` (skip, lease, small, walk) |
| Stage latency | p50 of `call1`, `walk`, `final` among walk routes | |
| Judge | mean calls and input tokens per routed route; error counts by `code` | |
| Walk | dead-end guard rate, deadline-hit rate, mean levels | over `route_mode = walk` |
| Lease | reuse rate = `lease-reuse` / (routes with a lease) | |
| Note | median `selected` size; share with `low_confidence` | over `status = routed` |
| Top surfaces | `--top` most selected content ids; most `use`/`skip` capabilities | ids no longer in the catalog marked `(gone)` |
| Health | corrupt lines, truncated records, `error` records by `type`/`where` | |
| Privacy | redaction counts by type (sum) | |

Nearest-rank percentile: `sorted(x)[ceil(q·n) − 1]`; `n < 20` prints percentiles with a `(n<20)` marker.

`--json` emits `StatsReport` (pydantic) for scripts. With no records: `no decisions recorded in .surf/logs (routing may be off, or logging disabled: log.enabled)` and exit 0.

### 4.6 Eval runs

`surf eval` routes through the same pipeline, but records go to `<run_dir>/decisions.jsonl` with `adapter=eval`, never to `.surf/logs/`. That keeps `surf stats` about real usage, and the eval run directory self-contained (16 §3.8).

---

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `log.enabled` | bool | `true` | Off: no records written; `--explain` still works |
| `log.max_bytes` | int | `10_000_000` | Spec §18.2 "10 MB" |
| `log.max_files` | int | `5` | Total files including the current one |
| `log.record_max_bytes` | int | `16384` | §4.1 step 4 |
| `log.lock_timeout_ms` | int | `50` | §4.2 |
| `privacy.log_prompt_text` | bool | `false` | Spec §16; stores redacted text (14 §4.8) |
| `explain.max_children` | int | `8` | Pruned children listed per walk node |

---

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| `.surf/logs/` missing | Created with `0700` on first append |
| Disk full / read-only FS / permission denied | `append` swallows the error, warns once per process on stderr, route result unaffected (00 §5) |
| Lock held by a crashed process | `flock` is released by the kernel on process death; no stale-lock cleanup needed |
| Lock not acquired within 50 ms | Append without lock; skip rotation |
| Two processes both see size ≥ max | The second finds the inode changed (§4.3 step 1) and returns |
| Record > `record_max_bytes` after truncation | Write a minimal record (`route_id, ts, status, truncated=true`) |
| Clock skew between writers | `ts` is informational; readers don't assume ordering; `--since` filters by `ts` |
| Corrupt or unknown-version line | Skipped, counted in `corrupt_lines`; `v > 1` lines skipped with one warning |
| Hook process killed mid-route | No record (appends happen after the payload is emitted) |
| `--explain` on `lease-reuse` | Shows call 1 and `lease action=reuse`; walk section `(not run: lease reuse)` |
| `--explain` on `error` | Prints whatever trace exists plus `error.type`/`where`; with `SURF_DEBUG=1` the traceback |
| `surf stats` over 50 MB of logs | Streams line by line; memory bounded by aggregation (top-N via counters) |
| Salt missing when reading | Not needed for reading; stats never reverses hashes |

---

## 7. Performance budget

| Operation | Budget |
|---|---|
| `build_record` | ≤ 0.3 ms |
| `append` (uncontended, local SSD) | ≤ 0.5 ms p95, includes open/lock/write/close |
| `append` with rotation | ≤ 5 ms |
| Worst case added to a hook (lock timeout) | 50 ms, only when contended |
| `render_explain` for a 40-candidate walk route | ≤ 10 ms |
| `surf stats` over 5 × 10 MB (~100k records) | ≤ 3 s, ≤ 100 MB RSS |

---

## 8. Test plan

| Test | Kind |
|---|---|
| Record round-trip: `DecisionRecord` → line → parse, for every `RouteStatus`; `exclude_none` behavior; unknown keys ignored on read | unit |
| Privacy: no raw prompt substring (≥ 8 chars) appears in any record when `log_prompt_text=false`; exception messages never stored (route through a judge that raises `ValueError("<prompt text>")`) | unit |
| **Concurrency**: 16 processes × 2,000 appends each with `max_bytes = 256 KiB`. Assert: every line parses; total parsed = 32,000 (± records lost only if the lock-timeout path raced rotation; expected 0); ≤ `max_files` files; no file > `max_bytes + record_max_bytes` | integration (multiprocessing) |
| Concurrency with a long-lived writer: one process appending in a loop while others rotate; its records continue to land in the current file after rotation | integration |
| Lock timeout path: hold the lock in another process for 1 s; `append` returns in ≤ 60 ms and the record is written | integration |
| Rotation arithmetic: sizes, names, oldest deleted | unit |
| Unwritable dir: `append` doesn't raise; one stderr warning | unit |
| `--explain` golden files: small-repo route, walk route with guard + deadline, lease reuse, error, flat mode (Phase 1) — fixture judge, fake clock (latency fixed) | golden |
| `--explain --json` validates against the `RouteTrace` model | unit |
| `surf stats` golden over a synthetic log of 500 records with known distribution; percentile correctness; `(gone)` marking; corrupt-line counting | golden |
| Hook ordering: stdout payload is flushed before the append (stub `DecisionLog.append` to sleep; measure time-to-payload) | integration |

---

## 9. Acceptance criteria

1. Phase 1: `surf route --explain` works in flat mode on both benchmark repos.
2. Phase 5: every route through every adapter writes exactly one record (verified by the fail-open test matrix in 00 §6.1 checking record count and status).
3. The concurrency test passes on Linux and macOS in CI; Windows runs the single-process tests.
4. `surf stats` reproduces spec §18.2 items: status mix, latency percentiles, top selected surfaces, dead-end rate.
5. Records meet the privacy rules of 14 (hash only by default; no exception text).

---

## 10. Deviations from the spec and open questions

**Deviations**

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-15-1 | §18.1 `prompt_hash: "sha256:…"` | `hmac-sha256` with a per-checkout salt (see D-14-1) | Dictionary reversal of short prompts |
| D-15-2 | §18.1 record fields | Adds `v`, `surf_version`, `adapter`, `prompt_chars`, `redactions`, `route_mode`, `low_confidence`, `note_lines`, `lease`, `judge.errors`, `walk.candidates`, `error`, `truncated`; optional fields omitted when null | Needed by `surf stats`, §13.3 failure logging, and debugging without prompt text |
| D-15-3 | §18.2 rotation "10 MB × 5 files" | Interpreted as 5 files total (current + 4 rotated), ≤ 50 MB | Spec is ambiguous; this is the common reading |
| D-15-4 | §18 is silent on eval | Eval records go to the run directory, not `.surf/logs/` | Keeps usage stats clean |
| D-15-5 | §15.7 `--explain` "prints walk trace" | Adds `--json`, `--cards`, `--show-requests`, `--dry-run` | Tooling for eval, privacy audit, and cost-free debugging |

**Open questions**

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-15-1 | Should `selected` ids be hashed too (paths can reveal what a developer works on)? | No; logs are local, `0600`, gitignored | User feedback / privacy review |
| Q-15-2 | Windows rotation can fail when another process holds the file open | Retry on next append; if the file reaches 2 × `max_bytes`, open with a new name `decisions.<ulid>.jsonl` and let the reader glob | Windows CI results |
| Q-15-3 | Log retention by age (e.g. 30 days) in addition to size? | No in v1 | User feedback |
| Q-15-4 | `RouteTrace` field list (§3.3) must be adopted by 09-router | As listed | Resolved: 09 §3.4 implements it (superset) |
