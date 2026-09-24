# 16 · Evaluation

**Status:** draft for review
**Spec sections:** §17 (all), §23 Phase 0 (and every phase exit that names an eval result), §2 principle 8, §3 D12, §19.4 (adversarial fixtures), §24 (v2 gates)
**Depends on:** 00-foundations (ids, F2/F3), 07-judge (`judge/fixture.py`, fixture keys), 09-router (`RouteTrace`, `mode`, selection), 10-lease, 13-config, 14-security-privacy (adversarial fixtures, redaction), 15-observability (`RouteTrace` field contract §3.3, decision records)
**Code:** `surf/eval/dataset.py`, `runner.py`, `metrics.py`, `attribution.py`, `report.py`; new: `surf/eval/bootstrap.py` (Phase 0 trivial cards), `surf/eval/flat.py` (Phase 0 A0 route), `surf/eval/label.py` (label/split/agree helpers), `surf/eval/ablations.yaml` (package resource); repo-level `bench/` (surf's own benchmark)

---

## 1. Purpose and scope

Evaluation is Phase 0: it exists before the indexer and sets every threshold, wording and v2 gate (spec §3 D12). This doc turns spec §17 into an implementable harness.

| In scope (v1) | Out of scope (v1) |
|---|---|
| Dataset schema, validation, label matching semantics | Online/implicit feedback (Stop hook, spec §24) |
| Runner for single queries and lease sequences | Agent-in-the-loop task success evaluation |
| Metrics, bootstrap and paired-bootstrap CIs | Per-user threshold learning |
| Failure attribution from `RouteTrace` | Automated labeling by an LLM |
| Ablations A0–A8 as config overlays; wording experiments | Hosted dashboards |
| Fixture record/replay and its regeneration workflow | |
| Report (markdown + JSON), baselines, CI gates, test-set hygiene | |
| Phase 0 bootstrap card builder and flat A0 route | |
| Labeling helpers, split, inter-labeler agreement | |
| surf's own benchmark repos (`bench/`) | |

---

## 2. Interfaces

### 2.1 CLI

```
surf eval [--set dev|test] [--final --reason TEXT [--force-rerun]]
          [--judge jev|llm|systemone-local|fixture|null] [--record] [--fixtures PATH]
          [--fixture-mode replay|append|rewrite] [--fixture-miss fail|null]
          [--ablate A0,A3,...|all] [--overlay FILE] [--wording KEY=ID ...] [--sweep FILE]
          [--cards catalog|bootstrap] [--dataset PATH] [--bench NAME|all]
          [--compare RUN_DIR|BASELINE.json] [--seed N] [--out DIR]
          [--category C] [--limit N] [--concurrency N] [--save-traces] [--adversarial]
surf eval validate [--strict]            # dataset checks (§3.2), incl. against the catalog
surf eval label [--out FILE] [--labeler NAME]
surf eval label --second --from dev.yaml --sample 0.2 --seed N --out labels.NAME.yaml
surf eval split [--ratio 0.6] [--seed N]
surf eval agree A.yaml B.yaml
surf eval compare RUN_A RUN_B [--metric recall_macro]
surf eval baseline --promote RUN_DIR --as NAME
surf eval fixtures check|prune [--bench NAME|all]
```

Exit codes (12 §4.12.2 table): 0 ok; 2 usage or hygiene refusal; 3 dataset missing or invalid; 4 gate failed (regression, fixture miss, invalid run); 8 judge unavailable (live runs).

### 2.2 Python

```python
# surf/eval/dataset.py
def load_dataset(path: Path) -> Dataset: ...                      # YAML -> pydantic; raises DatasetError with line numbers
def validate(ds: Dataset, *, catalog: CatalogReader | None, others: Sequence[Dataset] = (),
             redactor: Redactor | None = None, strict: bool = False) -> list[Issue]: ...
def dataset_sha256(path: Path) -> str: ...                        # over raw bytes

# surf/eval/runner.py
def run_eval(spec: RunSpec, *, clock: Clock | None = None) -> RunResult: ...
def run_unit(unit: Item, cfg: Config, env: EvalEnv) -> list[RowResult]: ...   # one query item or one whole sequence

# surf/eval/metrics.py
def score_row(row: RowResult, labels: RowLabels, cat: CatalogReader, cfg: EvalConfig) -> RowScore: ...
def aggregate(scores: Sequence[RowScore], *, weights: Mapping[str, float]) -> Metrics: ...
def bootstrap_ci(scores, metric: MetricName, *, n: int, seed: int, level: float) -> CI: ...
def paired_bootstrap(a, b, metric: MetricName, *, n: int, seed: int, level: float) -> PairedCI: ...

# surf/eval/attribution.py
def attribute_miss(label: SurfaceId, row: RowResult, cat: CatalogReader) -> Attribution: ...

# surf/eval/report.py
def write_report(res: RunResult, run_dir: Path) -> tuple[Path, Path]: ...    # (report.md, run.json)

# surf/eval/bootstrap.py  (Phase 0 only)
def build_trivial_catalog(repo_root: Path, *, excludes: Sequence[str]) -> TrivialCatalog: ...

# surf/eval/flat.py  (Phase 0 A0 before the router exists)
def route_flat(req: RouteRequest, cards: Sequence[Card], judge: Judge, th: Thresholds,
               *, chunk_size: int, max_pointers: int, wording: str) -> tuple[RouteResult, RouteTrace]: ...

# surf/eval/label.py
def split(items, *, ratio: float, seed: int) -> tuple[list[Item], list[Item]]: ...
def agreement(a: Dataset, b: Dataset, cat: CatalogReader | None) -> AgreementReport: ...
```

The runner calls the production pipeline with 09's signature: `route.pipeline.route(req, ctx, explain=True, enabled=True, on_route_done=...)`, where `ctx` is a `RouterContext` (09 §2) built by the runner from the eval config, catalog, judge stack, lease manager (or `None`) and clock; `on_route_done` receives the trace for decision records. It never re-implements routing, except `eval/flat.py` in Phase 0 (§4.1).

---

## 3. Data structures

### 3.1 Dataset models (`eval/dataset.py`)

The YAML top level is a list (spec §17.1). Unknown keys are errors (`extra="forbid"`).

```python
ItemId = Annotated[str, StringConstraints(pattern=r"^[a-z][a-z0-9_-]{0,31}$")]
QueryCategory = Literal["natural", "stack_trace", "cross_layer", "capability", "no_context", "doc_question"]

class Labels(BaseModel, extra="forbid"):
    must_include: list[SurfaceId] = []          # content ids only
    should_include: list[SurfaceId] = []        # content ids only
    must_exclude: list[SurfaceId] = []          # content or capability ids
    capabilities_use: list[SurfaceId] = []      # capability ids only
    capabilities_not_needed: list[SurfaceId] = []   # new (D-16-1)
    expect: Literal["capabilities_only_or_empty"] | None = None

class QueryItem(Labels):
    id: ItemId
    category: QueryCategory
    query: Annotated[str, StringConstraints(min_length=1, max_length=20_000)]
    previous_message: str | None = None         # optional context passed as last_message
    source: Literal["real", "synthetic"] = "real"
    labeler: str | None = None
    tags: list[str] = []                        # e.g. "adversarial", "layered", "phase0-skip"
    notes: str | None = None

class Turn(Labels):
    query: Annotated[str, StringConstraints(min_length=1, max_length=20_000)]
    expect_continuity: Literal["same", "extends", "new"]
    must_include_delta: list[SurfaceId] = []    # scored against the delta note only
    gap_minutes: float = 0.5                    # simulated time since previous turn (lease idle tests)

class SequenceItem(BaseModel, extra="forbid"):
    id: ItemId
    category: Literal["sequence"]
    turns: Annotated[list[Turn], Field(min_length=2, max_length=12)]
    source: Literal["real", "synthetic"] = "real"
    labeler: str | None = None
    tags: list[str] = []
    notes: str | None = None

Item = Annotated[QueryItem | SequenceItem, Field(discriminator="category")]
class Dataset(RootModel[list[Item]]): ...
```

Surface ids are validated with `ids.split_id` and normalized with `ids.norm_path` (00 §2.2) at load. Table ids are canonicalized per 00 §2.1 (default schema unqualified).

### 3.2 Validation rules

| Code | Rule | Severity |
|---|---|---|
| V1 | Item ids unique within a file and across dev + test | error |
| V2 | Normalized query (NFC, casefold, collapsed whitespace) unique across dev + test (leakage) | error |
| V3 | Prefix classes: `must/should_include` ∈ {code, doc, db, mig}; `capabilities_*` ∈ {mcp, skill, agent, cmd}; `must_exclude` any of these | error |
| V4 | `db:*` and `root:` are not valid labels | error |
| V5 | No path appears in both `must_include` and `should_include`; no `must_exclude` content label equals, contains, or is contained by a must/should label; `capabilities_use ∩ capabilities_not_needed = ∅`; `capabilities_use ∩ must_exclude = ∅` | error |
| V6 | `no_context` → `must_include` empty; `expect` defaults to `capabilities_only_or_empty` if omitted | error / auto-fill |
| V7 | `expect` set → `must_include` and `should_include` empty | error |
| V8 | `stack_trace`, `natural`, `cross_layer`, `doc_question` → ≥ 1 `must_include`; `capability` → ≥ 1 of `capabilities_use`/`capabilities_not_needed` | error |
| V9 | `cross_layer` → `must_include` spans ≥ 2 top-level directories; `doc_question` → ≥ 1 `doc:` label | warning |
| V10 | Sequence: turn 0 `expect_continuity == new`; `must_include_delta` only on `extends` turns | error |
| V11 | With a catalog: every label resolves (§3.3 resolution). Missing → warning (error with `--strict`); scored as a miss attributed `not_in_index` | warn / error |
| V12 | A content label whose path is secret-like or excluded (14 §3.2) | error |
| V13 | `Redactor.contains_secret(query)` or an email in the query → "scrub before committing" (14 T10) | warning |
| V14 | File label with the "wrong" prefix (`code:` on a `.md`) → matched by path (F2/F4); warn so it's fixed | warning |

### 3.3 Label semantics

Notation: `P(x) = ids.path_of(x)` (None for db and capability ids). `under(a, d)` is true when directory path `d` (ending `/`) is a prefix of path `a`. `files(d)` is the recursive file count of directory `d` from the catalog (`files_total`).

**Resolution of a label `l` against the catalog** (V11): a path label resolves if some catalog card has the same path, **ignoring prefix** (F2: directory prefix may flip; F3: `code:`↔`mig:` alias). Table and capability labels resolve by exact id.

**Satisfaction** `sat(l, p)` for a label `l` and one selected content pointer `p`:

| Label kind | Satisfied by pointer `p` when |
|---|---|
| file (`code:`/`doc:`/`mig:` without `/`) | `P(p) == P(l)` (any prefix), **or** `p` is a directory with `under(P(l), P(p))` and `files(p) ≤ eval.dir_credit_max_files` (collapse credit, D-16-2) |
| directory (ends `/`, any prefix) | `P(p) == P(l)` or `under(P(p), P(l))` (the dir or anything inside it, spec §17.1), **or** `p` is an ancestor directory with `files(p) ≤ eval.dir_credit_max_files` |
| table (`db:t`) | `p == l` |
| capability | never (capabilities are scored separately) |

**Row outcomes** (one row = one query item or one sequence turn):

| Quantity | Definition |
|---|---|
| `pointers` | `Selection.content` of the row (§4.3 for turns) |
| `satisfied(l)` | ∃ p ∈ pointers: `sat(l, p)` |
| `correct(p)` | ∃ l ∈ must ∪ should: `sat(l, p)` |
| exclusion violation for content `l` | ∃ p: `P(p) == P(l)` or (`l` is a dir and `under(P(p), P(l))`) or `p == l` (tables). An ancestor-dir pointer does **not** violate |
| exclusion violation for capability `l` | `l ∈ Selection.capabilities_use` |
| `expect: capabilities_only_or_empty` | satisfied iff `pointers == []` |
| `should_include` | only affects `correct(p)` (precision credit); never counts as a recall miss (spec §17.1) |

### 3.4 Row result and score

```python
class RowResult(BaseModel):
    row_key: str                   # "q017" or "s004#2"
    unit_id: str                   # bootstrap cluster: item id ("s004" for all its turns)
    category: str; config: str
    status: RouteStatus; route_mode: str | None
    selection: Selection
    delta: list[SurfaceId] = []    # lease additions this turn (extends)
    continuity_raw: str | None; continuity_conf: float | None
    continuity_effective: str | None   # what the pipeline acted on (§4.3)
    needs_context: float | None
    trace: RouteTrace | None       # dropped after scoring unless --save-traces
    latency_ms: LatencyMs; judge: JudgeStats | None
    fixture_misses: int = 0

class RowScore(BaseModel):
    row_key: str; unit_id: str; category: str
    recall: float | None           # None when must_include is empty
    precision: float | None        # None when pointers is empty
    n_must: int; n_must_hit: int; n_pointers: int; n_correct: int
    violations: int
    cap_acc: float | None; harmful_skips: int
    continuity_ok: bool | None     # None for single queries and turn 0
    delta_recall: float | None
    nocontext_ok: bool | None      # only rows with expect set
    misses: list[Attribution]
```

### 3.5 Ablation overlays (`surf/eval/ablations.yaml`, package resource)

Each ablation is a partial config (13-config keys) deep-merged over the effective project config.

```yaml
A0:  { router: { mode: flat } }                      # flat pass whatever the size: every leaf card, final wording
A1:  { router: { mode: walk, expand: { enabled_kinds: [] } } }                 # walk only
A2:  { router: { mode: walk, expand: { enabled_kinds: [contains] } } }
A3:  { router: { mode: walk, expand: { enabled_kinds: [contains, co_change] } } }
A4:  {}                                              # full v1 = defaults (flat within flat_max_tokens, walk above)
A4w: { router: { mode: walk } }                      # full walk pipeline on every repo; with A0 sets flat_max_tokens
A5:  { router: { mode: walk }, index: { cards: { coupled_dirs: false } } }   # changes cards -> per-config catalog rebuild
A6: { judge: { backend: llm } }                      # uses router.thresholds.llm (tuned first, §4.8)
A7: { judge: { backend: systemone-local } }          # uses router.thresholds.systemone-local
A8:
  sweep:                                             # coordinate-wise by default (§4.8)
    router.thresholds.{backend}.walk:  [0.20, 0.25, 0.30, 0.35, 0.40, 0.45, 0.50]
    router.thresholds.{backend}.final: [0.40, 0.45, 0.50, 0.55, 0.60, 0.65, 0.70, 0.75, 0.80]
    router.beam_max: [3, 6, 10]
```

These need three config keys that the spec lacks. They are defined in 13-config (`router.expand.enabled_kinds` is 04's key) and honored by 09/04/02 (Q-16-8):

| Key | Type | Default | Meaning |
|---|---|---|---|
| `router.mode` | `"auto" \| "flat" \| "walk"` | `"auto"` | `auto` = flat or walk by `router.flat_max_tokens` (09 §4.5); `flat` = A0; `walk` = A1–A3, A4w, A5 |
| `router.expand.enabled_kinds` | list[EdgeKind] | `[contains, co_change, schema_ref, defined_in, fk]` | Edge kinds used in expansion (§11.6 "tables from code" counts as `schema_ref`) |
| `index.cards.coupled_dirs` | bool | `true` | Render `coupled_dirs` on dir cards |

A4 includes `defined_in` and `fk` with schema refs (D-16-10).

### 3.6 Wordings file (`eval/wordings.yaml`)

Location: `.surf/eval/wordings.yaml` in a target repo (optional), `bench/wordings.yaml` for surf itself.

```yaml
version: 1
keys:
  walk:
    shipped: w1
    candidates:
      w1: "This item likely contains information needed for the request: {card}"
      w2: "The request can probably be answered using something inside this item: {card}"
  final:
    shipped: f1
    candidates:
      f1: "This item is needed to answer or complete the request: {card}"
      f2: "A developer doing this request would need to open this item: {card}"
  capability:
    shipped: c1
    candidates:
      c1: "Completing the request likely requires using this capability: {card}"
  needs_context:
    shipped: n1
    candidates:
      n1: "Answering the request requires information about this specific project's files, documentation, or database."
  continuity:
    shipped: k1
    candidates:
      k1:
        instructions: "How does the current request relate to the previous task?"
        options:
          same: "continues the same work with no new area of the project"
          extends: "same overall goal but involves a new area, file type, or capability"
          new: "a different task"
```

Validation: `walk`, `final`, `capability` candidates must contain exactly one `{card}`; `needs_context` and `continuity` must contain none; continuity options are exactly `same/extends/new`. The `shipped` candidate's text must equal the wording compiled into the router (09 owns the constants); a unit test enforces it. `--wording final=f2` overrides one key for a run. Any wording change changes fixture keys (§4.10).

### 3.7 Fixture file (eval view of `judge/fixture.py`)

07-judge owns the format; the eval side relies on:

```jsonc
// one line per recorded request, file sorted by "key"
{"key": "sha256:…",                       // canonical hash of {backend, model, state, questions}
 "backend": "jev", "model": "jev-1.13.0",
 "answers": {"file:code:src/a.ts": {"p": 0.81}, "continuity": {"choice": "new", "probs": {...}, "confidence": 0.9}},
 "qkeys": {"file:code:src/a.ts": "sha256:…"},   // question-level keys (Q-16-3)
 "latency_ms": 412,
 "request": null}                          // full redacted request only with --record-requests
```

Canonical hash: `sha256` over `json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False)` of NFC-normalized strings, where `obj = {backend, model, state, questions}` and `state`/`questions` are exactly what the judge adapter would send (after redaction). The question-level key hashes `{backend, model, state, key, question}` for one question.

### 3.8 Run directory

```
<out>/<run_id>/                   # default out: .surf/cache/eval-runs/ (user), bench/runs/ (surf; gitignored)
├── run.json                      # RunReport (§3.9)
├── report.md
├── items.jsonl                   # RowScore + selection per row (dev); items.test.jsonl for test
├── decisions.jsonl               # adapter=eval decision records (15 §4.6)
├── memo.jsonl                    # every judge answer seen in this run (same format as §3.7)
├── traces.jsonl.gz               # only with --save-traces
├── catalogs/<config>/            # only for overlays touching index.* (A5)
└── leases/<config>/              # sequence lease stores
```

`run_id` = `e_` + ULID.

### 3.9 `RunReport` (run.json)

```python
class CI(BaseModel):          point: float | None; lo: float | None; hi: float | None; n: int; n_eff: int
class PairedCI(BaseModel):    a: str; b: str; metric: str; delta: float; lo: float; hi: float
                              verdict: Literal["better", "worse", "no_difference"]
class ConfigResult(BaseModel):
    name: str; overlay: dict; config_hash: str; wordings: dict[str, str]
    metrics: dict[str, CI]                    # §4.5 names
    by_category: dict[str, dict[str, CI]]
    attribution: dict[str, int]               # top-level buckets
    attribution_detail: dict[str, int]        # "walk:pruned", "walk:deadline", ...
    latency: dict[str, dict[str, float]] | None   # path type -> {p50, p95, n}; None in fixture mode
    cost: dict[str, float]                    # mean calls, mean input tokens, est. USD per route
    walk_stats: dict[str, float]
    targets: dict[str, Literal["pass", "fail", "n/a"]]
class RunReport(BaseModel):
    v: Literal[1] = 1
    run_id: str; started_at: str; finished_at: str; complete: bool
    surf_version: str; git_sha: str; dirty: bool
    repo: dict                                # name, root, head
    dataset: dict                             # path, split, sha256, n_items, n_rows
    card_source: Literal["catalog", "bootstrap"]
    judge: dict                               # backend, model, mode: live|fixture|record, fixture_sha256
    seed: int; bootstrap_n: int; ci_level: float
    configs: list[ConfigResult]
    comparisons: list[PairedCI]
    rows: dict[str, list[RowScore]] | None    # per config; kept in baselines for paired comparison
    warnings: list[str]
```

### 3.10 surf's own benchmark (`bench/`)

Spec §9.1 puts datasets in the **target** repo's `.surf/eval/`. surf's own CI can't commit into external repos, so its benchmark lives in the surf repository (D-16-8):

```
bench/
├── manifest.yaml                 # repos to benchmark
├── repos/                        # synthetic repo specs, built by the tests/fixtures builder (00 §6.1)
│   ├── feature-shop.yaml         # feature-organized TS + Supabase app, ~400 files, scripted history
│   ├── layered-shop.yaml         # same domain, controllers/services/models layout
│   └── adversarial.yaml          # 14 §4.7 plants
├── configs/<repo>.toml           # per-repo .surf/config.toml used for the checkout
├── datasets/<repo>/dev.yaml, test.yaml
├── fixtures/<repo>/<split>/<backend>-<model>.jsonl
├── baselines/dev-fixture.json    # fixture-mode baseline for PR CI
├── baselines/v<version>.json     # release baselines (dev + test, with rows)
├── wordings.yaml
├── test-runs.jsonl               # test-set run log (§4.11)
└── checkouts/                    # gitignored; external repos at pinned SHAs
```

```yaml
# bench/manifest.yaml
version: 1
repos:
  - name: heimdall                # spec §23 Phase 0 candidate
    kind: external
    url: https://github.com/<owner>/heimdall
    commit: "<40-hex sha>"        # pinned; bumping it is a deliberate PR that regenerates fixtures
    organization: layered         # layered | feature
    license: "<spdx>"             # must allow cloning in CI; recorded for audit
    config: bench/configs/heimdall.toml
  - name: feature-shop
    kind: synthetic
    spec: bench/repos/feature-shop.yaml
    organization: feature
    config: bench/configs/feature-shop.toml
```

`surf eval --bench NAME` clones or updates `bench/checkouts/NAME` at `commit` (`git fetch --depth` sufficient for the co-change window, 07/04 decide depth), copies the config to `.surf/config.toml` in the checkout, runs `surf index`, then evaluates with `--dataset bench/datasets/NAME/<split>.yaml` and fixtures from `bench/fixtures/NAME/`. Synthetic repos are built fresh from their spec with fixed commit timestamps, so their catalogs are reproducible.

Both spec §17.2 coverage rules are met by the manifest: ≥ 2 repos, one per organization. Real (external) repos carry most of the queries; synthetic repos exist for CI speed, adversarial plants and Phase 0 smoke runs.

---

## 4. Behavior

### 4.1 Phase 0 bootstrap (A0 before the indexer)

Phase 0's exit is an A0 report on both repos' dev sets, and it comes before Phase 1. So A0 can't depend on `index/`. `eval/bootstrap.py` builds a **trivial catalog**:

1. Files: `git ls-files -co --exclude-standard -z`; drop spec §7.1 default excludes, `SECRET_LIKE_GLOBS` and `path_is_safe` failures (imported from `surf/redact.py`, which is therefore a Phase 0 deliverable), binaries (extension list + NUL byte in the first 8 KiB), files > `index.max_file_bytes`.
2. Ids per F4 (`doc:` by doc extension, else `code:`), via `surf/ids.py`.
3. Card text: code `"{path} [{lang}, {lines_bucket} lines]"`; doc `"{path} — \"{title}\""` with title = first H1 or file name, run through `sanitize_field`.
4. Directory entries (no card text, not judged) with F2 prefixes and `files_total`, so directory labels validate and `dir_credit` works.
5. Tables: a regex over `*.sql` in migration-like dirs for `CREATE TABLE [IF NOT EXISTS] [schema.]name` minus `DROP TABLE name` in filename order; card `"table {name}"`. No column parsing. Good enough for table labels to be satisfiable.
6. No capabilities, no edges. Capability metrics report `n/a`; sequences are skipped (tag `phase0-skip` is implied).

`eval/flat.py` `route_flat`: order content cards by id, split by the judge's `RequestLimits` (07 §4.2), `judge.ask_many` with the **final** wording, select `p ≥ router.thresholds.<backend>.final`, keep the top `router.max_pointers` by `(p desc, id asc)`. Trace: `mode="flat"`, `pool` = all, `final` = all scores, `budget_cut` = above-threshold beyond the cap. Attribution therefore only yields `final` or `budget`.

From Phase 2, A0 means `router.mode = flat` in the real pipeline with real cards (the same flat pass that `auto` runs within budget, 09 §4.8). `RunReport.card_source` distinguishes the two; reports never compare a bootstrap-card run with a catalog run without a warning.

Guard: A0 on a repo with more than `eval.a0_max_cards` content cards refuses without `--allow-large` (cost and rate limits: 5,000 cards is ~430k input tokens, ~15 requests and ~$0.018 per query, and more than one second of the account's 250k tokens/s).

### 4.2 Runner (single queries)

1. Load and validate the dataset (§3.2). Errors → exit 3.
2. Hygiene checks (§4.11).
3. Resolve the catalog: `--cards catalog` requires `surf index --check`-level freshness against the checkout HEAD, else it runs `surf index` (bench) or refuses with a hint (user repos). Overlays touching `index.*` build their own catalog under `catalogs/<config>/`.
4. Build the judge stack: `base = make_judge(cfg)` → `MemoJudge(base)` (run-scoped, keyed by the §3.7 hash) → `RecordingJudge` when `--record`. With `--judge fixture`: `FixtureJudge(path, miss=eval.fixture_miss, simulate_latency=True)`.
5. For each config, for each unit sorted by id, with `eval.concurrency_live` (default 1) or `eval.concurrency_fixture` (default 8) workers:
   `route(RouteRequest(request=item.query, session_id=None, previous_message=item.previous_message), ctx, explain=True)` with `ctx.leases=None` (and `lease.enabled=false`) for single queries (no continuity question).
6. Score each row (§4.4), attribute misses (§4.6), then aggregate (§4.5) and bootstrap (§4.7).
7. Compare: each config against `A4` if present, else against the first config, plus `--compare` target.
8. Write `run.json`, `report.md`, `items.jsonl`. Ctrl-C writes a partial report with `complete=false`.

`MemoJudge` means identical requests inside one invocation are answered once. Across ablations and sweep points this cuts cost and makes paired comparisons share judge noise on shared requests, which is what a paired test wants.

**Clock.** Eval passes a `HybridClock`: `wall()` (lease idle) is simulated; `monotonic()` (deadlines, latency) is real in live mode, and simulated in fixture mode, where `FixtureJudge` advances it by each request's recorded `latency_ms` (the max over a parallel `ask_many` batch). Deadline behavior in replay thus follows the recording (Q-16-3 for 07).

### 4.3 Sequence runner

```python
def run_sequence(seq, cfg, env):
    sid = f"eval-{env.run_id}-{cfg.name}-{seq.id}"        # fresh per sequence, per config, per run
    leases = LeaseManager(env.run_dir / "leases" / cfg.name, clock=env.clock)   # isolated store
    prev, rows = None, []
    for k, turn in enumerate(seq.turns):
        env.clock.advance_wall(minutes=turn.gap_minutes)
        ctx = env.router_context(cfg, leases=leases)          # RouterContext (09 §2): catalog, judge, clock, …
        res = route(RouteRequest(request=turn.query, session_id=sid, previous_message=prev),
                    ctx, explain=True, on_route_done=env.on_route_done)
        lease = leases.get(sid)
        sel = lease.selection if lease else (res.selection or Selection.empty())
        rows.append(RowResult(row_key=f"{seq.id}#{k}", unit_id=seq.id, selection=sel,
                              delta=res.trace.lease.delta if res.trace else [],
                              continuity_effective=effective_continuity(res, had_lease=k > 0), ...))
        prev = turn.query
    return rows
```

| Rule | Detail |
|---|---|
| Isolation | Lease store under the run dir; never touches `.surf/cache/leases/` |
| Selection scored | The **lease selection after the turn** (what the agent is working with), not only the note |
| Delta | `must_include_delta` scored against `delta` (additions on `extends`) → `delta_recall` |
| Effective continuity | `lease-reuse` or `skipped` with a lease → `same`; otherwise the continuity the pipeline acted on (low-confidence `same` → `extends`). Non-routed statuses (`error`, `judge-unavailable`, `deadline` before call 1) → `none` (wrong) |
| Raw continuity | Recorded too; the report shows raw accuracy as a secondary metric |
| Turn 0 | Not scored for continuity (no lease exists); its labels are scored like a query |
| Recall | Turns with `must_include` join the main recall rows (clustered by sequence, D-16-5) |

### 4.4 Row scoring

For each row with labels `L` and outcome `R`, per §3.3:

- `recall = n_must_hit / n_must` (None if `n_must == 0`)
- `precision = n_correct / n_pointers` (None if `n_pointers == 0`). `no_context` rows with pointers score precision 0 (no labels → nothing correct).
- `violations` = content + capability exclusion violations.
- `cap_acc`: over labeled capabilities `C = use ∪ not_needed ∪ (capability ids in must_exclude)`: a `use` label is correct iff in `R.use`; a `not_needed`/excluded label is correct iff **not** in `R.use` (unmentioned is fine, "skip" is advisory). `None` if `C` empty or card source is bootstrap.
- `harmful_skips` = |`capabilities_use` ∩ `R.skip`|. Target 0 (spec §11.4 asymmetry rationale).
- `nocontext_ok` = (`pointers == []`) for rows with `expect`.

### 4.5 Metrics

All row-level means are **macro over rows** (each query or scored turn weighs 1). Micro versions are reported alongside for recall and precision.

| Name | Formula | Rows | Spec target |
|---|---|---|---|
| `recall_macro` | mean(`recall`) | `recall ≠ None` | ≥ 0.85 (test) |
| `recall_micro` | Σ `n_must_hit` / Σ `n_must` | same | report |
| `recall_weighted` | Σ_c w_c · recall_macro_c / Σ_c w_c over categories present; `w` = `eval.category_weights` | same | used for wording/threshold choice |
| `precision_macro` | mean(`precision`) | `precision ≠ None` | ≥ 0.6 |
| `precision_micro` | Σ `n_correct` / Σ `n_pointers` | same | report |
| `violations_total` | Σ `violations` | all | 0 |
| `cap_accuracy` | mean(`cap_acc`) | `cap_acc ≠ None` | ≥ 0.9 |
| `harmful_skips_total` | Σ | all | 0 (new) |
| `continuity_accuracy` | mean(`continuity_ok`) | turns k ≥ 1 | ≥ 0.9 |
| `delta_recall` | mean(`delta_recall`) | extends turns with delta labels | report |
| `nocontext_correct` | mean(`nocontext_ok`) | rows with `expect` | ≥ 0.9 |
| `note_size_median`, `note_size_p90` | of `n_pointers` | rows excluding `expect` rows and lease-reuse turns | median ≤ 12 |
| `latency_p50/p95[path]` | nearest-rank over `latency_ms.total` by `route_mode` | live mode only | spec §11.9 |
| `judge_calls_mean`, `input_tokens_mean`, `usd_per_route` | mean over routed rows; USD = tokens × `eval.price_per_mtok[backend]` / 1e6 | routed | report |
| `walk_depth_mean`, `dead_end_rate`, `deadline_rate` | from trace | `route_mode = walk` | report |
| `judge_failure_rate` | rows with `judge-unavailable`/`error` | all | run invalid if > `eval.max_judge_failure_rate` (0.05) |
| `fixture_miss_rate` | rows with `fixture_misses > 0` | fixture mode | must be 0 in CI |

Category weights default to spec §17.2 shares: natural .35, cross_layer .15, stack_trace .10, capability .10, doc_question .10, no_context .05, sequence .15.

### 4.6 Failure attribution (`eval/attribution.py`)

For every unsatisfied `must_include` label `l` in a row, let `X(l)` be the catalog content ids that would satisfy it (file: the file id and its `mig:` alias; directory: the dir id and every id under it). Using the row's `RouteTrace` (fields in 15 §3.3), return the **first** matching bucket:

| # | Bucket (spec §17.4 in bold) | Condition | Detail recorded |
|---|---|---|---|
| 0a | `status` | row status ∈ {error, deadline (before final), judge-unavailable, index-missing} | status |
| 0b | `not_in_index` | `l` doesn't resolve (V11) | — |
| 0c | `lease` | turn resolved as `same` (reuse/skip) and the lease lacks `l` | lease generation |
| 0d | `no_context_gate` | status `no-context` (needs_context < threshold, no path hits) | needs_context |
| 1 | **budget** | `X ∩ trace.budget_cut ≠ ∅` | best p, rank vs `max_pointers` |
| 2 | **final** | `X ∩ keys(trace.final) ≠ ∅` | max p, threshold |
| 3 | **truncation** | `X ∩ trace.pool_cut ≠ ∅` | pool rank |
| 4 | **walk** | otherwise; sub-reason below | |

Walk sub-reasons, checked in order on the deepest ancestor directory `a` of `P(l)` present in `trace.walk.nodes`:

| Sub-reason | Condition |
|---|---|
| `walk:expansion_below` | some `x ∈ X` in `trace.expansion` with `admitted = false` (score recorded) |
| `walk:pruned` | `a.action == "pruned"` (record `a.id`, `a.p`, depth) |
| `walk:deadline` | `a` was expanded but its children weren't judged and `walk.deadline_hit` |
| `walk:depth_cap` | `a` expanded at `max_depth − 1` and `walk.depth_cap_hit` |
| `walk:beam` | `a`'s child on the path scored ≥ `tau_walk` but was cut by `beam_max` |
| `walk:small_mode` | `mode == small` and call-1 content Noul for `X` below threshold |
| `walk:unknown` | none of the above (trace incomplete; counts are reported so gaps surface) |

A directory label takes the **furthest** stage reached by any member of `X`. Each `must_exclude` violation also records the source of the offending pointer (`path_hit`, `walk`, `flatten`, `expansion`), and each failed `nocontext` row records `needs_context` and pointer sources.

The report shows bucket counts overall and per category, the top 20 detail keys, and (dev only) a per-miss table. The Phase 3 exit ("reduced walk losses on cross_layer") reads directly from `by_category.cross_layer` bucket counts across A1→A4.

### 4.7 Bootstrap confidence intervals

Resampling unit = **unit id** (a query item, or a whole sequence), so turns of one sequence stay together (cluster bootstrap).

Implementation (stdlib only; spec §22.1 has no numpy):

```python
def bootstrap_ci(scores, metric, *, n=1000, seed, level=0.95):
    units = sorted({s.unit_id for s in scores})
    agg = {u: metric.partial([s for s in scores if s.unit_id == u]) for u in units}  # e.g. (Σ value, count)
    rng = random.Random(_subseed(seed, metric.name))
    stats = []
    for _ in range(n):
        tot = metric.zero()
        for _ in range(len(units)):
            tot = metric.add(tot, agg[units[rng.randrange(len(units))]])
        v = metric.finish(tot)                 # None if count == 0 (e.g. no precision rows drawn)
        if v is not None: stats.append(v)
    lo, hi = _quantile(stats, (1 - level) / 2), _quantile(stats, 1 - (1 - level) / 2)
    return CI(point=metric.finish(sum of all agg), lo=lo, hi=hi, n=len(units), n_eff=len(stats))

def _subseed(seed, name): return int.from_bytes(hashlib.sha256(f"{seed}:{name}".encode()).digest()[:8], "big")
def _quantile(xs, q):                          # linear interpolation between order statistics (Hyndman–Fan type 7)
```

- Macro metrics decompose into per-unit `(Σ value, count)`; micro into `(Σ numerator, Σ denominator)`; so a resample is O(units) additions, not a re-score.
- `random.Random(int)` and `randrange` are stable across CPython 3.11+; a golden test pins the CI for a fixed input.
- `n_eff < 0.9 n` (many resamples with no defined rows) → warning in the report.
- Medians/percentiles (note size) use the percentile bootstrap over the pooled resampled rows.

**Paired bootstrap** for configs A and B:
1. Keep units present in both runs (warn and list the difference).
2. For each resample, draw one unit index sequence from `random.Random(_subseed(seed, "paired:" + metric))` and apply it to **both** A and B.
3. `delta_b = finish_B − finish_A`; CI from the deltas.
4. Verdict: `better` if `lo > 0`, `worse` if `hi < 0`, else `no_difference`. Never claim an improvement when the interval includes zero (spec §17.3).

Report the point delta, CI, and the verdict. For "lower is better" metrics (violations, latency), the sign convention is stated in the table header.

### 4.8 Ablations and sweeps

- `--ablate A1,A3` runs the listed overlays in one invocation, sharing `MemoJudge`.
- A5 rebuilds a catalog with `index.cards.coupled_dirs=false` (only directory card text changes).
- A6/A7: thresholds are per backend (spec §13.1). Before comparing, run an A8-style `final` and `walk` sweep for that backend on dev and use its best point; comparing Jev-tuned thresholds on another backend would be unfair. The report lists which thresholds each backend used.
- **A8 sweeps** are coordinate-wise by default (walk grid with other defaults, then final, then beam: 7 + 9 + 3 = 19 points); `--sweep-mode grid` runs the 189-point product.
- **Offline re-selection.** Parameters applied after the final pass (`final`, `path_hit_floor`, `max_pointers`, `cap_use`, `cap_skip`, `needs_context`) don't change judge requests. The runner re-runs `route/select.py` on stored traces instead of re-routing. This requires `select` to be a pure function of `(trace, params)` (contract for 09, Q-16-8).
- Objective for picking a sweep point: maximize `recall_weighted` subject to `precision_macro ≥ 0.6`, `violations_total == 0`, and `note_size_median ≤ 12`; ties go to the higher threshold. The incumbent is kept unless the paired CI on `recall_weighted` (or on precision, at equal recall) says `better`.

### 4.9 Report format

`report.md`:

```
# surf eval — heimdall / dev — e_01J…      (complete | INCOMPLETE)
surf 0.1.0 @ 3f2a1c (clean) · judge jev/jev-1.13.0 (live) · cards: catalog · seed 1729 · 1000 resamples · 95% CI
dataset bench/datasets/heimdall/dev.yaml sha256:ab12… · 58 items · 71 rows

## Headline
| config | recall (CI)        | precision (CI)     | violations | cap acc | continuity | no-ctx | note p50 | p50 / p95 ms (walk) | $/route |
|--------|--------------------|--------------------|-----------:|--------:|-----------:|-------:|---------:|---------------------|--------:|
| A4     | 0.87 (0.81–0.92)   | 0.64 (0.58–0.70)   | 0          | 0.93    | 0.91       | 1.00   | 6        | 1310 / 2480         | 0.0009  |
targets: recall PASS · precision PASS · violations PASS · ...

## Comparisons (paired, B − A)
| A  | B  | metric       | delta | 95% CI          | verdict        |

## By category
## Failure attribution (walk / truncation / final / budget / other) — overall and per category
## Top attribution details
## Misses (dev only): row, label, bucket, detail
## Warnings
```

`run.json` is the `RunReport` (§3.9). Both files are deterministic for fixture-mode runs given the same inputs (timestamps and run id aside), which makes them golden-testable.

### 4.10 Fixture record, replay and regeneration

Fixture keys hash the exact request, so they change whenever anything that shapes a request changes: card text (indexer or sanitizer change, repo SHA bump), wording, state composition, redaction, chunk composition, which nodes the walk expands, or the model pin. Replay is exact for code changes that don't alter requests and misses otherwise. Handling:

| Mode | Behavior |
|---|---|
| `replay` (CI) | Request-level hit → answer. Miss → question-level fallback: if every question hits in `qkeys`, assemble the answer (counted as `assembled`). Else miss → `eval.fixture_miss` (`fail` aborts with the first 10 missing `(row_key, stage, key)`; `null` answers "no decision" and marks the row) |
| `append` (regen default) | Live judge for misses only; existing entries untouched. Unchanged requests keep their old answers, so a metric change in the PR is attributable to the code, not to judge noise |
| `rewrite` | Start from empty; record everything (model pin bump, periodic refresh) |
| `prune` | `surf eval fixtures prune` drops entries not used by a full replay of all bench datasets |

Committed fixtures (`bench/fixtures/…`) store answers and keys only, sorted by key (diffable; ~1–2 MB per repo split). `--record-requests` stores full redacted requests for debugging in the run dir, never in `bench/`. User repos keep fixtures in `.surf/cache/eval-fixtures/` (gitignored; they contain prompts).

**Regeneration workflow** (`.github/workflows/eval-fixtures-regen.yml`):
1. Trigger: `workflow_dispatch` (input: branch) or a maintainer adding the `regen-fixtures` label to a same-repo PR. Fork PRs never get the secret; a maintainer pushes the branch to the main repo first (Q-16-4).
2. Check out the branch and the pinned external repos; build synthetic repos.
3. `surf eval --bench all --set dev --judge jev --record --fixture-mode append`, then `surf eval fixtures prune`.
4. Verify: `surf eval --bench all --set dev --judge fixture --fixture-miss fail` must pass.
5. Regenerate `bench/baselines/dev-fixture.json` from the verification run (it records `fixture_sha256` per file).
6. Commit fixtures + baseline to the branch ("eval: regenerate judge fixtures"), remove the label, and comment the metric diff vs. the previous `dev-fixture.json` on the PR.

The PR eval job prints, on a miss, the exact command and label to use, and which stage missed (call 1, walk level n, final), which usually identifies the cause.

### 4.11 Test-set hygiene

| Rule | Enforcement |
|---|---|
| Tune only on dev (spec §17.2 step 5) | `--set test` without `--final` → exit 2: "test is held out; use --final --reason" |
| Report on test once per release | `--final` requires `--reason`; appends to the test-run log; a second run for the same `(surf_version, dataset_sha256)` requires `--force-rerun` and the report header says `RERUN n` |
| No test-set tuning tools | `--sweep`, `--wording`, `--overlay` and ablations other than `A0`, `A4` (production) and `A6` are refused with `--set test` |
| No casual peeking | Test reports contain aggregates, CIs and category/attribution counts. Per-row detail goes only to `items.test.jsonl`; `--show-test-details` adds it to the markdown |
| No test fixtures in PR CI | Test fixtures are only recorded during a `--final` run; PR CI never uses the test split |

Test-run log (`.surf/eval/test-runs.jsonl` in user repos, committed; `bench/test-runs.jsonl` for surf):

```json
{"ts":"2026-10-20T10:00:00Z","run_id":"e_01J…","surf_version":"0.1.0","git_sha":"3f2a1c…","dirty":false,
 "dataset_sha256":"sha256:…","config_hash":"sha256:…","judge":"jev","model":"jev-1.13.0",
 "configs":["A4","A0"],"reason":"v0.1.0 release","user":"<git user.name>","rerun":0}
```

### 4.12 CI integration

| Job | Trigger | Command | Fails when |
|---|---|---|---|
| `eval-fixture` | PR touching `src/surf/**` or `bench/**` | `surf eval --bench all --set dev --judge fixture --fixture-miss fail` | any fixture miss; vs. `bench/baselines/dev-fixture.json`: `recall_macro` drop > `eval.regression_tolerance` (0.03), `violations_total` increase, `nocontext_correct` or `continuity_accuracy` drop > 0.05. Fixture runs are deterministic, so these are exact point comparisons. A PR that intends a change updates the baseline file (visible in review) |
| `eval-live-nightly` | schedule | `surf eval --bench all --set dev --judge jev --compare bench/baselines/v<last>.json` | dev `recall_macro` more than 0.03 below the last release's dev value **and** the paired CI vs. the stored baseline rows is `worse`; or `judge_failure_rate > 0.05`. Opens/updates an issue instead of blocking merges |
| `eval-release` | manual, on a release tag | `surf eval --bench all --set test --final --reason "v<ver> release" --ablate A4,A0` | test `recall_macro` more than 0.03 below the previous release's test value. Writes `bench/baselines/v<ver>.json` (dev + test sections, with `rows` for later paired comparisons) and the test-run log entry; §1.2 targets are reported, not blocking (spec §23 Phase 6 allows documented gaps) |
| `privacy` | PR | 14 §8 recording-transport + socket-guard tests | any assertion |

The first release has no previous baseline; the gate is skipped and the report says so.

Deviation D-16-3: spec §17.7 fails nightly on a **test** drop. Running test nightly contradicts "report on test once per release", so nightly compares dev, and the test comparison happens once, at release.

### 4.13 Labeling tooling

Minimal and offline; the router is never run while labeling (spec §17.2 "label before looking").

| Command | Behavior |
|---|---|
| `surf eval label` | Interactive loop: next free id (`q###`/`s###`), category menu, query (multi-line via `$EDITOR`), then label fields with tab completion over catalog paths and ids (stdlib `readline`; plain input on Windows). Each id is validated immediately (V3–V5, V11–V12). Appends to `.surf/eval/unsplit.yaml` with `labeler` set |
| `surf eval split` | Assigns each unsplit item to dev if `u(id) < ratio`, where `u = int(sha256(f"{seed}:{id}")[:16], 16) / 2^64`; stable, append-friendly (existing assignments never move). Warns if any category has < 2 items in either split. Sequences are units |
| `surf eval validate` | §3.2 against the current catalog |
| `surf eval label --second` | Samples `--sample` of dev items (seeded), writes them with labels blanked for a second labeler |
| `surf eval agree A B` | Agreement report (below) |

**Inter-labeler agreement** (spec §17.2 step 6):

| Measure | Definition |
|---|---|
| `must_f1` (per item) | Two labels match if their paths are equal, or one is a directory containing the other (tables and capabilities: equal ids). `P = matched(A)/|A|`, `R = matched(B)/|B|`, F1; both empty → 1.0. Report the mean and the share of items with F1 ≥ 0.8 |
| `must_should_f1` | Same over must ∪ should |
| `kappa_category` | Cohen's κ = (p_o − p_e)/(1 − p_e) over item categories |
| `kappa_continuity` | Cohen's κ over sequence turn `expect_continuity` |
| `kappa_capability` | Cohen's κ over (item, capability) decisions ∈ {use, not_needed, unlabeled} for capabilities labeled by either labeler |

Items with `must_f1 < 0.5` or differing category are listed for discussion; the resolution (relabel or drop) is recorded in the item's `notes`. The agreement summary is appended to the next eval report as a dataset-quality section.

---

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `eval.seed` | int | `1729` | Bootstrap, second-labeler sampling |
| `eval.split_seed` | int | `20260923` | `surf eval split` |
| `eval.bootstrap_n` | int | `1000` | Spec §17.3 |
| `eval.ci_level` | float | `0.95` | |
| `eval.dir_credit_max_files` | int | `8` | §3.3; equals the directory-collapse limit (spec §11.8) |
| `eval.category_weights` | map | spec §17.2 shares | §4.5 |
| `eval.concurrency_live` | int | `1` | Latency fidelity |
| `eval.concurrency_fixture` | int | `8` | |
| `eval.fixture_miss` | `"fail" \| "null"` | `"fail"` | |
| `eval.regression_tolerance` | float | `0.03` | Spec §17.7 |
| `eval.max_judge_failure_rate` | float | `0.05` | Invalid run above this |
| `eval.price_per_mtok` | map backend → float | `{jev: 0.042}` | Spec §11.9 |
| `eval.a0_max_cards` | int | `5000` | §4.1 |
| `router.mode`, `router.expand.enabled_kinds`, `index.cards.coupled_dirs` | see §3.5 | | New keys for ablations |

---

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| Dataset file missing or empty | Exit 3 with path; `surf init` step 7 points to `surf eval label` |
| No row has defined precision (all notes empty) | `precision_macro` = n/a; report warns |
| Label not in the catalog (renamed file) | V11 warning; miss attributed `not_in_index`; the report lists them first |
| Directory prefix flipped (`code:` ↔ `doc:`) since labeling | Matched by path (F2); V14 warning |
| Catalog changed since fixtures were recorded | Fixture misses → CI fails with the regen instruction (§4.10) |
| Judge rate-limited mid-run | Rows become `judge-unavailable` (counted as misses, attributed `status`); run invalid above 5 % |
| Live eval with `judge.provider` other than `typesafe` | Refused with a message: only TypeSafe direct pins `jev-1.13.0` exactly (OpenRouter pins the minor version, Vercel doesn't pin; 07 §3.2). Fixture replay of recordings made through TypeSafe is unaffected |
| Jev returns different probabilities for the same request across runs | Within a run, `MemoJudge` makes them identical; across runs it's noise, covered by CIs and the paired-CI condition in nightly |
| Sequence turn 0 routed `same` (impossible without a lease) | Scored as effective `new` |
| Duplicate query across dev and test | V2 error |
| `--set test --judge fixture` without `--final` | Refused (hygiene) |
| Overlay key unknown to config | Config validation error before any judge call |
| Bootstrap cards in Phase 0 with capability/sequence items | Scored `n/a`, listed in the report as skipped |
| A0 on a huge repo | Refused above `eval.a0_max_cards` without `--allow-large` |
| Interrupted run | Partial `run.json` with `complete=false`; not promotable to a baseline |
| Baseline from a different dataset sha | Comparison still runs on intersecting units; report warns; release gate requires matching sha or an explicit `--allow-dataset-change` |

---

## 7. Performance budget

| Operation | Budget |
|---|---|
| Load + validate 200 items against a 10k-card catalog | ≤ 1 s |
| Fixture-mode run, 100 items, one config | ≤ 30 s (routing CPU + scoring) |
| All bench repos, fixture mode, dev (PR CI job) | ≤ 5 min wall including synthetic repo builds and cached external checkouts |
| Bootstrap: 1,000 resamples × 15 metrics × 100 units | ≤ 1 s (per-unit partial sums) |
| Live dev run, 100 items, A4, concurrency 1 | ≈ items × route p50 ≈ 3 min |
| A8 coordinate sweep, live, 100 items | ≤ 19 × single run cost; memo and offline re-selection cut ~60 % in practice (estimate) |
| A0 live, 2,000-card repo, 100 items | ~700 requests, ~17.5M input tokens, ≈ $0.75; bound by the 250k tokens/s account limit (≥ 70 s of token budget), not by requests/min |

---

## 8. Test plan

| Area | Test | Kind |
|---|---|---|
| Dataset | Parse the spec §17.1 example verbatim; every V-rule with a failing and a passing case; line numbers in errors | unit |
| Label semantics | Table-driven `sat()` cases: file vs. file (prefix mismatch), dir label with file pointer, dir label with dir pointer, ancestor dir within/over credit, `mig:` ↔ `code:` alias, tables with schema qualification, exclusions with ancestor pointers | unit |
| Scoring | Hand-computed rows: recall/precision/violations/cap_acc/harmful_skips/nocontext; `should_include` affects precision only | unit |
| Metrics | macro vs. micro on a toy set; category weighting; nearest-rank percentiles | unit |
| Bootstrap | Pinned golden CI for a fixed seed (reproducibility across runs and Python 3.11–3.13); cluster resampling keeps sequence turns together; paired bootstrap with identical inputs gives delta 0 and `no_difference`; a constructed +0.2 improvement gives `better` | unit |
| Attribution | Synthetic `RouteTrace`s for each bucket and walk sub-reason; directory label takes the furthest stage | unit |
| Runner end-to-end | `feature-shop` synthetic repo + fixture judge: A1–A4 run; `run.json` and `report.md` golden (timestamps masked) | golden |
| Sequence runner | Scripted fixture answers for s004-like sequences: same → reuse, extends → delta scored, `gap_minutes: 60` → lease expiry → `new`; session ids unique; leases isolated from `.surf/cache` | integration |
| Fixture modes | replay hit, question-level assembly, miss with `fail` and `null`, append keeps old answers, prune removes unused, deterministic file ordering | unit (with 07) |
| Hygiene | test without `--final` refused; log entry written; rerun refused without `--force-rerun`; sweeps refused on test | unit |
| Phase 0 | `build_trivial_catalog` on a synthetic repo: excludes honored, F2/F4 ids, regex tables incl. drop; `route_flat` selection and trace | unit |
| Labeling | `split` stable when items are appended; `agree` κ and F1 on hand-computed examples | unit |
| Adversarial | `--adversarial` computes G1–G3 (14 §4.7) on the adversarial synthetic repo with fixtures | golden |
| Wordings | `shipped` text equals router constants; placeholder validation | unit |

---

## 9. Acceptance criteria

| Phase (spec §23) | Criterion from this doc |
|---|---|
| Phase 0 | `surf eval --cards bootstrap --ablate A0 --set dev --judge jev` produces `report.md`/`run.json` with recall/precision CIs and latency for both benchmark repos; the dataset validates; ≥ 60 queries per repo, 60/40 split, second-labeler agreement reported |
| Phase 2 | Paired comparison A0 vs. A4w on dev, reported by index size: sets `router.flat_max_tokens` to the largest size where A0's precision is `no_difference` or `better` with recall `no_difference` or `better` (spec D14). A1 vs. A0 on repos above the budget: precision `better` with recall `no_difference` or `better`, or a documented reason |
| Phase 3 | A2–A5 each have a paired comparison against its predecessor (all walk mode); `cross_layer` walk-bucket counts reported for A1 and A4w |
| Phase 4 | `continuity_accuracy ≥ 0.9` on dev sequences |
| Phase 5 | A6 and A7 reported with their own tuned thresholds; adversarial gates evaluated |
| Phase 6 | Exactly one logged `--final` test run per release (plus any logged reruns); `bench/baselines/v<ver>.json` committed; §1.2 targets marked pass/fail |
| Always | PR fixture job green with zero misses; the regen workflow reproduces a clean replay |

---

## 10. Deviations from the spec and open questions

**Deviations**

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-16-1 | §17.1 labels: `capabilities_use`, capability ids in `must_exclude` | Adds `capabilities_not_needed`; capability accuracy counts only labeled capabilities; adds `harmful_skips` | "Correct not-needed calls" needs labels; unlabeled capabilities would inflate accuracy |
| D-16-2 | §17.1 dir labels satisfied by the dir or a file inside it | Also: a file or dir label is satisfied by an ancestor directory pointer with ≤ 8 files | The router's directory collapse (spec §11.8) replaces ≥ 4 files with their small parent dir; without credit the collapse would register as recall loss |
| D-16-3 | §17.7 nightly live eval fails on a **test** recall drop | Nightly compares **dev** against the last release's dev; test compared once, at release | Spec §17.2 "report on test once per release" |
| D-16-4 | §17.3 recall/precision aggregation unspecified | Macro over rows (primary), micro reported; cluster bootstrap by item/sequence | Queries are the sampling unit; sequence turns are correlated |
| D-16-5 | §17.1 sequences test the lease | Turn-level `must_include` joins main recall; `must_include_delta` scored separately as `delta_recall` | Sequences are 15 % of the mix; their labels should count |
| D-16-6 | §23 Phase 0 "trivial file-card builder" | `eval/bootstrap.py` also emits directory entries (for label validation) and regex table cards | Lets directory and table labels be scored in Phase 0 |
| D-16-7 | §17.4 four buckets | Four buckets plus `status`, `not_in_index`, `lease`, `no_context_gate`, and walk sub-reasons | Every miss needs a cause; the added buckets are cases the four don't cover |
| D-16-8 | §9.1 datasets in the target repo's `.surf/eval/` | Same for users; surf's own benchmark lives in `bench/` of the surf repo with an external-repo manifest pinned by SHA | surf can't commit into external repos |
| D-16-9 | §17.6 pick wording "by the category-weighted metric" | Each wording is compared at its own best `final` threshold (offline re-selection), then paired bootstrap | Wordings shift the probability scale; a fixed threshold confounds wording with threshold |
| D-16-10 | §17.5 A4 = A3 + schema refs | A4 also enables `defined_in` and `fk` | They're schema edges; A4 is "full v1" |
| D-16-11 | §17.3 continuity accuracy | Scored on the **effective** continuity (post-threshold, skip-with-lease = `same`); raw Choice accuracy also reported | The effective value determines behavior |
| D-16-12 | §17.7 PR fixture run (no gate specified) | Gates on fixture misses and on point regressions vs. a committed fixture baseline | Deterministic replay makes exact gates possible |
| D-16-13 | §17.5: A0 flat brute force vs A1–A4 walk | A0 is the flat pass everywhere; A4 = defaults (flat within budget); new A4w forces the walk; A1–A3 and A5 force the walk | Flat is the default path below `flat_max_tokens` (spec D14), so the walk ablations must force walk mode to stay meaningful |

**Open questions**

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-16-1 | Is 8 files the right directory-credit limit? | Tie to the collapse limit (`eval.dir_credit_max_files = 8`) | If 09 changes the collapse rule, follow it |
| Q-16-2 | Live run-to-run noise vs. the 0.03 tolerance | Keep 0.03 but require the paired CI to say `worse` in nightly; measure noise with two live runs in Phase 0. Evidence (2026-09-23): Jev is not bit-reproducible. TypeSafe's self-consistency cookbook reports a mean per-question SD of 0.0102 over 15 repeats, with single answers spanning 0.43–0.53 (that run varied a `uid` field in state, so it bounds rather than measures identical-request noise). Items near a threshold can flip between live runs | Phase 0 noise measurement (07 §8.4) |
| Q-16-9 | TypeSafe's question-writing guidance suggests wordings v1 hasn't tried: one judgment per question; high value = yes; state paths referenced in backticks (`request`); structured `instructions` with the card as a named field; optional Noul `criteria` (`true`/`false` descriptions); explicit boundaries ("what it is not for") for similar options. Add them as candidates? | Yes, as **candidates** in `bench/wordings.yaml` for the §17.6 experiments, never as shipped wordings without a dev run. Structured candidates need object-valued `instructions` in the wordings schema and in 07 (Q-07-8) | Dev-set wording experiment (spec §17.6) |
| Q-16-10 | TypeSafe cookbooks rank candidates with a **Choice** over the shortlist and walk hierarchies with one Choice per node plus beam search. v1 uses one Noul per candidate | Not in v1: Nouls are absolute and can all be low, which surf's selection needs; Choice is relative. Flat mode (spec D14) already removes most walk questions. v2 ablation idea only | After the v1 test report |
| Q-16-3 | Question-level fixture fallback and replayed latency (`simulate_latency`) belong to 07's `judge/fixture.py` | Adopt both in 07 | Resolved: 07 D-07-3 |
| Q-16-4 | The regen workflow runs PR code with the judge API key | Maintainer label only, same-repo branches only, a budget-capped eval-only key | Maintainer decision |
| Q-16-5 | Test set is 24–40 queries per repo → recall CI width ≈ ±0.1; the 0.85 target is weakly tested | Headline test metric pooled across both repos (cluster bootstrap by item), per-repo also shown | User decision |
| Q-16-6 | Heimdall: license and pin | Record SPDX in the manifest; replace if clone-in-CI isn't allowed | Legal check in Phase 0 |
| Q-16-7 | Minimum inter-labeler agreement for a dataset to be usable | Report only; flag if mean `must_f1 < 0.7` | Phase 0 experience |
| Q-16-8 | New keys `router.mode`, `router.expand.enabled_kinds`, `index.cards.coupled_dirs`, and pure `route/select.py` over a trace | Adopt in 13 and 09 | Resolved: 13 §3, 09 §2 (`route/select.py`) |
