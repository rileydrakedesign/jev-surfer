# 08 · Path matching

**Status:** draft for review
**Spec sections:** §11.3 (also §8.5 anchor strength, §17.1 `stack_trace` category, §21 "incidental path hits")
**Depends on:** 00-foundations (ids, `norm_path`, F2/F3), 05-catalog-store (the `paths` table in `index.sqlite`), 09-router (consumer)
**Code:** `surf/route/pathmatch.py`, `surf/route/pathindex.py` (new: `PathIndex` protocol + SQLite and trie implementations)

---

## 1. Purpose and scope

Turn path-like text in a prompt (stack traces, compiler errors, test output, logs, diffs, links, prose mentions) into catalog surface ids, with a strength the router uses as `anchor_strength` (spec §8.5). Pure string work: no model, no filesystem access, no git.

**In scope (v1)**
- Extraction of path mentions from arbitrary text, with trace-format recognizers for Python, Node/JS/TS, Java/Kotlin/Scala, Go, Rust, .NET, Ruby, PHP, C/C++ (gcc/clang/MSVC), tsc, pytest, Jest, git diffs, URLs.
- Normalization (separators, schemes, bundler prefixes, line/col suffixes, URL-encoding, quoting).
- Matching against catalog file **and directory** paths by longest segment-aligned suffix, with a case-insensitive fallback.
- Classification: framework/vendor frames (dropped), test files (demoted), ambiguity (candidates, not anchors).
- Ranking, dedupe and the cap of 10 hits.

**Out of scope**
- Source-map resolution (`dist/x.js` → `src/x.ts`) (Q-08-1); symbol matching (`markShipped` → file) (v1.1 symbols); line numbers in the note (Q-08-2).
- Deciding what the router does with hits (09 §4.4).

Path matching runs on the **raw, unredacted** prompt. That's safe: it runs locally, and its output is catalog ids, never prompt text. Redaction could otherwise destroy matches (a 40-hex `/rustc/<hash>/` segment, or a long hashed path segment, looks like a high-entropy secret).

---

## 2. Interfaces

```python
# route/pathmatch.py
def match_paths(text: str, index: PathIndex, cfg: PathMatchConfig) -> PathMatchResult: ...

# internal stages, each unit-tested on its own
def extract_mentions(text: str, cfg: PathMatchConfig) -> list[Mention]: ...
def normalize(m: Mention) -> list[NormPath]: ...               # 0..2 variants (raw, url-decoded)
def classify_framework(np: NormPath, m: Mention) -> bool: ...
def is_test_path(path: str) -> bool: ...                       # also used by 09 selection trace
def resolve(np: NormPath, index: PathIndex) -> Resolution: ...
def rank_and_cap(res: list[Resolution], cfg) -> PathMatchResult: ...

# route/pathindex.py
class PathIndex(Protocol):
    def suffixes_of(self, rsegs: Sequence[str], *, fold: bool) -> list[IndexedPath]: ...
        # catalog paths that are a segment-suffix of the query, longest first
    def with_suffix(self, rsegs: Sequence[str], *, fold: bool, limit: int) -> tuple[list[IndexedPath], int]: ...
        # catalog paths that end with the query; returns (up to `limit` paths, total count)
    def extensionless_basenames(self) -> frozenset[str]: ...

class SqlitePathIndex(PathIndex): ...      # production: reads index.sqlite, zero build cost per process
class TriePathIndex(PathIndex): ...        # tests, eval, long-lived MCP server; built from the catalog in memory
```

Caller: `route/pipeline.py`, once per routed prompt, before call 1 (09 §4.4). Also `surf route --explain` prints the mention table (§3, `trace`).

---

## 3. Data structures

```python
class TraceKind(StrEnum):
    PY = "python"; NODE = "node"; JVM = "jvm"; GO = "go"; RUST = "rust"; DOTNET = "dotnet"
    RUBY = "ruby"; PHP = "php"; CC = "c_cpp"; TSC = "tsc"; PYTEST = "pytest"; JEST = "jest"
    DIFF = "diff"; URL = "url"; QUOTED = "quoted"; GENERIC = "generic"

class Mention(BaseModel, frozen=True):
    raw: str                     # exact substring (local only: trace/explain, never logged)
    path_text: str               # captured path part, before normalization
    span: tuple[int, int]
    kind: TraceKind
    line: int | None; col: int | None
    block_id: int | None         # id of the recognized trace block (§4.1 step 2)
    frame_rank: int              # 0 = frame nearest the error within its block; appearance order otherwise
    jvm_class: str | None        # "com.foo.Bar$Inner" for JVM frames

class NormPath(BaseModel, frozen=True):
    segs: tuple[str, ...]        # normalized POSIX segments, NFC, no empty/"."/".."
    is_dir_hint: bool            # trailing "/" in the text
    decoded: bool                # produced by URL-decoding
    mention: Mention

class MatchClass(StrEnum):
    SUFFIX = "suffix"            # whole catalog path is a segment-suffix of the query (includes exact)
    TAIL = "tail"                # query (≥ 2 segs) is a suffix of exactly one catalog path
    BASENAME = "basename"        # 1-segment query, unique basename
    CONTEXT = "context"          # ambiguous, disambiguated by other hits' directories (§4.6)
    FOLD = "fold"                # any of the above, but only via case-insensitive lookup
    AMBIGUOUS = "ambiguous"      # several catalog paths; not an anchor

class IndexedPath(BaseModel, frozen=True):
    id: SurfaceId; path: str; is_dir: bool

class PathHit(BaseModel, frozen=True):
    id: SurfaceId
    strength: float              # anchor_strength for §8.5
    match_class: MatchClass
    is_test: bool; is_dir: bool
    lines: tuple[int, ...]       # up to 3 distinct line numbers seen, ascending
    frame_rank: int              # best (lowest) across mentions
    first_offset: int
    n_mentions: int

class AmbiguousMention(BaseModel, frozen=True):
    query: str                   # normalized path text (local only)
    candidates: tuple[SurfaceId, ...]   # ≤ ambiguous_max, sorted by id
    total: int                   # real count (may exceed candidates)

class PathMatchResult(BaseModel):
    hits: list[PathHit]                  # ≤ max_hits, ranked (§4.7)
    candidates: list[SurfaceId]          # ambiguous candidates for the final pass, ≤ max_ambiguous_candidates
    ambiguous: list[AmbiguousMention]
    dropped: dict[str, int]              # reason -> count: framework, no_match, over_cap, too_ambiguous
    stats: PathMatchStats                # mentions_seen, unique_queries, lookups, elapsed_us, scanned_chars
    trace: list[MentionTrace] | None     # only when explain=True
```

### 3.1 Suffix index: reverse-segment trie

Conceptually, every catalog path (files and directories, content surfaces only: `code:` and `doc:` ids) is inserted into a trie keyed by its segments **in reverse** (basename first). Each node stores the ids that terminate there (`terminal`) and a subtree count.

```
ship.ts ─┬─ fulfillment ── src  [terminal: code:src/fulfillment/ship.ts]
         └─ legacy ── lib       [terminal: code:lib/legacy/ship.ts]
index.ts ── … (212 terminals below)
```

Two queries, both walking the query's reversed segments from the root:

| Query | Walk | Result |
|---|---|---|
| `suffixes_of(q)` | descend while segments match; collect every terminal passed | catalog paths that are a suffix of `q`; longest = deepest terminal |
| `with_suffix(q)` | descend all of `q`; if fully consumed, the node's subtree | catalog paths ending with `q`; count from the subtree counter |

**Production implementation (`SqlitePathIndex`).** The hook is a fresh process per prompt, so building a trie of 20k paths (~40 ms) on every prompt is wasted work. 05-catalog-store adds a table to `index.sqlite`:

```sql
CREATE TABLE paths (
  rpath      TEXT PRIMARY KEY,   -- reversed segments joined by '/': "ship.ts/fulfillment/src"; dirs end with '/' stripped, flag below
  rpath_fold TEXT NOT NULL,      -- casefold(rpath)
  id         TEXT NOT NULL,
  is_dir     INTEGER NOT NULL,
  basename   TEXT NOT NULL
) WITHOUT ROWID;
CREATE INDEX paths_fold ON paths(rpath_fold);
```

- `suffixes_of(q)`: for `k = len(q) … 1`, point lookups `rpath = join(q[:k])` (≤ number of query segments, typically ≤ 12). Hits collected longest-first.
- `with_suffix(q)`: `rpath = r OR (rpath >= r || '/' AND rpath < r || '0')` (`'0'` sorts right after `'/'`) with `LIMIT ambiguous_max + 1`, plus `COUNT(*)` of the same range (index-only).
- `fold=True`: same queries on `rpath_fold`.
- A file and a dir can't share a path, so `rpath` is unique.

`TriePathIndex` implements the same protocol in memory; both must pass the full corpus (§8.1).

---

## 4. Behavior / algorithm

### 4.1 Extraction

1. **Bound the input.** If `len(text) > scan_max_chars` (262,144), scan the first and last `scan_max_chars/2` characters (stack traces are usually at the end of long pastes). Fast gate: if the text contains none of `/`, `\`, and no `\.[A-Za-z0-9]{1,8}\b`, return an empty result.
2. **Detect trace blocks** (for `frame_rank`). A block is a maximal run of lines matched by one recognizer's frame regex, allowing ≤ 2 non-matching lines inside. Python blocks that follow `Traceback (most recent call last):` are **innermost-last**, so `frame_rank` counts from the end of the block; every other format (Node, JVM, Go, Rust, .NET, Ruby, PHP) is innermost-first. Mentions outside blocks get `frame_rank = 1000 + appearance index`.
3. **Run recognizers**, most specific first. A character span claimed by one recognizer isn't re-matched by a later one. All regexes are compiled once, with atomic groups / possessive quantifiers (Python 3.11) around repeated classes so matching stays linear.

| # | Kind | Pattern (sketch; `P` = path char class below) | Captures |
|---|---|---|---|
| 1 | PY | `File "(?P<p>[^"\n]{1,512})", line (?P<l>\d+)` | path with spaces allowed |
| 2 | JVM | `\bat (?P<cls>[\w$.]+)\.(?P<m>[\w$<>]+)\((?P<f>[\w$-]+\.(?:java\|kt\|kts\|scala\|groovy\|clj)):(?P<l>\d+)\)` | class, file, line |
| 3 | DOTNET | `\bat .+? in (?P<p>.+?):line (?P<l>\d+)` | path (may contain spaces; ends at `:line`) |
| 4 | RUST | `panicked at (?:'[^'\n]*', )?(?P<p>P+):(?P<l>\d+):(?P<c>\d+)` and backtrace `^\s+at (?P<p>P+):(?P<l>\d+):(?P<c>\d+)` | |
| 5 | GO | `^\s+(?P<p>P+\.go):(?P<l>\d+)(?: \+0x[0-9a-f]+)?$` | |
| 6 | NODE | `\bat (?:async )?(?:[^\s(]+ )?\(?(?P<p>(?:file://\|webpack(?:-internal)?://+\|node:)?P+):(?P<l>\d+):(?P<c>\d+)\)?` | |
| 7 | RUBY | `(?P<p>P+\.(?:rb\|rake\|erb)):(?P<l>\d+):in ` | |
| 8 | PHP | `(?P<p>P+\.php)(?:\((?P<l>\d+)\)\|:(?P<l2>\d+)\| on line (?P<l3>\d+))` | |
| 9 | CC / TSC / MSVC | `(?P<p>P+\.\w{1,8})(?:\((?P<l>\d+)(?:,(?P<c>\d+))?\)\|:(?P<l2>\d+)(?::(?P<c2>\d+))?)(?=[:\s]\|$)` | `file(12,5): error`, `file:12:5: error`, `file:12:5 - error TS…` |
| 10 | PYTEST | `(?P<p>P+\.py)::[\w\[\]\-.:]+` and `^(?P<p>P+\.py):(?P<l>\d+): ` | node id suffix stripped |
| 11 | JEST | `^\s*(?:PASS\|FAIL)\s+(?P<p>P+)` | |
| 12 | DIFF | `^(?:diff --git a/(?P<p>\S+) b/(?P<p2>\S+)\|--- a/(?P<p3>\S+)\|\+\+\+ b/(?P<p4>\S+))` | `a/`, `b/` stripped here |
| 13 | URL | `\b(?:https?\|file\|vscode\|idea)://[^\s<>"')\]]+` | whole URL; path extracted in normalization |
| 14 | QUOTED | `` `([^`\n]{1,512})` ``, `"([^"\n]{1,512})"`, `'([^'\n]{1,512})'` containing `/` or `\` or a dotted extension | only way to get a path with spaces outside PY/.NET |
| 15 | GENERIC | `(?<![\w@/\\.-])(?P<p>(?:[A-Za-z]:)?[\\/]?(?:P+[\\/])*P+\.[A-Za-z0-9]{1,10})(?P<pos>:\d+(?::\d+)?\|\(\d+(?:,\d+)?\))?` | any `seg/seg.ext`, bare `name.ext` |
| 16 | GENERIC-DIR | `(?<![\w@/\\.-])(?P<p>(?:P+/){1,}P*/?)` with ≥ 2 segments or a trailing `/` | directory mentions (`src/fulfillment/`) |
| 17 | GENERIC-NOEXT | bare token equal to a member of `index.extensionless_basenames()` (e.g. `Makefile`, `Dockerfile`, `Procfile`), or `seg/…/<that name>` | |

Path char class `P = [\w.\-+@~\[\]()$%!=,#&]` (`#` only mid-segment; a trailing `#L12` is treated as a fragment in normalization). Unicode letters are included via `\w`. Spaces are **not** in `P`: an unquoted path containing a space is not recognized (D-08-3).

4. **Trim wrappers** on the captured path: strip leading `(<[{'"` and trailing `)>]}'",;:.!?` characters when unbalanced (brackets are kept when balanced, so `app/(shop)/[id]/page.tsx` survives), and strip a trailing `:` left by `path:` prose.
5. **JVM mapping.** For a JVM frame, build the query from the class, not the bare file name: `com.foo.Bar$Inner` + `Bar.java` → segments `com/foo/Bar.java` (package dirs + file name; the nested class is dropped because the file name wins). Maven/Gradle layouts (`src/main/java/com/foo/Bar.java`, `module/src/test/kotlin/…`) then match as `TAIL`. Kotlin allows files outside their package directory; if the package query fails, fall back to the bare file name (`BASENAME` rules). Frames whose class starts with a JVM-runtime prefix (`java.`, `javax.`, `jdk.`, `sun.`, `kotlin.`, `kotlinx.`, `scala.`, `org.junit.`, `org.springframework.`, `org.gradle.`, `io.netty.`) are framework (§4.3) before any lookup.
6. **Dedupe mentions** by `path_text` (keeping all line numbers and the best `frame_rank`), and stop after `max_mentions` (500) distinct queries.

### 4.2 Normalization (per mention; pure function)

Applied in this order:

| # | Step | Example in → out |
|---|---|---|
| 1 | Strip scheme/bundler prefixes: `file://` (+ optional host), `webpack://<ns>/`, `webpack-internal:///`, `webpack:///`, `rollup://`, `vite:`/`/@fs/`, `turbopack://[project]/`, `app://`, `vscode://file/`, `idea://open?file=` | `webpack:///./src/x.ts` → `./src/x.ts` |
| 2 | URL: parse; keep the URL path; drop scheme, host, query, fragment. GitHub/GitLab/Bitbucket `…/blob/<ref>/`, `…/-/blob/<ref>/`, `…/src/<ref>/`, `…/tree/<ref>/` are **not** special-cased: longest-suffix matching absorbs them. `#L12` / `#L12-L20` → `line=12` | `https://github.com/o/r/blob/main/src/x.ts#L12` → `o/r/blob/main/src/x.ts`, line 12 |
| 3 | Strip position suffixes: `:L[:C]`, `(L[,C])`, `:line L`, `, line L`, ` +0x…`, `::test_id` (already captured into `line`/`col`) | `src/x.ts:88:21` → `src/x.ts` |
| 4 | Strip query `?v=…` on non-URL paths (Vite/webpack cache busting) | `src/x.ts?t=1690` → `src/x.ts` |
| 5 | Backslashes → `/`; strip Windows prefixes `\\?\`, `\\server\share\`, `C:` / `c:` | `C:\Users\me\repo\src\x.ts` → `/Users/me/repo/src/x.ts` |
| 6 | Strip `~/`, leading `./`, leading `/` (absolute-ness no longer matters) | |
| 7 | Split on `/`; drop empty and `.` segments; resolve `..` lexically (drop the segment and its predecessor; leading `..` dropped) | `a/b/../c.ts` → `a/c.ts` |
| 8 | If any segment contains `%[0-9A-Fa-f]{2}`: produce a second variant with percent-decoding (`decoded=True`); both are looked up, raw first | `src/api/orders/%5Bid%5D.ts` → also `[id].ts` |
| 9 | NFC-normalize every segment (00 §2.2) | |
| 10 | Diff prefixes: if the first segment is exactly `a` or `b` and the mention kind is DIFF, drop it (other kinds leave it to suffix matching) | |

Absolute prefixes (`/app/`, `/home/runner/work/repo/repo/`, `/Users/me/repo/`, `/usr/src/app/`, container roots, Go module paths `github.com/org/repo/`) are **not** stripped by rules: step 4.4 finds the longest catalog path that is a segment-suffix of the query, which strips exactly the right prefix whatever it is.

### 4.3 Framework / vendor classification

A normalized query is **framework** if any segment (case-insensitive) is in the marker set, or the raw text starts with a runtime marker:

| Marker | Ecosystem |
|---|---|
| `node_modules`, `.pnpm`, `.yarn`, `bower_components` | JS |
| `site-packages`, `dist-packages`, `.venv`, `venv`, `lib/python3.*` (two segments), `<frozen …>`, `<string>` | Python |
| `vendor` (Go, PHP, Ruby), `gems`, `.bundle`, `ruby/*/gems` | Go/Ruby/PHP |
| `.cargo/registry`, `.rustup`, `rustc/<hash>` (segment after `rustc` is 40 hex) | Rust |
| `go/pkg/mod`, `GOROOT`, `usr/local/go/src`, `runtime/*.go` under `go/src` | Go |
| `node:`, `internal/` directly after `node:`, `<anonymous>`, `native` | Node runtime |
| JVM runtime class prefixes (§4.1 step 5); `.m2/repository`, `.gradle/caches` | JVM |
| `Microsoft.NET`, `dotnet/shared` | .NET |

Framework queries are **dropped before lookup** (`dropped["framework"]`), unless `with_suffix`/`suffixes_of` returns a catalog path that itself contains the same marker segment (only possible if the user removed the default exclude for `vendor/`), in which case it's matched normally. Dropping first matters: `/app/node_modules/express/lib/router/index.js` must not become an ambiguous `index.js` candidate or a partial match on some `lib/router/`.

### 4.4 Resolution (per normalized query `q`, reversed segments `r`)

```
resolve(q, fold=False):
  if framework(q): return Dropped("framework")
  L        = index.suffixes_of(r, fold)                        # catalog paths that are suffixes of q, longest first
  best     = L[0] if L else None
  T, total = index.with_suffix(r, fold, limit=ambiguous_max)   # catalog paths ending with q
  B, btot  = index.with_suffix(r[:1], fold, limit=ambiguous_max)   # same basename (only needed for row 8)
  return classify(q, best, T, total, B, btot)                  # decision table below
resolve_mention(m):
  for variant in normalize(m):                                 # raw first, then URL-decoded
      res = resolve(variant)
      if res is not Dropped("no_match"): return res
  if fold_fallback: repeat with fold=True (§4.5)
  return Dropped("no_match")
```

Decision table (evaluated top to bottom; `|q|` = query segments, `best` = longest catalog path that is a suffix of `q`, `T` = catalog paths ending with `q`):

| # | Condition | Class | Strength | Anchor? |
|---|---|---|---|---|
| 1 | `best` exists and `|best| ≥ 2` | SUFFIX | 1.0 | hit |
| 2 | `best` exists, `|best| = 1` (a root-level file like `README.md`), and `|q| = 1` and `total(T) = 1` | SUFFIX | 1.0 | hit |
| 3 | `|q| ≥ 2` and `total(T) = 1` | TAIL | 0.9 | hit |
| 4 | `|q| ≥ 2` and `2 ≤ total(T) ≤ ambiguous_max` | AMBIGUOUS | – | candidates |
| 5 | `|q| = 1` and `total(T) = 1` | BASENAME | 0.8 | hit |
| 6 | `|q| = 1` and `2 ≤ total(T) ≤ ambiguous_max` | AMBIGUOUS | – | candidates |
| 7 | `total(T) > ambiguous_max` | – | – | dropped `too_ambiguous` (e.g. `index.ts`, `__init__.py`) |
| 8 | `best` exists, `|best| = 1` (a root-level file), `|q| ≥ 2`, `T` empty (query `/app/README.md` or `/opt/x/lib/README.md`) | apply rows 5–7 to the **basename** (`B`, `btot`): unique → BASENAME 0.8; 2..5 → AMBIGUOUS; more → dropped | 0.8 | hit / candidates |
| 9 | Nothing | – | – | fold fallback (§4.5), else dropped `no_match` |

Notes:
- Row 1 covers exact matches (`q == best`) and absolute paths with any junk prefix. It fires even when the query has further unmatched leading segments, which is the point: `/home/runner/work/repo/repo/src/x.ts` → `src/x.ts`. When several catalog paths are suffixes of `q` (`src/x.ts` and `packages/a/src/x.ts` for query `/app/packages/a/src/x.ts`), the longest wins.
- Row 8 exists because a one-segment catalog path can't tell an absolute prefix (`/app/README.md`, same file) from a different directory (`/opt/x/lib/README.md`, different file). It gets basename-level strength instead of 1.0; vendored cases are mostly removed earlier by framework markers.
- Row 1 with a directory `best` (query `src/fulfillment/` or `…/src/fulfillment`) produces a **dir hit** (`is_dir=True`). Directory hits use the same strengths.
- **Partial tails.** If rows 1–8 find nothing for a query with `|q| ≥ 3`, retry rows 3–4 with the query's tails `q[-k:]` for `k = |q|-1 … 2` (longest first). The first tail with 1..`ambiguous_max` matches yields **candidates only** (class AMBIGUOUS, never an anchor), because the leading segments disagree with the catalog: query `/build/lib/x.ts` vs catalog `src/lib/x.ts` (tail `lib/x.ts`) is plausibly a copied or built file. Single-segment tails are not tried (that would be a basename match from a conflicting path). Segments are compared exactly; extensions are part of the basename, so `x.js` never matches `x.ts`.

### 4.5 Case-insensitive fallback

Only if both case-sensitive lookups return nothing for **all** variants of a mention: rerun §4.4 with `fold=True` (keys `str.casefold()`). A resulting hit has class `FOLD` and strength × 0.9 (1.0 → 0.9, 0.9 → 0.81, 0.8 → 0.72). This covers Windows and macOS traces with lowercased drive or folder paths (`c:\users\me\repo\src\fulfillment\Ship.ts`). If two catalog paths differ only in case, fold lookups are ambiguous by construction and go to row 4/6/7.

### 4.6 Context disambiguation

After all mentions are resolved: let `H` = directories of the case-sensitive hits (rows 1–3, 5). For each ambiguous mention with candidates `C`: compute for each `c ∈ C` the longest common directory prefix (in segments) with any path in `H`. If exactly one candidate has the maximum and that maximum is ≥ 1 segment, promote it to a hit with class `CONTEXT`, strength 0.8. Example: a trace with frames `packages/api/src/jobs/worker.ts` (hit) and `utils/retry.ts` (ambiguous: `packages/api/src/utils/retry.ts`, `packages/web/src/utils/retry.ts`) promotes the `api` one. Deterministic; one pass (promoted hits don't feed further promotion).

### 4.7 Test detection, ranking, cap

`is_test_path(path)` is true if any directory segment is in {`test`, `tests`, `__tests__`, `spec`, `specs`, `e2e`, `testdata`, `fixtures`, `__mocks__`, `androidTest`} or the basename matches one of: `test_*.py`, `*_test.py`, `conftest.py`, `*_test.go`, `*.test.{js,jsx,ts,tsx,mjs,cjs}`, `*.spec.{js,jsx,ts,tsx,mjs,cjs}`, `*Test.java`, `*Tests.java`, `*Test.kt`, `*Spec.scala`, `*Tests.cs`, `*Test.cs`, `*_spec.rb`, `*_test.rb`, `*Test.php`, `test_*.rs` or `tests/` (covered). Also used by 09 for traces.

Ranking of hits (after dedupe by id; a hit's `strength` is the max over its mentions, `frame_rank` the min, `lines` the union capped at 3):

```
sort key = (is_test,                 # non-test first
            -strength,               # SUFFIX/TAIL before BASENAME/FOLD
            frame_rank,              # nearest-to-error frame first; prose mentions after trace frames
            first_offset,            # earlier in the prompt
            id)                      # stable tie-break
```

Cap: keep the first `max_hits` (10). **Test-slot reservation:** if any test hit exists and none survived the cap, the best test hit replaces the 10th item (a failing test is often a must-have). Everything over the cap is `dropped["over_cap"]`.

Ambiguous candidates: union of all ambiguous mentions' candidate lists minus ids already hits, ordered by (best frame_rank of the mention, id), capped at `max_ambiguous_candidates` (10).

Directory hits count against the same cap and rank with files.

### 4.8 Ids returned

- Files: the tree id (`code:` or `doc:`), never `mig:` (F3). The router maps a migration path hit to its `mig:` id via the `alias` edge (09 §4.4).
- Directories: the directory's current id; comparisons by path (F2).
- Excluded files (`.env`, `dist/…`, lockfiles) aren't in the index, so they can never be returned. This is also why secret files can't leak into a note via a pasted path.

---

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `router.pathmatch.enabled` | bool | true | ablation switch |
| `router.pathmatch.max_hits` | int | 10 | spec §11.3 cap |
| `router.pathmatch.max_ambiguous_candidates` | int | 10 | total ambiguous ids sent to the final pass |
| `router.pathmatch.ambiguous_max` | int | 5 | per-mention; above → dropped `too_ambiguous` |
| `router.pathmatch.scan_max_chars` | int | 262144 | head + tail beyond this |
| `router.pathmatch.max_mentions` | int | 500 | distinct queries resolved |
| `router.pathmatch.fold_fallback` | bool | true | |
| `router.pathmatch.extra_framework_markers` | list[str] | [] | added to §4.3 |
| `router.pathmatch.extra_test_patterns` | list[str] | [] | globs added to §4.7 |

Strengths (1.0 / 0.9 / 0.8, fold × 0.9) are module constants, not config: changing them changes expansion scores and needs an eval run anyway (Q-08-4).

---

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| No path-like characters | fast gate, empty result, < 50 µs |
| 5 MB log pasted | head/tail 256 KB scanned; `stats.scanned_chars` records it |
| Same frame repeated 50× (recursion) | one mention after dedupe; `n_mentions=50` |
| `index.ts` mentioned, 212 in repo | dropped `too_ambiguous` |
| `utils.py` in 3 places, another hit shares `pkg/a/` | CONTEXT promotion of `pkg/a/utils.py` |
| Windows path, different case from catalog | FOLD hit, strength × 0.9 |
| Two catalog files differ only by case (`Readme.md`, `README.md`), exact case in prompt | case-sensitive hit; fold never runs |
| Path with spaces, unquoted (`my docs/setup guide.md`) | not recognized (D-08-3) |
| Path with spaces in Python traceback or quotes | recognized |
| URL-encoded `%5Bid%5D.ts` | decoded variant matches `[id].ts` |
| Next.js `app/(shop)/[slug]/page.tsx` | brackets/parens kept (balanced) |
| Node frame `at fn (/app/src/x.ts:88:21)` | outer parens trimmed, `:88:21` stripped, SUFFIX hit `src/x.ts` |
| `webpack-internal:///./src/x.ts` | prefix stripped |
| Compiled output path `dist/x.js` (not indexed) | no match (Q-08-1) |
| `node_modules/foo/index.js` | framework, dropped before lookup |
| Go module path `github.com/org/repo/pkg/foo/bar.go:12` | SUFFIX `pkg/foo/bar.go` |
| Rust `/rustc/<hash>/library/core/src/panicking.rs` | framework |
| JVM `at com.acme.orders.OrderService.ship(OrderService.java:88)` | query `com/acme/orders/OrderService.java` → TAIL `src/main/java/com/acme/orders/OrderService.java` |
| Kotlin file outside package dir | package query fails → basename fallback |
| Git diff `a/src/x.ts` | `a/` dropped; SUFFIX |
| Prose "see ship.ts" | GENERIC, BASENAME if unique |
| Version numbers `1.2.3`, domains `example.com`, `Node.js`, `e.g.` | recognized as candidates by GENERIC but no catalog match → `no_match`; cost bounded by `max_mentions` |
| Directory mention `src/fulfillment` (no slash) | GENERIC-DIR, dir hit if catalog has the dir |
| Migration path pasted | `code:` hit; router maps to `mig:` |
| Path of an excluded/secret file | no match |
| Path escaping the repo (`../../other-repo/src/x.ts`) | `..` resolved lexically → `other-repo/src/x.ts`; matches only if this repo has that suffix (then it's likely the same layout; accepted risk) |
| Catalog index missing `paths` table (old cache) | `SqlitePathIndex` rebuilds the table from the catalog once (05), or falls back to `TriePathIndex` |
| Regex error / unexpected exception | pipeline guard: path matching returns empty result, route continues (fail open per stage, 09 §6) |

---

## 7. Performance budget

| Scenario (20k-file catalog) | Budget |
|---|---|
| Prompt with no paths | ≤ 0.1 ms |
| Typical prompt with a 15-frame stack trace | ≤ 3 ms (≤ 15 queries × ≤ 12 point lookups + range scans on SQLite) |
| 256 KB log, 500 distinct mentions | ≤ 40 ms |
| `TriePathIndex` build (tests, MCP server) | ≤ 60 ms, ≤ 30 MB |
| SQLite `paths` table size | ≈ 2 × catalog path bytes |

---

## 8. Test plan

### 8.1 Table-driven corpus

`tests/route/pathmatch_corpus.yaml` holds named repo layouts (file lists only, no content) and cases:

```yaml
layouts:
  ts_app: [src/fulfillment/ship.ts, src/api/orders/[id].ts, src/jobs/fulfillment-worker.ts, src/index.ts,
           lib/legacy/ship.ts, app/(shop)/[slug]/page.tsx, README.md, docs/setup guide.md, ...]
cases:
  - id: node-abs-app
    layout: ts_app
    text: "at markShipped (/app/src/fulfillment/ship.ts:88:21)"
    hits: [{id: "code:src/fulfillment/ship.ts", class: suffix, strength: 1.0, lines: [88]}]
```

Every case runs against **both** `SqlitePathIndex` and `TriePathIndex`. Minimum corpus (≥ 120 cases); representative rows:

| # | Group | Input (abridged) | Expected |
|---|---|---|---|
| 1 | Node | `at markShipped (/app/src/fulfillment/ship.ts:88:21)` | SUFFIX `src/fulfillment/ship.ts`, line 88 |
| 2 | Node | `at /home/runner/work/repo/repo/src/jobs/fulfillment-worker.ts:42:9` | SUFFIX |
| 3 | Node | `at async Promise.all (index 0)` | nothing |
| 4 | Node | `at Module._compile (node:internal/modules/cjs/loader:1105:14)` | framework |
| 5 | Node | `(/app/node_modules/express/lib/router/index.js:284:15)` | framework, no `index.js` candidates |
| 6 | Bundler | `webpack-internal:///./src/api/orders/[id].ts` | SUFFIX `[id].ts` |
| 7 | Bundler | `webpack:///src/x.ts?9a3f` | SUFFIX |
| 8 | URL | `file:///C:/Users/me/repo/src/x.ts` | SUFFIX |
| 9 | URL | `https://github.com/o/r/blob/main/src/fulfillment/ship.ts#L12-L20` | SUFFIX, line 12 |
| 10 | URL-enc | `src/api/orders/%5Bid%5D.ts` | SUFFIX via decoded variant |
| 11 | Python | `File "/usr/src/app/app/billing/invoice.py", line 41, in total` | SUFFIX |
| 12 | Python | full traceback, 4 frames | frame_rank innermost = last frame |
| 13 | Python | `File "/usr/lib/python3.11/site-packages/django/db/models.py"` | framework |
| 14 | Python | `File "<frozen importlib._bootstrap>"` | framework |
| 15 | Python | `File "/app/docs/setup guide.md", line 1` | SUFFIX (space inside quotes) |
| 16 | pytest | `tests/test_billing.py::test_total[eur] FAILED` | SUFFIX, is_test |
| 17 | pytest | `tests/test_billing.py:41: AssertionError` | SUFFIX, line 41 |
| 18 | JVM | `at com.acme.orders.OrderService.ship(OrderService.java:88)` | TAIL `src/main/java/com/acme/orders/OrderService.java` |
| 19 | JVM | `at com.acme.Outer$Inner.run(Outer.java:5)` | TAIL `…/com/acme/Outer.java` |
| 20 | JVM | `at java.base/java.lang.Thread.run(Thread.java:833)` | framework |
| 21 | Kotlin | `at com.acme.Util.f(Helpers.kt:3)` with file at `src/main/kotlin/Helpers.kt` | BASENAME fallback |
| 22 | Go | `\t/home/u/go/src/github.com/acme/svc/pkg/store/orders.go:123 +0x1d` | SUFFIX `pkg/store/orders.go` |
| 23 | Go | `/usr/local/go/src/runtime/panic.go:884` | framework |
| 24 | Go | `/home/u/go/pkg/mod/github.com/lib/pq@v1.10.9/conn.go:12` | framework |
| 25 | Rust | `thread 'main' panicked at src/main.rs:10:5:` | SUFFIX |
| 26 | Rust | `panicked at 'index out of bounds', src/lib.rs:12:5` (old format) | SUFFIX |
| 27 | Rust | `at /rustc/5680fa18feaa87f3ff04063800aec256c3d4b4be/library/core/src/panicking.rs:72:14` | framework |
| 28 | Rust | `~/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.32.0/src/runtime/mod.rs` | framework |
| 29 | .NET | `at Acme.Orders.Ship() in C:\src\Acme\Orders\Ship.cs:line 42` | SUFFIX, line 42 |
| 30 | Ruby | `/app/app/models/order.rb:12:in 'save!'` | SUFFIX |
| 31 | Ruby | `/usr/local/bundle/gems/activerecord-7.0/lib/x.rb:5` | framework |
| 32 | PHP | `#0 /var/www/src/Order.php(12): Order->ship()` | SUFFIX, line 12 |
| 33 | PHP | `in /var/www/vendor/laravel/framework/src/x.php on line 5` | framework |
| 34 | gcc | `src/net/sock.c:120:7: error: implicit declaration` | SUFFIX |
| 35 | MSVC | `src\net\sock.cpp(120,7): error C2065` | SUFFIX |
| 36 | tsc | `src/x.ts(12,5): error TS2345` and `src/x.ts:12:5 - error TS2345` | SUFFIX |
| 37 | Jest | `FAIL src/x.test.ts` | SUFFIX, is_test |
| 38 | Diff | `diff --git a/src/x.ts b/src/x.ts` | SUFFIX (once after dedupe) |
| 39 | Prose | `why does ship.ts double-send?` | BASENAME (unique) |
| 40 | Prose | `ship.ts` with `src/fulfillment/ship.ts` and `lib/legacy/ship.ts` | AMBIGUOUS, 2 candidates |
| 41 | Prose | `index.ts` with 212 matches | dropped too_ambiguous |
| 42 | Prose | `` `src/fulfillment/` `` | dir hit |
| 43 | Prose | `look in src/fulfillment` | dir hit |
| 44 | Prose | `README.md` (root only) | SUFFIX |
| 45 | Root file, deep query | `/app/README.md` with only root `README.md` (and no other `README.md`) | BASENAME 0.8 (row 8) |
| 45b | Root file, deep query, ambiguous | `/opt/pkg/lib/README.md` with root + 3 other `README.md` | AMBIGUOUS, 4 candidates |
| 45c | Partial tail | `/build/lib/x.ts:3` with catalog `src/lib/x.ts` only | AMBIGUOUS candidate `src/lib/x.ts` (not a hit) |
| 46 | Fold | `c:\users\me\repo\SRC\fulfillment\ship.ts` | FOLD 0.9 |
| 47 | Fold tie | `readme.md` with `Readme.md` and `README.md` present | AMBIGUOUS via fold |
| 48 | Context | trace hit `packages/api/src/jobs/w.ts` + `utils/retry.ts` (api, web) | CONTEXT `packages/api/src/utils/retry.ts` |
| 49 | Next.js | `app/(shop)/[slug]/page.tsx:3:1` | SUFFIX |
| 50 | Wrappers | `(see src/x.ts).` / `<src/x.ts>` / `"src/x.ts",` | SUFFIX |
| 51 | Noise | `v1.2.3`, `e.g.`, `Node.js`, `example.com`, `3.14` | nothing |
| 52 | Unquoted space | `my docs/setup guide.md` | no hit for `setup guide.md` (might match `guide.md` if one exists: documented) |
| 53 | Excluded | `.env.local:3` | nothing |
| 54 | Cap | 14 distinct user frames + 2 test frames | 10 hits, ≥ 1 test hit reserved, `over_cap=6` |
| 55 | Ranking | Python traceback | innermost frame first among equal strength |
| 56 | Migration | `supabase/migrations/20260611_add_shipments.sql:4` | `code:` hit (router maps to `mig:`) |
| 57 | Makefile | `make: *** [Makefile:12: build] Error 1` | SUFFIX `Makefile` via extensionless set |
| 58 | Big input | 5 MB log with trace at the end | tail scanned, hit found |

### 8.2 Property tests (`hypothesis`)

- For any catalog path `p` and random decoration (absolute prefix from a set, scheme, `:L:C`, `(L,C)`, wrapper punctuation, backslashes, URL-encoding of bracket chars), `match_paths` returns a hit with `id(p)` whenever `p`'s basename-with-2-segments suffix is unique.
- `normalize` is idempotent on its own output.
- Result is independent of mention order except `frame_rank`/`first_offset` fields.
- Sqlite and trie implementations return identical results on random layouts.

### 8.3 Performance tests

Benchmarks for §7 rows on a synthetic 20k-path layout; CI fails on > 2× regression.

---

## 9. Acceptance criteria

| Phase | Criterion |
|---|---|
| Phase 2 exit ("path matching with normalization tests") | Full corpus (≥ 120 cases) green on both index implementations; property tests green; `stack_trace` category recall of path-labeled `must_include` ≥ 0.95 on dev (path-matchable labels only) |
| Phase 2 exit | Zero framework-frame hits across the corpus; budget rows in §7 met |
| Phase 5 exit | Pipeline fail-open test: a raising `match_paths` doesn't break the route |

---

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-08-1 | Two strengths: exact suffix 1.0, unique basename 0.8 | Four: SUFFIX 1.0, TAIL (query is a unique tail of a longer catalog path) 0.9, BASENAME/CONTEXT 0.8, FOLD × 0.9 | "Exact suffix" is ambiguous about direction; `fulfillment/ship.ts` for `src/fulfillment/ship.ts` is weaker evidence than a full-path match |
| D-08-2 | Regex requires an extension | Also directory mentions and known extensionless basenames (`Makefile`, `Dockerfile`) | Users name directories in prose; build errors cite Makefiles |
| D-08-3 | Silent on spaces | Paths with spaces only inside quotes, backticks, Python `File "…"`, and .NET `in … :line` | Unquoted spaces make tokenization ambiguous; false positives are worse than rare misses |
| D-08-4 | Silent on JVM frames | JVM frames map `package.Class` to a package path, then fall back to the file name | JVM traces don't print paths |
| D-08-5 | Ambiguous basename → final-pass candidate | Only if ≤ 5 candidates; context disambiguation can promote one to a hit | 212 `index.ts` candidates would flood the pool |
| D-08-6 | "Prefer non-test, non-framework frames" | Framework frames dropped before lookup; tests demoted but one test slot reserved | Framework paths aren't in the catalog and only produce false basename matches; failing tests are often the point |
| D-08-7 | Silent on case | Case-insensitive fallback only when the case-sensitive pass finds nothing | Windows/macOS traces |
| D-08-8 | Match on prompt text | Match on the raw prompt before redaction | Redaction destroys hash-like path segments; output is ids only, so nothing leaks |
| D-08-9 | Suffix structure unspecified | SQLite reversed-path table in production, in-memory trie in tests/MCP | The hook is a fresh process per prompt; building a trie each time is wasted work |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-08-1 | Map compiled paths (`dist/x.js`, `build/x.js`) to sources via naming conventions or source maps | Not in v1 | Eval: share of stack_trace misses whose trace only has `dist/` frames |
| Q-08-2 | Put `:line` in the note for path hits | No (pointers only) | Agent behavior study in Phase 4 |
| Q-08-3 | Should path hits in prose (no trace) be as strong as trace frames? | Same strengths | Eval: precision of prose-mention hits vs trace hits |
| Q-08-4 | Strength constants (1.0/0.9/0.8/× 0.9) | As listed | A8-style sweep on stack_trace queries once expansion (Phase 3) lands |
| Q-08-5 | `ambiguous_max` 5 | 5 | Eval: truncation losses on ambiguous mentions |
| Q-08-6 | Should a directory hit seed the walk (start the walk at that dir) rather than only entering the pool? | Pool + flatten if small (09 §4.4) | Eval on prompts that name directories |
