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
- Backends: `jev` (providers `typesafe`, `openrouter`; D-07-9, D-07-10), `systemone-local`, `llm` (eval baseline), `null`, `fixture` (record/replay).
- Resilience: per-request timeout clipped to the route deadline, retry-once, concurrency semaphore, circuit breaker persisted across processes.
- Per-backend request limits (`RequestLimits`: questions and estimated tokens per request) and token-balanced auto-split.
- Token and cost accounting per route.
- Threshold-profile lookup (which `router.thresholds.<profile>` table applies).

**Out of scope**
- Redaction (14; the judge only asserts it was done), question wording (09, 16), threshold values (13, tuned by 16), answer caching in production (Q-07-5).

---

## 2. Interfaces

### 2.1 Why async (decision)

The router needs real parallelism in three places: the flat pass and walk levels fan out to N requests (`ask_many`); the flat pass or walk level 1 runs **speculatively alongside call 1** and must be **cancellable** when call 1 says `same`; and call 1 itself splits into parallel chunks when it exceeds the backend's request limits. Options considered:

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
    options: dict[str, str]                 # option key -> description; ≤ 255 (API limit); v1 uses 3
                                            # wire name is `criteria` (§3.1); option keys ARE shown to the model

Question = Annotated[NoulQ | ChoiceQ, Field(discriminator="type")]

class NoulA(BaseModel, frozen=True):
    p: float                                # P(yes), clamped to [0, 1]; wire field `noul`

class ChoiceA(BaseModel, frozen=True):
    choice: str                             # always a key of options
    probs: dict[str, float]                 # wire `probabilities`; renormalized to sum 1 over options
    confidence: float                       # backend-reported; required from `jev` (§4.3). Not max(probs)

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
    HTTP_4XX = "http_4xx"; AUTH = "auth"; MALFORMED = "invalid_response"; TOO_LARGE = "too_large"
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
    limits: RequestLimits                   # §4.2; per backend, config-overridable

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
    name: str                    # typesafe | openrouter | local
    base_url: str
    path: str
    auth_headers: Callable[[Mapping[str, str]], dict[str, str]]   # env -> headers
    key_env: tuple[str, ...]     # env vars required
    model_id: Callable[[str], str]                                # "jev-1.13.0" -> provider's model string
    normalize_model: Callable[[str], str]                         # echoed model -> comparable id (§3.2)
    parse_error: Callable[[int, bytes], str]                      # provider error body -> detail (no prompt text)

PROVIDERS: dict[str, ProviderSpec]
def encode(req: JudgeRequest, *, model: str, spec: ProviderSpec) -> tuple[bytes, KeyMap]: ...
def decode(body: bytes, *, keymap: KeyMap, req: JudgeRequest, spec: ProviderSpec) -> RawResult: ...
```

**Key mapping.** Caller keys contain `:` and arbitrary path characters (`w:code:src/api/orders/[id].ts`). The wire uses opaque keys `q000`, `q001`, … (zero-padded to 3 digits, 4 above 999) in request order and `KeyMap` maps back. The API documents question ids as free-form keys that are "not sent to the underlying model and not used in inference", but not their grammar or length, so opaque keys stay. Choice **option** keys are different: the model sees them with their descriptions, so they stay readable (`same`/`extends`/`new`).

**Native envelope (verified 2026-09-23 against the TypeSafe API reference; see [`../jev-reference.md`](../jev-reference.md) §3–4).** Request:

```json
{"model": "jev-1.13.0",
 "state": {"request": "...", "project": "..."},
 "questions": {"q000": {"type": "noul", "instructions": "..."},
               "q001": {"type": "choice", "instructions": "...", "criteria": {"same": "...", "extends": "...", "new": "..."}}}}
```

- `state` may be a string, object or array; surf always sends an object of strings (09 §3.2).
- `ChoiceQ.options` is encoded as `criteria` (option key → description; `null` allowed). Noul also accepts an optional `criteria: {"true": …, "false": …}`; v1 doesn't send it (wording change, 16).
- `instructions` and criteria values may also be JSON objects/arrays ("structured instructions"). v1 sends strings (Q-07-8).

Response:

```json
{"model": "jev-1.13.0",
 "answers": {"q000": {"type": "noul", "noul": 0.83},
             "q001": {"type": "choice", "choice": "same",
                      "probabilities": {"same": 0.91, "extends": 0.07, "new": 0.02}, "confidence": 0.86}},
 "usage": {"input_tokens": 1234, "output_tokens": 40}}
```

- `model` is the versioned id that answered. `decode` compares it with the requested model (§4.3).
- `usage.input_tokens` and `usage.output_tokens` are documented as required.
- A Score answer (`score`, `legend`, `probabilities`, `confidence`) exists but v1 never asks Score questions.

`decode` reads only the documented field names; the earlier alias list (`p`, `probs`, …) is dropped. The Phase 0 conformance test (§8.4) still runs before any eval, as a live check of the documented format.

### 3.2 Provider table (verified 2026-09-23; sources in [`../jev-reference.md`](../jev-reference.md) §14)

Every supported provider speaks the native System One envelope (§3.1), so there is no `chat` codec and `ProviderSpec.envelope` is always `"native"`.

| Provider | Base URL + path | Auth | Key env | `model` sent for `judge.model = "jev-1.13.0"` | Pin granularity | `model` echoed | Error body |
|---|---|---|---|---|---|---|---|
| `typesafe` | `https://api.typesafe.ai` + `/v1/systemone` | `Authorization: Bearer $KEY` | `TYPESAFE_API_KEY` | `jev-1.13.0` | exact version | `jev-1.13.0` | JSON, status per 07 §4.6 |
| `openrouter` | `https://openrouter.ai/api` + `/v1/systemone` | `Authorization: Bearer $KEY` | `OPENROUTER_API_KEY` | `typesafe/jev-1.13` | minor version (dated snapshot) | `typesafe/jev-1.13-20260917` | `{"error": {"code", "message"}}`; adds 402 (insufficient credits) |
| `local` (`systemone-local`) | `judge.local.base_url` + `/v1/systemone` | optional Bearer | `SYSTEMONE_LOCAL_API_KEY` (optional) | `judge.model` | – | implementation-defined | implementation-defined |

- `ProviderSpec.model_id` maps the pinned id to the provider string above; `ProviderSpec.normalize_model` maps the echoed string back to a comparable id (`typesafe/jev-1.13-20260917` → `jev-1.13`), used for the §4.3 model check at the provider's pin granularity. Decision records keep both the provider and the raw echoed value (15).
- Extra response fields (`id`, `provider`, `usage.cost` on OpenRouter) are ignored; cost is always computed from `usage.input_tokens` (§4.8).
- OpenRouter 402 → `AUTH` kind (not retried; breaker opens for `auth_cooldown_s`; `surf doctor` says "check credits").
- **Eval runs require `provider = "typesafe"`** (16): only TypeSafe direct pins `jev-1.13.0` exactly. `surf doctor` warns when the configured provider can't honor the pin.
- `local`: Laya (`pip install "laya[serve]"`, `laya-serve`, default `http://127.0.0.1:8321/v1/systemone`, Apache-2.0 code and weights, CPU) is the reference open reproduction for A7. Reproductions are community projects, not TypeSafe releases; calibration differs and several degrade on large Choices (Laya "past about 20 options"), which v1 doesn't ask.
- **Vercel AI Gateway is deferred** (D-07-10): it serves Jev natively at `https://ai-gateway.vercel.sh/typesafe/v1/systemone` (key `AI_GATEWAY_API_KEY`), but only as the unversioned `typesafe-ai/jev`, so thresholds can't be tied to a version and a silent upgrade can't be detected. Add it as a `ProviderSpec` when it exposes versioned ids.
- **Cloudflare is not a v1 provider** (D-07-9). Cloudflare AI Gateway has no TypeSafe provider; Jev is reachable only as the Workers AI third-party model `typesafe/jev` via `POST https://api.cloudflare.com/client/v4/accounts/{account_id}/ai/run` with a different envelope (`{"model", "input": {state, questions}}`) and no version pin, or through a user-registered custom provider. Either can be added later as a new codec in `jev_wire.py` with its own config keys.

Escape hatches (13-config): `judge.base_url`, `judge.path`.

### 3.3 Ledger

```python
class JudgeCallRecord(BaseModel):          # one per logical request (retries folded in)
    purpose: Purpose; n_questions: int; attempts: int
    outcome: Literal["ok", "partial", "error", "cancelled"]
    error_kind: ErrorKind | None; status_code: int | None
    usage: Usage; latency_ms: int
    request_id: str | None                 # `x-typesafe-request-id` header, if any (§4.8)

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

Format agreed with 16-evaluation §3.7 (07 owns it). One file per (repo, split, backend, model): `bench/fixtures/<repo>/<split>/<backend>-<model>.jsonl` for surf's own benchmark, `.surf/cache/eval-fixtures/…` in user repos (gitignored: records are derived from prompts), `tests/fixtures/judge/…` for unit tests. One line per recorded **request**, file sorted by `key`:

```jsonc
{"key": "sha256:…",                           // request-level key (§4.10)
 "backend": "jev", "model": "jev-1.13.0", "profile": "jev",
 "answers": {"file:code:src/a.ts": {"p": 0.81},
             "continuity": {"choice": "new", "probs": {"same": 0.04, "extends": 0.06, "new": 0.9}, "confidence": 0.9}},
 "qkeys": {"file:code:src/a.ts": "sha256:…", "continuity": "sha256:…"},   // question-level keys
 "latency_ms": 412, "input_tokens": 1830,
 "request": null}                              // full redacted request only with --record-requests
```

`request` is the only place prompt text could land on disk, which is why it's opt-in and never written under `bench/` (14, 16 §4.10).

---

## 4. Behavior / algorithm

### 4.1 `ask` pipeline (BaseJudge)

```
ask(req, ctx):
  1. assert req.redacted (network backends)                       -> ValueError (programming error)
  2. a = availability(); if not a.ok -> raise JudgeError(a.reason) (no network, no ledger failure count)
  3. if len(req.questions) > limits.max_questions
        or est_tokens(req) > limits.max_tokens:
        return merge(await ask_many(split(req), ctx))              # §4.2; partial -> missing keys
  4. async with semaphore (acquire bounded by ctx.deadline)        # §4.4
  5. t = request_timeout(ctx)                                      # §4.5; t < min_request_ms -> DEADLINE
  6. raw = await _send(req, timeout_ms=t)   with retry-once policy  # §4.6
  7. validate + map keys back                                       # §4.3
  8. ledger.add(record); breaker.on_result(...)                     # §4.7, §4.8
```

`ask_many(reqs, ctx)`: `asyncio.gather(*(self._ask_one(r, ctx) for r in reqs), return_exceptions=True)` where `_ask_one` converts `JudgeError` into a returned value; `CancelledError` propagates (the caller cancelled the whole batch). Results are in input order. Breaker bookkeeping is done **once per `ask_many` batch** (§4.7).

### 4.2 Request limits and splitting

Accuracy risk lives in the **state** (jaggedness #5), and questions are judged independently against it (`docs/jev-reference.md` §9), so limits bound request size and backend capacity, not accuracy (spec principle 5, D15). The router keeps state ≤ ~2k tokens (09 §3.2). Each backend exposes `limits: RequestLimits(max_questions, max_tokens)` (D-07-11):

| Backend | `max_questions` | `max_tokens` (estimated) | Why |
|---|---|---|---|
| `jev` | 500 | 30,000 | Fits OpenRouter's 32k context and TypeSafe's 64k (32k for state + longest question); 500 is a sanity bound, not an accuracy limit |
| `systemone-local` | 64 | 8,000 | Reproductions cap requests (openjev-sglang: 64 questions) and run on small hardware |
| `llm` | 40 | 8,000 | One chat completion answers every item; long item lists degrade generative models |
| `fixture`, `null` | inherit the recorded / configured backend | | |

`judge.max_questions_per_request` and `judge.max_request_tokens` override the backend defaults (13).

- `split(req)`: `k = max(ceil(n / max_questions), ceil(est_tokens / max_tokens))` chunks, token-balanced (greedy in question order, then rebalanced so chunk sizes differ by at most one question's tokens), preserving question order. **Choice questions are always in chunk 0.** Every chunk repeats the full `state`. Each question is asked exactly once.
- If a single question plus state exceeds `max_tokens` → `TOO_LARGE`, not sent (the router truncates state first, so this indicates a bug or a giant card).
- `merge`: answers union; keys from failed chunks go to `missing`; the merged response is `outcome="partial"` if any chunk failed, and `JudgeError` only if all failed.

The router pre-chunks explicitly (it needs chunk-level trace entries), so auto-split is a safety net, not the primary mechanism.

### 4.3 Answer validation

| Check | Action |
|---|---|
| Body not JSON / missing `answers` | `MALFORMED` for the whole request |
| Unknown wire key in answers | ignored, logged at debug |
| Asked key absent | added to `missing` |
| Answer `type` present and ≠ the asked type | invalid → `missing` |
| Noul `noul` NaN, non-numeric | that key → `missing`; if > 50 % of keys invalid → `MALFORMED` |
| Noul `noul` outside [0, 1] by ≤ 1e-6 | clamped; further outside → invalid |
| Choice `choice` not an option key | invalid → `missing` |
| Choice `probabilities` missing | `probs = {choice: confidence}` + others 0 |
| Choice `probabilities` not summing to 1 | renormalized (the API rounds to 2 decimals, so sums like 0.99 are normal) |
| Choice `confidence` missing | `jev`: invalid → `missing` (the field is required, and it is not `max(probs)`: the API derives it from the whole distribution, e.g. probabilities 0.88/0.12/0 → confidence 0.81). `systemone-local`, `llm`: `max(probs.values())`, which is only comparable within their own threshold profiles |
| Response `model` ≠ requested `judge.model`, when the request named a versioned id (not an alias such as `jev-latest`) | answers kept; `JudgeResponse.model` records the served id; warn once per process; the decision record carries both (15). Eval runs fail fast (16), because thresholds are model-qualified (§4.9) |

### 4.4 Concurrency

- One `asyncio.Semaphore(judge.max_concurrency)` (default 8, D-07-8) per `Judge` instance, i.e. per process. It is not coordinated across processes (two Claude Code sessions can each run 8); documented, not solved in v1.
- Why 8: the published `jev-1.13` limits are 1,200 requests per minute and 250,000 tokens per second per account, "adjusting dynamically", with no documented concurrency limit. TypeSafe's own cookbooks run 6–8 workers, one noting "the public endpoint rate-limits above roughly eight". A route sends a few to ~20 requests (the widest walk level has ≤ `max_frontier` 12 nodes, 09 §4.9), so RPM is not the constraint for one user; burst concurrency is. At the documented ~100 ms per request, queueing beyond 8 adds about one round trip, and only on the widest levels (09 §4.15). Phase 0 measures the 429 share at 8 and 16 (§8.4).
- Acquisition is bounded: `asyncio.timeout(ctx.deadline.remaining_ms() - min_request_ms)`; timing out → `DEADLINE` (not counted by the breaker).
- One `httpx.AsyncClient(http2=judge.http2)` per Judge with `Limits(max_connections=max_concurrency, max_keepalive_connections=max_concurrency)`. `api.typesafe.ai` negotiates HTTP/2, so all requests of a route share one connection and one TLS handshake (0.2–0.3 s cold) instead of up to 8. Dependency `httpx[http2]`; importing `h2` adds ~14 ms (measured 2026-09-24, Python 3.11), inside 12 §7's budget. Servers without HTTP/2 fall back to HTTP/1.1 through ALPN (D-07-12).
- Cancellation (speculative walk discard): the task is cancelled, httpx closes the connection, the semaphore slot is released in `finally`, and the ledger records `outcome="cancelled"` with the estimated tokens of the body already sent.

### 4.5 Timeouts

```
request_timeout(ctx) = min(judge.timeout_ms, ctx.deadline.remaining_ms())
if request_timeout < judge.min_request_ms (150): raise JudgeError(DEADLINE)   # don't start a request that can't finish
httpx.Timeout(connect=min(500, t), read=t, write=t, pool=t) and asyncio.timeout(t) around the attempt
```

- Default `judge.timeout_ms` = 1,200 (spec §13.3). `llm` default 20,000; `systemone-local` default 3,000. TypeSafe documents "most queries complete in about 100 ms"; its cookbooks measure 111–114 ms mean round trips and 90–310 ms for a 182-option Choice plus Nouls, so 1,200 ms leaves ample headroom (the SDK's own default is 10 s, sized for batch jobs).
- The route (3,000 ms) and walk (2,000 ms) budgets are enforced by the router passing the tighter `Deadline` in `ctx` (09 §4.2).
- Cold start: the hook process opens fresh TLS connections on every prompt (~100–250 ms on first request). The router fires call 1 and speculative walk level 1 together so the handshakes overlap. `surf doctor --live` reports cold vs warm latency.

### 4.6 Retry-once policy

| Failure | Retry? | Condition | Backoff |
|---|---|---|---|
| Timeout, connect error, 408, 5xx (incl. TypeSafe's `529 Overloaded`) | once | remaining deadline ≥ `min_request_ms` after backoff | 50–150 ms uniform jitter |
| 429 | once | server wait (if present) + `min_request_ms` ≤ remaining | `max(server wait, jitter)`; server wait = `retry-after-ms`, else `Retry-After` (seconds), else none. The API says only "retry after a short delay"; the header is not guaranteed |
| 401 / 403 | no | → `AUTH` | – |
| other 4xx (400, 404, 413, 422) | no | → `HTTP_4XX` (our bug or wire mismatch; the API uses 422 for body validation errors and returns a JSON body naming the field) | – |
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

- Provider-reported `usage.input_tokens` / `usage.output_tokens` are used when present (the TypeSafe API documents both as required); otherwise `estimated = ceil(len(body_utf8) / 4)` and `Usage.estimated = True`. TypeSafe publishes no tokenizer or counting endpoint.
- `est_tokens(req)` for splitting (§4.2) uses the same character heuristic on the encoded body. The documented examples show a fixed overhead per request of roughly 280–300 input tokens (a one-question request on a 12-word state reports 296), so splitting a request also repeats that overhead, not only the state. Phase 0 fits the heuristic against reported usage (Q-02-1).
- Cancelled requests are recorded with estimated tokens (they may be billed).
- `cost_usd = input_tokens × judge.price_per_mtok_input / 1e6` (default 0.042: "$42 per Btok / $0.042 per Mtok", "Charged per input token. Output tokens are free"). Output tokens are reported (tens per request) and priced at `judge.price_per_mtok_output` = 0.
- `JudgeCallRecord.request_id` keeps the `x-typesafe-request-id` response header when present (local logs only; useful when reporting an API issue).
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

Model-qualified tables (`[router.thresholds."jev:jev-1.14.0"]`) let a model upgrade ship new thresholds without breaking the pinned eval. Only `jev` defaults ship as calibrated in v1; the `Thresholds` field list is owned by 09 §5 and schema-validated by 13.

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

**`fixture` (`fixture.py`).** `FixtureJudge(path, *, mode, miss, simulate_latency, inner)`; mode from `judge.fixture.mode`, overridable by `SURF_FIXTURE_MODE` and `surf eval --fixture-mode`:

| Mode | Hit | Miss |
|---|---|---|
| `replay` (tests, CI) | request-level hit → answers; else question-level fallback: if **every** question's qkey is in the store, assemble the response (counted `assembled`) | `miss="fail"` → `FIXTURE_MISS` (eval aborts listing the first 10 misses); `miss="null"` → behaves like the `null` backend for that request and the row is marked |
| `append` (regeneration default) | from store (request-level, then assembled) | inner live judge for the whole request; appended; existing entries never modified, so unchanged requests keep their old answers |
| `rewrite` | ignored | everything re-recorded from empty (model pin bump) |

**Keying** (both keys use `canonical_json = json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False)` over NFC-normalized strings, and hash exactly the redacted state and questions the adapter would send):

```
key  = "sha256:" + sha256(canonical_json({"backend", "model", "state", "questions"}))   # caller keys included
qkey = "sha256:" + sha256(canonical_json({"backend", "model", "state", "key", "question"}))
```

`purpose`, provider, retries and wire keys are excluded. The question-level fallback lets re-chunking and beam/frontier changes replay as long as each individual question (same state) was recorded before (D-07-3). It assumes answers don't depend on co-batched questions (Q-07-2).

**Latency replay.** With `simulate_latency=True` (eval default), replay advances the injected clock by the record's `latency_ms` (for an `ask_many` batch: the max over its requests; assembled responses use the max over their source records). Deadline behavior in replay therefore follows the recording (16 §4.4 clock). Unit tests use `simulate_latency=False` (latency 0).

**Profile.** The record's `profile` is the fixture judge's `threshold_profile` (§4.9). A store mixing profiles is rejected at load.

Store loading is lazy, indexed in two dicts (key, qkey); a 50k-record store loads in < 200 ms. Files are rewritten sorted by key on close (deterministic diffs).

---

## 5. Configuration

Owned by 13-config; consumed here.

| Key | Type | Default | Notes |
|---|---|---|---|
| `judge.backend` | enum | `"jev"` | `jev` \| `systemone-local` \| `llm` \| `null` \| `fixture` |
| `judge.provider` | enum | `"typesafe"` | `typesafe` \| `openrouter` (D-07-9: no `cloudflare`; D-07-10: `vercel` deferred) |
| `judge.model` | str | `"jev-1.13.0"` | pinned |
| `judge.timeout_ms` | int | 1200 | per attempt; backend-specific defaults for `llm` (20000), `systemone-local` (3000) |
| `judge.min_request_ms` | int | 150 | don't start/retry below this remaining budget |
| `judge.max_concurrency` | int | 8 | per process (D-07-8) |
| `judge.max_questions_per_request` | int? | None → backend default (§4.2) | `router.chunk_size` ≤ the effective value |
| `judge.max_request_tokens` | int? | None → backend default (§4.2) | estimated tokens |
| `judge.http2` | bool | true | HTTP/2 via `h2` (§4.4, D-07-12) |
| `judge.retry` | bool | true | retry-once policy |
| `judge.breaker.failures` | int | 3 | consecutive failed batches |
| `judge.breaker.cooldown_s` | int | 60 | |
| `judge.breaker.auth_cooldown_s` | int | 600 | |
| `judge.price_per_mtok_input` | float | 0.042 | USD |
| `judge.price_per_mtok_output` | float | 0.0 | |
| `judge.base_url`, `judge.path` | str? | None | override provider defaults |
| `judge.local.base_url` | str | `"http://127.0.0.1:8080"` | `systemone-local` |
| `judge.llm.base_url`, `.model`, `.key_env` | str | none, required | `llm` |
| `judge.llm.allow_routing` | bool | false | |
| `judge.fixture.mode` | enum | `"replay"` | `replay` \| `append` \| `rewrite`; env `SURF_FIXTURE_MODE` overrides |
| `judge.fixture.miss` | enum | `"fail"` | `fail` \| `null` (16 uses `eval.fixture_miss`) |
| `judge.fixture.simulate_latency` | bool | true | |
| `judge.fixture.path` | path | `.surf/cache/eval-fixtures/` | 16 passes explicit paths for `bench/` |
| `judge.fixture.inner` | enum | `"jev"` | live backend used by `append`/`rewrite` |

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
| 529 Overloaded | 5xx class: retried once, then `HTTP_5XX`, counted |
| Response `model` differs from a pinned request | answers used; warning; both ids in the decision record; eval fails fast (§4.3) |
| Choice answer without `confidence` from `jev` | that key → `missing` (§4.3) |
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
| Duplicate question text with different caller keys | both asked (wire keys differ); separate qkeys (caller key is part of the qkey) |
| Request not redacted (`redacted=False`) to a network backend | `ValueError` before any I/O |

---

## 7. Performance budget

| Item | Budget |
|---|---|
| `make_judge` (import httpx + h2 + build client, no network) | ≤ 100 ms (httpx ~80–95 ms and h2 ~14 ms measured; the SDK is never imported) |
| `availability()` | ≤ 1 ms (stat + small JSON read) |
| Encode + decode per request (500 questions) | ≤ 10 ms |
| Breaker write | ≤ 2 ms, at most once per route, only on state change |
| Overhead per `ask` excluding network | ≤ 5 ms |
| Fixture replay per request | ≤ 0.5 ms after store load |
| Memory | < 10 MB for client + fixture store of 50k records |

---

## 8. Test plan

### 8.1 Unit (offline; `httpx.MockTransport`)

| Area | Cases |
|---|---|
| Split | by question cap (`systemone-local` 65 → 33/32) and by tokens (`jev` 45k estimated → two ~22.5k chunks); Choice always chunk 0; state repeated; order preserved; `TOO_LARGE` for one huge question |
| Validation | every row of §4.3 |
| Retry | table in §4.6: each status × remaining-deadline combination; jitter deterministic with seeded route id |
| Timeout | slow mock (sleep) vs `timeout_ms`; clipped by deadline; `min_request_ms` refusal |
| Semaphore | 40 concurrent asks with limit 8 → max 8 in flight (instrumented transport) |
| HTTP/2 | parallel requests share one connection (instrumented transport counts handshakes); `judge.http2 = false` uses HTTP/1.1 |
| Cancellation | cancel `ask_many` mid-flight → slots free within 10 ms, ledger `cancelled` |
| Breaker | 3 failed batches → open; batch of 6 timeouts counts once; success resets; half-open success/failure; AUTH opens 600 s; two `Breaker` instances on one file (simulated processes) see each other's state; corrupt file |
| Key map | caller keys with `:`, `[`, spaces, unicode round-trip |
| Ledger | tokens from provider vs estimated; cost math; cancelled tokens counted |
| Threshold lookup | model-qualified > backend > builtin > uncalibrated fallback |
| Fixture | key/qkey canonicalization (dict order, NFC); request-level hit; assembled hit across re-chunked requests; partial qkey coverage → miss; `miss=null` behaviour; `append` never mutates existing lines; `simulate_latency` advances the clock by the batch max; sorted rewrite is byte-stable |
| null | no network (transport asserts zero calls) |
| llm | schema fallback path; coarse probabilities parsed; Choice confidence = max |

### 8.2 Golden

- `jev_wire.encode` output for a fixed request, per provider → checked-in JSON bodies (reviewing a wire change means reviewing a golden diff).

### 8.3 Fail-open (shared with 00 §6.1)

- Judge that raises, times out, returns garbage, returns 500 twice: `route()` returns the right status with no exception.

### 8.4 Conformance (Phase 0, live, `@pytest.mark.live`, needs keys)

The documented format is already transcribed in §3.1 (verified against the docs on 2026-09-23). The live test confirms that the service matches its docs and measures what the docs don't say (list in [`../jev-reference.md`](../jev-reference.md) §15):

- Send a 3-question request (2 Nouls with obvious answers, 1 Choice) through each configured provider; assert decode succeeds, obvious answers are on the right side of 0.5, `usage` is parsed, `model` echoes the pinned id.
- If `typesafe-sdk==0.7.1` is installed (dev extra), encode the same request through the SDK with a capturing transport and diff against `jev_wire.encode`.
- Latency vs. question count (1, 10, 40, 100, 300), cold and warm, HTTP/1.1 vs HTTP/2, for Q-09-3, Q-09-13 and Q-07-3 (the docs claim adding questions "barely changes the response time"; cookbook data: 16 questions 0.32 s, 62 questions 0.51 s).
- Same request ×20: run-to-run spread of `noul` on identical requests (Q-16-2; the docs report SD ≈ 0.01 but with a varying `uid` field in state).
- Same question in two different batches ×20 (Q-07-2; the docs say answers are independent).
- Burst of 16 concurrent requests: 429 share and whether 429s carry `Retry-After`/`retry-after-ms` (D-07-8, Q-07-7).
- Reported `input_tokens` vs `ceil(bytes/4)` over the recorded requests (Q-02-1).

---

## 9. Acceptance criteria

| Phase | Criterion |
|---|---|
| Phase 0 exit | Conformance test passes for `typesafe` and at least one gateway; the live service matches the documented format in §3.1–3.2 (any drift is fixed in `jev_wire.py` only); fixture record/replay drives the A0 baseline deterministically (two replays produce byte-identical reports) |
| Phase 2 exit | Router uses `ask_many` with speculative cancellation; ledger feeds decision records |
| Phase 5 exit | All §6 rows covered by tests; breaker verified across two OS processes; `surf doctor --live` reports provider, model, cold/warm latency, breaker state |
| Always | No network call from `null`, `fixture` (replay) or when unavailable; judge overhead ≤ 5 ms per request |

---

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-07-1 | Sync `Judge.ask/ask_many(..., timeout_ms)` returning mappings (§13.1) | Async protocol + `SyncJudge` facade; `ask_many` returns per-request `JudgeResponse \| JudgeError`; timeout from `CallCtx.deadline` | Speculative walk needs cancellation; partial chunk failure must not discard the successful chunks; one deadline object per route (00 §5) |
| D-07-2 | Judge client = `typesafe-sdk-python` (pinned) + thin httpx adapters (§22.1) | httpx for every provider at runtime; the SDK (`typesafe-sdk`, import `typesafe_sdk`, 0.7.1, MIT) is a dev-only dependency used by the conformance test | Evidence 2026-09-23: the SDK is async and gateway-aware (`base_url` works for OpenRouter and Vercel), but `import typesafe_sdk` costs ~243 ms vs ~129 ms for httpx + pydantic (median of 7 cold runs, Python 3.11) in a per-prompt hook process; it depends on `httpx2`, a second HTTP stack beside httpx; it is pre-1.0 with two breaking minor releases in a week; its own retries (2 retries, 30 s budget) would need disabling for our deadline logic; and debug logging writes request bodies unredacted (14). One small codec in `jev_wire.py` stays simpler |
| D-07-3 | Fixture judge "replays recorded responses" (§13.2) | Request-level records with question-level keys (`qkeys`) as a fallback, `append`/`rewrite` modes and recorded-latency replay (adopts 16's Q-16-3) | Beam/chunk/frontier changes (A8) replay without new live calls; latency replay keeps deadline behavior deterministic. Assumes questions are independent given state (Q-07-2) |
| D-07-4 | Breaker "after 3 consecutive failures" (§13.3) | Counted per batch, persisted in `.surf/cache/judge_state.json`, AUTH opens for 600 s | The hook is a fresh process per prompt; per-request counting would open the breaker on one bad route |
| D-07-5 | Retry on 5xx or timeout | Also retry once on connect errors and on 429 when `Retry-After` fits the deadline; never on other 4xx or malformed | Transient classes only; deterministic failures don't improve on retry |
| D-07-6 | Thresholds per backend (§13.1) | Per `threshold_profile`, optionally model-qualified (`jev:jev-1.13.0`) | Model upgrades shift calibration |
| D-07-7 | `null` "returns no decision" | `null` is *unavailable*; router status `judge-unavailable` with reason `null-backend` | One code path for "no judge"; nothing to interpret |
| D-07-8 | "Concurrency limit: 16 in-flight requests" (§13.3) | Default `judge.max_concurrency` = 8 | TypeSafe documents RPM/TPS limits that change "without notice" and no concurrency limit; its cookbooks cap at 6–8 workers because "the public endpoint rate-limits above roughly eight". Phase 0 measures 8 vs 16 (§8.4) |
| D-07-9 | Providers include Cloudflare AI Gateway (§13.2) | No `cloudflare` provider in v1; `judge.cloudflare.*` keys removed | Cloudflare AI Gateway has no TypeSafe provider. Jev is only on Workers AI (`/ai/run`, a different envelope, no version pin) or via a user-defined custom provider. Can return later as its own codec |
| D-07-10 | Providers include Vercel AI Gateway (§13.2) | Deferred from v1 (spec D17) | Only an unversioned model id: thresholds can't be pinned and upgrades can't be detected |
| D-07-11 | ≤ 40 questions per request (§2 principle 5) | Per-backend `RequestLimits`: `jev` 500 questions / 30k tokens, `systemone-local` 64 / 8k, `llm` 40 / 8k (spec D15) | Questions are judged independently; the state, kept small by 09, is what affects accuracy |
| D-07-12 | HTTP/1.1 (implicit) | HTTP/2 via `httpx[http2]`, one connection per route (spec D18) | Saves up to 7 TLS handshakes per cold route; ~14 ms import |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-07-1 | Use the official SDK at runtime? | No (D-07-2) | Resolved 2026-09-23 by SDK inspection (D-07-2 evidence); revisit if its import cost drops and it reaches 1.0 |
| Q-07-2 | Are Jev answers independent of co-batched questions? If not, per-question fixtures drift from live | Independent (documented: "Every answer is independent. One question's answer is not hidden context for another. You can add or remove questions without changing the others' results.") | Narrowed to a Phase 0 sanity check: same question in two batches ×20, compared against the same-batch run-to-run spread (SD ≈ 0.01 per the docs); disable the question-level fallback only if the cross-batch gap clearly exceeds that noise |
| Q-07-3 | HTTP/2 multiplexing (adds `h2`) | Resolved 2026-09-24: adopted (D-07-12). The origin negotiates HTTP/2 and `h2` adds ~14 ms import | Phase 0 still measures cold HTTP/1.1 vs HTTP/2 latency |
| Q-07-4 | Escalating breaker cooldown (60 → 120 → 300 s) on repeated opens | Fixed 60 s | Production decision logs: flapping rate |
| Q-07-5 | Production answer cache (e.g. repeated walk level 1 for the same request in a session) | None in v1 | Latency data on `extends` routes |
| Q-07-6 | Exact wire format, gateway model ids, Cloudflare path, and whether OpenRouter/Vercel need a chat envelope | Resolved 2026-09-23 from the TypeSafe, OpenRouter and Vercel docs: §3.1–3.2 (all native; Cloudflare dropped, D-07-9) | Phase 0 still checks: whether OpenRouter accepts `jev-1.13.0`, and whether probabilities match TypeSafe direct on a fixed request |
| Q-07-7 | Cross-process concurrency cap (several sessions × 8) | None | Rate-limit errors (429 share) in decision logs. Limits are per account (1,200 RPM, 250k TPS for `jev-1.13`), so several sessions share one budget |
| Q-07-8 | Send structured `instructions` (card as a named field, question text referring to it and to `request` by backticked path), as TypeSafe recommends for questions that carry data? | No in v1: `NoulQ.instructions` stays `str` | Wording experiment (16 Q-16-9). If adopted, `instructions: str \| dict[str, str]` in §2.2 and fixture keys change |
| Q-07-9 | Spec §4 lists only jev-router as unlicensed; blink and jev-knowledge-base have no LICENSE either (shallow clones, 2026-09-23). MIT: jev-code-context-router, JevRouter, jev-codex-router, langchain-skill-router, jev-skillful | Update spec §4; copy nothing from the unlicensed three | Owner (spec edit) |
| Q-07-10 | Vercel serves only unversioned `typesafe-ai/jev` | Resolved 2026-09-24: deferred (D-07-10, spec D17) | – |
| Q-07-11 | Question and token caps differ per backend | Resolved 2026-09-24: `RequestLimits` per backend (§4.2, D-07-11) | – |
