# 13 · Configuration

**Status:** draft for review
**Spec sections:** §16 (all); keys referenced in §7.1, §7.2, §8.2, §8.3, §8.5, §9.5, §11.2–§11.8, §12.4, §13.2–§13.3, §14.3, §15.4, §18.2, §19.3
**Depends on:** 00-foundations. Every other design doc consumes keys defined here; each doc's §5 lists the subset it reads, and this doc is the single source of names, types and defaults.
**Code:** `surf/config.py` (models, loader, validation), `surf/config_write.py` (template generation and comment-preserving edits used by `surf init`)

---

## 1. Purpose and scope

One typed, validated configuration object (`SurfConfig`) built from layered TOML files, environment variables and CLI flags. It must be:

- **Complete:** every tunable in the spec and design docs has exactly one key here.
- **Deterministic where it matters:** keys that change the committed catalog come only from the committed project file, so `surf index --check` gives the same answer on every machine.
- **Fail-open at query time:** a bad config never breaks the host harness (hook/MCP → `status=error`, no note). It fails loudly in CLI commands (exit 5).

| In scope | Out of scope |
|---|---|
| Schema (pydantic v2), defaults, validation, cross-field rules | Secrets in config files (rejected; keys come from env) |
| File discovery and precedence, env overrides, CLI overrides | Remote/shared config services |
| Index fingerprint for rebuild detection | Hot-reload semantics beyond "reload when mtime changes" (12 §4.5.2) |
| Template for `surf init`; comment-preserving writes of `capabilities.describe` | A general TOML writer |

## 2. Interfaces

```python
# surf/config.py
CONFIG_VERSION: Final = 1

class LoadedConfig(BaseModel):
    config: SurfConfig
    origins: dict[str, ConfigLayer]          # dotted key → layer that set it (only non-default keys)
    warnings: list[ConfigWarning]            # unknown keys, ignored index-scoped keys, deprecated keys
    index_fingerprint: str                   # "sha256:<hex>" over index-scoped keys (§4.5)
    files: list[Path]                        # files actually read, in precedence order

ConfigLayer = Literal["default", "user", "project", "project-local", "extra", "env", "cli"]

class ConfigWarning(BaseModel):
    code: Literal["unknown-key", "ignored-index-key", "deprecated", "unknown-backend", "clamped"]
    key: str; layer: ConfigLayer; message: str; suggestion: str | None = None

class ConfigError(Exception):                # raised by load_config; carries every problem, not only the first
    problems: list[tuple[str, ConfigLayer, str]]   # (dotted key, layer, message)

def load_config(root: Path | None, *, env: Mapping[str, str] = os.environ,
                cli: Mapping[str, Any] | None = None,
                user_config: Path | None | Literal["auto"] = "auto") -> LoadedConfig: ...

def thresholds_for(cfg: SurfConfig, backend: str) -> Thresholds: ...
def raw_peek(root: Path, dotted: str) -> Any | None: ...     # tomllib only, no pydantic (hook fast path, 12 §4.4.3)
def api_key_env(cfg: JudgeConfig) -> str | None: ...          # env var name for the active provider
```

Callers: `surf/runtime.py` (12) loads once per process; the MCP server reloads when the files change; `surf config show|validate|path` exposes it. Per 00 §6, no module reads config on import. Components receive their sub-model (`cfg.router`, `cfg.lease`, …) as an argument.

## 3. Data structures: the complete schema

Conventions: all models `frozen=True`. Key names are `snake_case`. Durations carry their unit in the name (`_ms`, `_s`, `_minutes`, `_days`, `_months`). Probabilities are `Prob = Annotated[float, Field(ge=0, le=1)]`. "Owner" names the design doc that defines the key's semantics.

### 3.1 Root

```python
class SurfConfig(BaseModel):
    model_config = ConfigDict(frozen=True, extra="allow")   # extras → warnings, see §4.4
    version: int = CONFIG_VERSION
    project: ProjectConfig = ProjectConfig()
    index: IndexConfig = IndexConfig()
    capabilities: CapabilitiesConfig = CapabilitiesConfig()
    router: RouterConfig = RouterConfig()
    lease: LeaseConfig = LeaseConfig()
    judge: JudgeConfig = JudgeConfig()
    privacy: PrivacyConfig = PrivacyConfig()
    log: LogConfig = LogConfig()
    note: NoteConfig = NoteConfig()
    delivery: DeliveryConfig = DeliveryConfig()
    refresh: RefreshConfig = RefreshConfig()
    eval: EvalConfig = EvalConfig()
```

### 3.2 `[project]`

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `project.descriptor` | `str \| None` | `None` → auto-derived (09) | single line, 1–160 chars, sanitized like card text | 09 |

### 3.3 `[index]` (index-scoped, §4.5)

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `index.exclude` | `list[Glob]` | `[]` | gitignore-style globs; no absolute paths, no `..` | 01 |
| `index.include` | `list[Glob]` | `[]` | re-includes paths hidden by the **non-secret** default excludes; can't re-include secret-like files | 01 |
| `index.commit_catalog` | `bool` | `true` | | 05 |
| `index.max_file_bytes` | `int` | `1_000_000` | 1 KB–100 MB | 01 |
| `index.follow_symlinks` | `bool` | `false` | | 01 |

`[index.cards]`: card budgets (spec §7.2–§7.7).

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `index.cards.file_tokens` | int | `60` | 20–200 | 02 |
| `index.cards.dir_tokens` | int | `150` | 50–400 | 02 |
| `index.cards.cap_tokens` | int | `120` | 40–400 | 02 |
| `index.cards.file_tables` | int | `3` | 0–10 | 02 |
| `index.cards.changes_with` | int | `3` | 0–10 | 02 |
| `index.cards.doc_headings` | int | `8` | 0–20 | 02 |
| `index.cards.dir_child_dirs` | int | `12` | 0–40 | 02 |
| `index.cards.dir_child_files` | int | `15` | 0–40 | 02 |
| `index.cards.dir_tables` | int | `5` | 0–20 | 02 |
| `index.cards.dir_doc_titles` | int | `6` | 0–20 | 02 |
| `index.cards.coupled_dirs` | int | `3` | 0–10; `0` = ablation A5 | 02 |
| `index.cards.description_chars` | int | `200` | 40–500 (skills/agents/commands fallback paragraph) | 02 |
| `index.cards.table_columns` | int | `6` | 0–30 | 03 |

`[index.schema]`

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `index.schema.sources` | `list[Glob] \| None` | `None` = auto-detect (spec §7.5 table) | | 03 |
| `index.schema.formats` | `list[Literal["sql","prisma","rails","alembic","drizzle"]] \| None` | `None` = auto | | 03 |
| `index.schema.dialect` | `str` | `"postgres"` | a sqlglot dialect name (checked lazily at index time) | 03 |
| `index.schema.default_schema` | `str` | `"public"` | tables in it get unqualified ids (00 §2.1) | 03 |

`[index.cochange]`

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `index.cochange.enabled` | bool | `true` | | 04 |
| `index.cochange.window_months` | int | `24` | 1–120 | 04 |
| `index.cochange.max_commits` | int | `5000` | 100–100,000 | 04 |
| `index.cochange.max_files_per_commit` | int | `30` | 2–1,000 | 04 |
| `index.cochange.half_life_days` | float | `180` | > 0 | 04 |
| `index.cochange.min_shared_commits` | int | `2` | ≥ 1 | 04 |
| `index.cochange.min_coupling` | Prob | `0.15` | | 04 |
| `index.cochange.top_partners` | int | `10` | 1–50 | 04 |
| `index.cochange.follow_renames` | bool | `true` | | 04 |
| `index.cochange.exclude_authors` | `list[str]` | `["dependabot[bot]", "renovate[bot]", "github-actions[bot]"]` | case-insensitive substring match on author name or email | 04 |
| `index.cochange.exclude_message_patterns` | `list[Regex]` | `["^(chore\|style\|format)\\b", "\\bprettier\\b", "\\blint\\b"]` | Python regex, case-insensitive, matched against the subject line | 04 |
| `index.cochange.dir_top_k` | int | `5` | 1–20 | 04 |
| `index.cochange.dir_min_coupling` | Prob | `0.15` | | 04 |

`[index.schema_refs]`

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `index.schema_refs.enabled` | bool | `true` | | 04 |
| `index.schema_refs.stopwords` | `list[str]` | `["data","items","logs","status","type","meta"]` | lowercase | 04 |
| `index.schema_refs.min_name_len` | int | `4` | 1–20 | 04 |
| `index.schema_refs.max_tables_per_file` | int | `20` | 1–200 | 04 |
| `index.schema_refs.max_files_per_table` | int | `200` | 1–10,000 | 04 |
| `index.schema_refs.search` | `Literal["auto","rg","python"]` | `"auto"` | | 04 |
| `index.schema_refs.variant_weights` | `dict[str, float]` | `{exact=1.0, quoted=1.5, singular=0.6, pascal=0.6, camel=0.6}` | keys exactly these five; values 0–3 | 04 |

### 3.4 `[capabilities]` (index-scoped except `user_level`, §4.5)

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `capabilities.live_mcp` | `list[str]` | `[]` | server names; unknown names warn at index time | 02 |
| `capabilities.live_timeout_ms` | int | `10_000` | 1,000–60,000 | 02 |
| `capabilities.describe` | `dict[str, str]` | `{}` | value: single line, 1–200 chars; key: server name | 02 |
| `capabilities.exclude` | `list[str]` | `[]` | capability ids or globs (`mcp:internal-*`). `mcp:surf` is always excluded (12 §4.5.3) | 02 |
| `capabilities.user_level` | bool | `false` | opt-in to read user-level harness configs (`~/.claude.json`, `~/.codex/config.toml`, …); **per-user, never index-scoped** (§4.5) | 01, 02 |

### 3.5 `[router]`

The spec's single `deadline_ms` is split in two (D-13-1): `walk_deadline_ms` (spec §11.5 table: "walk time budget", 2,000) and `route_deadline_ms` (spec §11.9 / §13.3: "3 s end to end").

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `router.small_repo_cutoff` | int | `60` | 0–1,000 | 09 |
| `router.flatten_at` | int | `40` | 1–200 | 09 |
| `router.flatten_factor` | Prob | `0.9` | | 09 |
| `router.chunk_size` | int | `40` | 5–64 (> 40 warns: spec principle 5) | 09 |
| `router.tau_walk` | — | — | *not a key*: lives in `router.thresholds.<backend>.walk` | |
| `router.beam_max` | int | `6` | 1–40 | 09 |
| `router.beam_min` | int | `1` | 0–`beam_max` | 09 |
| `router.dead_end_min_needs_context` | Prob | `0.5` | spec §11.5 pseudocode constant | 09 |
| `router.max_depth` | int | `4` | 1–10 | 09 |
| `router.max_candidates` | int | `40` | 5–64 | 09 |
| `router.max_pointers` | int | `12` | 1–30 | 09, 10 |
| `router.walk_deadline_ms` | int | `2000` | 100–30,000; `< route_deadline_ms` | 09 |
| `router.route_deadline_ms` | int | `3000` | 200–60,000 | 09, 12 |
| `router.speculative_walk` | bool | `true` | | 09 |
| `router.max_caps_per_request` | int | `40` | 5–64 | 09 |
| `router.request_max_tokens` | int | `1500` | 200–8,000 (head+tail truncation) | 09, 14 |
| `router.max_path_hits` | int | `10` | 0–50 | 08 |
| `router.basename_hit_strength` | Prob | `0.8` | | 08 |
| `router.dir_collapse_min_selected` | int | `4` | 2–20; `0` disables | 09 |
| `router.dir_collapse_max_dir_files` | int | `8` | 1–100 | 09 |
| `router.type_diversity` | bool | `true` | | 09 |
| `router.wordings` | `dict[str, str]` | `{}` | keys ⊆ `walk, final, capability, continuity, needs_context, small_repo`; overrides built-in wordings; `{card}` placeholder required where applicable | 09, 16 |

`[router.skip]`

| Key | Type | Default | Owner |
|---|---|---|---|
| `router.skip.ack_words` | `list[str]` | `["ok","okay","thanks","thank you","continue","go ahead","yes","lgtm","sounds good","do it"]` | 09, 12 |
| `router.skip.ack_max_words` | int (1–10) | `4` | 09 |

`[router.expand]` (spec §8.5)

| Key | Type | Default | Validation | Owner |
|---|---|---|---|---|
| `router.expand.depth` | int | `1` | must be `1` in v1 (spec §8.5); other values are an error | 04, 09 |
| `router.expand.kind_factors` | `dict[str, float]` | `{co_change=1.0, schema_ref=0.9, defined_in=0.8, fk=0.5, contains=0.6}` | keys ⊆ `EdgeKind` minus `alias`, `dir_coupling`; 0–1 | 04 |
| `router.expand.tables_per_anchor` | int | `3` | 0–10 | 09 |

`[router.thresholds.<backend>]`: per judge backend (spec §13.1). One table per backend name.

```python
class Thresholds(BaseModel):
    walk: Prob = 0.35
    final: Prob = 0.60
    path_hit_floor: Prob = 0.20
    cap_use: Prob = 0.60
    cap_skip: Prob = 0.15
    needs_context: Prob = 0.25
    continuity_min_conf: Prob = 0.60
    expand: Prob = 0.30

class RouterConfig(BaseModel):
    ...
    thresholds: dict[str, Thresholds] = {"jev": Thresholds()}
```

`thresholds_for(cfg, backend)`: return `thresholds[backend]` if present; otherwise `thresholds["jev"]` (or the built-in defaults) and a one-time `unknown-backend` warning. `doctor` reports `judge.thresholds` (12 §4.11). Partial tables are merged over the defaults field by field.

### 3.6 `[lease]` (owner 10)

| Key | Type | Default | Validation |
|---|---|---|---|
| `lease.enabled` | bool | `true` | |
| `lease.idle_minutes` | int | `45` | 1–1,440 |
| `lease.lock_timeout_ms` | int | `250` | 10–2,000 |
| `lease.max_files` | int | `500` | ≥ 10 |
| `lease.max_request_chars` | int | `2000` | 200–8,000 |

### 3.7 `[judge]` (owner 07)

The spec's `timeout_ms` is **per request**. Retries and the circuit breaker are separate keys (D-13-2).

| Key | Type | Default | Validation |
|---|---|---|---|
| `judge.backend` | `Literal["jev","systemone-local","llm","null","fixture"]` | `"jev"` | `llm` allowed only for `surf eval` (runtime → error); `fixture` only for tests/eval |
| `judge.provider` | `Literal["typesafe","openrouter","vercel","cloudflare"]` | `"typesafe"` | only for `backend = "jev"` |
| `judge.model` | str | `"jev-1.13.0"` | non-empty; pinned version string (a warning if it looks unpinned, e.g. `jev-latest`) |
| `judge.base_url` | `HttpUrl \| None` | `None` | **required** for `systemone-local`; optional override for gateways; no credentials in the URL (error) |
| `judge.api_key_env` | `str \| None` | `None` → provider default (§4.3) | an env var **name**, `^[A-Z_][A-Z0-9_]*$` |
| `judge.timeout_ms` | int | `1200` | 100–30,000; `≤ router.route_deadline_ms` |
| `judge.retries` | int | `1` | 0–3; a retry only starts if the remaining route deadline ≥ `timeout_ms` |
| `judge.retry_backoff_ms` | int | `100` | 0–2,000 |
| `judge.max_concurrency` | int | `16` | 1–128 |
| `judge.breaker.failure_threshold` | int | `3` | 1–50 consecutive failures |
| `judge.breaker.cooldown_s` | int | `60` | 1–3,600 |
| `judge.breaker.persist` | bool | `true` | state in `.surf/cache/breaker.json`, shared across hook processes |
| `judge.llm.provider` | str | `"anthropic"` | eval baseline only (spec §13.2) |
| `judge.llm.model` | `str \| None` | `None` (must be set to run A6) | |
| `judge.fixture.path` | str | `".surf/eval/fixtures/judge.jsonl"` | |
| `judge.fixture.mode` | `Literal["replay","record"]` | `"replay"` | |

A key named `api_key`, `token` or `secret` anywhere under `[judge]` is an **error**: "put secrets in the environment; set `judge.api_key_env` to the variable name" (D-13-3).

### 3.8 `[privacy]` (owner 14)

| Key | Type | Default | Validation |
|---|---|---|---|
| `privacy.redact_prompt` | bool | `true` | `false` warns in `doctor` |
| `privacy.log_prompt_text` | bool | `false` | |
| `privacy.redact_emails` | bool | `true` | |
| `privacy.entropy_min_len` | int | `24` | 12–128 (spec §19.3) |
| `privacy.extra_patterns` | `list[RedactPattern]` | `[]` | `{name: str, regex: Regex}`; `name` matches `^[a-z0-9_-]{1,32}$` (becomes `[REDACTED:name]`); regex ≤ 500 chars, compiles, doesn't match the empty string |

### 3.9 `[log]` (owner 15)

| Key | Type | Default | Validation |
|---|---|---|---|
| `log.decisions` | bool | `true` | |
| `log.max_bytes` | int | `10_485_760` | ≥ 64 KiB (spec §18.2: 10 MB) |
| `log.backups` | int | `5` | 0–50 (spec §18.2: × 5 files) |
| `log.level` | `Literal["debug","info","warning","error"]` | `"warning"` | stderr logging |

### 3.10 `[note]` (owner 11)

| Key | Type | Default | Validation |
|---|---|---|---|
| `note.max_lines` | int | `15` | 4–30 |
| `note.max_line_chars` | int | `160` | 60–400 |
| `note.max_chars` | int | `2000` | 200–8,000 |
| `note.show_skip` | bool | `true` | |
| `note.ascii_only` | bool | `false` | |
| `note.root_hint` | bool | `true` | |

### 3.11 `[delivery]` (owner 12)

| Key | Type | Default | Validation |
|---|---|---|---|
| `delivery.control_commands` | bool | `true` | |
| `delivery.claude_code.prompt_timeout_s` | int | `10` | 3–60; `≥ route_deadline_ms/1000 + 2` |
| `delivery.claude_code.session_timeout_s` | int | `5` | 1–60 |
| `delivery.claude_code.stop_hook` | bool | `false` | |
| `delivery.claude_code.transcript_max_bytes` | int | `1_048_576` | 4 KiB–16 MiB |
| `delivery.mcp.host` | str | `"127.0.0.1"` | |
| `delivery.mcp.port` | int | `8765` | 1–65,535 |
| `delivery.instructions.files` | `list[str] \| None` | `None` = auto (12 §4.6) | repo-relative paths |

### 3.12 `[refresh]` (owner 06)

| Key | Type | Default | Validation |
|---|---|---|---|
| `refresh.session_start_min_interval_s` | int | `60` | 0–3,600 |
| `refresh.background_timeout_s` | int | `300` | 10–3,600; a background refresh that runs longer is killed and logged |
| `refresh.foreground_timebox_ms` | int | `3000` | 500–60,000 (spec §10.1 "time-boxed to 3 s"; used by `surf refresh` without `--background`) |

### 3.13 `[eval]` (owner 16)

| Key | Type | Default |
|---|---|---|
| `eval.dev_set` | str | `".surf/eval/dev.yaml"` |
| `eval.test_set` | str | `".surf/eval/test.yaml"` |
| `eval.wordings_file` | str | `".surf/eval/wordings.yaml"` |
| `eval.bootstrap_resamples` | int | `1000` |
| `eval.seed` | int | `0` |
| `eval.regression_tolerance` | Prob | `0.03` (spec §17.7) |

### 3.14 Complete default file (generated by `surf init`)

```toml
version = 1

[project]
# descriptor = "TypeScript + Supabase e-commerce backend"   # auto-derived if omitted

[index]
exclude = []
include = []
commit_catalog = true
max_file_bytes = 1_000_000
follow_symlinks = false

[index.cards]
file_tokens = 60
dir_tokens = 150
cap_tokens = 120
file_tables = 3
changes_with = 3
doc_headings = 8
dir_child_dirs = 12
dir_child_files = 15
dir_tables = 5
dir_doc_titles = 6
coupled_dirs = 3
description_chars = 200
table_columns = 6

[index.schema]
# sources = ["supabase/migrations/*.sql"]   # auto-detected if omitted
# formats = ["sql"]
dialect = "postgres"
default_schema = "public"

[index.cochange]
enabled = true
window_months = 24
max_commits = 5000
max_files_per_commit = 30
half_life_days = 180
min_shared_commits = 2
min_coupling = 0.15
top_partners = 10
follow_renames = true
exclude_authors = ["dependabot[bot]", "renovate[bot]", "github-actions[bot]"]
exclude_message_patterns = ['^(chore|style|format)\b', '\bprettier\b', '\blint\b']
dir_top_k = 5
dir_min_coupling = 0.15

[index.schema_refs]
enabled = true
stopwords = ["data", "items", "logs", "status", "type", "meta"]
min_name_len = 4
max_tables_per_file = 20
max_files_per_table = 200
search = "auto"
variant_weights = { exact = 1.0, quoted = 1.5, singular = 0.6, pascal = 0.6, camel = 0.6 }

[capabilities]
live_mcp = []
live_timeout_ms = 10_000
exclude = []
user_level = false

[capabilities.describe]
# linear = "Issue tracker: tickets, projects, cycles"

[router]
small_repo_cutoff = 60
flatten_at = 40
flatten_factor = 0.9
chunk_size = 40
beam_max = 6
beam_min = 1
dead_end_min_needs_context = 0.5
max_depth = 4
max_candidates = 40
max_pointers = 12
walk_deadline_ms = 2000
route_deadline_ms = 3000
speculative_walk = true
max_caps_per_request = 40
request_max_tokens = 1500
max_path_hits = 10
basename_hit_strength = 0.8
dir_collapse_min_selected = 4
dir_collapse_max_dir_files = 8
type_diversity = true

[router.skip]
ack_words = ["ok", "okay", "thanks", "thank you", "continue", "go ahead", "yes", "lgtm", "sounds good", "do it"]
ack_max_words = 4

[router.expand]
depth = 1
kind_factors = { co_change = 1.0, schema_ref = 0.9, defined_in = 0.8, fk = 0.5, contains = 0.6 }
tables_per_anchor = 3

[router.thresholds.jev]
walk = 0.35
final = 0.60
path_hit_floor = 0.20
cap_use = 0.60
cap_skip = 0.15
needs_context = 0.25
continuity_min_conf = 0.60
expand = 0.30

[lease]
enabled = true
idle_minutes = 45
lock_timeout_ms = 250
max_files = 500
max_request_chars = 2000

[judge]
backend = "jev"            # jev | systemone-local | llm (eval only) | null
provider = "typesafe"      # typesafe | openrouter | vercel | cloudflare
model = "jev-1.13.0"
timeout_ms = 1200          # per request
retries = 1
retry_backoff_ms = 100
max_concurrency = 16
# api_key_env = "TYPESAFE_API_KEY"   # the NAME of the env var; never the key itself

[judge.breaker]
failure_threshold = 3
cooldown_s = 60
persist = true

[privacy]
redact_prompt = true
log_prompt_text = false
redact_emails = true
entropy_min_len = 24
extra_patterns = []        # e.g. [{ name = "internal_id", regex = 'ACME-[0-9]{8}' }]

[log]
decisions = true
max_bytes = 10_485_760
backups = 5
level = "warning"

[note]
max_lines = 15
max_line_chars = 160
max_chars = 2000
show_skip = true
ascii_only = false
root_hint = true

[delivery]
control_commands = true

[delivery.claude_code]
prompt_timeout_s = 10
session_timeout_s = 5
stop_hook = false
transcript_max_bytes = 1_048_576

[delivery.mcp]
host = "127.0.0.1"
port = 8765

[refresh]
session_start_min_interval_s = 60
background_timeout_s = 300
foreground_timebox_ms = 3000
```

`surf init` writes a **short** file by default (detected values plus the `[project]`, `[index]`, `[capabilities]` and `[judge]` sections, with a comment pointing to `surf config show`). The full file above is `surf config show --defaults --toml`. A short file means default changes in later surf versions apply automatically.

## 4. Behavior

### 4.1 Discovery

| Layer | Path | Committed? | Notes |
|---|---|---|---|
| `default` | built-in | — | the pydantic defaults |
| `user` | `$SURF_USER_CONFIG`, else `$XDG_CONFIG_HOME/surf/config.toml`, else `~/.config/surf/config.toml`; Windows `%APPDATA%\surf\config.toml` | no | personal preferences (provider, log level, note style) |
| `project` | `<root>/.surf/config.toml` | yes | team settings; the **only** source of index-scoped keys |
| `project-local` | `<root>/.surf/config.local.toml` | no (in `.surf/.gitignore`) | personal per-repo overrides |
| `extra` | `--config PATH` or `SURF_CONFIG` | n/a | for CI/eval experiments |
| `env` | `SURF_<SECTION>__<KEY>…` | n/a | §4.3 |
| `cli` | command flags (`--judge`, `--no-lease`, …) | n/a | mapped to dotted keys by `cli.py` |

`<root>` is found by 12 §2.2. A missing file is skipped silently; an unreadable file is an error (it exists, so the user expects it to apply).

### 4.2 Precedence and merge

Later layers override earlier ones: `default < user < project < project-local < extra < env < cli`.

- **Tables** are deep-merged key by key.
- **Arrays replace** (no concatenation). `index.exclude` in `config.local.toml` replaces the project's list. That's predictable, and index-scoped keys can't be set there anyway (§4.5).
- **Dict-valued keys** (`capabilities.describe`, `router.thresholds`, `router.wordings`, `variant_weights`, `kind_factors`) merge per entry.
- `origins` records, for each non-default leaf, the layer that set it (`surf config show --origin`).

### 4.3 Environment variables

| Variable | Meaning |
|---|---|
| `SURF_<PATH>` where `<PATH>` has segments joined by `__` | override one leaf, e.g. `SURF_JUDGE__BACKEND=null`, `SURF_ROUTER__THRESHOLDS__JEV__FINAL=0.55`, `SURF_LEASE__ENABLED=false`. The value is parsed as a TOML literal (`tomllib.loads("v = " + raw)`); if that fails, it's taken as a string. Segments match case-insensitively, and `_` matches `-` so backend names like `systemone-local` are reachable. Only variables containing `__` are treated as config overrides. |
| `SURF_ROOT` | project root override (12) |
| `SURF_CONFIG` | extra config file |
| `SURF_USER_CONFIG` | user config file path |
| `SURF_SESSION` | CLI session id (spec §12.5) |
| `SURF_DISABLE=1` | disable routing for this process (12 §4.2) |
| `SURF_DEBUG=1` | stderr debug logging, including in the hook |
| `SURF_MCP_TOKEN` | Bearer token for non-loopback MCP HTTP |
| `TYPESAFE_API_KEY` | default key var, provider `typesafe` |
| `OPENROUTER_API_KEY` | provider `openrouter` |
| `AI_GATEWAY_API_KEY` | provider `vercel` |
| `CLOUDFLARE_API_TOKEN` (+ `CLOUDFLARE_ACCOUNT_ID`) | provider `cloudflare` (names unverified; Q-13-4) |

An env override of an unknown path produces an `unknown-key` warning, not an error.

### 4.4 Validation

Two stages:

1. **Per-layer parse**: `tomllib` → dict. A TOML syntax error is a `ConfigError` naming the file (tomllib's message includes the line).
2. **Merged model validation**: pydantic validates the merged dict. All errors are collected, not only the first. Each problem is reported as `dotted.key (layer): message`.

Unknown keys: models use `extra="allow"`. After validation, a walk collects `model_extra` at every level and emits `unknown-key` warnings with a `difflib` suggestion (`router.tresholds → router.thresholds?`). Unknown keys are warnings, not errors, so a config written for a newer surf keeps working with an older one. `doctor` shows them (12 §4.11, `config.unknown_keys`).

`version`: greater than `CONFIG_VERSION` → error "config requires a newer surf"; missing → 1.

Cross-field rules (`model_validator(mode="after")`):

| Rule | On violation |
|---|---|
| For each thresholds set: `cap_skip < cap_use` | error |
| For each thresholds set: `path_hit_floor ≤ final` | error |
| For each thresholds set: `walk ≤ final` | warning (spec principle 3: lenient early) |
| `router.walk_deadline_ms < router.route_deadline_ms` | error |
| `judge.timeout_ms ≤ router.route_deadline_ms` | error |
| `router.beam_min ≤ router.beam_max` | error |
| `router.flatten_at ≤ router.max_candidates` | warning |
| `router.chunk_size > 40` or `router.max_candidates > 40` or `router.max_caps_per_request > 40` | warning (spec principle 5, ≤ ~40 candidates per request) |
| `router.max_pointers > 12` | warning (spec §1.2 target) |
| `delivery.claude_code.prompt_timeout_s * 1000 < router.route_deadline_ms + 2000` | error (the hook would be killed before the deadline fires) |
| `judge.backend == "systemone-local"` and `judge.base_url is None` | error |
| `judge.backend == "llm"` outside `surf eval` | error at runtime load (`Runtime.open`), allowed by `eval` |
| `judge.base_url` contains userinfo or a query string with `key`/`token` | error |
| `router.expand.depth != 1` | error |
| `index.include` pattern would match a secret-like default exclude | warning; the secret exclude still wins |
| `privacy.extra_patterns[*].regex` fails to compile or matches `""` | error |
| `note.max_lines < 4` | error (header + one line + fallback + marker) |

Values outside ranges are errors, not clamps, except where the table says "warning".

Behavior on errors:

| Context | Behavior |
|---|---|
| CLI commands | print every problem to stderr; exit 5 |
| Claude Code hook | fail open: no note, exit 0; one decision record `status=error, error="config"` per process |
| MCP server | tool result with `status=error` and a message naming `surf config validate`; server keeps running and retries the load when files change |
| `surf doctor` | `config.parse` check fails with the problem list |

### 4.5 Index-scoped keys and the fingerprint

Keys that change what's written to the committed catalog are **index-scoped**:

`index.*` (all), `capabilities.live_mcp`, `capabilities.live_timeout_ms`, `capabilities.describe`, `capabilities.exclude`.

Rules:

1. Index-scoped keys are read **only** from the `project` layer (`.surf/config.toml`) and defaults. If set in any other layer, they're ignored with an `ignored-index-key` warning. Otherwise a developer's user config could make their `surf index` output differ from CI's `surf index --check`, and the check would fail for reasons that aren't in the repo (D-13-4).
2. `index_fingerprint = "sha256:" + sha256(canonical_json(index-scoped subset))`, where canonical JSON has sorted keys, compact separators and floats rounded to 6 dp. It's written to `meta.json` (05). `surf refresh` and `surf index --check` compare it: a mismatch forces a full rebuild (06).
3. `capabilities.user_level = true` is a per-user choice, so it's **not** index-scoped. Capability cards from user-level configs are machine-specific and must never enter the committed catalog. They're written to `.surf/cache/user_caps.jsonl` and merged at query time (flag to 02/05; Q-13-2).

### 4.6 Template generation (`surf init`)

`config_write.render_initial(detected: InitDetection) -> str` produces the short file: `version`, a commented `descriptor` (auto-derived value shown as a comment), `[index]` with `exclude = []`, `[index.schema]` with detected `sources`/`dialect` when detection was confident (else commented), `[capabilities]` with `live_mcp` from `--live-mcp`, an empty `[capabilities.describe]`, and `[judge]` with `backend`/`provider`/`model`. The output must parse and validate. A test asserts this for every fixture repo.

### 4.7 Comment-preserving edits

`tomllib` can't write, and a TOML writer would drop comments. surf only ever edits one thing after init: entries under `[capabilities.describe]`. `config_write.set_describe(path, name, text)`:

1. Read the text. Find the `[capabilities.describe]` header line (regex, ignoring whitespace and comments).
2. The key is rendered as a bare key if it matches `^[A-Za-z0-9_-]+$`, else as a basic string. The value is a TOML basic string with `\`, `"` and control characters escaped.
3. If an entry with that key exists inside the section, replace that line. Otherwise insert after the last non-blank line of the section. If the section doesn't exist, append `\n[capabilities.describe]\n<entry>\n`.
4. Re-parse the result with `tomllib` and validate; on failure, don't write (error).
5. Write atomically, preserving line endings.

Inline-table forms (`describe = { … }` under `[capabilities]`) are detected, and in that case the edit is refused with instructions.

### 4.8 Loading performance and the hook fast path

`load_config` runs once per process. The hook fast path (12 §4.4.3) must not import pydantic, so `raw_peek(root, "router.skip.ack_words")` reads only the project and project-local files with `tomllib` and returns the raw value or `None`. Only the few keys the fast path needs are read this way (`router.skip.*`, `delivery.control_commands`, `lease.enabled`, `lease.idle_minutes`). An invalid value there falls back to the default; the full validation happens on the slow path.

## 5. Configuration

This doc *is* the configuration. Each component doc's §5 must list only keys that appear in §3 with the same name, type and default. A test (`tests/test_config_docs.py`) parses the §5 tables of all design docs and compares them with `SurfConfig.model_json_schema()`, so drift fails CI.

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| No `.surf/config.toml` but `.surf/` exists | defaults + user layers; `doctor` warns |
| Empty config file | defaults |
| TOML syntax error in `config.local.toml` | error naming that file (CLI exit 5; hook fail-open) |
| BOM at file start | accepted (stripped before `tomllib`) |
| `SURF_JUDGE__BACKEND=nul` (typo) | validation error naming the env layer |
| `SURF_ROUTER__MAX_POINTERS=abc` | TOML literal parse fails → string → pydantic int error |
| Thresholds for backend `systemone-local` missing | jev thresholds used, warning; `doctor` warns |
| User config sets `index.exclude` | ignored with warning (§4.5) |
| `capabilities.describe` for a server that no longer exists | warning at index time; kept (harmless) |
| `judge.api_key = "sk-…"` in any file | error; the value is never echoed (message says "a secret-like key was found at judge.api_key") |
| Config file is world-writable | warning in `doctor` (it controls what's sent to the judge) |
| Very large `ack_words` list (> 200) | error (fast-path cost) |
| `version = 2` | error "requires newer surf" |
| Windows paths in globs (`src\gen\**`) | backslashes rejected with a hint to use `/` |

## 7. Performance budget

| Operation | Budget |
|---|---|
| `load_config` (4 files, after imports) | ≤ 10 ms |
| `raw_peek` (1–2 files, `tomllib` only) | ≤ 3 ms |
| `index_fingerprint` | ≤ 1 ms |

## 8. Test plan

- **Defaults:** `SurfConfig()` equals `load_config` of the §3.14 file (a golden test; the file is generated from the model and checked in, so doc and code can't drift).
- **Precedence:** a matrix test with one key set in each layer; `origins` is correct; arrays replace; dicts merge per entry.
- **Env parsing:** booleans, ints, floats, lists, strings, hyphenated backend names, unknown paths.
- **Validation:** one test per cross-field rule (error and pass cases); all errors reported together; secret-key rejection; unknown-key suggestions.
- **Index scoping:** index keys in user/local/env layers are ignored with warnings; the fingerprint changes only when an index-scoped key changes (property test over random non-index edits).
- **Writers:** `set_describe` on files with comments, CRLF, missing section, existing key, inline-table form (refused), unusual server names; round-trip validity.
- **Docs drift:** the §5-tables-vs-schema test described in §5.
- **Fail-open:** hook and MCP with an invalid config → no exception, `status=error`.

## 9. Acceptance criteria

1. Every key referenced in design docs 01–16 exists in `SurfConfig` with the same name and default (docs drift test green).
2. `surf index --check` gives identical results with and without a user config that sets any key (determinism, §4.5).
3. All validation errors in a file are reported in one run.
4. A config with only unknown extra keys loads with warnings and routes normally.
5. Phase 1 exit uses the index subset; Phase 4 exit needs the full schema (build plan P1.11 → complete by P4).

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-13-1 | §16: `router.deadline_ms = 3000`; §11.5: `deadline_ms = 2000` is the walk budget | Two keys: `router.walk_deadline_ms = 2000`, `router.route_deadline_ms = 3000` | The spec uses one name for two budgets (00 §5) |
| D-13-2 | §16 `judge.timeout_ms` only; §13.3 "retry once", "3 failures → 60 s" | `judge.timeout_ms` is per request; adds `judge.retries`, `retry_backoff_ms`, `judge.breaker.{failure_threshold, cooldown_s, persist}` | Make §13.3's constants tunable; breaker state must persist across hook processes |
| D-13-3 | §16: key "from env" (comment) | Secret-like keys in config are rejected; `judge.api_key_env` names the variable | Config is committed |
| D-13-4 | not specified | Index-scoped keys come only from `.surf/config.toml`; an index fingerprint in `meta.json` | Keeps `--check` reproducible across machines |
| D-13-5 | §16 sample only | Adds the keys listed in §3 that the spec uses as constants: card budgets, `beam_min`, `flatten_factor`, dead-end guard, collapse rule, path-hit caps, expansion kind factors, skip ack list, cochange message patterns and `dir_top_k`, schema-ref caps and variant weights, `capabilities.user_level`/`exclude`, lease/note/log/delivery/refresh/eval sections | Spec principle 8: every threshold must be tunable by evaluation; one place for all names |
| D-13-6 | §16: single `.surf/config.toml` | Layered discovery: user, project, project-local, extra, env, CLI | Personal settings (provider, logging) shouldn't require editing a committed file |
| D-13-7 | §16 `backend` list: `jev \| systemone-local \| llm \| null` | Adds `fixture` (tests/eval); `llm` rejected at runtime outside `surf eval` | Spec §13.2 says `llm` is an evaluation baseline only |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-13-1 | Should unknown keys be errors (catch typos) rather than warnings (forward compatibility)? | Warnings + `doctor`; `surf config validate --strict` makes them errors | User feedback on silent typos |
| Q-13-2 | Where do user-level capability cards live so they never reach the committed catalog? | `.surf/cache/user_caps.jsonl`, merged at query time (02/05 to confirm) | 02/05 review |
| Q-13-3 | Should thresholds for non-jev backends ship with defaults of their own? | No; fall back to jev with a warning until A6/A7 produce tuned sets | Ablations A6/A7 |
| Q-13-4 | Exact env var names for the Vercel AI Gateway and Cloudflare providers | `AI_GATEWAY_API_KEY`; `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` | Provider docs (not verifiable from this sandbox; confirm in P0.5) |
| Q-13-5 | Should `router.max_pointers` above 12 be allowed at all? | Allowed up to 30 with a warning | Precision eval |
| Q-13-6 | Should `index.exclude` also be settable per user for huge local-only dirs (e.g. `data/`)? | No (determinism); use `.git/info/exclude`, which discovery already honors as a gitignore source | User requests |
