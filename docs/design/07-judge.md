# 07 · Judge: protocol, backends, resilience

**Status:** draft for review
**Spec sections:** §13, §20 (also §11.9 cost, §16 `[judge]`, §17.7 fixture judge, §19.2)
**Depends on:** 00-foundations (Deadline, Clock, error model), 13-config (`judge.*`, `router.thresholds.*`), 14-security-privacy (redaction happens before a request reaches the judge), 15-observability (ledger → decision record)
**Code:** `surf/judge/base.py`, `surf/judge/jev.py`, `surf/judge/jev_wire.py` (new, see D-07-2), `surf/judge/systemone_local.py`, `surf/judge/llm.py`, `surf/judge/null.py`, `surf/judge/fixture.py`, `surf/judge/breaker.py` (new)

---

## 1. Purpose and scope

The judge is the only component that talks to a model. The router (09) speaks only the protocol defined here; everything provider-specific (URLs, auth, request/response JSON, key names) lives behind it.

**In scope (v1)**
- Async protocol with a sync facade; typed questions/answers; per-request outcomes for batches.
- Backends: `jev` (providers `typesafe`, `openrouter`, `vercel`, `cloudflare`), `systemone-local`, `llm` (eval baseline), `null`, `fixture` (record/replay).
- Resilience: per-request timeout clipped to the route deadline, retry-once, concurrency semaphore, circuit breaker persisted across processes.
- Question batching limits (≤ 40 questions per request, balanced auto-split).
- Token and cost accounting per route.
- Threshold-profile lookup (which `router.thresholds.<profile>` table applies).

**Out of scope**
- Redaction (14; the judge only asserts it was done), question wording (09, 16), threshold values (13, tuned by 16), answer caching in production (Q-07-5).

---

## 2. Interfaces

### 2.1 Why async (decision)

The router needs real parallelism in three places: walk levels fan out to N chunk requests (`ask_many`), walk level 1 runs **speculatively alongside call 1** and must be **cancellable** when call 1 says `same`, and call 1 itself splits into parallel chunks in small-repo mode. Options considered:

| Option | Parallelism | Cancellation of in-flight requests | Deadline enforcement | Fit |
|---|---|---|---|---|
| Sync `httpx.Client` + `ThreadPoolExecutor` | yes | **no** (threads can't be cancelled; a discarded speculative walk keeps its sockets and semaphore slots until timeout) | per-request only | poor |
| `asyncio` + `httpx.AsyncClient` | yes, one thread | `task.cancel()` closes the stream and frees the semaphore slot immediately | `asyncio.timeout()` around a whole phase | good |
| trio/anyio | yes | yes | yes | extra dependency; MCP SDK uses anyio but runs fine on asyncio |

**Decision:** the core protocol is `async`. The pipeline (`route_async`) is async end to end. Sync callers (CLI, hook entrypoint, simple tests) use `route()` / `SyncJudge`, which call `asyncio.run()` once per invocation. The MCP server (anyio on the asyncio backend) awaits `route_async` directly. No backend may block the event loop (file I/O for the breaker and fixtures is small and done synchronously on purpose, ≤ 1 ms; see §7).

### 2.2 Types (`judge/base.py`)

```python
class NoulQ(BaseModel, frozen=True):
    type: Literal["noul"] = "noul"
    instructions: str

class ChoiceQ(BaseModel, frozen=True):
    type: Literal["choice"] = "choice"
    instructions: str
    options: dict[str, str]                 # option key -> description; ≤ 255 (spec §20); v1 uses 3

Question = Annotated[NoulQ | ChoiceQ, Field(discriminator="type")]

class NoulA(BaseModel, frozen=True):
    p: float                                # P(yes), clamped to [0, 1]

class ChoiceA(BaseModel, frozen=True):
    choice: str                             # always a key of options
    probs: dict[str, float]                 # renormalized to sum 1 over options
    confidence: float                       # backend-reported; else max(probs)

Answer = NoulA | ChoiceA

class Purpose(StrEnum):                     # accounting / trace only; never part of any cache key
    CALL1 = "call1"; WALK = "walk"; FINAL = "final"; EVAL = "eval"; PING = "ping"

class JudgeRequest(BaseModel, frozen=True):
    state: dict[str, str]                   # ordered; keys from 09 §3.2 (request, project, ...)
    questions: dict[str, Question]          # ordered; caller's keys (e.g. "cap:mcp:supabase")
    purpose: Purpose
    redacted: bool                          # must be True for network backends (ValueError otherwise)

class Usage(BaseModel):
    input_tokens: int = 0
    output_tokens: int = 0
    estimated: bool = False                 # True when the provider didn't report usage

class JudgeResponse(BaseModel):
    answers: dict[str, Answer]              # keyed by the caller's question keys
    missing: list[str]                      # asked but not answered (treated as "unjudged")
    usage: Usage
    latency_ms: int
    attempts: int                           # 1 or 2
    backend: str; provider: str | None; model: str

class ErrorKind(StrEnum):
    TIMEOUT = "timeout"; CONNECT = "connect"; HTTP_5XX = "http_5xx"; RATE_LIMITED = "rate_limited"
    HTTP_4XX = "http_4xx"; AUTH = "auth"; MALFORMED = "malformed"; TOO_LARGE = "too_large"
    BREAKER_OPEN = "breaker_open"; NO_KEY = "no_key"; DEADLINE = "deadline"
    CANCELLED = "cancelled"; NULL_BACKEND = "null_backend"; FIXTURE_MISS = "fixture_miss"

class JudgeError(Exception):
    kind: ErrorKind; status_code: int | None; detail: str   # detail never contains prompt text

class Availability(BaseModel):
    ok: bool
    reason: ErrorKind | None                # NO_KEY | BREAKER_OPEN | NULL_BACKEND
    retry_at: datetime | None               # when an open breaker half-opens

@dataclass
class CallCtx:
    deadline: Deadline                      # 00 §5; route or walk deadline, whichever is tighter
    ledger: JudgeLedger                     # per-route accumulator (§3.3)
```

### 2.3 Protocol

```python
class Judge(Protocol):
    name: str                               # "jev" | "systemone-local" | "llm" | "null" | "fixture"
    provider: str | None
    model: str
    threshold_profile: str | None           # §4.9; None for null

    def availability(self) -> Availability: ...           # cheap, no network; reads breaker file
    async def ask(self, req: JudgeRequest, *, ctx: CallCtx) -> JudgeResponse: ...   # raises JudgeError
    async def ask_many(self, reqs: Sequence[JudgeRequest], *, ctx: CallCtx
                       ) -> list[JudgeResponse | JudgeError]: ...                   # never raises
    async def aclose(self) -> None: ...

class SyncJudge:                            # facade for sync callers; raises if a loop is running
    def __init__(self, judge: Judge): ...
    def ask(self, req, *, ctx) -> JudgeResponse: ...
    def ask_many(self, reqs, *, ctx) -> list[JudgeResponse | JudgeError]: ...

def make_judge(cfg: JudgeConfig, *, state_dir: Path | None, clock: Clock,
               env: Mapping[str, str]) -> Judge: ...   # the only place that reads env (keys)
```

Differences from spec §13.1 (see D-07-1): async; `ask_many` returns a per-request outcome instead of all-or-nothing; the timeout comes from `ctx.deadline` plus config, not a kwarg; `purpose` and `redacted` are carried on the request.

### 2.4 `BaseJudge` (shared machinery)

Every network backend subclasses `BaseJudge`, which implements `ask`/`ask_many` around one abstract hook:

```python
class BaseJudge(ABC):
    async def _send(self, req: JudgeRequest, *, timeout_ms: int) -> RawResult: ...  # one HTTP attempt
```

`BaseJudge` owns: splitting (§4.2), semaphore (§4.4), timeout computation (§4.5), retry (§4.6), breaker bookkeeping (§4.7), answer validation (§4.3) and ledger updates (§4.8). Backends only encode, send and decode.

Callers: `route/call1.py`, `route/walk.py`, `route/final.py` (via `pipeline.py`), `eval/runner.py`, `surf doctor` (`Purpose.PING`, one Noul).

---

## 3. Data structures

### 3.1 Wire isolation

All knowledge of the Jev/System One HTTP format is in **`judge/jev_wire.py`**, used by both `jev.py` and `systemone_local.py`:

```python
@dataclass(frozen=True)
class ProviderSpec:
    name: str                    # typesafe | openrouter | vercel | cloudflare | local
    base_url: str                # may contain {account_id}/{gateway_id} placeholders
    path: str
    auth_headers: Callable[[Mapping[str, str]], dict[str, str]]   # env -> headers
    key_env: tuple[str, ...]     # env vars required
    model_id: Callable[[str], str]                                # "jev-1.13.0" -> provider's model string
    envelope: Literal["native", "chat"]                           # body codec

PROVIDERS: dict[str, ProviderSpec]
def encode(req: JudgeRequest, *, model: str, spec: ProviderSpec) -> tuple[bytes, KeyMap]: ...
def decode(body: bytes, *, keymap: KeyMap, req: JudgeRequest, spec: ProviderSpec) -> RawResult: ...
```

**Key mapping.** Caller keys contain `:` and arbitrary path characters (`w:code:src/api/orders/[id].ts`). The wire uses opaque keys `q000`…`q039` in request order and `KeyMap` maps back. This removes any dependency on the provider's key grammar (UNVERIFIED) and keeps paths out of JSON keys.

**Assumed native envelope (UNVERIFIED; docs.typesafe.ai is unreachable from the design sandbox).** Request per spec §11.4 illustration:

```json
{"model": "jev-1.13.0",
 "state": {"request": "...", "project": "..."},
 "questions": {"q000": {"type": "noul", "instructions": "..."},
               "q001": {"type": "choice", "instructions": "...", "options": {"same": "...", "extends": "...", "new": "..."}}}}
```

Assumed response:

```json
{"answers": {"q000": {"p": 0.83},
             "q001": {"choice": "same", "probs": {"same": 0.91, "extends": 0.07, "new": 0.02}, "confidence": 0.91}},
 "usage": {"input_tokens": 1234}}
```

`decode` accepts a small set of aliases per field (`p` | `probability` | `yes`; `choice` | `answer`; `probs` | `probabilities` | `distribution`) so a field-name surprise found in Phase 0 is a one-line change. Everything in this subsection is **UNVERIFIED** and must be confirmed by the Phase 0 conformance test (§8.4) before any eval run.

### 3.2 Provider table (all values UNVERIFIED)

| Provider | Base URL (default) | Auth | Key env | Model string | Envelope |
|---|---|---|---|---|---|
| `typesafe` | `https://api.typesafe.ai` + `/v1/systemone` | `Authorization: Bearer $KEY` | `TYPESAFE_API_KEY` | `jev-1.13.0` | native |
| `openrouter` | `https://openrouter.ai/api/v1/...` | `Authorization: Bearer $KEY` | `OPENROUTER_API_KEY` | `typesafe/jev-1.13.0` (guess) | native or chat |
| `vercel` | `https://ai-gateway.vercel.sh/v1/...` | `Authorization: Bearer $KEY` | `AI_GATEWAY_API_KEY` | `typesafe/jev-1.13.0` (guess) | native or chat |
| `cloudflare` | `https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_id}/typesafe/...` | upstream key + optional `cf-aig-authorization: Bearer $CF_AIG_TOKEN` | `TYPESAFE_API_KEY` (+ `CF_AIG_TOKEN`) | `jev-1.13.0` | native (proxied) |
| `local` (`systemone-local`) | `judge.local.base_url` + `/v1/systemone` | optional Bearer | `SYSTEMONE_LOCAL_API_KEY` (optional) | `judge.model` | native |

Escape hatches (13-config): `judge.base_url`, `judge.path`, `judge.cloudflare.account_id`, `judge.cloudflare.gateway_id`. If a gateway turns out to require an OpenAI-style `chat` envelope, that codec is added inside `jev_wire.py` only.

### 3.3 Ledger

```python
class JudgeCallRecord(BaseModel):          # one per logical request (retries folded in)
    purpose: Purpose; n_questions: int; attempts: int
    outcome: Literal["ok", "partial", "error", "cancelled"]
    error_kind: ErrorKind | None; status_code: int | None
    usage: Usage; latency_ms: int

class JudgeLedger:                         # one per route, not thread-shared; asyncio-safe (single loop)
    records: list[JudgeCallRecord]
    def summary(self) -> JudgeUsageSummary: ...

class JudgeUsageSummary(BaseModel):        # → decision record "judge" block (15)
    backend: str; provider: str | None; model: str
    calls: int                             # logical requests sent (excl. retries, incl. cancelled)
    attempts: int; retries: int
    questions: int; input_tokens: int; output_tokens: int; tokens_estimated: bool
    cost_usd: float                        # input_tokens × judge.price_per_mtok_input / 1e6
    failures: dict[ErrorKind, int]
    cancelled: int                         # e.g. discarded speculative walk
```

### 3.4 Breaker state file

`.surf/cache/judge_state.json` (gitignored with the rest of `cache/`):

```json
{"version": 1,
 "breakers": {
   "jev|typesafe|jev-1.13.0": {
     "consecutive_failures": 2,
     "open_until": null,
     "last_failure_kind": "timeout",
     "last_failure_at": "2026-09-23T21:48:02Z",
     "opened_total": 3 }}}
```

Key = `backend|provider|model`. Timestamps are wall clock (UTC ISO), because monotonic clocks aren't comparable across processes.

### 3.5 Fixture store

```
<fixture_dir>/                     # tests/fixtures/judge/<set>/ in the surf repo; .surf/eval/recordings/ in user repos
  answers.jsonl                    # one record per (model, state, question), sorted by key
```

```json
{"key": "sha256:…", "model": "jev-1.13.0", "profile": "jev",
 "answer": {"p": 0.83}, "tokens": 31}
```

Optional debug sidecar `requests.jsonl` (off by default; `--record-requests`) stores the canonical state and question text per key. It is the only place raw prompts could land on disk, which is why it is opt-in (14).

---

## 4. Behavior / algorithm

### 4.1 `ask` pipeline (BaseJudge)

```
ask(req, ctx):
  1. assert req.redacted (network backends)                       -> ValueError (programming error)
  2. a = availability(); if not a.ok -> raise JudgeError(a.reason) (no network, no ledger failure count)
  3. if len(req.questions) > max_questions_per_request
        or est_tokens(req) > max_request_tokens:
        return merge(await ask_many(split(req), ctx))              # §4.2; partial -> missing keys
  4. async with semaphore (acquire bounded by ctx.deadline)        # §4.4
  5. t = request_timeout(ctx)                                      # §4.5; t < min_request_ms -> DEADLINE
  6. raw = await _send(req, timeout_ms=t)   with retry-once policy  # §4.6
  7. validate + map keys back                                       # §4.3
  8. ledger.add(record); breaker.on_result(...)                     # §4.7, §4.8
```

`ask_many(reqs, ctx)`: `asyncio.gather(*(self._ask_one(r, ctx) for r in reqs), return_exceptions=True)` where `_ask_one` converts `JudgeError` into a returned value; `CancelledError` propagates (the caller cancelled the whole batch). Results are in input order. Breaker bookkeeping is done **once per `ask_many` batch** (§4.7).

### 4.2 Question batching limits

- Hard cap `judge.max_questions_per_request` = 40 (spec §2 principle 5). Router `chunk_size` must be ≤ this (13 validates).
- Token cap `judge.max_request_tokens` = 8,000 (estimated, §4.8). With cards ≤ 150 tokens and state ≤ ~2,000 tokens, 40 dir cards fit; the cap exists to catch pathological state.
- `split(req)`: `k = ceil(n / cap)` balanced chunks (sizes differ by ≤ 1; 41 → 21 + 20, never 40 + 1), preserving question order. **Choice questions are always in chunk 0.** Every chunk repeats the full `state`. Each question is asked exactly once.
- If a single question plus state exceeds `max_request_tokens` → `TOO_LARGE`, not sent (the router truncates state first, so this indicates a bug or a giant card).
- `merge`: answers union; keys from failed chunks go to `missing`; the merged response is `outcome="partial"` if any chunk failed, and `JudgeError` only if all failed.

The router pre-chunks explicitly (it needs chunk-level trace entries), so auto-split is a safety net, not the primary mechanism.

### 4.3 Answer validation

| Check | Action |
|---|---|
| Body not JSON / missing `answers` | `MALFORMED` for the whole request |
| Unknown wire key in answers | ignored, logged at debug |
| Asked key absent | added to `missing` |
| Noul `p` NaN, non-numeric | that key → `missing`; if > 50 % of keys invalid → `MALFORMED` |
| Noul `p` outside [0, 1] by ≤ 1e-6 | clamped; further outside → invalid |
| Choice `choice` not an option key | invalid → `missing` |
| Choice `probs` missing | `probs = {choice: confidence}` + others 0 |
| Choice `probs` not summing to 1 | renormalized |
| Choice `confidence` missing | `max(probs.values())` |

### 4.4 Concurrency

- One `asyncio.Semaphore(judge.max_concurrency)` (default 16) per `Judge` instance, i.e. per process. It is not coordinated across processes (two Claude Code sessions can each run 16); documented, not solved in v1.
- Acquisition is bounded: `asyncio.timeout(ctx.deadline.remaining_ms() - min_request_ms)`; timing out → `DEADLINE` (not counted by the breaker).
- One `httpx.AsyncClient` per Judge with `Limits(max_connections=max_concurrency, max_keepalive_connections=max_concurrency)`, HTTP/1.1. HTTP/2 would need the `h2` extra (Q-07-3).
- Cancellation (speculative walk discard): the task is cancelled, httpx closes the connection, the semaphore slot is released in `finally`, and the ledger records `outcome="cancelled"` with the estimated tokens of the body already sent.

### 4.5 Timeouts

```
request_timeout(ctx) = min(judge.timeout_ms, ctx.deadline.remaining_ms())
if request_timeout < judge.min_request_ms (150): raise JudgeError(DEADLINE)   # don't start a request that can't finish
httpx.Timeout(connect=min(500, t), read=t, write=t, pool=t) and asyncio.timeout(t) around the attempt
```

- Default `judge.timeout_ms` = 1,200 (spec §13.3). `llm` default 20,000; `systemone-local` default 3,000.
- The route (3,000 ms) and walk (2,000 ms) budgets are enforced by the router passing the tighter `Deadline` in `ctx` (09 §4.9).
- Cold start: the hook process opens fresh TLS connections on every prompt (~100–250 ms on first request). The router fires call 1 and speculative walk level 1 together so the handshakes overlap. `surf doctor --live` reports cold vs warm latency.

### 4.6 Retry-once policy

| Failure | Retry? | Condition | Backoff |
|---|---|---|---|
| Timeout, connect error, 5xx | once | remaining deadline ≥ `min_request_ms` after backoff | 50–150 ms uniform jitter |
| 429 | once | `Retry-After` (if present) + `min_request_ms` ≤ remaining | `max(Retry-After, jitter)` |
| 401 / 403 | no | → `AUTH` | – |
| other 4xx (400, 404, 413, 422) | no | → `HTTP_4XX` (our bug or wire mismatch) | – |
| `MALFORMED` | no | deterministic parser problem | – |
| Cancelled / `DEADLINE` | no | – | – |

The retry uses `min(judge.timeout_ms, remaining)` as its timeout. Jitter uses `random.Random` seeded from the route id so fixture runs stay reproducible. `attempts` records 1 or 2.

### 4.7 Circuit breaker (persisted, cross-process)

The Claude Code hook runs a fresh Python process per prompt, so an in-memory breaker would never see 3 consecutive failures. State lives in `.surf/cache/judge_state.json` (§3.4); `breaker.py`:

```python
class Breaker:
    def __init__(self, path: Path | None, key: str, clock: Clock, cfg: BreakerCfg): ...
    def state(self) -> Literal["closed", "open", "half_open"]
    def on_batch(self, outcomes: Sequence[Outcome]) -> None
```

**What counts.** Bookkeeping is per *batch* (one `ask` or one `ask_many`), not per request, so a single bad route with 6 parallel timeouts counts as one failure, not six:

| Batch result | Effect |
|---|---|
| ≥ 1 request succeeded (ok or partial) | `consecutive_failures = 0`; close if half-open. Write only if the stored value was non-zero (avoid a write per prompt). |
| All requests failed with a **counted** kind (`TIMEOUT`, `CONNECT`, `HTTP_5XX`, `RATE_LIMITED`, `MALFORMED`) | `consecutive_failures += 1`; if ≥ `judge.breaker.failures` (3) or state was half-open → open: `open_until = now + judge.breaker.cooldown_s` (60) |
| Any `AUTH` | open immediately for `judge.breaker.auth_cooldown_s` (600) |
| Only uncounted kinds (`CANCELLED`, `DEADLINE`, `HTTP_4XX`, `TOO_LARGE`, `FIXTURE_MISS`) | no change |

**States.** `open` while `now < open_until` → `availability()` returns `BREAKER_OPEN` and the router returns `judge-unavailable` without network. When `open_until` has passed, the state is `half_open`: the next batch goes out; success closes, failure reopens for a full cooldown. Several processes may probe concurrently in half-open; acceptable at v1 scale.

**Persistence.** Read once at `Judge` construction and again at `availability()` if the file's mtime changed (the long-lived MCP server shares state with hook processes this way). Writes are `write tmp + os.replace` without locking; a lost update under a race costs at most one count. Unreadable or corrupt file → treated as closed and overwritten on the next write; a write failure (read-only FS) → in-memory only, warn once. `state_dir=None` (no `.surf/`) → in-memory.

### 4.8 Token accounting

- Provider-reported `usage.input_tokens` is used when present; otherwise `estimated = ceil(len(body_utf8) / 4)` and `Usage.estimated = True`.
- `est_tokens(req)` for splitting (§4.2) uses the same character heuristic on the encoded body.
- Cancelled requests are recorded with estimated tokens (they may be billed).
- `cost_usd = input_tokens × judge.price_per_mtok_input / 1e6` (default 0.042, spec §11.9); output is free per the spec and priced at `judge.price_per_mtok_output` = 0.
- `surf stats` and eval reports read `JudgeUsageSummary` from decision records (15, 16).

### 4.9 Threshold-profile lookup

Probabilities aren't comparable across backends (spec §13.1), so every judge exposes `threshold_profile`:

| Backend | `threshold_profile` |
|---|---|
| `jev` | `"jev"` |
| `systemone-local` | `"systemone-local"` |
| `llm` | `"llm"` |
| `fixture` | the profile stored in the recording (`"jev"` for Jev recordings) |
| `null` | `None` (router never reaches thresholds) |

```python
def thresholds_for(cfg: RouterConfig, judge: Judge) -> tuple[Thresholds, bool]:   # (values, calibrated)
    for key in (f"{judge.threshold_profile}:{judge.model}", judge.threshold_profile):
        if key in cfg.thresholds: return merge(BUILTIN[judge.threshold_profile] or BUILTIN["jev"], cfg.thresholds[key]), True
    if judge.threshold_profile in BUILTIN_CALIBRATED: return BUILTIN[judge.threshold_profile], True
    return BUILTIN["jev"], False            # uncalibrated: doctor warns, decision record flags it
```

Model-qualified tables (`[router.thresholds."jev:jev-1.14.0"]`) let a model upgrade ship new thresholds without breaking the pinned eval. Only `jev` defaults ship as calibrated in v1; the `Thresholds` field list is owned by 09 §5.2 and schema-validated by 13.

### 4.10 Backends

**`jev` (`jev.py`).** `BaseJudge` + `ProviderSpec` from `jev_wire.PROVIDERS[judge.provider]`. Missing key env → `availability() = NO_KEY` (router: `judge-unavailable`, no network). The official `typesafe-sdk-python` is **not** imported at runtime (D-07-2).

**`systemone-local` (`systemone_local.py`).** Same codec with `ProviderSpec("local", base_url=judge.local.base_url, ...)`. No breaker auth rule for a missing key (the key is optional). Profile `systemone-local`, uncalibrated until A7 runs.

**`llm` (`llm.py`, eval baseline).** OpenAI-compatible `POST {judge.llm.base_url}/chat/completions` via httpx (works with OpenRouter and most gateways; no new dependency). One chat call per `JudgeRequest`:
- System prompt: fixed text in the module: "You are a classifier. For each numbered item, estimate the probability that the statement is true given the state. Answer only with JSON matching the schema." Items are `q000…` with their instructions; state is rendered as `key: value` lines inside a delimited block marked as data.
- `response_format = {"type": "json_schema", ...}` with schema `{answers: {q000: {p: number}, q001: {probs: {...}}}}`; if the endpoint rejects `json_schema`, fall back to `{"type": "json_object"}` + validation (§4.3).
- `temperature = 0`, `seed = 0` where supported.
- Choice: `probs` from the model; `confidence = max(probs)`.
- Probabilities are verbalized, therefore coarse and needing their own threshold profile (`llm`).
- `make_judge` refuses `llm` outside `surf eval` unless `judge.llm.allow_routing = true` (it is slow and sends cards to a general LLM, which changes the §19.1 privacy statement).

**`null` (`null.py`).** `availability() = Availability(ok=False, reason=NULL_BACKEND)`; `ask` raises `NULL_BACKEND`. No network, no breaker file writes. The router maps it to `judge-unavailable` with `reason="null-backend"` (the kill switch of spec §19.2).

**`fixture` (`fixture.py`).** Wraps an optional inner judge; mode from `judge.fixture.mode` or `SURF_FIXTURE_MODE`:

| Mode | Hit | Miss |
|---|---|---|
| `replay` (default in tests, CI) | answer from store, `latency_ms = 0`, tokens from record | the whole request raises `FIXTURE_MISS` (tests fail loudly; eval counts it) |
| `record` | ignored; always calls inner judge | calls inner judge and appends |
| `record-missing` | from store | calls inner judge for **only the missing questions**, appends |

**Keying.** One record per question, not per request (D-07-3):

```
key = "sha256:" + sha256(canonical_json({
        "model": model,                              # "jev-1.13.0"
        "state": state,                              # dict, keys sorted
        "q": {"type": ..., "instructions": ..., "options": {...sorted}}
      }))
canonical_json = json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False) on NFC-normalized strings
```

The caller's question key, `purpose`, provider and chunking are **excluded**, so re-chunking, changing `beam_max` or reordering questions still replays. A request replays only if every question hits. `answers.jsonl` is rewritten sorted by key on close (deterministic diffs). Store loading is lazy and indexed in a dict; a 50k-record store loads in < 200 ms.

---

## 5. Configuration

Owned by 13-config; consumed here.

| Key | Type | Default | Notes |
|---|---|---|---|
| `judge.backend` | enum | `"jev"` | `jev` \| `systemone-local` \| `llm` \| `null` \| `fixture` |
| `judge.provider` | enum | `"typesafe"` | `typesafe` \| `openrouter` \| `vercel` \| `cloudflare` |
| `judge.model` | str | `"jev-1.13.0"` | pinned |
| `judge.timeout_ms` | int | 1200 | per attempt; backend-specific defaults for `llm` (20000), `systemone-local` (3000) |
| `judge.min_request_ms` | int | 150 | don't start/retry below this remaining budget |
| `judge.max_concurrency` | int | 16 | per process |
| `judge.max_questions_per_request` | int | 40 | hard cap; `router.chunk_size` ≤ this |
| `judge.max_request_tokens` | int | 8000 | estimated |
| `judge.retry` | bool | true | retry-once policy |
| `judge.breaker.failures` | int | 3 | consecutive failed batches |
| `judge.breaker.cooldown_s` | int | 60 | |
| `judge.breaker.auth_cooldown_s` | int | 600 | |
| `judge.price_per_mtok_input` | float | 0.042 | USD |
| `judge.price_per_mtok_output` | float | 0.0 | |
| `judge.base_url`, `judge.path` | str? | None | override provider defaults |
| `judge.cloudflare.account_id`, `.gateway_id` | str? | None | required for `cloudflare` |
| `judge.local.base_url` | str | `"http://127.0.0.1:8080"` | `systemone-local` |
| `judge.llm.base_url`, `.model`, `.key_env` | str | none, required | `llm` |
| `judge.llm.allow_routing` | bool | false | |
| `judge.fixture.mode` | enum | `"replay"` | env `SURF_FIXTURE_MODE` overrides |
| `judge.fixture.dir` | path | `.surf/eval/recordings` | |
| `judge.fixture.inner` | enum | `"jev"` | backend used in record modes |

Keys come only from env (spec §16); names per §3.2. `make_judge` is the only function that reads env.

---

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| No API key | `availability()` → `NO_KEY`; router `judge-unavailable`; `surf doctor` names the missing env var |
| Breaker open | `BREAKER_OPEN`, zero network, router `judge-unavailable`; decision record carries `retry_at` |
| Breaker file corrupt / unreadable | treat as closed; overwrite on next write |
| `.surf/cache/` read-only | in-memory breaker; warn once on stderr |
| Deadline remaining < 150 ms before send | `DEADLINE` without sending; not a breaker failure |
| Timeout on attempt 1, success on retry | `ok`, `attempts=2`, breaker reset |
| 429 with `Retry-After: 30` | no retry (doesn't fit); `RATE_LIMITED`, counted |
| 401/403 | `AUTH`, breaker opens for 600 s; doctor says "check key" |
| 400/422 (wire mismatch) | `HTTP_4XX`, not counted by breaker (it's our bug); logged with status; the eval run fails fast |
| Response missing some keys | those keys in `missing`; router treats them as unjudged |
| All keys invalid / not JSON | `MALFORMED`, counted |
| 41+ questions passed to `ask` | auto-split balanced; Choice in chunk 0 |
| One chunk of a split fails | merged `partial`; failed keys in `missing` |
| Speculative batch cancelled | `CancelledError` propagates to the task; records `cancelled`; slots released |
| `SyncJudge` called inside a running loop | `RuntimeError` (programming error, caught by the pipeline guard in tests) |
| `llm` backend selected for routing without opt-in | `make_judge` raises config error; `surf route` exits with message, hook fails open |
| Fixture miss in `replay` | `FIXTURE_MISS`; eval reports the miss count and fails CI if > 0 |
| Provider returns usage in an unknown field | `estimated=True`; no failure |
| Duplicate question text with different caller keys | both asked (wire keys differ); fixture store holds one record, both keys answered from it |
| Request not redacted (`redacted=False`) to a network backend | `ValueError` before any I/O |

---

## 7. Performance budget

| Item | Budget |
|---|---|
| `make_judge` (import httpx + build client, no network) | ≤ 60 ms (httpx import dominates; lazy-import `h2`/SDK never) |
| `availability()` | ≤ 1 ms (stat + small JSON read) |
| Encode + decode per request (40 questions) | ≤ 2 ms |
| Breaker write | ≤ 2 ms, at most once per route, only on state change |
| Overhead per `ask` excluding network | ≤ 5 ms |
| Fixture replay per request | ≤ 0.5 ms after store load |
| Memory | < 10 MB for client + fixture store of 50k records |

---

## 8. Test plan

### 8.1 Unit (offline; `httpx.MockTransport`)

| Area | Cases |
|---|---|
| Split | 40 → 1 chunk; 41 → 21/20; 81 → 27/27/27; Choice always chunk 0; state repeated; order preserved; `TOO_LARGE` for one huge question |
| Validation | every row of §4.3 |
| Retry | table in §4.6: each status × remaining-deadline combination; jitter deterministic with seeded route id |
| Timeout | slow mock (sleep) vs `timeout_ms`; clipped by deadline; `min_request_ms` refusal |
| Semaphore | 40 concurrent asks with limit 16 → max 16 in flight (instrumented transport) |
| Cancellation | cancel `ask_many` mid-flight → slots free within 10 ms, ledger `cancelled` |
| Breaker | 3 failed batches → open; batch of 6 timeouts counts once; success resets; half-open success/failure; AUTH opens 600 s; two `Breaker` instances on one file (simulated processes) see each other's state; corrupt file |
| Key map | caller keys with `:`, `[`, spaces, unicode round-trip |
| Ledger | tokens from provider vs estimated; cost math; cancelled tokens counted |
| Threshold lookup | model-qualified > backend > builtin > uncalibrated fallback |
| Fixture | key canonicalization (dict order, NFC); per-question hit across re-chunked requests; `record-missing` only asks missing questions; sorted rewrite is byte-stable |
| null | no network (transport asserts zero calls) |
| llm | schema fallback path; coarse probabilities parsed; Choice confidence = max |

### 8.2 Golden

- `jev_wire.encode` output for a fixed request, per provider → checked-in JSON bodies (reviewing a wire change means reviewing a golden diff).

### 8.3 Fail-open (shared with 00 §6.1)

- Judge that raises, times out, returns garbage, returns 500 twice: `route()` returns the right status with no exception.

### 8.4 Conformance (Phase 0, live, `@pytest.mark.live`, needs keys)

- Send a 3-question request (2 Nouls with obvious answers, 1 Choice) through each configured provider; assert decode succeeds, obvious answers are on the right side of 0.5, usage is parsed or flagged estimated.
- If `typesafe-sdk-python` is installed (dev extra), encode the same request through the SDK with a capturing transport and diff against `jev_wire.encode`. This is how the UNVERIFIED wire assumptions get verified.
- Measure latency vs. question count (1, 10, 20, 40) for Q-09-3 and Q-07-3.

---

## 9. Acceptance criteria

| Phase | Criterion |
|---|---|
| Phase 0 exit | Conformance test passes for `typesafe` and at least one gateway; every UNVERIFIED item in §3.1–3.2 is either confirmed or corrected in `jev_wire.py` only; fixture record/replay drives the A0 baseline deterministically (two replays produce byte-identical reports) |
| Phase 2 exit | Router uses `ask_many` with speculative cancellation; ledger feeds decision records |
| Phase 5 exit | All §6 rows covered by tests; breaker verified across two OS processes; `surf doctor --live` reports provider, model, cold/warm latency, breaker state |
| Always | No network call from `null`, `fixture` (replay) or when unavailable; judge overhead ≤ 5 ms per request |

---

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-07-1 | Sync `Judge.ask/ask_many(..., timeout_ms)` returning mappings (§13.1) | Async protocol + `SyncJudge` facade; `ask_many` returns per-request `JudgeResponse \| JudgeError`; timeout from `CallCtx.deadline` | Speculative walk needs cancellation; partial chunk failure must not discard the successful chunks; one deadline object per route (00 §5) |
| D-07-2 | Judge client = `typesafe-sdk-python` (pinned) + thin httpx adapters (§22.1) | httpx for every provider at runtime; the SDK is a dev-only dependency used by the conformance test | One wire codec for all providers (isolation), async unknown for the SDK, import cost in a per-prompt hook process. Revisit if the SDK is async, light and exposes the gateways (Q-07-1) |
| D-07-3 | Fixture judge "replays recorded responses" (§13.2) | Per-question records keyed by `(model, state, question)` | Threshold/beam/chunk sweeps (A8) replay without new live calls; request-level keys would miss on any re-chunking. Assumes questions are independent given state (Q-07-2) |
| D-07-4 | Breaker "after 3 consecutive failures" (§13.3) | Counted per batch, persisted in `.surf/cache/judge_state.json`, AUTH opens for 600 s | The hook is a fresh process per prompt; per-request counting would open the breaker on one bad route |
| D-07-5 | Retry on 5xx or timeout | Also retry once on connect errors and on 429 when `Retry-After` fits the deadline; never on other 4xx or malformed | Transient classes only; deterministic failures don't improve on retry |
| D-07-6 | Thresholds per backend (§13.1) | Per `threshold_profile`, optionally model-qualified (`jev:jev-1.13.0`) | Model upgrades shift calibration |
| D-07-7 | `null` "returns no decision" | `null` is *unavailable*; router status `judge-unavailable` with reason `null-backend` | One code path for "no judge"; nothing to interpret |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-07-1 | Use the official SDK at runtime? | No (D-07-2) | SDK inspection in Phase 0: async support, import time, gateway support |
| Q-07-2 | Are Jev answers independent of co-batched questions? If not, per-question fixtures drift from live | Assume independent | Phase 0: ask the same question in two different batches ×20; if |Δp| > 0.02 median, switch fixtures to per-request keys |
| Q-07-3 | HTTP/2 multiplexing (adds `h2`) | Off | Conformance latency test: cold-start cost of N parallel TLS handshakes vs one h2 connection |
| Q-07-4 | Escalating breaker cooldown (60 → 120 → 300 s) on repeated opens | Fixed 60 s | Production decision logs: flapping rate |
| Q-07-5 | Production answer cache (e.g. repeated walk level 1 for the same request in a session) | None in v1 | Latency data on `extends` routes |
| Q-07-6 | Exact wire format, gateway model ids, Cloudflare path, and whether OpenRouter/Vercel need a chat envelope | Assumptions in §3.1–3.2 (UNVERIFIED) | Phase 0 conformance test against the live docs/APIs |
| Q-07-7 | Cross-process concurrency cap (several sessions × 16) | None | Rate-limit errors (429 share) in decision logs |
