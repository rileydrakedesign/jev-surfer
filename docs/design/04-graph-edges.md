# 04 · Graph edges and expansion scoring

**Status:** draft for review
**Spec sections:** §8 (8.1–8.5), §11.6, §7.3/§7.7 (edge-derived card fields), §22.2 `graph/`
**Depends on:** 00-foundations, 01-discovery (file set, excludes), 03-schema-extraction (table set, FKs, defining migrations), 05-catalog-store (read API), 06-refresh (incremental driver)
**Code:** `surf/graph/containment.py`, `surf/graph/cochange.py`, `surf/graph/schema_refs.py`, `surf/graph/expand.py`

---

## 1. Purpose and scope

Build every edge in `edges.jsonl` deterministically, and score depth-1 neighbours at query time.

| In scope (v1) | Out of scope |
|---|---|
| Tree structure (parents, children, dir prefix per F2, recursive counts) and `contains` edges | Symbol references, imports, doc→code path mentions (§8.4) |
| `co_change` (file level) and `dir_coupling` (directory level) from git history | Per-package co-change windows (v2) |
| `schema_ref` (table ↔ code/doc file), `defined_in`, `fk`, `alias` | Comment-aware matching (see Q-04-4) |
| `expand()` used by the router at Step 4 (§11.6) | Depth > 1 expansion |
| Per-file churn counts consumed by 02-cards | Churn bucketing (owned by 02) |

The core rule for everything in this doc: **every edge set is a pure function of explicit inputs** (file set, file contents, table set, the co-change window, config). Incremental refresh (06) supplies the same inputs from caches, so incremental and full builds produce identical bytes (proof in §4.2.7).

---

## 2. Interfaces

```python
# surf/graph/containment.py
@dataclass(frozen=True, slots=True)
class TreeNode:
    id: SurfaceId                     # code:src/ , doc:docs/x.md , db:* , db:orders
    path: str | None                  # None for db:* and tables
    parent: SurfaceId                 # "root:" for top level
    is_dir: bool
    files_total: int                  # recursive; 1 for files; table count for db:*
    children: tuple[SurfaceId, ...]   # sorted by id

class Tree:
    nodes: Mapping[SurfaceId, TreeNode]
    def dir_id(self, dir_path: str) -> SurfaceId | None          # "src/api/" -> "code:src/api/"
    def file_id(self, path: str) -> SurfaceId | None
    def ancestors(self, sid: SurfaceId) -> tuple[SurfaceId, ...]  # nearest first, excludes root:
    def index_files(self, dir_id: SurfaceId, names: IndexNames) -> tuple[SurfaceId, ...]

def build_tree(files: Sequence[DiscoveredFile], tables: Sequence[SurfaceId],
               cfg: IndexConfig) -> Tree
def containment_edges(tree: Tree) -> Iterator[Edge]              # cache builder only (D-04-1)

# surf/graph/cochange.py
def read_commits(git: Git, *, rev_range: str, since_ct: int | None,
                 cfg: CochangeConfig) -> list[CommitRecord]
def update_window(prev: CochangeWindow | None, git: Git, head: str,
                  cfg: CochangeConfig) -> CochangeWindow          # full build: prev=None
def compute_cochange(window: CochangeWindow, tree: Tree,
                     cfg: CochangeConfig) -> CochangeResult

# surf/graph/schema_refs.py
def table_variants(t: TableRef, cfg: SchemaRefsConfig) -> tuple[Variant, ...]
def build_matcher(tables: Sequence[TableRef], cfg: SchemaRefsConfig) -> Matcher
def prefilter(paths: Sequence[str], matcher: Matcher, engine: Literal["rg", "python"]) -> set[str]
def scan_text(text: str, matcher: Matcher) -> dict[SurfaceId, MentionStats]
def schema_ref_edges(mentions: Mapping[SurfaceId, Mapping[SurfaceId, MentionStats]],
                     n_files: int, cfg: SchemaRefsConfig) -> list[Edge]
def schema_struct_edges(schema: SchemaModel, tree: Tree, cfg: SchemaConfig) -> list[Edge]  # defined_in, fk, alias:
                                                  # validates 03's edge_intents (drops missing endpoints), applies caps

# surf/graph/expand.py
def expand(anchors: Mapping[SurfaceId, float], store: CatalogReader,
           cfg: ExpandConfig, *, exclude: Collection[SurfaceId] = (),
           collect_rejected: int = 0) -> tuple[list[ExpansionCandidate], list[ExpansionCandidate]]
                                   # (admitted, rejected); rejected = top-N below threshold, [] when 0 (09 trace)

# 02's EdgeLookup (top()) and ChurnCounts (file_commits(), dir_commits()) protocols are implemented
# here over the in-memory build results: edges, CochangeResult.churn and CochangeResult.dir_churn.
```

Callers: `index/build.py` (all builders), `index/cards.py` (reads `Tree`, `CochangeResult.churn`, and edges for `tables`, `changes_with`, `coupled_dirs`), `route/pipeline.py` (`expand`), `adapters/mcp_server.py` `surface_info` (edges via store).

---

## 3. Data structures

```python
class DiffEntry(BaseModel, frozen=True):
    status: Literal["A", "M", "D", "R", "C", "T"]
    path: str                         # post-commit path
    old_path: str | None = None       # R and C only
    score: int | None = None          # similarity 0..100 for R/C

class CommitRecord(BaseModel, frozen=True):
    hash: str                         # full 40-hex
    ct: int                           # committer time, unix seconds (D-04-3)
    excluded: Literal["author", "message"] | None
    entries: tuple[DiffEntry, ...]    # raw, sorted by (path, old_path)

class CochangeWindow(BaseModel, frozen=True):   # persisted as cache/cochange.state (05)
    head: str
    ref_time: int                     # ct(head)
    config_fp: str                    # sha256 of CochangeConfig + exclude rules
    git_version: str
    commits: tuple[CommitRecord, ...] # ALL non-merge commits in window, sorted (ct desc, hash asc)

class CochangeResult(BaseModel, frozen=True):
    file_edges: tuple[Edge, ...]      # kind=co_change, from < to
    dir_edges: tuple[Edge, ...]       # kind=dir_coupling, from < to
    churn: Mapping[SurfaceId, int]    # eligible commits touching each file
    dir_churn: Mapping[SurfaceId, int] # distinct eligible commits touching each dir subtree (02 dir churn)
    stats: CochangeStats              # commits in window, eligible, dropped by filter (for doctor/explain)

class Variant(BaseModel, frozen=True):
    kind: Literal["exact", "singular", "pascal", "camel"]
    text: str
    mult: float                       # 1.0 / 0.6 / 0.6 / 0.6 ; quoted-SQL context = 1.5 (§4.4.3)

class MentionStats(BaseModel, frozen=True):
    n: int                            # counted occurrences
    best_mult: float                  # max multiplier over counted occurrences

class ExpansionCandidate(BaseModel, frozen=True):
    id: SurfaceId
    score: float
    via_anchor: SurfaceId
    via_kind: EdgeKind                # "contains" for README/index siblings
```

### 3.1 Edge storage conventions (all kinds)

| Kind | Stored direction in `edges.jsonl` | Weight | `evidence` |
|---|---|---|---|
| `contains` | **not stored** (D-04-1); derived from `Card.parent` by the cache | 1.0 | – |
| `co_change` | once, `from < to` (symmetric) | coupling, 4 dp | `{"shared_commits": n}` |
| `dir_coupling` | once, `from < to` (symmetric) | coupling, 4 dp | `{"shared_commits": n}` |
| `schema_ref` | table → file | 4 dp | `{"mentions": n, "variant": "quoted"\|"exact"\|…}` |
| `defined_in` | table → `mig:` | 1.0 | `{"role": "create"\|"alter"}` |
| `fk` | table → referenced table | 0.7 | `{"columns": ["customer_id"]}` |
| `alias` | once, `code:<p>` → `mig:<p>` | 1.0 | – |

The SQLite cache materialises the reverse direction of every edge (05 §3), so `edges_from(id)` works for either end regardless of stored direction.

---

## 4. Behaviour / algorithms

### 4.1 Containment and the tree

1. Input: discovered files (01) after excludes, plus table ids from 03.
2. Directory set = every proper ancestor directory of an included file. No empty or excluded-only directories exist as nodes.
3. Directory prefix (F2): `doc:` iff > 50 % of recursive files are doc files and the dir is not the repo root; else `code:`. File prefix by type (F4).
4. Parents: top-level dirs/files → `root:`; `db:*` → `root:` (only if ≥ 1 table); tables → `db:*`. **`mig:` cards are not tree nodes** (parent `None`, see §10 note to foundations).
5. `files_total`: recursive included-file count; `db:*` = table count.
6. Children sorted by id (the router re-orders chunks by churn; §11.5).
7. `index_files(dir)`: children files of `dir` whose **basename** matches `router.expand.index_names` (case-insensitive glob). Default: `README`, `README.*`, `index.*`, `_index.md`, `__init__.py`, `mod.rs`. Sorted: README first, then id.

Single-child directory chains are kept (no path compression in v1; the walk's flattening handles them).

### 4.2 Co-change

#### 4.2.1 Window definition (pure function of `H` = co-change head)

- `T_ref = ct(H)` (committer time).
- `start = T_ref − window_months × 2_629_746` s (mean Gregorian month).
- `W(H)` = all **non-merge** commits reachable from `H` with `ct ≥ start`, sorted by `(ct desc, hash asc)`, truncated to the first `max_commits`.
- The count cap applies to **all** non-merge commits in the time window, before content filters. Filters decide evidence, not window membership, so the window does not move when filter config changes.

#### 4.2.2 Extraction

```
git -c core.quotepath=off -c diff.renames=true -c log.showSignature=false \
    log --no-merges -z --name-status -M50% -C50% -l<rename_limit> \
    --no-ext-diff --no-textconv --no-color \
    --format=%x1e%H%x1f%ct%x1f%an%x1f%ae%x1f%s  <rev_range> [--since-as-filter=<start>]
```

- Invoked via `surf/proc.py` (`LC_ALL=C`, timeout). All user diff/log config that changes output is overridden on the command line.
- `--since-as-filter` (git ≥ 2.37) visits all commits instead of stopping at the first old one, which plain `--since` does. On older git: omit it and filter `ct ≥ start` in Python. Same result, slower.
- Full build: `rev_range = H`. Incremental: `rev_range = H0..H1` (06).
- Parse per record: header, then NUL-separated `status\0path\0` (`R/C` carry `score`, `old\0new\0`). Merge commits never appear. Entries sorted by `(path, old_path)`.
- Commit filters evaluated **once at ingest**, stored as `excluded`:
  - `author`: casefolded `%an` or `%ae` contains any `index.cochange.exclude_authors` entry (substring, casefolded).
  - `message`: any `index.cochange.exclude_message_patterns` regex matches the subject (`re.search`, `re.IGNORECASE`).
  - Subjects and author identities are **not** stored (privacy; 14).

#### 4.2.3 Resolving historic paths to current paths (renames)

Computed at `compute_cochange` time over the whole window, so it depends only on the window set (§4.2.7).

```python
alias: dict[str, str | None] = {}              # historic path -> current path, None = tombstone
def resolve(p): return alias.get(p, p)

for c in window.commits:                        # newest -> oldest, canonical order (ct desc, hash asc)
    c.current = {resolve(e.path) for e in c.entries if not is_pure_rename(e)}   # D entries resolve to
    # paths absent from the domain (or tombstoned) and are dropped by the domain filter below
    # then update alias for OLDER commits:
    for e in c.entries:
        if e.status == "R":
            alias[e.old_path] = resolve(e.path); alias[e.path] = None
        elif e.status in ("A", "C"):
            alias[e.path] = None                # before its creation this path was another file
        # "D", "M", "T": no change
```

- `is_pure_rename(e)`: `status == "R" and score == 100`. A pure move is not evidence of co-change (a mass `git mv` would otherwise couple 200 files); it still updates `alias`.
- Rename chains resolve naturally (`a → b` then `b → c` walking backwards maps `a → c`).
- Files that resolve to `None` or to a path not in the **co-change domain** are dropped from `c.current`.
- **Co-change domain** = file ids in the current `Tree` whose file is tracked at `H` (untracked working-tree files have no history). Mapped to ids via `tree.file_id(path)`.
- Known approximation: history is linearised by commit time, so a rename on one branch and edits to the old path on a parallel branch may not reconnect. Deterministic, documented, accepted.

#### 4.2.4 Eligibility (per commit, stable)

A commit contributes evidence iff:
1. `excluded is None`, and
2. `k_c ≤ max_files_per_commit`, where `k_c` = number of entries that are **not pure renames** and whose **at-commit path** is not excluded by `index.exclude` / default excludes (01's matcher). Excluded paths (lockfiles, generated, vendored) **do not count** toward the cap, so a dependency bump touching `package.json` + lockfile counts as 1. `k_c` is fixed at ingest and does not change when files are later deleted.
3. `|c.current| ≥ 1` after resolution.

Ineligible commits stay in the window (they count toward `max_commits` and their renames still apply).

#### 4.2.5 Weighting (integer fixed point, order independent)

```
age_days(c) = max(0, T_ref − ct(c)) / 86400
w(c)        = round(exp(−age_days(c) · ln2 / half_life_days) · 2**40)     # int
S(a)        = Σ w(c)  over eligible c with a ∈ c.current                   # Python int, exact
S(a,b)      = Σ w(c)  over eligible c with {a,b} ⊆ c.current
n(a,b)      = count of such c
coupling    = round(S(a,b) / sqrt(S(a) · S(b)), 4)                          # float
```

- **Half-life, not time constant.** The spec writes `exp(−t/H)` and calls `H` a half-life. We use `exp(−t·ln2/H)` so `half_life_days` is a true half-life (D-04-2).
- Decay reference is `T_ref`, never wall clock (foundations §4.4).
- Because `w(c) ∝ exp(ct/τ)`, the reference time cancels in the cosine ratio: coupling depends on `T_ref` only through window membership. That's why "apply decay at read time" (spec §10.2 step 5) needs no special handling: we store raw commits and recompute.
- Integer sums make accumulation order irrelevant. `exp` may differ by 1 ulp across platforms' libm; the `2**40` quantisation absorbs that except in astronomically rare cases, and `--check` tolerates ±0.0001 on weights (06).
- `churn[a]` = count of eligible commits with `a ∈ c.current`; `dir_churn[d]` = count of eligible commits touching any file below `d` (distinct commits, not a sum).

#### 4.2.6 Pruning and emission

1. Candidate pairs: `n(a,b) ≥ min_shared_commits` and `coupling ≥ min_coupling` (on the rounded value).
2. Per file, rank partners by `(coupling desc, partner id asc)`; keep top `top_partners`.
3. A pair is kept if it is in the top-K of **either** endpoint (union; D-04-4).
4. Emit one record per kept pair with `from = min(a,b)`, `to = max(a,b)`.

Pair generation per commit is `O(|c.current|²)`, bounded by `max_files_per_commit` (≤ 435 pairs per commit).

#### 4.2.7 Incremental equals full: argument

Let `F(W, Tree, cfg)` be `compute_cochange`. The full build computes `W(H1)` from git and calls `F`. Incremental (06 §4.4) computes `W' = trim(stored W(H0) ∪ read_commits(H0..H1))` and calls `F`. Claim: `W' = W(H1)` whenever `H0` is an ancestor of `H1`, `ct(H1) ≥ ct(H0)`, and the config and git version fingerprints are unchanged.

- Reachable(H1) = Reachable(H0) ∪ (H0..H1).
- `start(H1) ≥ start(H0)`, so every commit in `W(H1)` that is reachable from `H0` also passed `H0`'s time filter. Truncating the union to the top `max_commits` by the total order `(ct desc, hash asc)` cannot need a commit that `W(H0)`'s truncation dropped, because adding elements to a set only pushes existing elements down the order.
- `CommitRecord` is intrinsic to the commit (its diff against its only parent, pinned git options, filter flags under the same config). So stored records equal freshly read ones.
- `F` depends only on the set `W` (canonical order is re-derived by sorting).

If any precondition fails (not an ancestor, clock went backwards, config or git version changed, or the state is missing or corrupt), 06 does a full co-change rebuild. Tested by the property test in §8.

#### 4.2.8 Directory coupling (spec §8.2.5, defined precisely)

"Normalised the same way" means **the same recency-weighted cosine, computed on directory touch sets at the commit level**, rather than summing file-pair couplings, which is unbounded and double counts:

```
D_c   = { d ∈ tree dirs : d is an ancestor of some f ∈ c.current } \ {root:}
S(D)  = Σ w(c) over eligible c with D ∈ D_c ;   S(D,E) likewise for D,E ∈ D_c
dir_coupling(D,E) = round(S(D,E) / sqrt(S(D)·S(E)), 4)
```

- Only pairs where neither is an ancestor of the other.
- Commits with `|D_c| > max_dirs_per_commit` (default 40) are skipped for directory coupling only (bounds the per-commit cost to 780 pairs).
- Pruning: `n ≥ dir_min_shared_commits` (2), `coupling ≥ dir_min_coupling` (0.15), top `dir_top_partners` (5) per directory by `(coupling desc, id asc)`, union, emitted once with `from < to`, kind `dir_coupling`.
- `doc:` and `code:` directories both participate. `db:*` does not.
- 02-cards takes `coupled_dirs` = top 3 `dir_coupling` partners.

### 4.3 Structural schema edges (from 03's `SchemaModel`)

| Edge | Rule |
|---|---|
| `defined_in` | Table → `mig:<file>` for the creating migration and the **3 most recent** altering migrations in replay order (cap `index.schema.max_defined_in` = 4; D-04-5). Weight 1.0. |
| `fk` | Table → referenced table, one edge per referenced table (multiple FK columns merged into `evidence.columns`, sorted). Weight 0.7. Self-references dropped. |
| `alias` | For every migration file that is also a tree file: `code:<p>` → `mig:<p>`, weight 1.0. |

Non-migration schema sources (`schema.prisma`, `db/schema.rb`) get `defined_in` from each table they define (no `mig:` card exists for them; target is the `code:` id). 03 owns which is which.

### 4.4 Schema references

#### 4.4.1 Scanned files

All `code_file` and `doc_file` tree nodes ≤ `index.max_file_bytes`, **excluding schema source files** (migrations, `schema.prisma`, `schema.rb`), which already have `defined_in` edges. `N` = number of scanned files. Decoding: UTF-8 with `errors="surrogateescape"`.

#### 4.4.2 Variants (per table, on each unqualified name `t` in 03's `search_names`, which includes a Prisma `model_name`)

| Kind | Text | Mult | Case rule | Boundaries |
|---|---|---|---|---|
| exact | `t` and `t.upper()` | 1.0 | case-sensitive (either form) | identifier: prev and next char ∉ `[A-Za-z0-9_]` |
| singular | `sing(t)` | 0.6 | case-sensitive | identifier |
| pascal | `Pascal(sing(t))` | 0.6 | case-sensitive | left: start, non-`[A-Za-z0-9]`, or prev ∈ `[a-z0-9]` (camel hump); right: end or next ∉ `[a-z0-9]` |
| camel | `camel(t)` (only if `t` contains `_`) | 0.6 | case-sensitive | left: start or prev ∉ `[A-Za-z0-9]`; right: end or next ∉ `[a-z0-9]` |

- `sing` (on the last `_` segment only): `ies→y`; `(s|x|z|ch|sh)es→\1`; trailing `s` not preceded by `s`/`u`/`i` → drop; otherwise **no** singular (the variant is omitted, as are pascal and camel variants that collapse onto `exact`).
- Duplicate variant texts collapse to the highest multiplier.
- Non-ASCII table names: only the exact and quoted forms, case-sensitive.
- Examples for `order_items`: `order_items`/`ORDER_ITEMS` (1.0), `order_item` (0.6), `OrderItem` (0.6, matches in `createOrderItem` and `OrderItemService`, not in `OrderItems`), `orderItems` (0.6).

#### 4.4.3 Quoted-SQL context (mult 1.5)

An occurrence of `t` (optionally schema-qualified `s.t`, `"s"."t"`) is **quoted** when either:
- (a) it is enclosed by the same quote pair immediately before and after the (qualified) token: `"…"`, `'…'`, `` `…` ``, or `[…]`. The name must match case-sensitively; or
- (b) it is preceded by `(?i)\b(FROM|JOIN|INTO|UPDATE|TABLE|REFERENCES)\s+(?:ONLY\s+)?(?:IF\s+(?:NOT\s+)?EXISTS\s+)?` (whitespace may include newlines) and followed by an identifier boundary. The name is compared ASCII case-insensitively, since unquoted SQL identifiers fold.

Each text span is classified **once**, as the highest-multiplier kind that matches it (quoted > exact > singular = pascal = camel).

#### 4.4.4 Noise guards

- Tables with `len(t) < min_name_len` or `t ∈ stopwords` count **only** quoted occurrences.
- Comments are not stripped in v1 (Q-04-4).
- Ambiguous unqualified names (same `t` in two schemas): occurrences are attributed to both tables.

#### 4.4.5 Matching engine (identical results with or without ripgrep)

1. **Literal set** `L` = ASCII-lowercased texts of all variants.
2. **Prefilter** (which files could match), a superset only:
   - `rg`: `rg --files-with-matches --fixed-strings --ignore-case --text --no-config --no-ignore --hidden --no-messages -f <L-file> -- <paths…>` (paths batched under ARG_MAX). Files rg errors on are passed through as candidates.
   - `python`: all scanned files.
3. **Verify** (always Python, the only place results are decided): build one `pyahocorasick` automaton over `L`; scan `ascii_lower(text)` (`str.translate` of A–Z only, so length is preserved); for each hit, check the case rule and boundaries of every variant sharing that literal against the **original** text; then apply quoted-context detection and span classification.

Because rg only narrows the file set and never decides a match, `engine=rg` and `engine=python` produce identical mentions. A test asserts this (§8).

#### 4.4.6 Weights

```
df(t)        = # scanned files with MentionStats(t).n ≥ 1
spec(t)      = ln(1 + N / (1 + df(t)))
spec_norm(t) = min(1, spec(t) / ln(1 + N/2))            # divide by the value at df = 1 (D-04-6)
base(n)      = min(1, 0.25 + 0.25 · ln(1 + n))
weight(f,t)  = round(min(1, base(n) · best_mult · spec_norm(t)), 4)
```

- Drop `weight < index.schema_refs.min_weight` (0.05).
- Caps: first per file, top `max_tables_per_file` (20) by `(weight desc, table id asc)`; then per table, top `max_files_per_table` (200) by `(weight desc, file id asc)`.
- Normalising by the df = 1 value (not the empirical max over tables) means adding or removing an unrelated rare table doesn't shift every weight.

### 4.5 Expansion scoring (spec §8.5, query time)

```python
def expand(anchors, store, cfg, *, exclude=(), collect_rejected=0):
    anchors = canonicalise(anchors)             # alias: code:<mig> and mig:<mig> share strength (max)
    best: dict[SurfaceId, ExpansionCandidate] = {}
    for a, strength in sorted(anchors.items()):
        for nb, kind, w in neighbours(a, store, cfg):          # see traversal table
            s = w * cfg.kind_factor[kind] * strength
            if s + 1e-9 < cfg.threshold:
                if collect_rejected: note_rejected(nb, s, a, kind)       # best per id, for the trace
                continue
            nb = canonical_id(nb)                               # code:<mig path> -> mig:<path>
            if nb in anchors or nb in exclude: continue
            cand = ExpansionCandidate(id=nb, score=s, via_anchor=a, via_kind=kind)
            if better(cand, best.get(nb)): best[nb] = cand     # score desc, KIND_ORDER, anchor id asc
    out = sorted(best.values(), key=lambda c: (-c.score, c.id))
    return out[: cfg.max_total], top_rejected(collect_rejected)  # (admitted, rejected); 09 passes 20
```

**Traversal table** (per anchor; `max_per_anchor` applies per kind after sorting by weight desc, id asc):

| Kind | Directions traversed | kind_factor | Allowed anchors → neighbours |
|---|---|---|---|
| `co_change` | both | 1.0 | file → file |
| `schema_ref` | table → file, file → table | 0.9 | table ↔ code/doc file |
| `defined_in` | table → mig, mig → table | 0.8 | table ↔ `mig:` |
| `fk` | both | 0.5 | table ↔ table |
| `contains` | file → sibling index files; dir → own index files (edge weight 1.0) | 0.6 | never generic children, never upward |
| `dir_coupling` | not traversed | 0 | used only on dir cards |
| `alias` | identity, not scored | – | canonicalisation only |

- Anchor strength: walk cumulative score or 1.0 for path hits (router passes it in).
- Dedup: a neighbour reached via several anchors or kinds keeps the max score. Ties are broken by `KIND_ORDER = co_change, schema_ref, defined_in, fk, contains`, then anchor id.
- `cfg.enabled_kinds` lets eval ablations A1–A4 switch kinds off.
- Output feeds final-pass pool ranking (§11.6, 09-router).

---

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `index.cochange.enabled` | bool | `true` | forced off for non-git |
| `index.cochange.window_months` | int | 24 | |
| `index.cochange.max_commits` | int | 5000 | counted before filters |
| `index.cochange.max_files_per_commit` | int | 30 | excluded paths and pure renames not counted |
| `index.cochange.half_life_days` | float | 180 | true half-life (D-04-2) |
| `index.cochange.min_shared_commits` | int | 2 | |
| `index.cochange.min_coupling` | float | 0.15 | |
| `index.cochange.top_partners` | int | 10 | union rule |
| `index.cochange.exclude_authors` | list[str] | `["dependabot", "renovate", "github-actions"]` | casefolded substring on name/email (Q-04-2) |
| `index.cochange.exclude_message_patterns` | list[regex] | `["^(chore\|style\|format)\\b", "\\bprettier\\b", "\\blint\\b"]` | subject only, case-insensitive (Q-04-3) |
| `index.cochange.rename_limit` | int | 2000 | git `-l` |
| `index.cochange.dir_top_partners` | int | 5 | |
| `index.cochange.dir_min_coupling` | float | 0.15 | |
| `index.cochange.dir_min_shared_commits` | int | 2 | |
| `index.cochange.max_dirs_per_commit` | int | 40 | |
| `index.schema.max_defined_in` | int | 4 | D-04-5 |
| `index.schema_refs.stopwords` | list[str] | spec §16 | |
| `index.schema_refs.min_name_len` | int | 4 | |
| `index.schema_refs.max_tables_per_file` | int | 20 | |
| `index.schema_refs.max_files_per_table` | int | 200 | |
| `index.schema_refs.min_weight` | float | 0.05 | new |
| `index.schema_refs.engine` | `auto\|rg\|python` | `auto` | `auto` = rg if on PATH |
| `router.thresholds.<profile>.expand` | float | 0.30 | spec §16 |
| `router.expand.kind_factors` | table | §4.5 values | |
| `router.expand.enabled_kinds` | list | all five | ablations |
| `router.expand.max_per_anchor` | int | 8 | per kind; new (Q-04-5) |
| `router.expand.max_total` | int | 40 | |
| `router.expand.index_names` | list[glob] | `README`, `README.*`, `index.*`, `_index.md`, `__init__.py`, `mod.rs` | |

Any change to an `index.cochange.*` key changes `config_fp`, which forces a full co-change rebuild.

---

## 6. Edge cases and failure behaviour

| Case | Behaviour |
|---|---|
| Not a git repo | No `co_change`/`dir_coupling`; `churn` empty; `meta.cochange_status="no_git"` |
| Shallow clone | Use the available history; `cochange_status="shallow"`; doctor warns; `--check` refuses (06) |
| Repo with 0–1 commits | No edges; no error |
| `git` missing or too old for `-z --name-status` | Index fails loudly with a message (index time may fail) |
| git < 2.37 | No `--since-as-filter`; Python time filter; same output |
| Commit touching only excluded paths | `k_c = 0`, `current = ∅`: ineligible, still in the window |
| Mass rename commit (500 × R100) | No evidence; renames applied |
| Rename + edit (R87) | Counts as touching the new path |
| Rename limit exceeded in a commit | git reports A/D; history of those files breaks at that commit (deterministic); doctor reports count |
| File deleted and later re-created at same path | History before the re-creation is tombstoned |
| Clock skew (child ct < parent ct) | Order still total by `(ct, hash)`; ages clamped ≥ 0 |
| Submodules | Gitlink entries ignored (not files in the tree) |
| Untracked file | In the tree and schema refs; no co-change (not in the domain) |
| Paths with tabs/newlines/non-UTF-8 | `-z` parsing; non-UTF-8 decoded with surrogateescape; ids NFC-normalised (00 §2.2) |
| No tables | No schema edges; matcher not built |
| Table name that is a common English word (`events`) | Specificity lowers weight; stopword list if configured |
| Non-ASCII table name | Exact and quoted, case-sensitive only |
| rg present but fails (exit 2) | Log a warning, fall back to the python prefilter (same result) |
| File unreadable during scan | Skipped with a warning in `meta.warnings`; still counts toward `N` (`N` is the length of the scanned-file list, not the number of successes) |
| Expansion anchor not in catalog | Ignored |
| Table anchor with 200 schema_ref files | `max_per_anchor` keeps the 8 heaviest |

---

## 7. Performance budget

| Operation | Target (5k files, 5k commits) | Note |
|---|---|---|
| `git log` full window | ≤ 15 s | dominated by `-M -C` in git |
| `compute_cochange` | ≤ 1 s | integer sums, ≤ 2.2 M pair increments worst case |
| Incremental co-change (≤ 20 new commits) | ≤ 300 ms | git log on the range + recompute |
| Schema-ref full scan (100 tables) | ≤ 3 s | rg prefilter + Aho-Corasick |
| Schema-ref rescan of 20 changed files | ≤ 100 ms | |
| `expand()` for 40 anchors | ≤ 10 ms | SQLite indexed lookups |
| `cochange.state` size | ≤ 5 MB gzipped | |

All of it fits inside the Phase 1 exit budget (full index < 60 s for 5k files).

---

## 8. Test plan

**Unit**
- `resolve()` rename chains: `a→b→c`, rename then re-create `a`, copy vs rename, R100 exclusion.
- Eligibility: cap counts exclude lockfiles and pure renames; bot and message filters; a filtered commit's renames still apply.
- Weight maths: hand-computed 3-commit example; coupling ∈ [0,1] (hypothesis property); invariance of coupling under shifting all `ct` and `T_ref` by a constant.
- Union top-K: asymmetric partner lists.
- Directory coupling: ancestor pairs excluded; `max_dirs_per_commit` skip.
- Variants: table of 20 names (`order_items`, `categories`, `addresses`, `status`, `people`, `billing.invoices`, non-ASCII) → expected variant sets.
- Boundary matrix: `OrderItemService` ✓ pascal, `OrderItems` ✗ pascal, `getorderItems` ✗ camel, `order_items_id` ✗ exact, `"order_items"` quoted, `from('order_items')` quoted, `FROM public.order_items` quoted, `ORDER_ITEMS` exact.
- Stopword table counted only in quoted context.
- Specificity: weights independent of an added unrelated table.
- Expansion: traversal table rows; alias canonicalisation; tie-break order; `enabled_kinds`.

**Golden** (`tests/fixtures/repos/` built from YAML)
- `layered`: controllers/services/models with scripted history. Expected `edges.jsonl` checked in.
- `renames`: a directory move plus edits across the move. Expected partners follow the new paths.
- `schema`: Supabase migrations and TS code. Expected schema_ref weights.

**Determinism and equivalence**
- `rg` vs `python` engine produce byte-identical `edges.jsonl` (CI job with and without rg on PATH).
- Property test: random commit DAG (adds, edits, renames, deletes, bot commits, skewed clocks), random split point `H0`. `F(trim(W(H0) ∪ H0..H1)) == F(W(H1))` byte for byte. Also covers window roll-off, where old commits fall out as new ones arrive.
- Two builds of the same fixture produce identical bytes.

---

## 9. Acceptance criteria

1. Full edge build for a 5k-file, 5k-commit repo within the §7 budgets (Phase 1 exit contributes).
2. Property test (§8) passes 500 examples: incremental co-change == full.
3. rg / python parity test passes.
4. On the `layered` fixture, a cross-layer query's `must_include` service file is reachable by depth-1 expansion from the controller anchor (supports the Phase 3 exit: reduced "walk" losses on cross_layer).
5. Ablations A2–A5 can be run purely through `router.expand.enabled_kinds` and a `coupled_dirs` card toggle (02).
6. No edge references an id missing from the catalog (referential-integrity check in 05's `write_catalog`).

---

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-04-1 | Record `contains` edges (§8.1) | Not written to `edges.jsonl`; the cache derives them from `Card.parent` | Pure duplication of `parent`; avoids two sources of truth and thousands of diff lines |
| D-04-2 | `w = exp(−t/H)`, "half-life H" | `w = exp(−t·ln2/H)` | Makes `half_life_days` mean what it says; the spec formula has a half-life of 0.69·H |
| D-04-3 | `%at` (author time) in the git log format | Committer time `%ct` for ages and the window | Committer time is closer to when the change landed; author time survives rebases and can predate its parent. Either would be deterministic. |
| D-04-4 | "Keep the top 10 partners per file. Store symmetrically." | Keep the pair if it's in the top-K of either endpoint; one record `from<to`; cache materialises both directions | Deterministic symmetric rule; halves the file |
| D-04-5 | `defined_in` for migrations (unbounded) | Creating migration + 3 most recent alters | A table altered by 40 migrations would flood expansion |
| D-04-6 | "normalized specificity" undefined | Divide by `ln(1+N/2)` (the df = 1 value) | Stable when the table set changes |
| D-04-7 | Window via `git log --since` | Window defined over the commit set by `(ct desc, hash asc)`; `--since-as-filter` or a Python filter | `--since` stops at the first old commit, so the result depends on graph shape and clock skew |
| D-04-8 | Comment stripping "when a cheap regex exists" | Never in v1 | Determinism and simplicity; specificity absorbs most noise (Q-04-4) |
| D-04-9 | Schema refs scan "all indexed files" | Schema source files excluded; they get `defined_in` | Otherwise every migration couples to every table it mentions |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-04-1 | Add a PascalCase **plural** variant (`OrderItems`, `getOrderItems`)? | Not in v1 | Eval: schema-ref recall on TS/Java repos (A4 per-table misses) |
| Q-04-2 | Exclude all `[bot]` authors? Some bots (coding agents) make real changes. | Only the three named bots | Inspect bot share of commits on the two eval repos |
| Q-04-3 | `^chore` excludes real work in some conventional-commit repos | Keep the spec default | A3 ablation with and without the message filter |
| Q-04-4 | Strip comments per extension? | No | Eval shows schema_ref false positives from comments |
| Q-04-5 | `max_per_anchor = 8` and `expand = 0.3`. Walk anchors with cumulative score < 0.5 rarely clear 0.3 × kind_factor, so expansion is dominated by path hits and confident walk picks | Keep; report expansion yield per anchor source | A2–A4 ablations and failure attribution |
| Q-04-6 | Should `fk` edges get a weight from FK multiplicity instead of a flat 0.7? | Flat 0.7 | Ablation |
| Q-04-7 | git rename-detection heuristics can differ across git versions, so co-change can differ across machines | Record `git_version` in meta; `--check` warns (and doesn't fail) on a version mismatch | Real-world `--check` failures |
