# 13 · Configuration

**Status:** draft for review
**Spec sections:** §16 (all); keys referenced in §7.1–§7.7, §8.2, §8.3, §8.5, §9.5, §10.1, §11.2–§11.9, §12.4, §13.2–§13.3, §14.3, §15.4, §17, §18.2, §19.3
**Depends on:** 00-foundations. Every other design doc consumes keys defined here; each doc's §5 lists the subset it reads, and this doc is the single registry of names, types and defaults. Where docs disagreed, the resolution is recorded in §10 (D-13-8).
**Code:** `surf/config.py` (models, loader, validation, fingerprints), `surf/config_write.py` (template generation and comment-preserving edits used by `surf init`)

---

## 1. Purpose and scope

One typed, validated configuration object (`SurfConfig`) built from layered TOML files, environment variables and CLI flags. It must be:

- **Complete:** every tunable in the spec and design docs has exactly one key here, with the name its owning doc uses.
- **Deterministic where it matters:** the indexer only sees an **index view** built from defaults plus the project file, so every machine builds the same catalog from the same commit.
- **Fail-open at query time:** a bad config never breaks the host harness (hook/MCP → `status=error`, no note). It fails loudly in CLI commands (exit 5).

| In scope | Out of scope |
|---|---|
| Schema (pydantic v2), defaults, validation, cross-field rules | Secrets in config files (rejected; keys come from env, 07 §3.2) |
| File discovery, precedence, env and CLI overrides, deprecated aliases | Remote/shared config services |
| Index view and per-section fingerprints (05 `config_fingerprint`, 06 cache invalidation) | Hot-reload beyond "reload when mtime changes" (12 §4.5.2) |
| `surf init` template; comment-preserving writes of `capabilities.describe` | A general TOML writer |

Deliberately **not** config: card budgets and churn thresholds (02: code constants tied to `CARD_FORMAT_VERSION`), path-hit strengths (08), note wording (11 §4.8, overridable only through eval wordings).

## 2. Interfaces

```python
# surf/config.py
CONFIG_VERSION: Final = 1

class LoadedConfig(BaseModel):
    config: SurfConfig                       # merged view: router, judge, lease, note, delivery, log, eval, …
    index_view: SurfConfig                   # defaults + project layer only (+ capabilities.user_level from merged); §4.5
    origins: dict[str, ConfigLayer]          # dotted key → layer that set it (non-default leaves only)
    warnings: list[ConfigWarning]
    fingerprints: dict[str, str]             # per index section → "sha256:<hex>"; plus "all"
    files: list[Path]                        # files actually read, in precedence order

ConfigLayer = Literal["default", "user", "project", "project-local", "extra", "env", "cli"]

class ConfigWarning(BaseModel):
    code: Literal["unknown-key", "ignored-index-key", "deprecated", "unknown-profile", "soft-limit"]
    key: str; layer: ConfigLayer; message: str; suggestion: str | None = None

class ConfigError(Exception):                # carries every problem found, not only the first
    problems: list[tuple[str, ConfigLayer, str]]   # (dotted key, layer, message)

def load_config(root: Path | None, *, env: Mapping[str, str] = os.environ,
                cli: Mapping[str, Any] | None = None,
                user_config: Path | None | Literal["auto"] = "auto",
                purpose: Literal["runtime", "index", "eval"] = "runtime") -> LoadedConfig: ...

def thresholds_table(cfg: SurfConfig) -> Mapping[str, Thresholds]: ...   # consumed by 07 §4.9 lookup
def raw_peek(root: Path, dotted: str) -> Any | None: ...                  # tomllib only; hook fast path (12 §4.4.3)
```

- `surf/runtime.py` (12) loads once per process with `purpose="runtime"`. `surf index`/`refresh` use `purpose="index"`, and `surf eval` uses `purpose="eval"`. `purpose` changes validation only, e.g. `judge.backend = "llm"` is allowed for `eval` (§4.4).
- Per 00 §6, no module reads config on import. Components receive their sub-model (`cfg.router`, `cfg.lease`, …) as an argument. The indexer (01–04, 06) receives `index_view`, never `config`.
- `surf config show [--origin] [--defaults] [--toml] [--json]`, `surf config validate [--strict]`, `surf config path` (12 §4.12) expose all of this.
- Threshold-profile **resolution** (which table applies to a backend/model) is 07 §4.9. This doc only defines and validates the tables.

## 3. Data structures: the complete schema

Conventions: all models `frozen=True`, `extra="allow"` (extras → warnings, §4.4). Key names are `snake_case`. Durations carry their unit (`_ms`, `_s`, `_minutes`, `_days`, `_months`). `Prob = Annotated[float, Field(ge=0, le=1)]`. `Glob` = gitignore-style pattern with `/` separators (01 §3.4). `Regex` = Python regex ≤ 500 chars that compiles. **Owner** = the doc that defines the semantics. **Scope**: `I` = index-scoped (read from the index view, part of a fingerprint), `R` = runtime.

### 3.1 Root

```python
class SurfConfig(BaseModel):
    version: int = CONFIG_VERSION
    project: ProjectConfig; index: IndexConfig; capabilities: CapabilitiesConfig
    router: RouterConfig; lease: LeaseConfig; judge: JudgeConfig; privacy: PrivacyConfig
    log: LogConfig; explain: ExplainConfig; note: NoteConfig; delivery: DeliveryConfig
    refresh: RefreshConfig; store: StoreConfig; eval: EvalConfig
```

### 3.2 `[project]`

| Key | Type | Default | Validation | Scope | Owner |
|---|---|---|---|---|---|
| `project.descriptor` | `str \| None` | `None` → derived at index time and stored in `meta.json` (02 §4.10) | single line, 1–160 chars, sanitized like card text | R | 02, 09 |

### 3.3 `[index]`

| Key | Type | Default | Validation | Scope | Owner |
|---|---|---|---|---|---|
| `index.exclude` | `list[Glob]` | `[]` | no absolute paths, no `..`, no `\` | I | 01 |
| `index.include` | `list[Glob]` | `[]` | re-includes default/user excludes; can't re-include always-excluded or capability-definition paths (01 §5); secret-like re-includes are listed by `doctor` (14) | I | 01, 14 |
| `index.default_excludes` | bool | `true` | | I | 01 |
| `index.include_untracked` | bool | `true` | untracked-file cards go to the overlay (00 §4.1) | I | 01 |
| `index.max_file_bytes` | int | `1_000_000` | 1 KB–100 MB | I | 01 |
| `index.respect_linguist` | bool | `true` | | I | 01 |
| `index.exclude_generated` | bool | `true` | | I | 01 |
| `index.commit_catalog` | bool | **`false`** | see D-13-6 | R | 05, 06 |
| `index.max_catalog_mb_warn` | int | `20` | ≥ 1 | R | 05 |

`[index.cards]`

| Key | Type | Default | Scope | Owner |
|---|---|---|---|---|
| `index.cards.coupled_dirs` | bool | `true` | I | 02, 16 (ablation A5) |

`[index.schema]`

| Key | Type | Default | Validation | Scope | Owner |
|---|---|---|---|---|---|
| `index.schema.enabled` | bool | `true` | | I | 03 |
| `index.schema.sources` | `list[Glob] \| None` | `None` = auto-detect (spec §7.5) | | I | 03 |
| `index.schema.dialect` | str | `"postgres"` | sqlglot dialect name (checked at index time); fallback only | I | 03 |
| `index.schema.ignore_schemas` | `list[str]` | `["pg_catalog","information_schema","supabase_migrations","supabase_functions","extensions","graphql","graphql_public","realtime","_realtime","vault","pgsodium","net","cron","pgbouncer"]` | | I | 03 |
| `index.schema.external_stubs` | bool | `true` | | I | 03 |
| `index.schema.max_statements_per_file` | int | `20000` | ≥ 100 | I | 03 |
| `index.schema.max_defined_in` | int | `4` | 1–10 | I | 04 |

`[index.cochange]` (any change forces a full co-change rebuild, 04 §5)

| Key | Type | Default | Validation | Scope | Owner |
|---|---|---|---|---|---|
| `index.cochange.enabled` | bool | `true` | forced off for non-git | I | 04 |
| `index.cochange.window_months` | int | `24` | 1–120 | I | 04 |
| `index.cochange.max_commits` | int | `5000` | 100–100,000 | I | 04 |
| `index.cochange.max_files_per_commit` | int | `30` | 2–1,000 | I | 04 |
| `index.cochange.half_life_days` | float | `180` | > 0 | I | 04 |
| `index.cochange.min_shared_commits` | int | `2` | ≥ 1 | I | 04 |
| `index.cochange.min_coupling` | Prob | `0.15` | | I | 04 |
| `index.cochange.top_partners` | int | `10` | 1–50 | I | 04 |
| `index.cochange.exclude_authors` | `list[str]` | `["dependabot", "renovate", "github-actions"]` | casefolded **substring** of author name or email, so `github-actions[bot]`, `dependabot[bot]` and `renovate[bot]` all match (settles the spec's §8.2.2 vs §16 difference) | I | 04 |
| `index.cochange.exclude_message_patterns` | `list[Regex]` | `['^(chore\|style\|format)\b', '\bprettier\b', '\blint\b']` | subject line only, case-insensitive | I | 04 |
| `index.cochange.rename_limit` | int | `2000` | git `-l` | I | 04 |
| `index.cochange.dir_top_partners` | int | `5` | 1–20 | I | 04 |
| `index.cochange.dir_min_coupling` | Prob | `0.15` | | I | 04 |
| `index.cochange.dir_min_shared_commits` | int | `2` | ≥ 1 | I | 04 |
| `index.cochange.max_dirs_per_commit` | int | `40` | ≥ 2 | I | 04 |

`[index.schema_refs]`

| Key | Type | Default | Validation | Scope | Owner |
|---|---|---|---|---|---|
| `index.schema_refs.stopwords` | `list[str]` | `["data","items","logs","status","type","meta"]` | lowercase | I | 04 |
| `index.schema_refs.min_name_len` | int | `4` | 1–20 | I | 04 |
| `index.schema_refs.max_tables_per_file` | int | `20` | 1–200 | I | 04 |
| `index.schema_refs.max_files_per_table` | int | `200` | 1–10,000 | I | 04 |
| `index.schema_refs.min_weight` | Prob | `0.05` | | I | 04 |
| `index.schema_refs.engine` | `Literal["auto","rg","python"]` | `"auto"` | results must be identical (04 parity test), so not fingerprinted | R | 04 |

### 3.4 `[capabilities]`

| Key | Type | Default | Validation | Scope | Owner |
|---|---|---|---|---|---|
| `capabilities.harnesses` | `list[str]` | `["claude","cursor","codex","opencode"]` | known harness names | I | 01 |
| `capabilities.user_level` | bool | `false` | opt-in to user-level harness configs; results go to the overlay only (00 §4.1). Taken from the **merged** config (a per-user choice) | R | 01, 02 |
| `capabilities.live_mcp` | `list[str]` | `[]` | server names; live listing is off by default (14 D-14-5) | I | 02, 14 |
| `capabilities.live_timeout_ms` | int | `10_000` | 1,000–60,000 | I | 02 |
| `capabilities.live_env` | `Literal["inherit","minimal"]` | `"inherit"` | | I | 02 |
| `capabilities.describe` | `dict[str, str]` | `{}` | value: single line, 1–200 chars | I | 02 |
| `capabilities.exclude` | `list[str]` | `[]` | capability ids or globs (`mcp:internal-*`). `mcp:surf` is always excluded (12 §4.5.3) | I | 02, 12 |

### 3.5 `[router]`

The spec's `deadline_ms` names two budgets (00 §5). It's split into `walk_deadline_ms` and `route_deadline_ms`, and `router.deadline_ms` is accepted as a **deprecated alias** of `route_deadline_ms` (warning `deprecated`; if both are set, `route_deadline_ms` wins).

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `router.mode` | `Literal["auto","flat"]` | `"auto"` | `flat` = ablation A0 | 16, 09 |
| `router.small_repo_cutoff` | int | `60` | 0–1,000 | 09 |
| `router.flatten_at` | int | `40` | 1–200 | 09 |
| `router.flatten_factor` | Prob | `0.9` | | 09 |
| `router.chunk_size` | int | `40` | 5 – `judge.max_questions_per_request` | 09 |
| `router.beam_max` | int | `6` | 1–40 | 09 |
| `router.beam_min` | int | `1` | 0 – `beam_max` | 09 |
| `router.max_frontier` | int | `12` | 1–100 | 09 |
| `router.max_depth` | int | `4` | 1–10 | 09 |
| `router.max_candidates` | int | `40` | 5–64 | 09 |
| `router.max_pointers` | int | `12` | 1–30 (> 12 soft-limit warning) | 09, 10 |
| `router.max_caps_use` | int | `6` | 0–40 | 09, 11 |
| `router.max_caps_skip` | int | `10` | 0–40 | 09, 11 |
| `router.collapse_min_files` | int | `4` | 2–20 | 09 |
| `router.collapse_max_dir_files` | int | `8` | 1–100 | 09 |
| `router.walk_deadline_ms` | int | `2000` | 100–30,000; `< route_deadline_ms` | 09 |
| `router.route_deadline_ms` | int | `3000` | 200–60,000 | 09, 12 |
| `router.deadline_ms` | int | — | deprecated alias → `route_deadline_ms` | 13 |
| `router.min_level_ms` | int | `400` | ≥ 50 | 09 |
| `router.min_final_ms` | int | `300` | ≥ 50 | 09 |
| `router.speculative_walk` | bool | `true` | | 09 |
| `router.request_max_tokens` | int | `1500` | 200–8,000 | 09, 14 |
| `router.context_max_tokens` | int | `300` | 50–2,000 (`previous_task`, `last_message`) | 09 |
| `router.final_max_request_tokens` | int | `6000` | ≤ `judge.max_request_tokens` | 09 |

`[router.skip]`

| Key | Type | Default | Owner |
|---|---|---|---|
| `router.skip.ack_max_words` | int 1–10 | `4` | 09 |
| `router.skip.ack_words` | `list[str]` (≤ 200) | the list in 09 §4.3 | 09, 12 |

`[router.wording]` (09; values chosen by eval, 16 §3.6)

| Key | Type | Default | Validation |
|---|---|---|---|
| `router.wording.walk` / `final` / `capability` | str | spec §11.5 / §11.7 / §11.4 texts | must contain `{card}` |
| `router.wording.needs_context` / `continuity` | str | spec §11.4 texts | |
| `router.wording.continuity_options` | `dict[str, str]` | spec texts | keys exactly `same`, `extends`, `new` |

`[router.expand]` (04, spec §8.5)

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `router.expand.enabled_kinds` | `list[EdgeKind]` | `["contains","co_change","schema_ref","defined_in","fk"]` | subset of those five; ablations A1–A4 | 04, 16 |
| `router.expand.kind_factors` | `dict[str, float]` | `{co_change=1.0, schema_ref=0.9, defined_in=0.8, fk=0.5, contains=0.6}` | same key set; 0–1 | 04 |
| `router.expand.max_per_anchor` | int | `8` | per kind; 1–50 | 04 |
| `router.expand.max_total` | int | `40` | 1–200 | 04 |
| `router.expand.index_names` | `list[Glob]` | `["README","README.*","index.*","_index.md","__init__.py","mod.rs"]` | | 04 |

`[router.pathmatch]` (08)

| Key | Type | Default |
|---|---|---|
| `router.pathmatch.enabled` | bool | `true` |
| `router.pathmatch.max_hits` | int | `10` |
| `router.pathmatch.max_ambiguous_candidates` | int | `10` |
| `router.pathmatch.ambiguous_max` | int | `5` |
| `router.pathmatch.scan_max_chars` | int | `262144` |
| `router.pathmatch.max_mentions` | int | `500` |
| `router.pathmatch.fold_fallback` | bool | `true` |
| `router.pathmatch.extra_framework_markers` | `list[str]` | `[]` |
| `router.pathmatch.extra_test_patterns` | `list[Glob]` | `[]` |

`[router.thresholds.<profile>]`: keyed by the judge's **threshold profile** (07 §4.9, D-07-6), optionally model-qualified: `[router.thresholds.jev]` or `[router.thresholds."jev:jev-1.13.0"]`.

```python
class Thresholds(BaseModel):                 # all Prob; jev defaults
    walk: Prob = 0.35
    walk_guard: Prob = 0.50                  # dead-end guard needs_context minimum (09)
    final: Prob = 0.60
    path_hit_floor: Prob = 0.20
    cap_use: Prob = 0.60
    cap_skip: Prob = 0.15
    needs_context: Prob = 0.25
    continuity_min_conf: Prob = 0.60
    expand: Prob = 0.30

class RouterConfig(BaseModel):
    ...
    thresholds: dict[str, PartialThresholds] = {}   # user tables; partial tables allowed
```

Profile keys must match `^(jev|systemone-local|llm|fixture)(:[A-Za-z0-9._-]+)?$`. Other keys are warnings (`unknown-profile`). Partial tables are allowed. 07 merges them over its built-in table for the profile (07 §4.9).

### 3.6 `[lease]` (10)

| Key | Type | Default | Validation |
|---|---|---|---|
| `lease.enabled` | bool | `true` | |
| `lease.idle_minutes` | int | `45` | 1–1,440 |
| `lease.lock_timeout_ms` | int | `250` | 10–2,000 |
| `lease.max_files` | int | `500` | ≥ 10 |
| `lease.max_request_chars` | int | `2000` | 200–8,000 |

### 3.7 `[judge]` (07)

| Key | Type | Default | Validation |
|---|---|---|---|
| `judge.backend` | `Literal["jev","systemone-local","llm","null","fixture"]` | `"jev"` | §4.4 purpose rules |
| `judge.provider` | `Literal["typesafe","openrouter","vercel"]` | `"typesafe"` | `jev` backend only; no `cloudflare` in v1 (07 D-07-9) |
| `judge.model` | str | `"jev-1.13.0"` | non-empty; an alias (`*latest*`, `*preview*`) → soft-limit warning (pin versions; aliases move when TypeSafe ships a release) |
| `judge.timeout_ms` | `int \| None` | `None` → per backend: jev 1200, systemone-local 3000, llm 20000 | 100–60,000; per attempt |
| `judge.min_request_ms` | int | `150` | 0–5,000 |
| `judge.max_concurrency` | int | `8` | 1–128; per process (07 D-07-8) |
| `judge.max_questions_per_request` | int | `40` | 5–255 (Choice cap) |
| `judge.max_request_tokens` | int | `8000` | 1,000–64,000 (`jev-1.13` context: 64k per request, 32k for state + longest question) |
| `judge.retry` | bool | `true` | retry-once policy (spec §13.2) |
| `judge.breaker.failures` | int | `3` | 1–50 consecutive failed batches |
| `judge.breaker.cooldown_s` | int | `60` | 1–3,600 |
| `judge.breaker.auth_cooldown_s` | int | `600` | 1–86,400 |
| `judge.price_per_mtok_input` | float | `0.042` | ≥ 0, USD |
| `judge.price_per_mtok_output` | float | `0.0` | ≥ 0 |
| `judge.base_url`, `judge.path` | `str \| None` | `None` | override provider defaults; no userinfo, no secret-looking query params |
| `judge.local.base_url` | str | `"http://127.0.0.1:8080"` | `systemone-local` |
| `judge.llm.base_url`, `judge.llm.model`, `judge.llm.key_env` | `str \| None` | `None` | required when `backend = "llm"`; `key_env` is an env var **name** `^[A-Z_][A-Z0-9_]*$` |
| `judge.llm.allow_routing` | bool | `false` | |
| `judge.fixture.mode` | `Literal["replay","append","rewrite"]` (07 §4.10) | `"replay"` | `SURF_FIXTURE_MODE` overrides |
| `judge.fixture.miss` | `Literal["fail","null"]` | `"fail"` | 16 sets it from `eval.fixture_miss` |
| `judge.fixture.simulate_latency` | bool | `true` | |
| `judge.fixture.path` | str | `".surf/cache/eval-fixtures/"` | 07 §3.5; 16 passes explicit `bench/` paths |
| `judge.fixture.inner` | str | `"jev"` | live backend used by `append`/`rewrite` |

Breaker state lives in `.surf/cache/judge_state.json` (07 §4.7). It isn't configurable. Keys named `api_key`, `key`, `token`, `secret` or `password` anywhere under `[judge]` are an **error** (D-13-3), and the value is never echoed.

### 3.8 `[privacy]` (14)

| Key | Type | Default | Validation | Scope |
|---|---|---|---|---|
| `privacy.redact_prompt` | bool | `true` | `false` → `doctor` warning | R |
| `privacy.log_prompt_text` | bool | `false` | | R |
| `privacy.redact_patterns` | `list[{name: str, regex: Regex}]` | `[]` | `name` `^[a-z0-9_-]{1,32}$` (→ `[REDACTED:name]`); regex must not match `""` | I and R (§4.5) |
| `privacy.redact_emails` | bool | `true` | | I and R |
| `privacy.entropy_min_len` | int | `24` | 12–128 | I and R |
| `privacy.entropy_min_bits` | float | `3.5` | 1.0–6.0 bits/char | I and R |
| `privacy.injection_filter` | bool | `true` | changes card hashes → full rebuild | I |

### 3.9 `[log]` (15) and `[explain]` (15)

| Key | Type | Default | Validation |
|---|---|---|---|
| `log.enabled` | bool | `true` | |
| `log.max_bytes` | int | `10_000_000` | ≥ 64 KiB |
| `log.max_files` | int | `5` | 1–50 (total, including the current file) |
| `log.record_max_bytes` | int | `16384` | 1 KiB–1 MiB |
| `log.lock_timeout_ms` | int | `50` | 1–1,000 |
| `log.level` | `Literal["debug","info","warning","error"]` | `"warning"` | stderr logging level (12) |
| `explain.max_children` | int | `8` | 1–100 |

### 3.10 `[note]` (11)

| Key | Type | Default | Validation |
|---|---|---|---|
| `note.max_lines` | int | `15` | 4–30 |
| `note.max_line_chars` | int | `160` | 60–400 |
| `note.max_chars` | int | `2000` | 200–8,000 |
| `note.max_migrations` | int | `2` | 0–5 |
| `note.show_skip` | bool | `true` | |
| `note.ascii_only` | bool | `false` | |
| `note.root_hint` | bool | `true` | |

### 3.11 `[delivery]` (12)

| Key | Type | Default | Validation |
|---|---|---|---|
| `delivery.control_commands` | bool | `true` | |
| `delivery.claude_code.prompt_timeout_s` | int | `10` | 3–60 |
| `delivery.claude_code.session_timeout_s` | int | `5` | 1–60; `≥ refresh.session_start_wait_ms/1000 + 1` |
| `delivery.claude_code.stop_hook` | bool | `false` | |
| `delivery.claude_code.transcript_max_bytes` | int | `1_048_576` | 4 KiB–16 MiB |
| `delivery.mcp.host` | str | `"127.0.0.1"` | |
| `delivery.mcp.port` | int | `8765` | 1–65,535 |
| `delivery.instructions.files` | `list[str] \| None` | `None` = auto (12 §4.6) | repo-relative |

### 3.12 `[refresh]` (06)

| Key | Type | Default | Validation |
|---|---|---|---|
| `refresh.git_hooks` | bool | `true` | |
| `refresh.hooks` | `list[str]` | `["post-commit","post-merge","post-checkout","post-rewrite"]` | subset of those four |
| `refresh.session_start_wait_ms` | int | `3000` | 0–30,000 |
| `refresh.check_interval_s` | int | `60` | 0–3,600 |
| `refresh.max_loops` | int | `3` | 1–10 |
| `refresh.foreground_lock_wait_s` | int | `30` | 0–600 |
| `refresh.background_nice` | int | `10` | 0–19 |

### 3.13 `[store]` (05)

| Key | Type | Default |
|---|---|---|
| `store.card_lru` | int | `4096` |
| `store.open_rebuild_timeout_ms` | int | `1500` |

### 3.14 `[eval]` (16)

| Key | Type | Default |
|---|---|---|
| `eval.seed` | int | `1729` |
| `eval.split_seed` | int | `20260923` |
| `eval.bootstrap_n` | int | `1000` |
| `eval.ci_level` | Prob | `0.95` |
| `eval.dir_credit_max_files` | int | `8` |
| `eval.category_weights` | `dict[str, float]` | spec §17.2 shares |
| `eval.concurrency_live` | int | `1` |
| `eval.concurrency_fixture` | int | `8` |
| `eval.fixture_miss` | `Literal["fail","null"]` | `"fail"` |
| `eval.regression_tolerance` | Prob | `0.03` |
| `eval.max_judge_failure_rate` | Prob | `0.05` |
| `eval.price_per_mtok` | `dict[str, float]` | `{jev = 0.042}` |
| `eval.a0_max_cards` | int | `5000` |

Dataset and wordings paths are fixed by spec §9.1 and 16 §3.6 (`.surf/eval/{dev,test,wordings}.yaml`), not config.

### 3.15 `surf init` template and the full defaults file

`surf init` writes a **short** file, so later changes to defaults apply automatically:

```toml
version = 1

[project]
# descriptor = "TypeScript + Supabase e-commerce backend"   # derived: "<auto value>"

[index]
exclude = []
commit_catalog = false        # true = maintain a committed baseline with `surf index --baseline`

[index.schema]
sources = ["supabase/migrations/*.sql"]    # detected; delete to auto-detect
dialect = "postgres"

[capabilities]
live_mcp = []                 # servers listed live at index time (spawns them; opt-in per server)

[capabilities.describe]
# linear = "Issue tracker: tickets, projects, cycles"

[judge]
backend = "jev"               # jev | systemone-local | null
provider = "typesafe"         # key from env: TYPESAFE_API_KEY
model = "jev-1.13.0"

# All other keys and defaults: `surf config show --defaults --toml`
```

`surf config show --defaults --toml` prints every key in §3 with its default. That output is generated from the models and checked in as `docs/config-defaults.toml`. A test diffs it, so this doc, the file and the code can't drift silently (§8).

## 4. Behavior

### 4.1 Discovery

| Layer | Path | Committed? | Notes |
|---|---|---|---|
| `default` | built-in | — | model defaults |
| `user` | `$SURF_USER_CONFIG`, else `$XDG_CONFIG_HOME/surf/config.toml`, else `~/.config/surf/config.toml`; Windows `%APPDATA%\surf\config.toml` | no | personal preferences (provider, log level, note style) |
| `project` | `<root>/.surf/config.toml` | yes | team settings; the only non-default source for the index view |
| `project-local` | `<root>/.surf/config.local.toml` | no (in `.surf/.gitignore`) | personal per-repo overrides |
| `extra` | `--config PATH` or `SURF_CONFIG` | n/a | CI and eval experiments |
| `env` | `SURF_<SECTION>__<KEY>…` | n/a | §4.3 |
| `cli` | command flags (`--judge`, `--no-lease`, …) | n/a | mapped to dotted keys in `cli.py` |

`<root>` comes from 12 §2.2. A missing file is skipped silently. An existing but unreadable file is an error, because the user expects it to apply.

### 4.2 Precedence and merge

`default < user < project < project-local < extra < env < cli`.

- Tables are deep-merged key by key.
- **Arrays replace**; they never concatenate.
- Dict-valued keys (`capabilities.describe`, `router.thresholds`, `router.wording.continuity_options`, `router.expand.kind_factors`, `eval.category_weights`, `eval.price_per_mtok`) merge per entry.
- Deprecated aliases are rewritten before merging (`router.deadline_ms` → `router.route_deadline_ms`) in the layer where they appear. Precedence is then by layer as usual.
- `origins` records the winning layer per non-default leaf.

### 4.3 Environment variables

| Variable | Meaning |
|---|---|
| `SURF_<SEG>__<SEG>…` | override one leaf, e.g. `SURF_JUDGE__BACKEND=null`, `SURF_ROUTER__THRESHOLDS__JEV__FINAL=0.55`. Only variables containing `__` are config overrides. Segments match case-insensitively, and `_` matches `-` (so `SURF_ROUTER__THRESHOLDS__SYSTEMONE_LOCAL__WALK` works). A model-qualified profile (`jev:jev-1.13.0`) can't be expressed; use a file. The value is parsed as a TOML literal (`tomllib.loads("v = " + raw)`), falling back to a plain string. |
| `SURF_ROOT`, `SURF_CONFIG`, `SURF_USER_CONFIG` | root, extra file, user file |
| `SURF_SESSION` | CLI session id (spec §12.5) |
| `SURF_DISABLE=1` | disable routing for this process (12 §4.2) |
| `SURF_DEBUG=1` | stderr debug logging, including in the hook |
| `SURF_SKIP_HOOKS=1` | git hooks no-op (06 §4.5) |
| `SURF_FIXTURE_MODE` | overrides `judge.fixture.mode` (07) |
| `SURF_MCP_TOKEN` | Bearer token for non-loopback MCP HTTP (12) |
| `TYPESAFE_API_KEY`, `OPENROUTER_API_KEY`, `AI_GATEWAY_API_KEY`, `SYSTEMONE_LOCAL_API_KEY` | judge credentials; names and provider mapping owned by 07 §3.2 |

An override of an unknown path is an `unknown-key` warning.

### 4.4 Validation

1. **Per-layer parse**: `tomllib` (BOM stripped). A syntax error is a `ConfigError` naming the file and tomllib's line/column.
2. **Merged validation**: pydantic over the merged dict. All errors are collected, and each is reported as `dotted.key (layer): message`.
3. **Unknown keys**: a walk over `model_extra` at every level produces `unknown-key` warnings with a `difflib` suggestion (`router.tresholds → router.thresholds?`). They're warnings, so a config written for a newer surf keeps working with an older one. `surf config validate --strict` and `doctor --strict` turn them into errors.
4. `version` > `CONFIG_VERSION` → error "config requires a newer surf"; missing → 1.

Cross-field rules (`model_validator(mode="after")`):

| Rule | Violation |
|---|---|
| Each thresholds table: `cap_skip < cap_use`; `path_hit_floor ≤ final` | error |
| Each thresholds table: `walk ≤ final` | warning (spec principle 3) |
| `router.walk_deadline_ms < router.route_deadline_ms` | error |
| effective `judge.timeout_ms ≤ router.route_deadline_ms` | error |
| `router.chunk_size ≤ judge.max_questions_per_request` | error |
| `router.final_max_request_tokens ≤ judge.max_request_tokens` | error |
| `router.beam_min ≤ router.beam_max` | error |
| `router.flatten_at ≤ router.max_candidates` | warning |
| `router.chunk_size > 40` or `router.max_candidates > 40` | warning (spec principle 5) |
| `delivery.claude_code.prompt_timeout_s * 1000 ≥ router.route_deadline_ms + 2000` | error otherwise (Claude Code would kill the hook before the deadline fires) |
| `delivery.claude_code.session_timeout_s * 1000 ≥ refresh.session_start_wait_ms + 1000` | error otherwise |
| `judge.backend == "llm"` requires `judge.llm.base_url`, `model`, `key_env` | error |
| `judge.backend == "llm"` with `purpose="runtime"` and not `judge.llm.allow_routing` | error (spec §13.2: eval baseline) |
| `judge.backend == "fixture"` with `purpose="runtime"` | warning (tests only) |
| `privacy.redact_patterns[*].regex` fails to compile or matches `""` | error |
| `router.expand.enabled_kinds` ⊄ `kind_factors` keys | error |
| `router.max_pointers > 12` | soft-limit warning (spec §1.2) |

Values outside ranges are errors. Nothing is silently clamped.

| Context | On `ConfigError` |
|---|---|
| CLI commands | all problems to stderr; exit 5 |
| Claude Code hook | fail open: no note, exit 0; one `status=error, error="config"` decision record per process |
| MCP server | tool result `status=error` naming `surf config validate`; reload attempted when files change |
| `surf doctor` | `config.parse` fails with the problem list |

### 4.5 Index view and fingerprints

**Index view.** `index_view` = defaults + the `project` layer, plus `capabilities.user_level` from the merged config. The indexer, refresh and `--check` (01–06) receive only this view. So keys that shape the catalog (all `I`-scoped keys in §3) can only come from the committed `.surf/config.toml`, and every machine and CI builds the same thing from the same commit. `I`-scoped keys set in other layers get an `ignored-index-key` warning.

Keys scoped **I and R** (`privacy.redact_patterns`, `redact_emails`, `entropy_min_*`) are used twice: the index-time sanitizer (14) reads them from the index view, and query-time redaction reads them from the merged config. A developer's personal extra pattern therefore redacts their prompts without changing the catalog.

**User-level capabilities.** `capabilities.user_level` is per user. Cards from user-level configs, and from untracked files, go to `.surf/cache/overlay.jsonl` and never to the committed catalog (00 §4.1, 01 D-01-1).

**Fingerprints.** `fingerprints[section] = "sha256:" + sha256(canonical_json(index_view subset))`, where canonical JSON has sorted keys, compact separators and floats rounded to 6 dp. Sections:

| Fingerprint key | Covers | Invalidates (06 §4.2) |
|---|---|---|
| `discovery` | `index.exclude`, `include`, `default_excludes`, `include_untracked`, `max_file_bytes`, `respect_linguist`, `exclude_generated` | everything |
| `schema` | `index.schema.*` | schema facts, table cards, schema edges |
| `cochange` | `index.cochange.*` | co-change edges, `changes_with`, `coupled_dirs` |
| `schema_refs` | `index.schema_refs.*` except `engine` | `schema_ref` edges |
| `cards` | `index.cards.*`, `privacy.injection_filter`, the index-view privacy sanitizer keys | card text and hashes |
| `capabilities` | `capabilities.*` except `user_level` | capability cards |
| `all` | hash of the section hashes | `meta.config_fingerprint` (05) |

`project.descriptor` isn't fingerprinted. A configured descriptor overrides the derived one at query time (09).

### 4.6 Template generation (`surf init`)

`config_write.render_initial(detected) -> str` produces §3.15's short file. Detected `sources`/`dialect` are written only when detection is confident, and are otherwise left as comments. `--live-mcp NAME` values go into `live_mcp`. The derived descriptor is shown as a comment. The output must parse and validate; a test checks this for every fixture repo.

### 4.7 Comment-preserving edits

After init, surf only ever edits entries under `[capabilities.describe]`. `config_write.set_describe(path, name, text)`:

1. Find the `[capabilities.describe]` header (regex; whitespace and trailing comments allowed).
2. Render the key bare if it matches `^[A-Za-z0-9_-]+$`, otherwise as a basic string. The value is a basic string with `\`, `"` and control characters escaped.
3. Replace an existing entry line for that key inside the section, or insert after the section's last non-blank line, or append a new section at the end of the file.
4. Re-parse with `tomllib` and validate. On failure, don't write.
5. Write atomically, preserving line endings.

An inline-table form (`describe = { … }` under `[capabilities]`) is detected, and the edit is refused with instructions.

### 4.8 Hook fast path

The hook's stdlib-only path (12 §4.4.3) must not import pydantic. `raw_peek(root, dotted)` reads the project and project-local files with `tomllib` and returns the raw value or `None`. It's used only for `router.skip.ack_words`, `router.skip.ack_max_words`, `delivery.control_commands`, `lease.enabled` and `lease.idle_minutes`. A value of the wrong type falls back to the default; full validation happens on the slow path.

## 5. Configuration

This doc *is* the configuration registry. Every component doc's §5 must list only keys that appear in §3, with the same name, type and default. `tests/test_config_docs.py` parses the §5 tables of all design docs and compares them with `SurfConfig.model_json_schema()` (normalizing the router doc's unprefixed key column to `router.`). Any drift fails CI.

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| `.surf/` exists but no `config.toml` | defaults + user layers; `doctor` warns |
| Empty config file | defaults |
| TOML syntax error in `config.local.toml` | error naming the file (CLI exit 5; hook fail-open) |
| `SURF_JUDGE__BACKEND=nul` (typo) | validation error naming the env layer |
| `SURF_ROUTER__MAX_POINTERS=abc` | string → pydantic int error |
| Both `router.deadline_ms` and `route_deadline_ms` in one file | `route_deadline_ms` wins; `deprecated` warning |
| No thresholds table for the active profile | 07's built-in table; if uncalibrated, 09 marks routes `UNCALIBRATED` (low confidence) |
| User config sets `index.exclude` | ignored for indexing, with a warning (§4.5) |
| User config adds a `privacy.redact_patterns` entry | applies to that user's prompts only; the catalog is unchanged |
| `capabilities.describe` for a server that no longer exists | warning at index time; kept |
| `judge.api_key = "…"` in any file | error; value never echoed |
| Config file world-writable | `doctor` warning (it controls what's sent to the judge) |
| `version = 2` | error "requires newer surf" |
| Backslashes in a glob | error with a hint to use `/` |

## 7. Performance budget

| Operation | Budget |
|---|---|
| `load_config` (≤ 4 files, after imports) | ≤ 10 ms |
| `raw_peek` | ≤ 3 ms |
| fingerprints | ≤ 1 ms |

## 8. Test plan

- **Defaults:** `surf config show --defaults --toml` equals the checked-in `docs/config-defaults.toml`, and loading that file gives `SurfConfig()`.
- **Precedence matrix:** one key per layer; `origins` correct; arrays replace; dicts merge per entry; deprecated alias in each layer.
- **Env parsing:** bools, ints, floats, lists, strings, hyphenated profile names, unknown paths.
- **Validation:** one pass/fail test per cross-field rule; multiple errors reported together; secret-key rejection without echo; unknown-key suggestions; `purpose` rules for `llm`/`fixture`.
- **Index view:** `I` keys in user/local/env layers don't change `index_view` or fingerprints (property test over random non-project edits); each section's fingerprint changes only when its keys change.
- **Writers:** `set_describe` on files with comments, CRLF, missing section, existing key, inline-table form (refused), odd server names; the result always re-parses.
- **Docs drift test** (§5).
- **Fail-open:** hook and MCP with an invalid config → no exception, `status=error`.

## 9. Acceptance criteria

1. Every key referenced in design docs 01–16 exists in `SurfConfig` with the same name and default (drift test green).
2. `surf index --check` gives identical results with and without a user config that sets arbitrary keys (determinism, §4.5).
3. All validation errors in a run are reported together.
4. A config with only unknown extra keys loads with warnings and routes normally.
5. Build plan: index subset by P1.11 (Phase 1); full schema by Phase 4 exit.

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-13-1 | §16: `router.deadline_ms = 3000`; §11.5: `deadline_ms = 2000` is the walk budget | `router.walk_deadline_ms = 2000`, `router.route_deadline_ms = 3000`; `router.deadline_ms` is a deprecated alias of the latter | One name, two budgets (00 §5, 09) |
| D-13-2 | §16: `judge.timeout_ms` only; §13.3 constants | Adds `judge.retry`, `min_request_ms`, `max_questions_per_request`, `max_request_tokens` and `judge.breaker.{failures, cooldown_s, auth_cooldown_s}` (07) | Make §13.3's constants tunable; `timeout_ms` is per attempt |
| D-13-3 | §16: key "from env" (comment) | Secret-like keys in config files are rejected | Config is committed |
| D-13-4 | not specified | Index view (defaults + project file only) and per-section fingerprints | `--check` and incremental refresh must be reproducible across machines |
| D-13-5 | §16 sample only | Adds every key the spec uses as a constant and every key introduced by 01–16 (§3) | Spec principle 8; one registry |
| D-13-6 | §9.5 / §16: `index.commit_catalog = true` | Default `false`; committed baseline via `surf index --baseline` (05 D-05-1, 06 D-06-1; pending user decision Q-06-1) | Hook-driven refresh would otherwise dirty the tree after every commit |
| D-13-7 | §16: single `.surf/config.toml` | Layered discovery: user, project, project-local, extra, env, CLI | Personal settings shouldn't require editing a committed file |
| D-13-8 | n/a (design docs disagreed) | Resolved names: `capabilities.user_level` (not `user_configs`, 14); `router.expand.enabled_kinds` (not `router.expand_kinds`, 16); no `tables_per_anchor` key (09 dropped its own; tables per anchor are bounded by 04's `router.expand.max_per_anchor`); `router.wording.*` (09); `judge.fixture.{mode,miss,simulate_latency,path}` (07, not `fixture.dir`); `log.max_files` (15); `privacy.redact_patterns` (14); exclude-author substrings (04) | One name per key; the owning component's name wins, and where two owners overlap, the more structured name wins |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-13-1 | Unknown keys: errors (catch typos) or warnings (forward compatibility)? | Warnings; `--strict` makes them errors | User feedback on silent typos |
| Q-13-2 | Should the committed `project.descriptor` be fingerprinted (it's only used at query time)? | No | — |
| Q-13-3 | `judge.price_per_mtok_input` (07) and `eval.price_per_mtok` (16) overlap. Should eval derive jev's price from `judge.*`? | Yes: `eval.price_per_mtok` only for non-active backends | 07/16 owners |
| Q-13-4 | Should `router.max_pointers` above 12 be allowed at all? | Up to 30 with a warning | Precision eval |
| Q-13-5 | Should a user be able to exclude a huge local-only dir without touching the committed config? | Yes, through `.git/info/exclude` (discovery honors git's exclude sources), not surf config | User requests |
| Q-13-6 | Hook kill switch naming: 06 uses `SURF_SKIP_HOOKS`, 14's hook sketch used `SURF_HOOK_DISABLE` | `SURF_SKIP_HOOKS` (06 owns the hook block) | Resolved: 14 §4.6 now reproduces 06's block |
