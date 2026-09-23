# 05 · Catalog store and meta

**Status:** draft for review
**Spec sections:** §9 (9.1–9.5), §10.1 (atomicity), §15.2 `surface_info`, §11.3 (path lookup needs)
**Depends on:** 00-foundations (types, determinism rules), 04-graph-edges (edge conventions), 06-refresh (writers, locking, commit-policy decision)
**Code:** `surf/catalog/store.py`, `surf/catalog/meta.py`

---

## 1. Purpose and scope

The on-disk catalog: canonical JSONL files (source of truth), `meta.json`, and the derived SQLite read cache that every query-time component uses.

| In scope | Out of scope |
|---|---|
| File layout under `.surf/` for both commit policies | Deciding *when* to rebuild (06) |
| Canonical serialisation of `Card`, `Edge`, `Meta` | Card and edge content (02, 03, 04) |
| SQLite schema, build, open, and version checks | Lease files (10), decision log (15) |
| Read API used by router, path matching, MCP `surface_info`, eval | Build-state caches' contents (06 owns `build.sqlite` semantics; this doc gives its location) |
| Integrity checks and fail-open on corruption | |

---

## 2. Interfaces

```python
# surf/catalog/store.py
class CatalogPaths(BaseModel, frozen=True):
    root: Path                         # repo root
    surf_dir: Path                     # <root>/.surf
    catalog_dir: Path                  # always .surf/cache/ : the working catalog every reader uses (§3.1)
    baseline_dir: Path | None          # .surf/ when index.commit_catalog = true, else None
    cache_dir: Path                    # .surf/cache/ (same as catalog_dir in v1; kept separate for clarity)
    @classmethod
    def resolve(cls, root: Path, cfg: Config) -> CatalogPaths: ...

class EdgeView(NamedTuple):
    other: SurfaceId                   # the neighbour, whichever end it was stored on
    kind: EdgeKind
    weight: float
    outgoing: bool                     # True if stored as (this -> other)
    evidence: Mapping[str, Any]

class CatalogReader(Protocol):
    meta: Meta
    def card(self, sid: SurfaceId) -> Card | None: ...
    def cards(self, sids: Iterable[SurfaceId]) -> dict[SurfaceId, Card]: ...
    def children(self, sid: SurfaceId) -> list[Card]: ...          # tree children, sorted by id; sid may be "root:"
    def subtree_files(self, dir_id: SurfaceId) -> list[SurfaceId]: ... # recursive file ids (flatten), sorted
    def edges_from(self, sid: SurfaceId, kinds: Collection[EdgeKind] | None = None, *,
                   direction: Literal["out", "in", "both"] = "both",
                   limit_per_kind: int | None = None) -> list[EdgeView]: ...  # weight desc, other asc
    def id_for_path(self, path: str) -> SurfaceId | None: ...      # tree node only (code:/doc:), exact
    def ids_for_suffix(self, suffix: str, *, casefold: bool = False) -> list[SurfaceId]: ...
    def ids_for_basename(self, name: str, *, casefold: bool = False) -> list[SurfaceId]: ...
    def capabilities(self) -> list[Card]: ...
    def content_cards(self) -> Iterator[Card]: ...                 # small-repo mode, eval A0
    def content_card_count(self) -> int: ...
    def close(self) -> None: ...

def open_catalog(paths: CatalogPaths, *, allow_rebuild: bool = True) -> CatalogReader | None
def write_catalog(paths: CatalogPaths, cards: Sequence[Card], edges: Sequence[Edge], meta: Meta) -> None
def build_cache(paths: CatalogPaths, cards: Sequence[Card], edges: Sequence[Edge], meta: Meta,
                *, file_state: Iterable[FileState] = ()) -> Path   # writes index.sqlite.tmp, returns it
def publish(paths: CatalogPaths, staged: StagedBuild) -> None       # atomic replace sequence (§4.4)
def load_jsonl(paths: CatalogPaths) -> tuple[list[Card], list[Edge], Meta]   # verifies meta.files digests
def serialize_card(c: Card) -> bytes
def serialize_edge(e: Edge) -> bytes

# surf/catalog/meta.py
def read_meta(path: Path) -> Meta | None
def meta_equal_for_check(a: Meta, b: Meta) -> list[str]            # differing field names, ignoring volatile ones
def content_fingerprint(entries: Iterable[tuple[str, str]]) -> str  # (path, blob-or-content sha) pairs
```

`open_catalog` returns `None` when no usable index exists. The router maps that to `index-missing` (fail open). It never raises for missing or corrupt data.

---

## 3. Data structures

### 3.1 Layout

Default (`index.commit_catalog = false`, the recommendation in 06 §4.1 / Q-06-1):

```
.surf/
├── config.toml          committed
├── eval/{dev,test}.yaml committed
├── .gitignore           committed, written by init: "cache/\nlogs/\n"
├── cache/               ignored
│   ├── catalog.jsonl    canonical catalog (local build)
│   ├── edges.jsonl
│   ├── meta.json
│   ├── index.sqlite     read snapshot, replaced atomically
│   ├── build.sqlite     build state: extraction cache, file fingerprints, schema mentions (06)
│   ├── cochange.state   gzip JSONL: CochangeWindow (04 §3)
│   ├── refresh.lock     flock target (06)
│   ├── refresh.pending  coalescing marker (06)
│   └── leases/
└── logs/                ignored
    ├── decisions.jsonl
    └── refresh.log
```

With `index.commit_catalog = true`, `catalog.jsonl`, `edges.jsonl` and `meta.json` **also** exist at `.surf/` as the committed **baseline**. The local working copy is always in `cache/` and is what the router reads (06 §4.1 option c′). `CatalogPaths.catalog_dir` is always `cache/` for reads. The committed path is written only by `surf index --baseline`.

### 3.2 Card and edge records

- Card: foundations `Card` fields in model order `id, type, path, parent, card, fields, hash`. **No `indexed_at`** (D-F5).
- Edge: `from, to, kind, weight, evidence`. Symmetric kinds stored once with `from < to`. `contains` not stored (04 D-04-1).
- `mig:` cards and capability cards have `parent: null`.

### 3.3 Meta (`meta.json`)

```python
class FileDigest(BaseModel, frozen=True):
    sha256: str
    lines: int

class Meta(BaseModel, frozen=True):
    schema_version: int = 1
    surf_version: str                                  # volatile for --check (warn only)
    index_head: str | None                             # full 40-hex commit whose tree was indexed; None = non-git
    worktree_dirty: bool                               # True if uncommitted content was indexed
    content_fingerprint: str                           # sha256 over sorted (path, content id) of indexed files
    config_fingerprint: str                            # sha256 of index-affecting config (canonical JSON)
    cochange_head: str | None
    cochange_ref_time: int | None                      # ct(cochange_head)
    cochange_status: Literal["ok", "shallow", "no_git", "disabled", "empty"]
    git_version: str | None                            # volatile (warn only)
    counts: dict[SurfaceType, int]                     # sorted keys
    content_cards: int
    walk_mode: Literal["flat", "walk"]
    files: dict[str, FileDigest]                       # "catalog.jsonl", "edges.jsonl"
    warnings: list[str]                                # sorted, deterministic (unparseable migrations, …)
    built_at: str                                      # RFC 3339 UTC — volatile
    build: dict[str, Any]                              # {"kind": "full"|"incremental", "ms": int} — volatile
```

Volatile fields (`built_at`, `build`, `surf_version`, `git_version`) are ignored by `--check` equality. The spec's short hash (`"a1b2c3d"`) becomes a full hash (D-05-3).

`content_fingerprint`: for each indexed file, `(path, id)` where `id` is the git blob sha from `git ls-files -s` for clean tracked files, or `git hash-object`-equivalent sha1 of the working-tree bytes for dirty and untracked files. Non-git: sha256 of the bytes. Sorted by path, joined `path\0id\n`, sha256. Because git blob ids are used for clean files, this is computable in CI from a checkout without reading every file.

### 3.4 Serialisation (determinism rules, foundations §4)

```python
def _canon(v):   # nested dicts sorted by key; lists preserved; floats already rounded by producers
    return {k: _canon(v[k]) for k in sorted(v)} if isinstance(v, dict) else \
           [_canon(x) for x in v] if isinstance(v, list) else v

def serialize_card(c):
    d = c.model_dump(mode="json", by_alias=True)
    d["fields"] = _canon(d["fields"])
    return (json.dumps(d, ensure_ascii=False, separators=(",", ":"), allow_nan=False) + "\n").encode()
```

- Top-level keys in model order; nested `fields`/`evidence` keys sorted.
- `weight` is rounded to 4 dp in the `Edge` validator (`round(x, 4)`), so every producer is covered.
- Files: UTF-8, `\n`, trailing newline, sorted (`catalog` by id; `edges` by `(from, to, kind)`), with byte-wise ordering of the UTF-8 strings (`sorted(key=lambda s: s.encode())`) so it doesn't depend on locale.
- `write_catalog` validates before writing: sorted and unique ids; unique `(from, to, kind)`; every edge endpoint exists as a card; every `parent` exists or is `root:`; weights in [0, 1]. A violation is a bug and raises (index time may fail).

### 3.5 SQLite cache (`cache/index.sqlite`)

```sql
PRAGMA user_version = 1;           -- bumped on any DDL change; mismatch => rebuild
PRAGMA journal_mode = OFF;         -- built once into a tmp file, never mutated after publish
PRAGMA page_size = 4096;

CREATE TABLE meta_kv (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL              -- JSON; includes "meta" (full meta.json) and "source_digests"
) WITHOUT ROWID;

CREATE TABLE cards (
  id          TEXT PRIMARY KEY,
  type        TEXT NOT NULL,
  path        TEXT,                -- NULL for db / capability cards
  parent      TEXT,                -- NULL for mig: and capabilities; 'root:' for top level
  is_dir      INTEGER NOT NULL,    -- 1 for code_dir, doc_dir, schema_root
  is_content  INTEGER NOT NULL,    -- 0 for capability types
  in_tree     INTEGER NOT NULL,    -- 0 for mig: and capabilities
  files_total INTEGER NOT NULL,    -- recursive (dirs), 1 (files), table count (db:*)
  churn       INTEGER NOT NULL DEFAULT 0,   -- eligible commits (04); used for chunk ordering
  hash        TEXT NOT NULL,
  card        TEXT NOT NULL,
  fields      TEXT NOT NULL        -- canonical JSON
) WITHOUT ROWID;
CREATE INDEX cards_parent ON cards(parent, id) WHERE in_tree = 1;
CREATE INDEX cards_type   ON cards(type, id);

-- tree nodes only (code:/doc:), one row per path; migration files resolve to their code: id
CREATE TABLE paths (
  path        TEXT PRIMARY KEY,    -- 'src/api/' for dirs, 'src/a.ts' for files
  id          TEXT NOT NULL,
  basename    TEXT NOT NULL,       -- last segment without trailing '/'
  basename_cf TEXT NOT NULL,       -- casefold(basename)
  path_cf     TEXT NOT NULL
) WITHOUT ROWID;
CREATE INDEX paths_basename    ON paths(basename);
CREATE INDEX paths_basename_cf ON paths(basename_cf);
CREATE INDEX paths_path_cf     ON paths(path_cf);

-- every segment-aligned suffix of every FILE path: 'src/a/b.ts' -> 'src/a/b.ts','a/b.ts','b.ts'
CREATE TABLE suffixes (
  suffix     TEXT NOT NULL,
  suffix_cf  TEXT NOT NULL,
  id         TEXT NOT NULL,
  nseg       INTEGER NOT NULL,     -- segments in the suffix
  PRIMARY KEY (suffix, id)
) WITHOUT ROWID;
CREATE INDEX suffixes_cf ON suffixes(suffix_cf);

-- both directions of every stored edge + contains edges derived from cards.parent
CREATE TABLE edges (
  src       TEXT NOT NULL,
  dst       TEXT NOT NULL,
  kind      TEXT NOT NULL,
  weight    REAL NOT NULL,
  outgoing  INTEGER NOT NULL,      -- 1: stored as src->dst ; 0: materialised reverse
  evidence  TEXT NOT NULL,         -- canonical JSON, '{}' if empty
  PRIMARY KEY (src, kind, dst, outgoing)
) WITHOUT ROWID;
CREATE INDEX edges_by_weight ON edges(src, kind, weight DESC, dst);

-- non-git change detection and build bookkeeping snapshot (06); may be empty
CREATE TABLE file_state (
  path     TEXT PRIMARY KEY,
  size     INTEGER NOT NULL,
  mtime_ns INTEGER NOT NULL,
  content  TEXT NOT NULL           -- blob sha / sha256 as in content_fingerprint
) WITHOUT ROWID;
```

`meta_kv.source_digests` = `meta.files` of the JSONL the snapshot was built from. That's how a reader detects that the snapshot and the JSONL diverged (§4.3).

Symmetric kinds (`co_change`, `dir_coupling`, `alias`) produce two rows, both `outgoing=1`, because the relation has no direction. For directed kinds the reverse row has `outgoing=0`.

---

## 4. Behaviour

### 4.1 Read API semantics

| Call | Query | Notes |
|---|---|---|
| `card(id)` | `SELECT … FROM cards WHERE id=?` | In-process LRU (4,096 entries) in long-lived processes (MCP server) |
| `children("root:")` | `WHERE parent='root:' AND in_tree=1 ORDER BY id` | Includes `db:*` when tables exist |
| `children(dir)` | same with `parent=?` | The router re-sorts by `churn` for chunking |
| `subtree_files(code:src/x/)` | `SELECT id FROM paths WHERE path > 'src/x/' AND path < 'src/x0' AND path NOT LIKE '%/'` | `'0'` is the byte after `'/'`; `db:*` → all tables |
| `edges_from(id, kinds, direction)` | `WHERE src=? AND kind IN (…) [AND outgoing=?] ORDER BY kind, weight DESC, dst` | `contains` rows come from `cards.parent` at build time; `limit_per_kind` applied in Python |
| `id_for_path(p)` | `paths` PK | Exact, case-sensitive |
| `ids_for_suffix(s)` | `suffixes` PK or `suffix_cf` | 08 queries suffixes longest-first: first non-empty answer wins, more than one id = ambiguous |
| `ids_for_basename(b)` | `paths_basename[_cf]` | Files only |

Path matching (08) uses only `ids_for_suffix` and `ids_for_basename`. The suffix table costs Σ(depth) rows (≈ 5 × files) and makes absolute-prefix stripping a series of O(1) lookups.

### 4.2 Opening

```python
def open_catalog(paths, *, allow_rebuild=True):
    meta = read_meta(paths.catalog_dir / "meta.json")                  # None -> missing
    if meta is None or meta.schema_version != SCHEMA_VERSION: return None
    db = paths.cache_dir / "index.sqlite"
    conn = try_open_ro(db)                                              # uri 'file:…?mode=ro'
    if conn and user_version(conn) == SQLITE_VERSION:
        if source_digests(conn) == meta.files: return SqliteReader(conn, meta)
        return SqliteReader(conn, meta_from(conn), stale=True)          # consistent older snapshot; 06 refreshes
    if not allow_rebuild: return None
    return rebuild_under_lock_or_none(paths, timeout_ms=1500)           # §4.3
```

- A snapshot whose digests differ from the current `meta.json` is still **internally consistent** (it was built in one go). Serving it is better than failing. `stale=True` is exposed in `surf_status` and decision records.
- The reader holds one read-only connection per process. POSIX `os.replace` of the file leaves open readers on the old inode, so a route in flight is never torn.

### 4.3 Rebuilding the cache from JSONL

Used when `index.sqlite` is missing, has the wrong `user_version`, or is corrupt (`sqlite3.DatabaseError`), e.g. after a fresh clone in committed mode, or after the user deleted `cache/`.

1. Try to take `refresh.lock` without blocking. If it's held, another process is building: poll for `index.sqlite` for up to `timeout_ms`, then give up (`None`, meaning `index-missing` for this route).
2. `load_jsonl`: stream-parse, validate each line with pydantic, verify `meta.files[*].sha256`. On a digest mismatch (a torn write should be impossible given the §4.4 order, but a hand-edited file is possible), return `None` and schedule a full rebuild (06).
3. `build_cache` into `index.sqlite.tmp`: one transaction, `executemany`, create the indexes after the inserts, `ANALYZE`, close.
4. `os.replace(tmp, index.sqlite)`.

In committed mode, the first open after a clone copies the committed baseline into `cache/` (byte copy, digests verified) and then runs step 3. 06 then refreshes incrementally from the baseline's `index_head`.

### 4.4 Publish order (writers; always under `refresh.lock`, see 06)

```
stage:   cache/tmp-<pid>/{catalog.jsonl, edges.jsonl, meta.json, index.sqlite, cochange.state}
fsync each staged file
os.replace  cochange.state                       (build state first: a crash here only costs recompute)
os.replace  catalog.jsonl, edges.jsonl
os.replace  index.sqlite                         (readers switch here, atomically)
os.replace  meta.json                            (last: its digests describe the files above)
rmdir tmp-<pid>
```

- A crash between steps leaves `meta.json` describing the old JSONL while the JSONL is new. `load_jsonl` detects the digest mismatch, and `open_catalog` keeps serving the (old or new) SQLite snapshot, which is internally consistent. The next refresh repairs everything.
- `build.sqlite` is updated in place inside its own transaction before staging (06).
- Windows: `os.replace` onto an open file can raise `PermissionError`. Retry 5 × 100 ms. If it still fails, leave the staged directory, write `refresh.pending`, and exit 0 (the next refresh retries). Readers open with `mode=ro&nolock=1` to minimise sharing conflicts (Q-05-3).
- Leftover `tmp-*` directories older than 1 h are removed at the start of the next refresh.

### 4.5 Commit policy mechanics

The policy decision is 06 §4.1 (Q-06-1). This doc owns what `surf init` writes:

| Policy | `.surf/.gitignore` | `.gitattributes` additions | Committed files |
|---|---|---|---|
| `commit_catalog=false` (default) | `cache/`, `logs/` | none | `config.toml`, `eval/` |
| `commit_catalog=true` | same | `.surf/catalog.jsonl -diff linguist-generated=true`, `.surf/edges.jsonl -diff linguist-generated=true` | plus `catalog.jsonl`, `edges.jsonl`, `meta.json` at `.surf/` |

`-diff` keeps review noise down. On a merge conflict in a committed baseline, `surf index --baseline` regenerates it and the user commits the result (no custom merge driver in v1; Q-05-2). `surf doctor` warns when a committed `catalog.jsonl` exceeds 20 MB.

---

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `index.commit_catalog` | bool | **`false`** | D-05-1; spec default `true` |
| `index.max_catalog_mb_warn` | int | 20 | doctor warning threshold |
| `store.card_lru` | int | 4096 | long-lived processes only |
| `store.open_rebuild_timeout_ms` | int | 1500 | wait for a concurrent cache build on open |

---

## 6. Edge cases and failure behaviour

| Case | Behaviour |
|---|---|
| `.surf/` missing | `open_catalog → None` → `index-missing` |
| `meta.json` unparseable or wrong `schema_version` | `None`; doctor says "run `surf index`"; SessionStart schedules a full build |
| `index.sqlite` missing or corrupt | Rebuild from JSONL (§4.3) within 1.5 s, else `None` for this route |
| Snapshot older than `meta.json` | Serve the snapshot (`stale=True`) |
| JSONL digest mismatch | Serve the snapshot if present; schedule a full rebuild |
| Card line fails validation | Treated as a digest-level corruption (whole file rejected) |
| Read-only filesystem / cache not writable | Reads work if `index.sqlite` exists; else rebuild in memory (`:memory:`) for this process only |
| Two processes rebuild the cache simultaneously | Lock; the loser waits or serves `None` |
| Path with `'` or `%` | Parameterised queries only; `LIKE` never takes user input |
| Paths differing only in case (`Readme.md`, `README.md`) | Both in `paths`; `casefold` lookups return both (08 treats as ambiguous) |
| `code:` and `mig:` share a path | `paths` maps to `code:` only; `mig:` reached via `alias` edge |
| Committed baseline present but `cache/` empty (fresh clone) | Copy baseline → build snapshot → incremental refresh (06) |
| Catalog > 20 MB | Works; doctor warns in committed mode |

---

## 7. Performance budget

| Operation | Target (10k cards, 60k stored edges) |
|---|---|
| `open_catalog` (warm) | ≤ 5 ms |
| `card`, `children`, `edges_from` | ≤ 0.2 ms each |
| `ids_for_suffix` × 8 lookups | ≤ 1 ms |
| `build_cache` from in-memory records | ≤ 1.5 s (≈ 170k rows incl. reverse edges and suffixes) |
| `load_jsonl` + validate | ≤ 1 s |
| `index.sqlite` size | ≈ 3× JSONL size |

---

## 8. Test plan

- **Serialisation golden:** a fixture catalog serialised → byte-compared with a checked-in file; round-trip `load → write` is byte-identical; nested key sorting; non-ASCII paths (NFC); floats (`0.1 + 0.2` rounds to `0.3`).
- **Validation:** unsorted input is sorted on write; a duplicate id raises; a dangling edge raises; a bad parent raises.
- **Read API:** `children("root:")` includes `db:*`, excludes `mig:`; `subtree_files` prefix boundaries (`src/x/` vs `src/x-y/`, `src/x0`); `edges_from` returns both directions for `schema_ref`, one row per partner for `co_change`; suffix lookup longest-first with ambiguity.
- **Concurrency:** a reader thread loops `open_catalog` + queries while a writer publishes 50 times. No exception, and every observed snapshot has matching card/edge counts (from `meta_kv`).
- **Crash injection:** kill the writer between each `os.replace` step (monkeypatched). The next `open_catalog` serves a consistent snapshot, and the next refresh repairs.
- **Corruption:** truncated `index.sqlite`, truncated JSONL, wrong `user_version` → fail-open behaviour per §6.
- **Performance:** a synthetic 10k-card catalog meets §7 on CI hardware (marked `perf`, not gating on PRs).

---

## 9. Acceptance criteria

1. Phase 1 exit: complete catalogs for both eval repos written and reloaded byte-identically.
2. All reads used by 08/09/10/12 are served from SQLite in ≤ 0.2 ms each (p95).
3. The concurrency and crash-injection tests pass (never a torn read, never a raised exception at query time).
4. Deleting `.surf/cache/` and routing again works: rebuild or `index-missing`, never an error.

---

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-05-1 | Default: commit catalog, edges, meta (§9.5) | Default `commit_catalog=false`; committed mode is an opt-in baseline | Hook-driven refresh would dirty the working tree after every commit, and CI can't reproduce co-change in shallow clones (full analysis 06 §4.1) |
| D-05-2 | `cache/index.sqlite` "fast lookup" (unspecified) | Immutable snapshot, rebuilt and replaced atomically per refresh; plus `build.sqlite` for mutable build state | Readers never see partial state; no WAL or locking on the read path |
| D-05-3 | `index_head: "a1b2c3d"` | Full 40-hex hash, plus `content_fingerprint`, `config_fingerprint`, `worktree_dirty`, `cochange_status`, `files` digests | Needed for freshness checks, `--check` and torn-write detection |
| D-05-4 | Card record has `indexed_at` | Dropped (foundations D-F5) | Determinism |
| D-05-5 | Committed files live at `.surf/` | Router always reads `cache/`; committed files are a baseline only | Keeps hooks from touching tracked files |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-05-1 | Should `index.sqlite` be patched in place instead of rebuilt, for very large repos (> 50k files)? | Rebuild; revisit if refresh > 3 s is dominated by `build_cache` | Phase 5 refresh timing on the larger eval repo |
| Q-05-2 | Ship a git merge driver for committed catalogs? | No; `surf index --baseline` after conflicts | User feedback from committed-mode teams |
| Q-05-3 | Windows `os.replace` over an open SQLite file: is `nolock=1` plus retries enough? | Yes, with pending-marker fallback | Windows CI job |
