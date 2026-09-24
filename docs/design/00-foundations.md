# 00 · Foundations: shared model, conventions and cross-cutting rules

**Status:** draft for review
**Spec sections:** §2, §5, §6, §9, §13.1, §22
**Read this first.** Every other design doc in `docs/design/` builds on the types, id grammar and rules defined here. If a component doc disagrees with this one, this one wins until it's amended.

---

## 1. Purpose

The v1 spec (`docs/spec/architecture-v1.md`) describes *what* `surf` does. The design docs describe *how* each component is built: module boundaries, Python interfaces, algorithms in implementable detail, edge cases, tests and acceptance criteria.

This document pins down the things every component shares:

1. The surface **id grammar** and path normalization.
2. The **core data types** (`Card`, `Edge`, `Meta`, `Selection`, `RouteResult`, …) and the module that owns them.
3. **Determinism rules** for anything written to `.surf/`.
4. The **fail-open error model** and deadline handling.
5. Code, test and repository conventions.
6. Spec ambiguities that affect more than one component, with the proposed resolution.

---

## 2. Surface ids

### 2.1 Grammar

```
surface_id   := prefix ":" locator
prefix       := "code" | "doc" | "db" | "mig" | "mcp" | "skill" | "agent" | "cmd"
locator      := repo-relative POSIX path            (code, doc, mig)
              | repo-relative POSIX path + "/"      (code dirs, doc dirs)
              | table name                          (db)
              | "*"                                 (db schema root; see 2.3)
              | capability name                     (mcp, skill, agent, cmd)
```

| Surface type | Example id |
|---|---|
| `code_dir` | `code:src/fulfillment/` |
| `code_file` | `code:src/fulfillment/ship.ts` |
| `doc_dir` | `doc:docs/fulfillment/` |
| `doc_file` | `doc:docs/fulfillment/shipping-lifecycle.md` |
| `schema_root` | `db:*` |
| `db_table` | `db:orders` (schema-qualified when not the default schema: `db:billing.invoices`) |
| `db_migration` | `mig:supabase/migrations/20260611_add_shipments.sql` |
| `mcp_server` | `mcp:supabase` |
| `skill` | `skill:db-migrations` |
| `subagent` | `agent:code-reviewer` |
| `command` | `cmd:deploy` |

The **virtual root** of the content tree has the reserved id `root:` and no card. It is never selected and never sent to the judge.

### 2.2 Path normalization (all path-bearing ids)

- Repo-relative, POSIX separators, no leading `./` or `/`.
- Case preserved exactly as git stores it. Comparisons are case-sensitive; path matching (§11.3 of the spec) does its own case-insensitive fallback.
- Unicode normalized to NFC.
- Directories always end in `/`; files never do.
- Paths containing `:` are legal (the id is split on the **first** `:` only).

Helpers live in `surf/ids.py`:

```python
def make_id(prefix: Prefix, locator: str) -> SurfaceId: ...
def split_id(sid: SurfaceId) -> tuple[Prefix, str]: ...
def path_of(sid: SurfaceId) -> str | None: ...        # None for db/mcp/skill/agent/cmd
def norm_path(p: str, *, is_dir: bool) -> str: ...
```

### 2.3 Resolved ambiguities in the id scheme

These are gaps in the spec. The resolutions below are **proposals** and are also listed in `docs/open-questions.md`.

| # | Gap | Proposed resolution |
|---|---|---|
| F1 | The schema root's id isn't given (`db:` prefix only). | `db:*`. `*` can't be a table name. |
| F2 | `doc_dir` vs `code_dir` is decided by "docs dominate", so a directory's **id prefix can flip** when its contents change. That would churn edges, leases and eval labels. | Ids for **directories** keep the prefix chosen by a stable rule: `doc:` if > 50 % of recursive files are doc files **and** the dir isn't the repo root, else `code:`. The flip is treated as remove + add by refresh. Anything that compares directories by identity (eval labels, lease re-validation, path hits) compares by **path**, via `path_of()`, never by prefix. |
| F3 | Migrations are "also indexed as files", so one file could have two ids (`code:…sql` and `mig:…sql`). | The file keeps **one tree node**: `code:<path>` (walked, path-hit, co-changed like any file). A separate `mig:<path>` card exists **outside the tree** (`parent = None`, no `contains` edge) and carries `defined_in` edges from tables. An `alias` edge links `code:<path>` ↔ `mig:<path>` so either one resolves to the other. The note renders migrations from the `mig:` id. |
| F4 | Which prefix does a `.md` file inside `src/` get? | Files: prefix by **file type**, never by location. Any doc extension (§7.4) → `doc:`; everything else → `code:`. |

---

## 3. Core data types

All shared types live in **`surf/model.py`** (a module the spec's §22.2 layout doesn't list; it is added so that `index`, `graph`, `catalog`, `route`, `lease` and `eval` don't import each other for types). Types are `pydantic` v2 models with `frozen=True` unless noted. Field names match the JSONL records in spec §9 so records round-trip without mapping code.

```python
# surf/model.py  (sketch; exact field lists are owned by the component docs cited)
SurfaceId = NewType("SurfaceId", str)

class SurfaceType(StrEnum):
    CODE_DIR = "code_dir"; CODE_FILE = "code_file"; DOC_DIR = "doc_dir"; DOC_FILE = "doc_file"
    SCHEMA_ROOT = "schema_root"; DB_TABLE = "db_table"; DB_MIGRATION = "db_migration"
    MCP_SERVER = "mcp_server"; SKILL = "skill"; SUBAGENT = "subagent"; COMMAND = "command"

    @property
    def is_capability(self) -> bool: ...
    @property
    def is_dir(self) -> bool: ...          # code_dir, doc_dir, schema_root

class Card(BaseModel):                     # one line of catalog.jsonl   (02-cards.md)
    id: SurfaceId
    type: SurfaceType
    path: str | None                       # None for db / capability cards
    parent: SurfaceId | None               # "root:" for top-level nodes; None for mig: and capabilities
    card: str                              # exact text the judge sees
    fields: dict[str, Any]                 # typed per surface type, see 02-cards.md
    hash: str                              # "sha256:<hex>" over card inputs (spec §7.8)

class EdgeKind(StrEnum):
    CONTAINS = "contains"; CO_CHANGE = "co_change"; SCHEMA_REF = "schema_ref"
    DEFINED_IN = "defined_in"; FK = "fk"; ALIAS = "alias"; DIR_COUPLING = "dir_coupling"

class Edge(BaseModel):                     # one line of edges.jsonl    (04-graph-edges.md)
    from_: SurfaceId = Field(alias="from")
    to: SurfaceId
    kind: EdgeKind
    weight: float                          # [0, 1], rounded to 4 dp on write
    evidence: dict[str, Any] = {}

class Meta(BaseModel): ...                 # meta.json                  (05-catalog-store.md)

class Selection(BaseModel):                # what a route selected; also stored in the lease
    content: list[SurfaceId]
    capabilities_use: list[SurfaceId]
    capabilities_not_needed: list[SurfaceId]

class RouteStatus(StrEnum):                # spec §18.1
    ROUTED = "routed"; LEASE_REUSE = "lease-reuse"; SKIPPED = "skipped"; NO_CONTEXT = "no-context"
    NO_CANDIDATES = "no-candidates"; DEADLINE = "deadline"; JUDGE_UNAVAILABLE = "judge-unavailable"
    INDEX_MISSING = "index-missing"; ERROR = "error"

class RouteRequest(BaseModel):
    request: str
    session_id: str | None = None
    previous_message: str | None = None
    cwd: str | None = None                 # reserved for v2 monorepo scoping

class RouteResult(BaseModel):
    route_id: str                          # "r_" + ULID
    status: RouteStatus
    note: str | None                       # None → inject nothing
    selection: Selection | None
    continuity: Literal["same", "extends", "new"] | None
    low_confidence: bool
    trace: RouteTrace | None               # only populated when explain=True (09-router.md)
```

Two new edge kinds are introduced relative to the spec:

- `alias` — links the two ids of a migration file (F3). Weight 1.0, never used in expansion scoring.
- `dir_coupling` — directory-level co-change (spec §8.2.5). The spec computes it but never says where it's stored. Storing it as edges keeps `coupled_dirs` on directory cards reproducible and lets `surface_info` explain it.

---

## 4. Determinism rules for `.surf/`

The spec commits `catalog.jsonl`, `edges.jsonl` and `meta.json`, and CI runs `surf index --check` against a fresh build (spec §10.1). That only works if a build is **byte-for-byte reproducible** from the same commit. Rules:

1. **Sorted output.** `catalog.jsonl` is sorted by `id`; `edges.jsonl` by `(from, to, kind)`. JSON keys are written in a fixed order (model field order), compact separators, UTF-8, `\n` line endings, trailing newline.
2. **Rounded floats.** All weights are rounded to 4 decimal places on write. Comparisons in code use the rounded value after load, so a refresh and a full build agree.
3. **No wall-clock time in content files.** The spec's card record has `indexed_at`; it is **dropped** from `catalog.jsonl` (proposal D-F5). Build timestamps live only in `meta.json`, and `--check` ignores `meta.built_at`.
4. **Recency is relative to the indexed commit, not "now".** Co-change decay (spec §8.2.3) uses `t_c = commit_time(index_head) − commit_time(c)`. Using wall-clock age would make the committed catalog drift daily and fail `--check` with no code change. (Proposal D-F6.)
5. **No environment leakage.** Absolute paths, usernames, hostnames and env values never appear in committed files. Live-MCP listings are stored with the server's `command` basename only; URLs have credentials and query strings stripped.
6. **Stable tie-breaks.** Every "top N" (partners, tables, child files) sorts by score desc, then id asc.

### 4.1 Commit policy and the local-only overlay (amended after 01/05/06/14)

- **Proposed default: `index.commit_catalog = false`** (05 D-05-1, 06 D-06-1, open question Q-06-1). The catalog lives in `.surf/cache/`, and the router **always reads `cache/`** (05 D-05-5). A committed catalog is an opt-in *baseline* that changes only via `surf index --baseline`; git hooks only ever write to `cache/`. The determinism rules above still apply to both, because `--check` and the eval fixtures rely on them.
- **Local-only cards.** A card is committable only if every input it was built from is committable (01 D-01-1). Cards built from untracked files or user-level capability configs go to **`.surf/cache/overlay.jsonl`** and are merged at load time. This is the only name for that file; 13 and 14 refer to it.
- **Stored edge set.** `contains` edges aren't written to `edges.jsonl` (they're derived from `Card.parent`), and symmetric kinds (`co_change`, `dir_coupling`, `alias`) are stored once with `from < to` (04 D-04-1, D-04-4). **Nothing outside `catalog/` reads the JSONL files directly**; use the store API in 05.
- **Flat set.** `mig:` cards and external stub tables (03 D-03-3) are not content cards and are never asked in the flat pass (09 §4.5, 05 `flat_cards()`).

### 4.2 Clock

The injected `Clock` exposes `monotonic()` (deadlines only) and `wall()` (lease idle expiry, log timestamps) separately, so the eval runner can simulate wall time across sequence turns without affecting deadlines (16).

---

## 5. Error model: fail open, always

The spec's principle 6 is enforced structurally:

- **Index time** may fail loudly (`surf index` exits non-zero with a message). A partial index is never written: builds write to `*.tmp` and `os.replace()` atomically.
- **Query time never raises to the caller.** `route()` is wrapped by a single guard in `route/pipeline.py`. Any exception becomes `RouteResult(status="error", note=None)` plus a decision-log entry. Adapters additionally wrap their own entrypoints so a crash in `surf` itself can't break the host harness (for the Claude Code hook: exit 0 with empty output).
- **Degradation ladder.** When something is missing, route with what's left rather than failing:

  | Missing | Behavior |
  |---|---|
  | Index | `index-missing`, no note |
  | Judge (timeout, breaker open, no key) | `judge-unavailable`, no note. **Path hits alone are not emitted** in v1: they would be un-judged pointers. (Open question Q-F7.) |
  | Lease store | Route as `new` without a lease |
  | Decision log unwritable | Route normally; warn once per process on stderr |

- **Deadlines.** A single `Deadline` object (monotonic clock) is created per route and passed down. Components check `deadline.remaining_ms()` before starting work and pass `min(per_request_timeout, remaining)` to the judge. See `09-router.md` for the two budgets (walk budget vs. route budget), which the spec conflates under one `deadline_ms` name.

---

## 6. Repository and code conventions

| Topic | Convention |
|---|---|
| Distribution name | `jev-surfer` on PyPI (the name `surf` is very likely taken); import package `surf`; console script `surf`. (Q-F1) |
| Python | 3.11+. `from __future__ import annotations` everywhere. |
| Tooling | `uv` for env and lockfile; `ruff` (lint + format); `pyright` in strict mode for `surf/`; `pytest`. |
| Layout | `src/surf/…` (src layout) with the module tree from spec §22.2, plus `surf/model.py`, `surf/ids.py`, `surf/deadline.py`, `surf/proc.py`, and modules added by the design docs: `surf/runtime.py`, `surf/control.py`, `adapters/_cc_transcript.py`, `adapters/_jsonfile.py`, `install/manifest.py` (12); `surf/config_write.py` (13); `lease/logic.py`, `surf/filelock.py` (10); `judge/jev_wire.py`, `judge/breaker.py` (07); `route/pathindex.py` (08); `route/state.py`, `route/trace.py`, `route/wordings.py` (09); `route/note_text.py` (11); `index/globs.py`, `index/frontmatter.py` (01, 02; below); `eval/bootstrap.py`, `eval/flat.py`, `eval/label.py`, `log/explain.py`, `log/stats.py` (16, 15). surf's own benchmark (pinned external repos, synthetic repos, datasets, fixtures, baselines) lives in top-level `bench/` (16 D-16-8). `redact.py` is a Phase 0 deliverable because the bootstrap card builder uses its sanitizer. |
| Dependencies | Only those in spec §22.1. Anything new needs a line in the relevant design doc saying why. |
| Pure core | `index/`, `graph/`, `route/`, `lease/` take their inputs as arguments (catalog, config, judge, clock). No module reads config or env on import. This makes every piece unit-testable with the fixture judge. |
| Clock | Injected `Clock` protocol (`wall()`, `monotonic()`; see §4.2); tests use a fake clock. |
| Subprocesses | `git` and `rg` only, invoked through `surf/proc.py` with timeouts and `LC_ALL=C`. Never `shell=True`. **One exception:** opt-in live MCP listing (`index/extract_caps.py`) spawns the user-configured server commands, under the isolation rules in `02-cards.md` and `14-security-privacy.md`. |
| Small shared helpers | `index/globs.py` (gitignore-style matcher, so `pathspec` isn't needed) and `index/frontmatter.py` (minimal YAML-subset frontmatter parser, so PyYAML isn't a runtime dependency). |
| Logging | stdlib `logging` to stderr for humans; decision records are a separate channel (`log/decisions.py`). Nothing is ever printed to stdout from the hook or MCP paths except the protocol payload. |

### 6.1 Test strategy (shared)

- **Unit tests** per module, offline, using the `fixture` judge (`judge/fixture.py`) and synthetic repos under `tests/fixtures/repos/` built by a script from a YAML description (files + commit history), so co-change is testable without checked-in `.git` dirs.
- **Golden tests** for card rendering, note rendering and `catalog.jsonl` output: a fixture repo's expected catalog is checked in and diffed.
- **Property tests** (`hypothesis`, dev-only dependency) for path normalization, id round-trips and the co-change math bounds.
- **Fail-open tests**: every adapter entrypoint is run against a judge that raises, times out, and returns garbage; the assertion is always "no exception, no note, a decision record with the right status".

---

## 7. Design doc map

| Doc | Component(s) | Code |
|---|---|---|
| `00-foundations.md` | shared model, ids, conventions | `model.py`, `ids.py`, `deadline.py`, `proc.py` |
| `01-discovery.md` | file enumeration, excludes, agent-config detection | `index/discover.py` |
| `02-cards.md` | code, doc, dir, capability cards; budgets; hashing | `index/extract_code.py`, `extract_docs.py`, `extract_caps.py`, `cards.py` |
| `03-schema-extraction.md` | migration replay, table cards | `index/extract_schema.py` |
| `04-graph-edges.md` | containment, co-change, dir coupling, schema refs, expansion scoring | `graph/*` |
| `05-catalog-store.md` | JSONL + SQLite, meta, commit policy | `catalog/*` |
| `06-refresh.md` | incremental rebuild, triggers, `--check` | `index/build.py`, `adapters/git_hooks.py` |
| `07-judge.md` | judge protocol, backends, resilience | `judge/*` |
| `08-path-matching.md` | path extraction and normalization | `route/pathmatch.py`, `route/pathindex.py` |
| `09-router.md` | pipeline, skip, call 1, walk, expansion, final pass, selection | `route/*` |
| `10-lease.md` | task lease | `lease/manager.py`, `lease/logic.py`, `filelock.py` |
| `11-note.md` | note rendering | `route/note.py`, `route/note_text.py` |
| `12-delivery.md` | CLI, MCP server, instruction snippet, Claude Code adapter, init/uninstall | `cli.py`, `runtime.py`, `control.py`, `adapters/*`, `install/manifest.py` |
| `13-config.md` | `config.toml` schema and validation | `config.py`, `config_write.py` |
| `14-security-privacy.md` | redaction, sanitization, injection defenses | `redact.py` |
| `15-observability.md` | decision log, `--explain`, `surf stats` | `log/decisions.py`, `log/explain.py`, `log/stats.py` |
| `16-evaluation.md` | datasets, runner, metrics, attribution, ablations | `eval/*` |
