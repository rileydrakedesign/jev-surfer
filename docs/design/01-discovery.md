# 01 · Discovery: file enumeration, excludes, classification, agent-config detection

**Status:** draft for review
**Spec sections:** §7.1, §7.6 (detection only), §6.2, §6.3, §10.1 (fingerprints), §19.3 (secret-like files)
**Depends on:** 00-foundations (ids F2/F4, determinism rules, `proc.py`), 13-config
**Consumed by:** 02-cards, 03-schema-extraction, 04-graph-edges, 05-catalog-store, 06-refresh
**Code:** `src/surf/index/discover.py`, `src/surf/index/globs.py` (new, see §3.4)

---

## 1. Purpose and scope

Discovery answers four questions for one build, without reading anything it doesn't need to:

1. Where is the project root, and is it a git repo (and how deep is its history)?
2. Which files are indexable content, and what kind (`code` / `doc`) is each?
3. Which directories exist in the content tree, and which id prefix (`code:` / `doc:`) does each get (F2)?
4. Where are the agent-config files that define capability surfaces (MCP servers, skills, subagents, commands)?

It also defines the **content fingerprint** used for change detection outside git (§3.3).

| In scope (v1) | Out of scope (v1) |
|---|---|
| git and non-git roots, tracked + untracked-not-ignored files | git submodules (skipped, warned) |
| default, linguist, generated-header, secret-like and user excludes | following symlinks |
| binary and oversize detection, line counting in the same read | parsing capability files (→ 02 `extract_caps.py`) |
| doc/code classification (F4), directory tree and prefixes (F2) | schema source detection (→ 03 §4.1) |
| capability source detection for Claude Code, Cursor, Codex, OpenCode | monorepo package scoping (v2) |
| content fingerprints for refresh | file watching (v2) |

## 2. Interfaces

```python
# src/surf/index/discover.py
def resolve_root(start: Path) -> RootInfo: ...
def discover(root: RootInfo, cfg: IndexConfig, caps_cfg: CapabilitiesConfig,
             *, home: Path | None = None) -> Discovery: ...
def classify_paths(root: RootInfo, rel_paths: Sequence[str], cfg: IndexConfig
                   ) -> list[FileEntry | Excluded]: ...          # refresh: subset only
def build_tree(files: Sequence[FileEntry]) -> list[DirEntry]: ...  # pure, sorted by path
def fingerprint(root: RootInfo, rel_path: str, prev: Fingerprint | None) -> Fingerprint: ...
def detect_capability_sources(root: RootInfo, caps_cfg: CapabilitiesConfig,
                              *, home: Path | None) -> list[CapSource]: ...
```

| Caller | Uses |
|---|---|
| `index/build.py` (full) | `resolve_root`, `discover` |
| `index/build.py` (incremental, 06) | `classify_paths` on the changed set, `build_tree` on the merged file list, `fingerprint` for non-git and untracked files |
| `extract_code.py` / `extract_docs.py` (02) | `FileEntry` (lines, kind) |
| `extract_schema.py` (03) | `Discovery.files` to match schema-source globs |
| `extract_caps.py` (02) | `Discovery.cap_sources` |
| `graph/cochange.py` (04) | `RootInfo.is_git`, `is_shallow`; the excluded-path predicate `Discovery.is_excluded(path)` so historic paths are filtered with the same rules |
| `surf doctor`, `surf init` plan | `Discovery.stats`, `warnings` |

`home` is injectable so tests never touch the real `~`. It is only read when `capabilities.user_level = true`.

## 3. Data structures

```python
class RootInfo(BaseModel, frozen=True):
    path: Path                     # absolute, resolved; never written to .surf/
    is_git: bool
    head: str | None               # full sha; None for non-git or unborn HEAD
    is_shallow: bool
    git_ok: bool                   # False if .git exists but git is missing/broken

class FileKind(StrEnum):
    CODE = "code"; DOC = "doc"

class FileEntry(BaseModel, frozen=True):
    path: str                      # norm_path(..., is_dir=False)
    kind: FileKind
    tracked: bool                  # False = untracked-not-ignored (git) ; always True for non-git
    size: int                      # bytes
    lines: int | None              # None when oversize
    oversize: bool                 # size > index.max_file_bytes
    mtime_ns: int                  # cache only; never in committed output

class ExcludeReason(StrEnum):
    DEFAULT_DIR = "default_dir"; DEFAULT_FILE = "default_file"; USER = "user"
    LINGUIST = "linguist"; GENERATED = "generated_header"; BINARY = "binary"
    SECRET_LIKE = "secret_like"; SYMLINK = "symlink"; SUBMODULE = "submodule"
    CAP_DEFINITION = "capability_definition"; UNREADABLE = "unreadable"; NFC_COLLISION = "nfc_collision"

class Excluded(BaseModel, frozen=True):
    path: str
    reason: ExcludeReason

class DirEntry(BaseModel, frozen=True):
    path: str                      # "src/services/" ; never "" (root has no DirEntry)
    prefix: Literal["code", "doc"] # F2
    files_total: int               # recursive, indexed files only
    doc_files_total: int
    has_untracked_only: bool       # every file below is untracked → overlay-only dir (§4.7)

class Discovery(BaseModel):
    root: RootInfo
    files: list[FileEntry]         # sorted by path
    dirs: list[DirEntry]           # sorted by path
    excluded_counts: dict[ExcludeReason, int]
    cap_sources: list[CapSource]   # sorted (§4.8)
    warnings: list[str]            # human text, no absolute paths
    def is_excluded(self, path: str) -> ExcludeReason | None: ...  # pattern-only check (no I/O)
```

### 3.1 Capability sources

```python
class Harness(StrEnum):
    CLAUDE = "claude"; CURSOR = "cursor"; CODEX = "codex"; OPENCODE = "opencode"

class CapSourceKind(StrEnum):
    MCP_CONFIG = "mcp_config"; SETTINGS = "settings"; SKILL = "skill"
    AGENT = "agent"; COMMAND = "command"

class CapLevel(StrEnum):
    PROJECT = "project"; PROJECT_LOCAL = "project_local"; USER = "user"; PLUGIN = "plugin"

class CapSource(BaseModel, frozen=True):
    harness: Harness
    kind: CapSourceKind
    level: CapLevel
    display_path: str        # repo-relative ("./.mcp.json" style NOT used: ".mcp.json"),
                             # or "~/.codex/config.toml" for user level. Never absolute.
    abs_path: Path           # runtime only; never serialized
    namespace: str | None    # plugin name for PLUGIN level, command sub-dir for commands
    committable: bool        # True only for PROJECT level files tracked by git
```

### 3.2 Committable vs local-only inputs

A committed `catalog.jsonl` must be reproducible by CI from a clean checkout (00 §4). Three kinds of input can't be:

| Input | Why not reproducible | Treatment |
|---|---|---|
| Untracked (not ignored) files | not in the checkout | `FileEntry.tracked = False` |
| `.claude/settings.local.json`, any git-ignored config | not in the checkout | `CapSource.committable = False` |
| User-level and plugin capability sources | per developer | `committable = False` |

Discovery only **labels** these. The rule applied downstream (proposal D-01-1, handed to 05 and 06): *a card is committed only if every input it was built from is committable.* Leaf cards built from local-only inputs (untracked files, user-level capabilities) go to a local overlay (`.surf/cache/overlay.jsonl`); committed directory cards are built from tracked files only, so a local untracked file never changes committed text. Directories that contain only untracked files (`has_untracked_only`) exist only in the overlay. For non-git projects everything is committable (there is no CI check to satisfy).

### 3.3 Content fingerprint (cache only)

Card `hash` (02 §4.9) covers card inputs, not bytes, so it can't detect every change, and non-git projects have no `git diff`. Refresh therefore needs a separate fingerprint, stored only in `.surf/cache/index.sqlite`, never in a card:

```python
class Fingerprint(BaseModel, frozen=True):
    size: int
    mtime_ns: int
    sha256: str | None       # computed lazily
```

`fingerprint(root, path, prev)`: stat the file; if `prev` exists and `(size, mtime_ns)` both match, return `prev` unchanged (no read); otherwise read and hash. A file is **changed** iff the new `sha256` differs from `prev.sha256` (a `touch` doesn't count). Used for: non-git roots (all files), untracked files in git roots, and the capability-config change trigger (spec §10.1). Table: `fingerprints(path TEXT PRIMARY KEY, size INT, mtime_ns INT, sha256 TEXT)`. The same function is used on capability source files with `path = display_path`.

### 3.4 Glob matcher (`index/globs.py`)

Config excludes, default excludes, `.gitignore` for non-git roots, and schema-source globs (03) all need gitignore-style matching. Python 3.11 has no `glob.translate`, and `pathspec` isn't in spec §22.1, so we add a ~100-line translator instead of a dependency:

```python
class GlobSet:
    def __init__(self, patterns: Sequence[str]) -> None: ...   # compiled once
    def match(self, path: str, *, is_dir: bool) -> bool: ...   # last matching pattern wins (for "!")
```

Semantics (gitignore subset): `*` (no `/`), `**` (any depth), `?`, `[...]`; a pattern with no `/` except a trailing one matches the basename at any depth; a leading `/` or an inner `/` anchors to the root; a trailing `/` matches directories only (and everything beneath); `!` negates. Case-sensitive. Tested against a table of git's own `check-ignore` results (§8).

## 4. Behavior / algorithm

### 4.1 Resolve root

1. Run `git rev-parse --show-toplevel` from `start` via `proc.run` (timeout 5 s).
   - Success: `is_git = True`, `path` = output. Then `git rev-parse --verify -q HEAD` (empty result on an unborn branch → `head = None`) and `git rev-parse --is-shallow-repository`.
   - `git` binary missing but a `.git` entry exists at or above `start`: `is_git = False`, `git_ok = False`, warning "git not found; co-change disabled". Root = directory containing `.git`.
   - Not a repo: `is_git = False`, root = `start` resolved (the `surf` commands take `--root`; default cwd).
2. Bare repository or `start` inside `.git/`: `SurfError("run surf from a working tree")`, exit non-zero (index time may fail loudly, 00 §5).

### 4.2 Enumerate candidates

- **git:** `git ls-files -z --cached --others --exclude-standard --full-name` from the root, plus `git ls-files -z --deleted` to subtract tracked files missing from the working tree. The `--others` subset (second call, `--others --exclude-standard` only) marks `tracked = False`. `git ls-files -z -s` gives mode `160000` for submodules and `120000` for symlinks; those are dropped (`SUBMODULE`, `SYMLINK`). Sparse-checkout paths not on disk are dropped silently.
- **non-git:** `os.walk(root, followlinks=False)` in sorted order, pruning default-excluded directories before descent (so `node_modules/` isn't walked). `.gitignore` files are honored with `GlobSet` (root and nested, nested patterns anchored to their dir), so a copied-out repo behaves like its git version. Symlinks are skipped.
- `index.include_untracked = false` drops the `--others` call.

All candidate paths go through `norm_path` (NFC). Two distinct raw paths that normalize to the same NFC path: keep the lexicographically smaller raw path, exclude the other with `NFC_COLLISION`, warn.

### 4.3 Exclude (pattern stage, no I/O)

Evaluated in this order; first hit wins and is recorded as the reason:

| # | Rule | Patterns / behavior | Overridable by `index.include`? |
|---|---|---|---|
| 1 | Secret-like (spec §7.1) | 14 §3.2 `SECRET_LIKE_GLOBS` (basename, case-insensitive: `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `id_ed25519*`, `id_ecdsa*`, …), except that `*credentials*` / `*secret*` apply **only when the extension isn't a source-code extension** (D-01-2) | Only by an explicit `index.include` entry for that path, which `doctor` lists (14 §3.2 rule 6, D-14-8). Otherwise never read, never carded |
| 2 | Always | `.git/`, `.surf/`; any path for which `redact.path_is_safe` is false (control characters, newlines or bidi controls in any segment; 14 §4.3, D-01-7), logged | No |
| 3 | Capability definitions (D-01-3) | `.claude/skills/`, `.claude/agents/`, `.claude/commands/`, `.opencode/agent/`, `.opencode/agents/`, `.opencode/command/`, `.opencode/commands/`, `.codex/skills/`, `.codex/prompts/` | No (they become capability cards) |
| 4 | Default dirs (any depth) | spec: `node_modules/ vendor/ .venv/ venv/ target/ dist/ build/ .next/ out/ coverage/ __pycache__/`; added: `__snapshots__/ .tox/ .mypy_cache/ .pytest_cache/ .ruff_cache/ .gradle/ .turbo/ .nuxt/ .svelte-kit/ .parcel-cache/ .terraform/ bower_components/ Pods/ .dart_tool/ .idea/` | Yes |
| 5 | Default files | `*.lock`, `package-lock.json`, `npm-shrinkwrap.json`, `pnpm-lock.yaml`, `go.sum`, `*.min.*`, `*.map`, `*.snap`, `*.pyc` | Yes |
| 6 | Binary by extension | images, fonts, audio/video, archives, compiled objects, `*.sqlite`, `*.db`, `*.pdf`, `*.wasm`, `*.jar`, `*.class` (list in code, `BINARY_EXTS`) | Yes |
| 7 | User | `index.exclude` | Yes (`index.include` wins; include is checked last) |
| 8 | Linguist | `git check-attr -z --stdin linguist-generated linguist-vendored` on survivors (one batched call); `set`/`true` → excluded. Git only; `index.respect_linguist = true` | Yes |

Rules 4–6 are skipped entirely when `index.default_excludes = false`. The secret-like and always rules can't be turned off.

Source-code extensions for rule 1 are the keys of the language map in 02 §4.2 (so `src/secrets/rotate.ts` stays, `config/secrets.yaml` and `aws_credentials` are excluded).

### 4.4 Read stage (one read per surviving file)

For each surviving path, `stat` (lstat; symlink → `SYMLINK`), then:

1. `size > index.max_file_bytes` (default 1,000,000): `oversize = True`, `lines = None`; read only the first 8 KiB for the binary check. The file stays (path hits still need it) but no content-derived fields are extracted anywhere (02, 04 schema refs skip it).
2. Else read the whole file as bytes once:
   - null byte in the first 8,192 bytes → `BINARY`;
   - generated header: any of the first 5 lines matches `^\s*(//|#|/\*|--)\s*Code generated .* DO NOT EDIT\.?` or contains `@generated` → `GENERATED` (`index.exclude_generated = true`);
   - `lines = count(b"\n") + (1 if data and not data.endswith(b"\n") else 0)`.
3. `OSError`/`PermissionError` → `UNREADABLE`, warning, continue.

Reads use a thread pool of `min(8, os.cpu_count())`; results are re-sorted by path, so pool scheduling can't affect output.

### 4.5 Classify (F4)

`kind = DOC` iff any of:
- extension (case-insensitive) in `.md .mdx .markdown .rst .adoc .asciidoc`;
- basename matches `README*`, `CHANGELOG*`, `CONTRIBUTING*` with no extension or `.txt`;
- extension `.txt` and some parent dir is named `adr`, `adrs`, `decisions` or `rfcs` (ADRs).

Everything else is `CODE`. Location never matters (`src/foo/NOTES.md` is `doc:`; `docs/build.py` is `code:`).

### 4.6 Build tree and prefixes (F2)

`build_tree(files)` (pure):

1. For each file, add 1 to `files_total` of every ancestor dir, and to `doc_files_total` if `kind == DOC`.
2. A directory exists iff `files_total ≥ 1`. Directories whose every file is excluded produce no node.
3. `prefix = "doc"` iff `doc_files_total * 2 > files_total` (strictly more than half), else `"code"`. The root is not a `DirEntry` and never gets a prefix.
4. `has_untracked_only` = no tracked file below.
5. Parent of a top-level file or dir is `root:` (00 §2.1). Parent ids use the parent's prefix, so a prefix flip re-parents all direct children; refresh treats it as remove + add (F2).

**The root has no card.** The walk's first frontier is `root:`; its children are the top-level dir cards, top-level file cards, and `db:*` when a schema exists (03). Jev sees those cards at walk level 1, with `project` (the descriptor, 02 §4.10) in state. That descriptor stands in for a root card. A repo whose top level has more than `router.chunk_size` children is chunked like any wide node (09).

### 4.7 Local-only labeling

`tracked` from 4.2; `DirEntry.has_untracked_only` from 4.6. Committed dir cards count tracked files only: `build_tree` runs twice when untracked files exist — once on tracked files (committed tree) and once on all files (overlay tree; only dirs missing from the committed tree are emitted to the overlay). Cost is linear and negligible.

### 4.8 Detect capability sources (§7.6 step 1)

Only existence checks and directory listings here; parsing is in 02. Paths per harness:

| Harness | Level | Kind | Location | Notes |
|---|---|---|---|---|
| Claude Code | project | mcp_config | `.mcp.json` | `mcpServers` |
| | project | settings | `.claude/settings.json` | read for `enabledMcpjsonServers`, `disabledMcpjsonServers`, `enableAllProjectMcpServers`, and any `mcpServers` key (tolerated) |
| | project_local | settings | `.claude/settings.local.json` | usually git-ignored → not committable |
| | project | skill | `.claude/skills/*/SKILL.md` | one level deep only |
| | project | agent | `.claude/agents/**/*.md` | |
| | project | command | `.claude/commands/**/*.md` | `namespace` = sub-dir path |
| | user | mcp_config | `~/.claude.json` | top-level `mcpServers` **and** `projects["<abs root>"].mcpServers` (Claude's "local" scope) |
| | user | settings / skill / agent / command | `~/.claude/settings.json`, `~/.claude/skills/*/SKILL.md`, `~/.claude/agents/**/*.md`, `~/.claude/commands/**/*.md` | |
| | plugin | all | `~/.claude/plugins/installed_plugins.json` → each enabled plugin's install path (enabled = listed in any settings `enabledPlugins` with value `true`); inside: `skills/*/SKILL.md`, `agents/**/*.md`, `commands/**/*.md`, `.mcp.json` | `namespace` = plugin name; best-effort (Q-01-2) |
| Cursor | project | mcp_config | `.cursor/mcp.json` | `mcpServers` |
| | user | mcp_config | `~/.cursor/mcp.json` | |
| Codex | project | mcp_config | `.codex/config.toml` | `[mcp_servers.<name>]` |
| | user | mcp_config | `~/.codex/config.toml` (or `$CODEX_HOME/config.toml`) | spec's `~/.codex/config.*`: only `.toml` exists |
| | user | command | `~/.codex/prompts/*.md` | Codex custom prompts |
| | project/user | skill | `.codex/skills/*/SKILL.md`, `~/.codex/skills/*/SKILL.md` | best-effort |
| OpenCode | project | mcp_config | `opencode.json`, `opencode.jsonc` (root), `.opencode/opencode.json(c)` | key `mcp`; JSONC comments stripped |
| | project | agent / command | `.opencode/agent(s)/**/*.md`, `.opencode/command(s)/**/*.md`; also inline `agent` / `command` keys in `opencode.json` | |
| | project | skill | `.opencode/skill(s)/*/SKILL.md` | |
| | user | all | `$XDG_CONFIG_HOME/opencode/` (default `~/.config/opencode/`) same layout | |

Rules:
- Harnesses scanned = `capabilities.harnesses` (default all four). Detection doesn't require the harness to be installed; the files are the evidence.
- User and plugin levels are scanned **only** when `capabilities.user_level = true` (opt-in, spec §7.6). They are never committable.
- `committable = (level == PROJECT) and (non-git or git ls-files reports the file tracked)`.
- A path that is both a Claude and an OpenCode source (e.g. OpenCode reading `.claude/skills/`) is reported once per harness that reads it; 02 dedupes by resolved file (02 §4.8).
- Output sorted by `(kind, harness, level, display_path)`.
- Missing directories are normal and not warned. Unreadable files: warning, skipped.

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `index.exclude` | list[str] | `[]` | gitignore-style (§3.4) |
| `index.include` | list[str] | `[]` | new; overrides default and user excludes and (explicitly, doctor-listed) secret-like; never always/cap-definition |
| `index.default_excludes` | bool | `true` | new; disables rules 4–6 |
| `index.include_untracked` | bool | `true` | new |
| `index.max_file_bytes` | int | `1_000_000` | spec §16 |
| `index.respect_linguist` | bool | `true` | new |
| `index.exclude_generated` | bool | `true` | new |
| `capabilities.harnesses` | list[str] | `["claude","cursor","codex","opencode"]` | new |
| `capabilities.user_level` | bool | `false` | new; the spec's "user-level configs if the user opts in" |

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| Not a git repo | Index everything not excluded; `tracked = True`; co-change off; `doctor` notes it |
| Unborn HEAD (no commits) | `head = None`; all files come from `--others`/`--cached` as usual; co-change off |
| Shallow clone | `is_shallow = True`; passed to 04; `doctor` suggests `git fetch --unshallow` |
| `git` hangs | `proc.run` timeout (5 s per call, 30 s for `ls-files` on huge repos) → `SurfError`; index fails loudly |
| Run from a subdirectory | Root resolved to toplevel; ids are always toplevel-relative |
| Git worktree | Works (`--show-toplevel` returns the worktree) |
| Submodule | Excluded (`SUBMODULE`), one warning listing the count |
| Symlink (file or dir) | Excluded, counted; never followed (cycles, out-of-repo reads) |
| Path with `:` or spaces or non-ASCII | Legal; NFC-normalized; `-z` output avoids quoting issues |
| Path with a newline, control or bidi character | Excluded (rule 2, `path_is_safe`), counted and logged; never opened |
| NFC collision | §4.2 |
| Case-only duplicates (`Readme.md`, `README.md`) | Both kept; ids differ by case (00 §2.2) |
| Tracked file deleted in working tree | Dropped (from `--deleted`) |
| File modified in working tree | Working-tree content is used; `--check` on a dirty tree warns (06) |
| UTF-16 text file | Null bytes → treated as binary (accepted loss) |
| Oversize file | Kept, `oversize = True`, no content fields |
| Empty file | `lines = 0`, kept |
| `.env.example` | Excluded (secret-like rule is by name; safe default) |
| `src/secrets/` directory of `.ts` files | Kept (D-01-2) |
| All files of a dir excluded | No dir node |
| Doc-only dir inside `src/` | `doc:` prefix (F2); its code parent stays `code:` |
| Exactly 50 % docs | `code:` (strict majority required) |
| Malformed `.gitignore` line (non-git) | Ignored line, no warning |
| `~/.claude.json` huge (MBs of history) | Streamed JSON parse is not needed: file is read once, capped at 20 MB; larger → warning, skipped |
| User-level opt-in but `home` unreadable | Warning, project level only |
| Plugin install path missing | Warning, plugin skipped |

## 7. Performance budget

| Repo | Candidates | Budget (warm cache, SSD) |
|---|---|---|
| 5k files | 5k reads, avg 10 KB | ≤ 2 s total discovery (spec Phase 1 exit: full index < 60 s) |
| 50k files | 50k reads | ≤ 10 s |
| `classify_paths` for a 20-file commit | 20 reads | ≤ 50 ms |

Git calls: `rev-parse` ×3, `ls-files` ×3, `check-attr` ×1 (stdin batch). No per-file subprocesses. Memory: `FileEntry` objects only (content bytes are dropped after the read stage, except docs, which 02 re-reads).

## 8. Test plan

**Unit**
- `globs.py`: table-driven against `git check-ignore --no-index` output generated once and checked in (≈150 cases: anchoring, `**`, trailing `/`, negation, character classes).
- Exclusion order: each rule, first-hit-wins reason, `index.include` overriding rules 1 and 4–7 only (rule 1 only by an explicit path, reported by `doctor`); rules 2–3 never overridable.
- Classification: F4 table (`src/NOTES.md` → doc, `docs/conf.py` → code, `README` → doc, `adr/0001.txt` → doc).
- `build_tree`: prefix threshold at 50 %/51 %, root excluded, empty dirs absent, untracked-only dirs.
- Line counting: empty file, no trailing newline, CRLF (counts `\n`), oversize.
- `fingerprint`: unchanged stat → no read (mock `open`), touch without change → not changed, content change → changed.
- Capability detection: a fake `home` tree with every location in §4.8; `user_level = false` returns project only; committable flags.

**Fixture repos** (`tests/fixtures/repos/*.yaml`, built by the shared script):
- `discovery-basic`: node_modules, dist, lockfiles, a PNG, a null-byte `.txt`, `.env`, `config/secrets.yaml`, `src/secrets/rotate.ts`, a `// Code generated` Go file, `.gitattributes` with `linguist-generated`, a symlink, an untracked file, a git-ignored file.
- `discovery-nongit`: same tree without `.git`, with nested `.gitignore`.
- Golden: `Discovery` serialized (minus `mtime_ns`, abs paths) compared to a checked-in JSON.

**Property** (`hypothesis`): `discover` output is identical under permutation of `ls-files` order; `norm_path` idempotent.

## 9. Acceptance criteria

1. Both Phase 0 target repos: discovery output matches a hand-reviewed exclude report (no `node_modules`, lockfiles, binaries or secret-like files in the catalog; no source file excluded by the secret rule).
2. Two runs on the same commit produce byte-identical serialized `Discovery` (excluding `mtime_ns`).
3. The §7 budgets hold on a 5k-file repo in CI.
4. No absolute path, username or home directory appears in any `display_path`, warning or serialized output.
5. With `capabilities.user_level = false`, nothing under `home` is opened (asserted with a sandboxed fake home that raises on access).

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-01-1 | Enumerate tracked + untracked files (§7.1); commit the catalog (§9.5) | Label untracked files and local-only capability sources; downstream, their cards go to a local overlay, never to `catalog.jsonl` | Otherwise a developer's scratch files and personal MCP servers land in the committed catalog and CI `--check` can't reproduce it |
| D-01-2 | `*credentials*`, `*secret*` excluded | Only when the extension isn't a source-code extension | `src/secrets/manager.ts` is legitimate code that tasks ask about; the name rule targets data/config files |
| D-01-3 | Silent | Capability definition dirs (`.claude/skills/` etc.) are excluded from the content tree | They are represented by capability cards; indexing them twice produces duplicate, competing candidates |
| D-01-4 | Excludes list only | Added: linguist attributes, `Code generated … DO NOT EDIT` headers, more tool cache dirs | Generated code is a common distractor; `.gitattributes` is committed so it stays deterministic |
| D-01-5 | `max_file_bytes` purpose unstated | Oversize files keep a path-only card | Stack traces still point at them; content fields would be noise |
| D-01-6 | `~/.codex/config.*` | `config.toml` (+ `$CODEX_HOME`), plus `.codex/config.toml` and `~/.codex/prompts/` | Current Codex layout |
| D-01-7 | All non-excluded files are indexed (§7.1) | Paths containing control characters, newlines or bidi controls are always excluded at discovery (14 `path_is_safe`) | A crafted file name could otherwise inject lines into the note or judge prompts (14 T5) |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-01-1 | Should `.svg` and large text data (`*.csv`, `*.json` > 100 KB) be excluded by default? | Keep; rely on `max_file_bytes` | Eval: share of final-pass candidates that are data/asset files |
| Q-01-2 | Claude Code plugin layout (`installed_plugins.json`, `enabledPlugins`) is not a stable public contract | Partly resolved 2026-09-23. Documented: `enabledPlugins` maps `"<plugin>@<marketplace>": bool` at any settings scope (managed > local > project > user), and a plugin with no entry falls back to its manifest `defaultEnabled` (default `true`), so "enabled = listed true" is wrong; plugins root is `$CLAUDE_CODE_PLUGIN_CACHE_DIR` or `$CLAUDE_CONFIG_DIR/plugins` (default `~/.claude/plugins`), with `cache/` and `synced/`. Undocumented: the `installed_plugins.json` schema. Reader: install paths from that file if parseable, else enumerate `<root>/cache/**/.claude-plugin/plugin.json`; inside a plugin read `skills/*/SKILL.md`, `commands/**/*.md`, `agents/**/*.md`, `.mcp.json` or inline `mcpServers` in `plugin.json` | Phase 4 check of the `installed_plugins.json` schema only |
| Q-01-5 | Harness layouts changed since the design pass (verified 2026-09-23; details in `jev-docs-checks.md` §3 H-C9, H-O1, H-O3): Claude Code merged commands into skills (a skill shadows a same-name command; namespaced commands are `frontend:component`), loads nested `<subdir>/.claude/skills/`, takes agent identity from frontmatter `name`, and documents no `mcpServers` key in `settings*.json` (project-local servers live in `~/.claude.json` under `projects[<root>]`, with `disabledMcpServers`); Codex skills moved to `.agents/skills/` (walk up to the repo root) and `~/.agents/skills/`, agents are `.codex/agents/*.toml` / `$CODEX_HOME/agents/*.toml`, `~/.codex/prompts/` is deprecated; OpenCode uses plural `agents/`/`commands/`/`skills/` (singular is legacy), global config `~/.config/opencode/opencode.json`, MCP `command` is an array with `environment`, and also reads `.claude/skills` and `.agents/skills` | Update the §4.8 source table and D-01-6 with these paths in P1 (discovery), with fixtures per harness; honor `CLAUDE_CONFIG_DIR` for every `~/.claude` path | Discovery implementation review |
| Q-01-3 | Cursor `.cursor/commands/` and `.cursor/rules/` as capability surfaces? | Out of v1 (spec lists only Cursor MCP) | User demand |
| Q-01-4 | Should directories that are single-child chains (`a/b/c/` with one child each) be collapsed? | No collapse in v1; the walk's flattening handles small subtrees | Walk depth distribution on target repos |
