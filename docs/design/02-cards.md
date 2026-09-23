# 02 · Cards: code, doc, directory and capability cards; budgets; hashing

**Status:** draft for review
**Spec sections:** §7.2, §7.3, §7.4, §7.6, §7.7, §7.8, §9.2, §19.3, §19.4
**Depends on:** 00-foundations, 01-discovery, 03-schema-extraction (table facts), 04-graph-edges (edge lookups, churn counts), 14-security-privacy (`redact.py`)
**Code:** `src/surf/index/extract_code.py`, `extract_docs.py`, `extract_caps.py`, `cards.py`, `src/surf/index/frontmatter.py` (new, §4.5)

---

## 1. Purpose and scope

Turn discovered files, parsed schema and capability configs into `Card` records (00 §3) whose `card` text is the only thing the judge ever sees about a surface. Cards are deterministic, budgeted and sanitized.

| In scope (v1) | Out of scope (v1) |
|---|---|
| Code file, doc file, code/doc dir cards | Symbols, header comments (spec §7.3, v1.1) |
| MCP server (static, live, described), skill, subagent, command cards | Calling MCP tools, OAuth flows |
| Token approximation, budgets, deterministic truncation | Table / migration / schema-root card *content* (specified in 03; rendered by functions in `cards.py`) |
| Card `hash`, build order w.r.t. edges | Edge computation (04) |
| Project descriptor auto-derivation (spec §16) | Card storage (05) |

## 2. Interfaces

```python
# extract_code.py
def extract_code_file(entry: FileEntry) -> CodeFacts: ...          # no I/O; uses FileEntry.lines

# extract_docs.py
def extract_doc_file(root: RootInfo, entry: FileEntry) -> DocFacts: ...

# extract_caps.py
def extract_capabilities(sources: Sequence[CapSource], listings: McpListings,
                         cfg: CapabilitiesConfig) -> list[CapFacts]: ...
async def list_mcp_server(defn: McpServerDef, env: Mapping[str, str], *,
                          timeout_s: float) -> McpListing | ListingFailure: ...
def load_listings(path: Path) -> McpListings: ...                  # .surf/mcp-listings.json
def save_listings(path: Path, listings: McpListings) -> None: ...  # sorted, sanitized

# cards.py
def approx_tokens(text: str) -> int: ...
def fit_card(header: str, sections: Sequence[Section], budget: int) -> str: ...
def card_hash(type_: SurfaceType, inputs: Mapping[str, Any]) -> str: ...
def render_all(facts: IndexFacts, edges: EdgeLookup, churn: ChurnCounts,
               cfg: IndexConfig) -> list[Card]: ...
def derive_descriptor(facts: IndexFacts) -> str: ...
```

`render_all` is called by `index/build.py` after the edge builder (§4.1). `EdgeLookup` and `ChurnCounts` are protocols owned by 04:

```python
class EdgeLookup(Protocol):
    def top(self, sid: SurfaceId, kind: EdgeKind, n: int, *, direction: Literal["out","in","any"]
            ) -> list[tuple[SurfaceId, float]]: ...   # sorted weight desc, id asc; rounded weights
class ChurnCounts(Protocol):
    def file_commits(self, path: str) -> int | None: ...   # None when no git
    def dir_commits(self, dir_path: str) -> int | None: ...# distinct commits touching the subtree
```

## 3. Data structures

Per-type facts are intrinsic (no edge data). `Card.fields` is `FieldsModel.model_dump(mode="json", exclude_none=True)`.

```python
class CodeFacts(BaseModel, frozen=True):
    path: str; lang: str; lines: int | None; oversize: bool

class DocFacts(BaseModel, frozen=True):
    path: str; title: str; title_source: Literal["frontmatter","h1","filename"]
    headings: list[str]            # already selected + capped (§4.4)

class CapFacts(BaseModel, frozen=True):
    id: SurfaceId; type: SurfaceType; name: str
    description: str | None        # skills/agents/commands
    mcp: McpStatic | None          # transport, pkg_or_host
    listing: McpListing | None     # live listing, if any and not stale-marked
    purpose: str | None            # capabilities.describe[name]
    sources: list[CapSourceRef]    # (harness, level, display_path), sorted
    conflict: bool                 # same name, differing definitions (§4.8)
    committable: bool

class CodeFileFields(BaseModel):   # fields for code_file
    lang: str; lines_bucket: str; oversize: bool | None
    tables: list[str]; changes_with: list[str]; churn: Literal["low","med","high"] | None
class DocFileFields(BaseModel):
    title: str; headings: list[str]; tables: list[str]; changes_with: list[str]; churn: ... | None
class DirFields(BaseModel):
    files_total: int; langs: list[str]; child_dirs: list[str]; child_files: list[str]
    tables: list[str]; doc_titles: list[str]; coupled_dirs: list[str]; churn: ... | None
class CapFields(BaseModel):
    name: str; sources: list[dict]; description: str | None
    transport: str | None; target: str | None; tools: list[str] | None; tool_count: int | None
    listing: Literal["static","live","described"] | None; conflict: bool | None
```

A `Section` for `fit_card`:

```python
class Section(BaseModel, frozen=True):
    label: str                 # "tables", "changes with", "sections", …
    items: list[str]           # already ordered by relevance
    sep: str = ", "            # " · " for doc headings
    min_items: int             # truncation floor before the section is dropped
    drop_rank: int             # lower = shrunk/dropped first
    total: int | None = None   # true count if > len(items) (for "+N")
```

## 4. Behavior / algorithm

### 4.1 Build order and what goes into `hash`

Card text contains edge-derived fields (`tables`, `changes_with`, `coupled_dirs`, `churn`-based ordering), so rendering must follow edges:

```
1. discover                              (01)
2. extract intrinsic facts               code, docs, schema (03), capabilities
   → intrinsic hash per surface          (§4.9; no edge data)
3. build edges                           (04) containment, co_change (+churn counts),
                                         dir_coupling, schema_ref (needs table set from 03),
                                         fk / defined_in / alias (from 03 facts)
4. render leaf cards                     code, doc, table, migration, capability
5. render dir cards bottom-up            post-order over DirEntry (deepest path first),
                                         then schema root (03)
6. hand Card list to catalog store       (05)
```

**`hash` = intrinsic inputs only** (spec §7.8's list, minus edge-derived data). **`card` = intrinsic + edge-derived.** Consequences, handed to 06:

| Event | `hash` changes? | `card` changes? | Refresh action |
|---|---|---|---|
| Whitespace edit, same line bucket | no | no | nothing |
| Edit crossing a line bucket | yes | yes | re-render node + ancestors |
| New co-change partner / table reference | no | yes | re-render node (spec §10.2 step 7) |
| Renderer change (`CARD_FORMAT_VERSION` bump) | yes, all | yes, all | full rebuild |

Lease staleness (spec §10.3, 10-lease) keys on `hash` changes and deletions, so an edge-only change doesn't mark a lease stale.

### 4.2 Code file cards (spec §7.3)

`lang`: extension map in `extract_code.LANG_BY_EXT` (lowercased extension → short code: `ts tsx js jsx mjs cjs py go rs java kt rb php cs cpp c h hpp swift scala sql sh yaml json toml html css scss vue svelte proto graphql tf prisma` …) plus `LANG_BY_NAME` for extensionless names (`Dockerfile→docker`, `Makefile→make`, `Gemfile/Rakefile→ruby`, `Procfile→procfile`). Unknown: the lowercased extension without the dot (cap 10 chars), or `text` if none. This map also defines "source-code extension" for 01 D-01-2.

`lines_bucket`: `<50`, `<200`, `<500`, `<1500`, `1500+` over `FileEntry.lines`; oversize → `1500+`.

`churn` (see §4.7). `tables`: `edges.top(id, SCHEMA_REF, 3, direction="any")` mapped to table display names (03: unqualified for the default schema). `changes_with`: `edges.top(id, CO_CHANGE, 3)` rendered with `short_path` (§4.6).

Render:

```
{path} [{lang}, {lines_bucket} lines]
tables: {tables}
changes with: {changes_with}
```

Empty sections are omitted. Oversize files render `{path} [{lang}, large file]`. Budget 60.

### 4.3 Doc file cards (spec §7.4)

Parsing is line-based, over UTF-8 decoded with `errors="replace"`, skipping fenced code blocks (```` ``` ````/`~~~`) and HTML comments.

| Format | Title candidates | "H2" | "H3" |
|---|---|---|---|
| Markdown / MDX (`.md .mdx .markdown`, README etc.) | ATX `# x`, setext `x\n===` | `## x`, setext `---` | `### x` |
| reStructuredText | first adorned title (over+underline or underline) | 2nd distinct adornment style | 3rd |
| AsciiDoc | `= x` | `== x` | `=== x` |
| `.txt` (ADR) | first non-empty line | none | none |

MDX: lines starting with `import `/`export ` and JSX-only lines (`^\s*<[A-Z]`) are skipped.

- **title**: frontmatter `title` → first H1 → filename stem (D-02-4). `README*` with no title → `README`.
- **headings**: all H2s in order; if fewer than 3 H2s, H2s and H3s in document order; then the first 8.
- Heading cleanup: strip trailing `#`s, `{#anchor}`, inline code backticks, emphasis markers, links `[t](u)`→`t`, images, HTML tags; sanitize (§4.11); cap 60 chars (title 100). Duplicate headings are kept once (first occurrence). Empty after cleanup → dropped.

Render (budget 60):

```
{path} — "{title}"
sections: {h1} · {h2} · …
tables: {tables}
changes with: {changes_with}
```

### 4.4 Directory cards (spec §7.7)

Built after all child cards, using `DirEntry` (01) and child facts.

| Field | Rule (all ties → name/id asc) |
|---|---|
| `files_total` | `DirEntry.files_total` (tracked files only for committed cards, 01 §4.7) |
| `langs` | top 3 `lang` by recursive file count (docs count under their lang, e.g. `md`) |
| `child_dirs` | direct child dirs by recursive file count desc; up to 12; shown as `name/` |
| `child_files` | direct child files by churn rank desc (`high`>`med`>`low`>none), then line count desc, then name; up to 15; basenames |
| `tables` | sum of `schema_ref` weights over all files in the subtree, per table; top 5 |
| `doc_titles` | only for `doc:` dirs: recursive doc files by depth asc, churn rank desc, path asc; titles quoted; up to 6 |
| `coupled_dirs` | `edges.top(id, DIR_COUPLING, 3)`, full dir paths |
| `churn` | bucket of `dir_commits` (§4.7) |

Ordering files by line **count** (not bytes) keeps whitespace-only edits in one file from reordering its parent's card only when the line count really changes.

Render (budget 150):

```
{path} — {files_total} files · {langs joined ", "}
dirs: a/, b/, +N
files: x.ts, y.ts, +N
tables: …
docs: "Title A", "Title B"
changes with: src/api/orders/, supabase/migrations/
```

`+N` counts are exact direct-child counts beyond those listed. The root has no dir card (01 §4.6).

### 4.5 Frontmatter (`frontmatter.py`)

PyYAML isn't in spec §22.1, and full YAML (tags, anchors) is more attack surface than needed. A minimal parser handles the frontmatter we read (`title`, `name`, `description`, `argument-hint`):

- Block delimited by `---` on line 1 and the next `---`/`...` line; max 200 lines, else ignored.
- Top-level `key: value` only; values: plain, single/double-quoted (with `\"`, `''` escapes), block scalars `|`/`>` (with `-`/`+` chomping) collected by indentation. Lists and nested maps are skipped (kept as `None`).
- Unknown syntax never raises; the key is skipped. Duplicate keys: last wins.

TOML frontmatter (`+++`) is not supported (Q-02-4).

### 4.6 Short paths for partners

`changes_with` shows "basenames when unambiguous" (spec §7.3). Rule: `short_path(p)` = the shortest suffix of whole path segments of `p` that is a suffix of no other indexed file path (committed tree). Example: `src/api/orders/[id].ts` → `orders/[id].ts` if another `[id].ts` exists elsewhere but no other `orders/[id].ts`. Computed from a suffix → count map built once per build (O(total segments)). Directory partners (`coupled_dirs`) always use the full path. Trade-off: adding a same-named file elsewhere can lengthen other cards' partner strings; accepted, since it's deterministic and only affects text, not `hash`.

### 4.7 Churn buckets

`churn` is used to order `child_files`, walk chunks (09) and `doc_titles`. It isn't shown in card text.

Options considered:

| Option | Problem |
|---|---|
| Percentile within repo | A file's bucket changes when *other* files change, rewriting unrelated dir cards on every commit |
| Recency-weighted mass W(a) (04) | Changes for every file whenever HEAD moves (ages are relative to the index head, 00 §4 rule 4) |
| **Absolute raw commit count in the co-change window, after 04's commit filters** | Changes only when the file is touched or a commit leaves the window |

Chosen: absolute counts (D-02-2). Thresholds:

| Surface | `low` | `med` | `high` |
|---|---|---|---|
| file (`file_commits`) | 0–2 | 3–14 | ≥ 15 |
| directory (`dir_commits`, distinct commits in subtree) | 0–9 | 10–49 | ≥ 50 |
| table (migrations touching it, 03) | 0–1 | 2–4 | ≥ 5 |

No git → `churn = None`, and ordering falls back to the next key. Thresholds are constants (not config) because they change card text; revisit with eval data (Q-02-2).

### 4.8 Capability cards (spec §7.6)

#### 4.8.1 Parsing sources

| Source | Parse | Yields |
|---|---|---|
| Claude `.mcp.json`, `~/.claude.json`, Cursor `mcp.json` | JSON; `mcpServers: {name: {command,args,env} | {type:"http"|"sse", url, headers}}` | `McpServerDef` |
| Codex `config.toml` | `tomllib`; `[mcp_servers.<name>]` with `command,args,env` or `url`; `enabled = false` skipped | `McpServerDef` |
| OpenCode `opencode.json(c)` | JSONC (strip `//` and `/* */` outside strings, trailing commas); `mcp: {name: {type:"local", command:[…], environment} | {type:"remote", url, headers}}`; `enabled: false` skipped | `McpServerDef` |
| Claude settings | `disabledMcpjsonServers` removes those `.mcp.json` names; other keys ignored | filter |
| `SKILL.md` | frontmatter `name`, `description`; body | skill |
| agent `.md` | frontmatter `name`, `description` | subagent |
| command `.md` / Codex prompt | frontmatter `description` (fallback body) | command |
| OpenCode inline `agent`/`command` keys | `description` field | subagent / command |

#### 4.8.2 Names and ids

| Type | Name | Id |
|---|---|---|
| MCP | config key, NFC, stripped | `mcp:{name}`; plugin: `mcp:{plugin}:{name}` |
| skill | frontmatter `name` if it matches `[A-Za-z0-9][A-Za-z0-9_.-]{0,63}`, else directory name | `skill:{name}`; plugin: `skill:{plugin}:{name}` |
| subagent | same rule, else file stem | `agent:{name}` |
| command | relative path under the commands dir without `.md` (`frontend/component`) | `cmd:{rel}`; card shows `/component (frontend)` |

Names containing control characters or longer than 64 chars → source skipped with a warning.

#### 4.8.3 Collisions and precedence

Several sources can define the same id (same MCP server in `.mcp.json` and `.cursor/mcp.json`; a skill read by both Claude and OpenCode). One card per id:

1. Collect all definitions per id.
2. Dedupe definitions pointing at the same resolved file.
3. Winner by `(level rank, harness rank, display_path)`: level `project` < `project_local` < `user` < `plugin`; harness in `capabilities.harnesses` order.
4. `fields.sources` lists every definition `(harness, level, display_path)`, sorted.
5. MCP definitions are **equivalent** if transport and `target` (§4.8.4) match. Non-equivalent → `conflict = True`, winner's definition used, `doctor` warns. No suffixed ids: the note names a capability, and the agent resolves it by name in whatever harness it runs.
6. `committable` = winner's source is committable **and** no user-level definition was merged (otherwise the card goes to the overlay, 01 D-01-1). With `user_level = false`, only project sources exist.

#### 4.8.4 MCP modes

- **Static (default):** `transport` ∈ `stdio|http|sse`; `target` = for stdio the package token (first non-flag argument after a launcher `npx|bunx|pnpm dlx|yarn dlx|uvx|pipx run|docker run`, else the command basename), for HTTP the URL host only (no scheme, path, query, credentials). Arguments, env and headers are never read into facts.
- **Described:** `capabilities.describe[name]` (one line, sanitized, cap 160 chars) → `purpose`.
- **Live (opt-in):** §4.8.5. Listing results come from `.surf/mcp-listings.json`; `surf index` itself never contacts a server.

Render (budget 120):

```
mcp {name} — tools: {t1}, {t2}, …, +N           # live
mcp {name} — {transport} server ({target})       # static
purpose: {purpose or first sentence of listing.instructions, cap 160}
```

Skills/agents/commands: `skill {name} — {description}`, `agent {name} — {description}`, `cmd /{leaf} ({namespace}) — {description}`. Description = frontmatter `description`, else the first body paragraph that isn't a heading, capped at 200 chars (spec §7.6). No description → `skill {name}` only.

#### 4.8.5 Live MCP listing

Triggered only by `surf init --live-mcp` (per-server y/N prompt) or `surf index --live-mcp [NAME…]`, for servers named in `capabilities.live_mcp`.

| Concern | Rule |
|---|---|
| Protocol | MCP Python SDK client. Send `initialize` (client capabilities **empty**: no roots, sampling or elicitation), `notifications/initialized`, `tools/list` (follow `nextCursor`, max 10 pages). Nothing else. `tools/call`, `resources/*`, `prompts/*` are never sent; server→client requests are answered with a JSON-RPC error. |
| stdio process | `command` + `args` from config, `cwd` = repo root, own process group, stdin/stdout pipes, stderr to a 64 KiB ring buffer that is discarded (never logged: it can echo tokens). |
| Env | `capabilities.live_env = "inherit"` (default: parent env, as harnesses do) or `"minimal"` (PATH, HOME, USER, LANG, LC_*, TMPDIR/TEMP/TMP, SYSTEMROOT, APPDATA, USERPROFILE, XDG_*, proxy and CA vars). Config `env` values are expanded (`${VAR}`, `${VAR:-default}`) and added. Secrets exist only in the child env; nothing from env, args or headers is persisted or logged. |
| HTTP / SSE | Configured headers expanded the same way. 401/403 or `WWW-Authenticate` → `needs_auth`; no OAuth flow is started. TLS verification on; proxy env respected. |
| Timeouts | `initialize` ≤ `capabilities.live_timeout_ms` (10 s), `tools/list` total ≤ same, hard cap 20 s per server. Then SIGTERM to the group, 2 s, SIGKILL. Servers listed concurrently, max 4. |
| Failures | `timeout`, `spawn_error`, `needs_auth`, `protocol_error` → no listing; card stays static; `surf init` asks for the one-line purpose (spec §7.6 step 4). |
| Foundations exception | 00 §6 allows only `git`/`rg` subprocesses. Live listing spawns user-configured servers; it lives in `extract_caps.py`, runs only on explicit opt-in commands, and is flagged for 00. |

Persisted `McpListing` (in `.surf/mcp-listings.json`, committed when the server's source is committable; otherwise `.surf/cache/mcp-listings.json`):

```json
{"version":1,"servers":{"supabase":{"def_fp":"a1b2c3d4e5f6a7b8","server":{"name":"supabase-mcp","version":"0.4.1"},
 "instructions":"…≤1000 chars, sanitized…","tool_count":9,
 "tools":[{"name":"apply_migration","description":"…≤300 chars…"}, …]}}}
```

- Tools sorted by name, deduped, max 200 stored (`tool_count` is the true count). Tool names must match `[A-Za-z0-9_.:/-]{1,64}`, else dropped.
- Descriptions and instructions pass §4.11 sanitization. No timestamps, no args, no env, no URLs.
- `def_fp` = first 16 hex of sha256 over `(transport, target)` only, so a secret-bearing arg can't be brute-forced from it. When the current definition's `def_fp` differs, the listing is ignored for the card (static render) and `doctor` reports "listing stale, re-run `surf index --live-mcp NAME`".
- The file is an **input** to the index, like a lockfile: `surf index --check` reads it and never re-lists, so CI is reproducible without network or secrets.

### 4.9 Hash

```python
CARD_FORMAT_VERSION = 1
def card_hash(type_, inputs) -> str:
    payload = {"v": CARD_FORMAT_VERSION, "type": type_.value, **inputs}
    blob = json.dumps(payload, sort_keys=True, separators=(",", ":"), ensure_ascii=False)
    return "sha256:" + hashlib.sha256(unicodedata.normalize("NFC", blob).encode()).hexdigest()
```

| Type | Inputs |
|---|---|
| code_file | `path, lang, lines_bucket, oversize` |
| doc_file | `path, title, headings` (post-cleanup, pre-budget) |
| code_dir / doc_dir | `path, prefix, children: sorted [(child_id, child_hash)]` (Merkle: any descendant change changes every ancestor) |
| capability | `id, name, description, transport, target, purpose, listing (tools+instructions) or null, sources` |
| db_table, schema_root, db_migration | 03 §4.7 |

Spec §7.8 includes "extracted tables" in the file hash; tables are edge-derived here and excluded (D-02-3). Change **detection** doesn't use `hash` alone: git diff, or the cache fingerprint (01 §3.3) for non-git/untracked files, decides what to re-extract; `hash` then decides whether the card and its ancestors changed.

### 4.10 Project descriptor (spec §16)

If `project.descriptor` is unset: `"{L1} + {L2} project"` from the top two code languages by file count (display names: `TypeScript`, `Python`, …; one language → `"{L1} project"`), plus `", {stack} schema"` when 03 found a schema source (`Supabase`, `Prisma`, `Rails`, `Alembic`, `SQL`). Example: `"TypeScript + SQL project, Supabase schema"`. Stored in `meta.json` (05), not in a card. Deterministic; ties by name.

### 4.11 Sanitization

`sanitize(s, cap)` applied to every free-text item (titles, headings, descriptions, tool descriptions, instructions, purposes):

1. NFC; drop C0/C1 controls except space, bidi overrides (U+202A–202E, U+2066–2069), zero-width chars (U+200B–200D, U+FEFF).
2. Collapse whitespace runs to one space; strip.
3. `redact.redact(s)` (14) → matches become `[REDACTED:type]`. For **list items** (headings, tool names) an item containing a redaction is dropped instead (spec §7.2 "drop").
4. Truncate to `cap` chars at a word boundary if one exists within the last 20 chars, append `…`.

Paths and identifiers (file names, table names, capability names) are not redacted: pointers must be exact, and secret-like files are already excluded (01). They are control-char stripped only.

### 4.12 Budgets and `fit_card`

**Token approximation** (D-02-1): `approx_tokens(s) = ceil(len(s.encode("utf-8")) / 4)`. Deterministic, no dependency. It over-counts non-ASCII (conservative) and roughly matches BPE tokenizers on paths and identifiers. It's a budget unit, not a cost estimate; `surf stats` can report real tokenizer counts in dev (Q-02-1).

| Card | Budget (approx tokens) | ≈ bytes |
|---|---|---|
| code_file, doc_file, db_migration | 60 | 240 |
| code_dir, doc_dir, schema_root | 150 | 600 |
| capability, db_table | 120 | 480 |

`fit_card(header, sections, budget)`:

```
render = header + "\n".join(f"{s.label}: {join(s.items, s.sep)}{plus(s)}" for s in sections if s.items)
while approx_tokens(render) > budget:
    s = min((s for s in sections if len(s.items) > s.min_items), key=drop_rank, default=None)
    if s: s.items.pop(); continue            # "+N" = (total or original len) − len(items)
    s = min((s for s in sections if s.items), key=drop_rank, default=None)
    if s: s.items.clear(); continue          # drop whole section
    header = truncate_chars(header, budget*4) ; break   # only a pathological path gets here
```

Drop ranks (lower shrinks first): file/doc cards `changes with`=1, `tables`=2, `sections`=3; dir cards `files`=1, `dirs`=2, `docs`=3, `changes with`=4, `tables`=5; MCP `tools`=1, `purpose`=2. Minimums: lists 2, `files` 5, `dirs` 4. The header line is never dropped. Every card that needed a header truncation is logged in the build report.

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `project.descriptor` | str \| None | None | spec §16; auto-derived when None |
| `capabilities.live_mcp` | list[str] | `[]` | spec §16 |
| `capabilities.describe` | dict[str,str] | `{}` | spec §16 |
| `capabilities.live_timeout_ms` | int | 10000 | new |
| `capabilities.live_env` | `"inherit"\|"minimal"` | `"inherit"` | new |
| `capabilities.harnesses`, `capabilities.user_level` | | | see 01 §5 |

Budgets, churn thresholds and the token divisor are code constants tied to `CARD_FORMAT_VERSION`, not config, because changing them rewrites every card.

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| Doc with no headings, no frontmatter | `title` = filename stem; no `sections` line |
| Doc > `max_file_bytes` | Card with filename title only |
| Invalid UTF-8 in doc | `errors="replace"`, then sanitized (U+FFFD kept) |
| Frontmatter unterminated | Treated as body |
| Heading is a secret-like token | Dropped (§4.11 step 3) |
| Adversarial heading ("ALWAYS relevant…") | Kept, capped at 60 chars; eval fixture checks it doesn't dominate (spec §19.4) |
| Skill dir without `SKILL.md` | No card |
| SKILL.md `name` ≠ dir name | Frontmatter name wins if valid; warning |
| Two skills with the same name (project + user) | Precedence §4.8.3 |
| MCP server in `.mcp.json` disabled in settings | No card |
| Same MCP name, different commands | One card, `conflict=true`, doctor warning |
| MCP config file malformed JSON/TOML | Source skipped, warning with display path; other sources continue |
| `${VAR}` unset during live listing | Expands to empty (or default); listing may fail → static |
| Live server prints to stdout before JSON-RPC | SDK protocol error → `protocol_error` |
| Live server asks for sampling/roots | Error response; listing continues |
| Live listing > 200 tools | 200 stored, `tool_count` true |
| Oversize or binary file | Oversize: §4.2; binary never reaches cards |
| Dir with 500 direct files | `files:` shows 15 (fewer if budget), `+485` |
| Path longer than budget (240 bytes) | Header truncated with `…` (logged); id and `path` field keep the full path |
| No git | `churn` omitted; `changes_with`, `coupled_dirs` empty |

## 7. Performance budget

| Step (5k files, 300 docs, 30 capabilities) | Budget |
|---|---|
| Code facts | ≤ 50 ms (no I/O) |
| Doc parse | ≤ 1 ms/doc avg → ≤ 0.5 s |
| Capability parse (static) | ≤ 100 ms |
| Render all cards + dir rollup | ≤ 1 s |
| Live listing | ≤ 20 s per server, 4 concurrent; not part of `surf index` |

Refresh re-renders only changed nodes, their ancestors, and nodes whose edges changed; that's ≤ 50 ms for a typical commit.

## 8. Test plan

**Unit**
- `approx_tokens`: ASCII, multibyte, empty.
- `fit_card`: property test (`hypothesis`) that output ≤ budget unless only the header remains; "+N" counts are exact; deterministic under repeated calls.
- Doc parsing: table of Markdown/MDX/RST/AsciiDoc snippets → (title, headings), including setext, fenced blocks containing `#`, HTML comments, H3 fallback, frontmatter precedence.
- `frontmatter.py`: quoted, block scalars, nested (skipped), duplicates, unterminated, 201-line block.
- `short_path`: uniqueness cases incl. `[id].ts` in several dirs.
- Churn thresholds at 2/3/14/15.
- Capability precedence and conflict matrix; disabled servers; plugin namespacing; command ids with sub-dirs.
- Sanitization: bidi/zero-width, redaction drop vs in-place, word-boundary truncation.
- Hash: whitespace-only edit same hash; bucket crossing changes hash and all ancestor dir hashes; edge change leaves hash and changes `card`.

**Live MCP** (integration, offline): a fake stdio server script (Python, in `tests/fixtures/mcp/`) with modes: normal, slow (timeout), >200 tools, paginated, sends a `sampling/createMessage` request, logs a fake token to stderr, exits early. Assertions: no `tools/call` ever sent (server fails the test if received), stderr token never appears in output/logs, persisted file byte-identical across two runs, process group killed on timeout. An HTTP fake returns 401 → `needs_auth`.

**Golden**: the `feature-organized` and `layered` fixture repos' full catalogs (all card texts) checked in; any renderer change must update them with `CARD_FORMAT_VERSION` bumped.

**Determinism**: build twice, and build with shuffled discovery order → byte-identical `catalog.jsonl`.

## 9. Acceptance criteria

1. Phase 1 exit: 100 % of cards in both target repos are within budget by `approx_tokens` (header-truncation cases listed and reviewed; target 0).
2. Card texts match the spec's examples in shape (§7.3, §7.4, §7.6, §7.7) on the golden fixtures.
3. Two builds of the same commit produce identical `card` and `hash` for every surface; a whitespace-only commit changes no `hash`.
4. `surf index --check` passes in CI with live listings present and no network access.
5. No card contains a string matched by `redact.py` patterns (scan over golden and target catalogs).
6. The live-listing fake server never receives a method other than `initialize`, `notifications/initialized`, `tools/list`.

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-02-1 | Budgets in "tokens", no tokenizer | `ceil(utf8_bytes / 4)` | Deterministic, dependency-free, conservative for non-ASCII |
| D-02-2 | `churn` buckets `low/med/high`, undefined | Absolute filtered commit counts in the window with fixed thresholds; not rendered in text | Percentiles and decayed weights change cards on unrelated commits |
| D-02-3 | File hash includes extracted tables (§7.8) | `hash` = intrinsic inputs only; tables, partners, coupled dirs affect `card` text only | Tables come from schema-ref edges (§8.3), which are computed after extraction; a single definition keeps lease staleness meaningful |
| D-02-4 | Doc title: first H1, else frontmatter | Frontmatter `title` first | Static-site docs (Docusaurus, MkDocs) set the title in frontmatter and often have no H1 or a different one |
| D-02-5 | Live MCP results stored in the catalog, re-listed at init | Persisted to `.surf/mcp-listings.json` (new file, lockfile-like); index reads it, never lists | `--check` must be reproducible without network or secrets |
| D-02-6 | Record server command or URL | Only a package token or URL host (`target`) | Args, env and URLs routinely carry tokens |
| D-02-7 | Silent on name collisions across harnesses | One card per id, precedence + `conflict` flag, no suffixed ids | The note names capabilities the agent resolves by name |
| D-02-8 | Silent on dir hash | Merkle over children | Lets refresh and leases detect subtree changes cheaply |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-02-1 | Is bytes/4 close enough to Jev's real tokenization? | Yes; report real counts in `surf stats` if a tokenizer is available | Measure on target catalogs; switch divisor with a format-version bump if error > 25 % |
| Q-02-2 | Churn thresholds 3/15 (files) | As §4.7 | Distribution on the two target repos |
| Q-02-3 | Should `churn` or line counts appear in card text for Jev? | No | A/B on walk recall (spec §17.6) |
| Q-02-4 | TOML (`+++`) frontmatter for Hugo docs | Not supported | User demand |
| Q-02-5 | Plugin-provided capability id namespacing (`skill:{plugin}:{name}`) | As §4.8.2 | Match Claude Code's displayed names once verified |
| Q-02-6 | Should a stale live listing still render (with a marker) instead of falling back to static? | Fall back to static | Eval of capability accuracy with stale listings |
