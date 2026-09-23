# 06 · Refresh: incremental builds, triggers, `--check`

**Status:** draft for review
**Spec sections:** §10 (10.1–10.3), §9.5 (commit policy), §15.4 `SessionStart`, §8.2.6, §23 Phase 5
**Depends on:** 00-foundations, 01-discovery, 02-cards, 03-schema-extraction, 04-graph-edges, 05-catalog-store, 10-lease (staleness contract), 12-delivery (hook entrypoints, `surf init`)
**Code:** `surf/index/build.py`, `surf/adapters/git_hooks.py`

---

## 1. Purpose and scope

Keep the catalog consistent with the working tree and history, cheaply and without ever blocking the user or the agent.

| In scope | Out of scope |
|---|---|
| Commit policy decision and its workflow (§4.1) | Card and edge algorithms themselves (02–04) |
| Full and incremental build orchestration; proof that they agree | File watchers (v2, §21) |
| Change detection, git and non-git | Lease state machine (10); this doc only defines what refresh exposes |
| Git hook scripts and hook-manager integration | Claude Code hook wiring (12) |
| Locking, background execution, coalescing, SessionStart time box | |
| `surf index --check` semantics for both policies | |

---

## 2. Interfaces

```python
# surf/index/build.py
class BuildInputs(BaseModel, frozen=True):
    root: Path
    history_ref: str | None            # commit used for co-change; None = no git
    content: Literal["worktree", "commit"]   # "commit": read blobs of history_ref (baseline/check)

def build_full(inputs: BuildInputs, cfg: Config, *, clock: Clock) -> BuildOutput           # pure; no publish
def refresh(paths: CatalogPaths, cfg: Config, *, mode: Literal["changed", "full"],
            reason: str, clock: Clock) -> RefreshReport                                    # takes lock, publishes
def trigger_background(paths: CatalogPaths, *, reason: str, full: bool = False) -> TriggerResult
def check_freshness(paths: CatalogPaths, cfg: Config, *, deep: bool) -> Freshness            # no lock, read-only
def ensure_fresh(paths: CatalogPaths, cfg: Config, *, wait_ms: int) -> Freshness            # SessionStart / first call
def check(paths: CatalogPaths, cfg: Config, *, trust_cochange: bool = False,
          allow_lag: bool = False) -> CheckReport                                          # `surf index --check`
def write_baseline(paths: CatalogPaths, cfg: Config) -> RefreshReport                      # `surf index --baseline`

# surf/adapters/git_hooks.py
def detect_hook_setup(root: Path) -> HookSetup
def plan_install(setup: HookSetup) -> list[HookAction]          # shown in the `surf init` plan
def apply(actions: Sequence[HookAction]) -> InstallManifest
def uninstall(root: Path) -> list[str]                          # reads the manifest, removes only our blocks
def verify(root: Path) -> list[HookProblem]                     # `surf doctor`
```

CLI (12 owns parsing): `surf index [--full] [--check [--trust-cochange] [--allow-lag]] [--baseline]`, `surf refresh [--changed|--full] [--background] [--reason R] [--quiet]`.

---

## 3. Data structures

```python
class Freshness(BaseModel, frozen=True):
    state: Literal["fresh", "stale", "missing", "refreshing", "unknown"]
    reasons: list[str]                 # "head_moved", "worktree_changed", "config_changed", "caps_changed", …
    index_head: str | None
    head: str | None

class RefreshReport(BaseModel, frozen=True):
    kind: Literal["full", "incremental", "noop"]
    changed_ids: list[SurfaceId]       # cards whose `hash` changed, sorted
    added_ids: list[SurfaceId]
    removed_ids: list[SurfaceId]
    edges_changed: int
    cochange: Literal["incremental", "full", "skipped", "unavailable"]
    ms: int

class CheckReport(BaseModel, frozen=True):
    exit_code: Literal[0, 1, 2]        # 0 ok · 1 mismatch or stale · 2 cannot verify
    reasons: list[str]
    diffs: list[str]                   # first 20 differing ids / edge keys
```

### 3.1 Build state (`cache/build.sqlite`, mutable, only written under the refresh lock)

```sql
PRAGMA user_version = 1;
CREATE TABLE extract (                 -- per-file extraction results (01/02/03 extractors)
  path TEXT NOT NULL, extractor TEXT NOT NULL,       -- e.g. "code@1", "docs@1"
  content_id TEXT NOT NULL, cfg_fp TEXT NOT NULL,
  result TEXT NOT NULL,                              -- canonical JSON
  PRIMARY KEY (path, extractor)) WITHOUT ROWID;
CREATE TABLE mentions (                -- schema-ref scan results (04 §4.4)
  path TEXT PRIMARY KEY, content_id TEXT NOT NULL, matcher_fp TEXT NOT NULL,
  tables_scanned TEXT NOT NULL,        -- sorted JSON list of "table_id:variant_fp"
  result TEXT NOT NULL) WITHOUT ROWID; -- {table_id: [n, best_mult]}
CREATE TABLE file_state (              -- stat cache for content ids
  path TEXT PRIMARY KEY, size INTEGER NOT NULL, mtime_ns INTEGER NOT NULL,
  content_id TEXT NOT NULL) WITHOUT ROWID;
CREATE TABLE inputs (key TEXT PRIMARY KEY, value TEXT NOT NULL) WITHOUT ROWID;
  -- "schema_sources_fp", "caps_sources" (path,size,mtime,sha list incl. user-level configs),
  -- "dirty_paths", "config_fps" (per section), "last_head"
```

`content_id` = git blob id (`sha1("blob <len>\0" + bytes)`). For clean tracked files it comes free from `git ls-files -s`. For dirty and untracked files it's computed from the working-tree bytes. Non-git: the same formula over the bytes. Blob ids match whenever no clean/smudge filter is in play. Filters only cause cache misses, never wrong results.

---

## 4. Behaviour

### 4.1 Commit policy (critical workflow issue): options and recommendation

**The problem.** The spec commits `catalog.jsonl`/`edges.jsonl`/`meta.json` by default *and* has `post-commit` hooks run `surf refresh`. Consequences:

1. After every commit, the hook rewrites tracked files, so the working tree is permanently dirty with a catalog that describes the previous commit.
2. `meta.index_head` can never equal the commit that contains it (a commit can't contain its own hash).
3. Co-change weights change on almost every commit, so every pair of parallel branches conflicts in `edges.jsonl`.
4. CI `surf index --check` with `actions/checkout` defaults (`fetch-depth: 1`) has no history, so co-change can't be reproduced and the check always fails.

**Options**

| | (a) Local-only catalog | (b) Pre-commit regenerate + stage | (c′) Committed baseline + local working copy | (d) CI bot commits catalog on main |
|---|---|---|---|---|
| Mechanism | `commit_catalog=false`; catalog in `.surf/cache/`; built by `surf init`, hooks, SessionStart | `pre-commit` hook rebuilds from the staged tree with co-change at `HEAD` (the parent) and stages the files | Committed files are a baseline refreshed only by `surf index --baseline` (human or bot); hooks and SessionStart refresh a local copy in `cache/` | Post-merge CI job builds and pushes a catalog commit |
| Dirty tree after commit | Never | Never | Never | Never locally |
| `index_head` meaning | HEAD at build | Parent of the commit (content = the commit's tree) | Baseline commit B; fresh if no indexed file changed in B..HEAD | Commit before the bot commit |
| Merge conflicts | None | On every parallel branch (edges change per commit) | Only when two branches both regenerate the baseline | Branches rarely touch it; the bot's commit can race |
| Commit latency | 0 (background) | +0.5–3 s per commit, blocking; bypassed by `--no-verify` | 0 | 0 |
| Fresh clone | Cold build (≈ 5–60 s, background) | Ready | Ready (baseline → incremental) | Ready on main; stale on branches |
| CI `--check` | Nothing committed to check; optional determinism self-check | Needs full history; flaky with amend/rebase | Well defined (§4.7), needs `fetch-depth: 0` | Mostly moot; the bot needs push rights |
| Complexity | Lowest | Medium; fights rebases, amends, stashes | Medium (two locations, one code path) | High (CI permissions, bot noise) |
| Teammates without surf | Unaffected | Every committer needs surf, or catalogs drift | Unaffected | Unaffected |

**Recommendation: (a) as the default, with (c′) as the supported opt-in** (`index.commit_catalog = true`). (d) is just (c′) with a bot running `surf index --baseline`, documented as a recipe rather than built in. (b) is rejected: blocking commits and guaranteed merge conflicts are worse than a cold build.

Rationale: routing needs a fresh local catalog anyway (a teammate's committed catalog doesn't describe my working tree), and hooks keep a local one fresh at no commit-time cost. A local build is cheap (Phase 1 exit: < 60 s for 5k files, typically 5–15 s). Committing buys a warm start for fresh clones and reviewable diffs, which (c′) still offers to teams that want it. **Flagged for the user as Q-06-1** (this reverses the spec default §9.5; foundations §4 needs a matching note).

In every mode, hooks, SessionStart and routes read and write **only** `.surf/cache/`. Tracked files change only through an explicit `surf index --baseline`.

### 4.2 Build model (why incremental equals full)

A build is `assemble(I)` over six inputs. Each is either computed or loaded from a content-addressed cache:

| Input | Full build | Incremental | Cache key |
|---|---|---|---|
| I1 file list (01) | enumerate | enumerate (always; ≈ 50 ms) | – |
| I2 per-file extraction (02) | extract all | reuse on key hit | `(path, extractor, content_id, cfg_fp)` |
| I3 schema model (03) | replay migrations | replay iff `schema_sources_fp` changed, else reuse | fp over schema sources' `(path, content_id)` + dialect |
| I4 schema mentions (04 §4.4) | scan all | per file: rescan if `content_id` or `matcher_fp` changed; else scan only tables missing from `tables_scanned`; drop removed tables | `(path, content_id, matcher_fp, table)` |
| I5 co-change window (04 §4.2) | read the whole window | `update_window` (preconditions in 04 §4.2.7, else full) | `(head, config_fp, git_version)` |
| I6 capability listings (02) | extract | re-extract iff `caps_sources` changed; live listings reused | stat + sha of config files |

`assemble` (tree → co-change compute → schema edges → **render every card** → sort → serialise) is a pure function and runs in full every time. Rendering 10k cards costs ≈ 200 ms, and it removes the spec's steps 6–7 (ancestor roll-up, `changes_with` recompute) as a source of bugs. Every per-file result depends only on its key, and mentions are per table and independent (04 §4.4.3 classifies spans per table), so the cached inputs equal freshly computed ones. It follows that `assemble(I_incremental) == assemble(I_full)` byte for byte. The property test in §8 enforces this.

### 4.3 Change detection

**Git projects**

1. `git rev-parse HEAD`; `git --no-optional-locks status --porcelain=v2 -z --untracked-files=all` (never takes `index.lock`, so it can't break a concurrent `git commit`).
2. `git ls-files -s -z` → `content_id` for clean tracked files.
3. Dirty and untracked paths: `(size, mtime_ns)` compared with `file_state`; hash only on mismatch.
4. The changed set is implicit: files whose `content_id` misses the I2/I4 caches. No `git diff <index_head>..HEAD` is needed for content (D-06-2). That makes checkout, rebase, stash and reset safe by construction. History ranges are only used for co-change (I5).

**Non-git projects**

- Walk the root with 01's excludes (no `.gitignore` semantics beyond what 01 implements); `content_id` via the `file_state` stat cache. No co-change (`cochange_status="no_git"`), no hooks. Freshness is checked by the stat walk.

### 4.4 Incremental algorithm (replaces spec §10.2)

```
refresh(mode="changed"):
  0. acquire lock (§4.6); read meta + build.sqlite; on any cache corruption -> mode="full"
  1. I1  = discover()                                   # 01
  2. ids = content ids (§4.3)
  3. cfg fingerprints per section; a changed section invalidates exactly its caches
  4. I2  = [cache.get(k) or extract(f) for f in I1]     # write misses back
  5. I3  = schema_sources_fp changed ? replay() : cached
  6. I4  = mentions(I1, I3.tables)                      # 04 §4.4, incremental per table
  7. I5  = git ? update_window(state, HEAD) : None      # full rebuild if preconditions fail
  8. I6  = caps_sources changed ? extract_caps() : cached
  9. out = assemble(I1..I6)
 10. if out.catalog_bytes == old and out.edges_bytes == old: publish meta only (index_head, fps)
     else: stage + publish (05 §4.4)
 11. report changed/added/removed ids (by card hash), write cache/last_refresh.json
 12. release lock; handle pending marker (§4.6)
```

Removed files simply don't appear in I1. Their cards and every edge touching them vanish in `assemble`. A directory whose prefix flips (F2) shows up as remove + add in the report.

### 4.5 Triggers

| Trigger | What runs | Wait |
|---|---|---|
| `post-commit` | `trigger_background("post-commit")` | none |
| `post-merge` | same | none |
| `post-checkout` | only when `$3 = 1` (branch checkout) and `$1 ≠ $2` | none |
| `post-rewrite` (`amend`, `rebase`) | same | none |
| Claude Code `SessionStart` (12) | `ensure_fresh(wait_ms = refresh.session_start_wait_ms)` | ≤ 3 s |
| MCP server start / new `session_id` | `ensure_fresh(wait_ms=0)` | none |
| CLI `surf route` | cheap check (HEAD + stamp) at most every `refresh.check_interval_s`; triggers background if stale | none |
| `surf refresh` / `surf index` | foreground | – |
| CI | `surf index --check` (§4.7) | – |

Hooks skip (touch `refresh.pending`, exit 0) while a rebase, merge, cherry-pick, revert or bisect is in progress: `rebase-merge/`, `rebase-apply/`, `MERGE_HEAD`, `CHERRY_PICK_HEAD`, `REVERT_HEAD` or `BISECT_LOG` exists under `git rev-parse --git-path`. The final `post-rewrite`/`post-commit`/`post-checkout` catches up. Also skipped: `SURF_SKIP_HOOKS=1`, `refresh.git_hooks=false`, and worktrees without `.surf/cache/meta.json` (surf never built here, so no surprise cold builds for teammates).

**`ensure_fresh(wait_ms)`**

1. `check_freshness(deep=True)`: compares `HEAD` with `meta.index_head`; the dirty-path set plus stats with `inputs.dirty_paths`/`file_state`; config fingerprints; and `caps_sources` stats (including user-level configs such as `~/.codex/config.*` when opted in). Target ≤ 150 ms.
2. `fresh`: return. `missing`: trigger a full background build. `stale`: trigger a background refresh.
3. Poll every 50 ms until `meta.json`'s `index_head`/fingerprints match, or the lock is released with no pending marker, or `wait_ms` elapses. Return the resulting `Freshness`. On timeout the refresh **keeps running** in the background (never killed mid-build). Routes serve the previous consistent snapshot, and decision records carry `index_stale: true`.

The time box is a *wait* bound, not a work bound, so a partial index is never produced (D-06-3).

### 4.6 Locking, background execution, coalescing

- Lock file: `.surf/cache/refresh.lock`, `fcntl.flock(LOCK_EX|LOCK_NB)` on POSIX and `msvcrt.locking` on Windows. It contains `pid`, `host`, `started_at` for diagnostics only. Kernel locks die with the process, so no stale-lock handling is needed. On filesystems without `flock` (some NFS), fall back to `O_CREAT|O_EXCL` with a 10-minute staleness takeover.
- `trigger_background`:
  1. Touch `refresh.pending`.
  2. Try the lock. If busy, exit 0; the holder will see the pending marker.
  3. If acquired, spawn `surf refresh --changed` via `subprocess.Popen(start_new_session=True, pass_fds=[lock_fd], stdin=DEVNULL, stdout/err → logs/refresh.log)`. The child inherits the locked file description, and the parent exits without unlocking. There is no window where the lock is free between parent and child.
- Runner loop (inside the lock): `while pending exists and loops < refresh.max_loops: delete pending; run once`. After release, if `pending` exists again, retry the lock once and loop. This closes the race where a trigger lands between the last check and the release.
- The background runner applies `os.nice(10)` and `ionice` best effort.
- Foreground `surf refresh`/`surf index` block on the lock for up to 30 s, then error with "refresh in progress".
- **Readers never take the lock.** They read the immutable `index.sqlite` snapshot (05 §4.2). Swapping it with `os.replace` is atomic, and open connections keep the old inode.
- `build.sqlite` and `cochange.state` are only touched under the lock.

### 4.7 `surf index --check` semantics

**Common comparison rules**

| Artifact | Compared | Ignored |
|---|---|---|
| `catalog.jsonl` | byte-exact | – |
| `edges.jsonl` | same `(from,to,kind)` key set; `evidence` exact; `|weight Δ| ≤ 0.0001` | – |
| `meta.json` | all fields | `built_at`, `build`, `surf_version`, `git_version` (warn if they differ) |

Output: exit code (0 ok, 1 mismatch or stale, 2 cannot verify) plus up to 20 diff lines.

**Default policy (`commit_catalog=false`)**

- `--check` compares the local `cache/` catalog with a fresh `build_full` of the current working tree and `HEAD`. It catches incremental-refresh bugs and staleness. With no local catalog it performs a build, validates invariants (05 §3.4) and exits 0 with "nothing to compare".
- CI needs no check. Teams running `surf eval` in CI need `fetch-depth: 0` for co-change to match local results. In a shallow clone, `surf index` works but meta records `cochange_status="shallow"`, and eval reports flag it.

**Committed baseline (`commit_catalog=true`)**

1. Load the committed `.surf/meta.json`; verify file digests (mismatch → 1).
2. `B = meta.index_head`. If `B` is missing locally or not an ancestor of `HEAD` → **2** ("baseline commit not in history; use `fetch-depth: 0`").
3. Staleness: `git diff --name-only -z B HEAD` filtered to paths that would be indexed (01 excludes applied), schema sources, capability config files and `.surf/config.toml`. Any hit → **1** ("stale: N indexed files changed since B; run `surf index --baseline`"), unless `--allow-lag`.
   - This is what makes the committed baseline workable. A baseline built at `B` and committed in `C`, where `C` touches only `.surf/`, is **fresh at `C`**. `index_head` never needs to equal the commit that contains it.
4. Co-change needs the full window history behind `B`. If the repo is shallow → **2**, unless `--trust-cochange`. In that case, `co_change`/`dir_coupling` edges and each card's `churn` field are taken from the committed files as inputs, and everything else is rebuilt and compared.
5. Build `build_full(BuildInputs(history_ref=B, content="worktree"))`. Step 3 guarantees the indexed content at `HEAD` equals `B`'s. The working tree must be clean for indexed paths (else **2**). With `--allow-lag`, build from a temporary `git worktree add --detach <tmp> B` instead, and remove it afterwards.
6. Capability cards: baselines include only repo-level capability sources (never user-level configs). Live MCP listings are taken from the committed catalog as inputs (CI can't start servers). See Q-06-4.
7. Compare with the common rules above.

CI recipe (documented, committed mode only):

```yaml
- uses: actions/checkout@v4
  with: { fetch-depth: 0 }          # required: co-change reads up to 24 months / 5,000 commits
- run: uvx jev-surfer index --check  # exit 2 is a setup problem, not a stale catalog
```

`surf index --baseline` refuses to run on a working tree that is dirty for indexed paths. It builds with `history_ref=HEAD`, writes `.surf/{catalog.jsonl,edges.jsonl,meta.json}` (05 §4.5) and prints "commit these files". The bot recipe (option d) runs exactly this on `main` and commits the result.

### 4.8 Hook installation

**Block** (identical for every hook; inserted **after the shebang line** so an existing `exit`/`exec` at the end can't skip it; our block never exits and never fails):

```sh
# >>> surf >>> managed by `surf init`; remove with `surf uninstall`
if [ -z "$SURF_SKIP_HOOKS" ] && [ -f .surf/cache/meta.json ] && command -v surf >/dev/null 2>&1; then
  ( surf refresh --changed --background --quiet --reason "<hook>" "$@" </dev/null >/dev/null 2>&1 & ) || true
fi
# <<< surf <<<
```

`surf refresh` receives the hook arguments to apply the `post-checkout` `$3` rule. Shell-level guards keep the hook under ~5 ms when surf isn't in use.

**Where it goes** (`detect_hook_setup` → `plan_install`, shown in the init plan; never replace a file):

| Setup detected | Action |
|---|---|
| No manager, no `core.hooksPath` | `<git-common-dir>/hooks/<hook>`: create (`#!/bin/sh` + block, chmod +x) or insert the block into the existing shell hook |
| `core.hooksPath` set to a custom dir | Same, in that dir |
| Existing hook with a non-shell shebang (python, node) | Don't modify; print a manual snippet; `doctor` reports "manual step pending" |
| **husky** v9 (`.husky/` + `core.hooksPath=.husky/_`) | Insert the block into `.husky/<hook>` (create if absent; husky v9 files need no shebang). These files are **committed**, so the plan says so and asks; the shell guards make them inert for teammates without surf |
| husky v4 (`"husky": {"hooks"}` in `package.json`) | Don't edit `package.json`; print a snippet |
| **lefthook** (`lefthook.yml`, `.lefthook.yml`, `lefthook.yaml`) | Never edit the committed YAML. Create or append `lefthook-local.yml` (local, conventionally ignored) with `post-commit`/`post-merge`/`post-checkout`/`post-rewrite` → `commands: surf-refresh: run: surf refresh --changed --background --quiet {0}` (hook args); if that file already defines one of these hooks → print a snippet. Then run `lefthook install` if on PATH, else tell the user to |
| **pre-commit** framework (`.pre-commit-config.yaml`) | It only owns hook types installed with `pre-commit install -t <type>`. If `.git/hooks/<hook>` was generated by pre-commit (contains `File generated by pre-commit`), write our block into `<hook>.legacy`, which pre-commit's generated hook runs first. Otherwise install natively; a later `pre-commit install -t <hook>` moves our file to `.legacy` automatically (Q-06-5: verify on current pre-commit) |
| Multiple managers | Use the one that owns `core.hooksPath`, else the native dir; report |

- The install manifest is written to `<git-common-dir>/surf/manifest.json` (it survives `cache/` deletion): files touched, whether created, and a block hash. `uninstall` removes only the marked block, deletes a file only if surf created it and it's now empty except for a shebang, and deletes `.legacy` files only if surf created them.
- Idempotent: an existing block with the same hash means no-op; a different hash means the block is replaced in place.
- Git worktrees share the common hooks dir. The `.surf/cache/meta.json` guard makes the hook act per worktree.

### 4.9 Contract with the lease (10) and observability (15)

- Leases should store the `hash` of each selected card. The lease is stale if any selected id is missing or its hash differs (no push notification needed). `cache/last_refresh.json` (`RefreshReport`) is also written for `surf status` and optional use by 10.
- Every refresh appends one line to `logs/refresh.log`: reason, kind, ms, counts, co-change mode, errors. No paths from outside the repo.

---

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `index.commit_catalog` | bool | `false` | §4.1, D-06-1 |
| `refresh.git_hooks` | bool | `true` | installed by `surf init` |
| `refresh.hooks` | list | `["post-commit","post-merge","post-checkout","post-rewrite"]` | |
| `refresh.session_start_wait_ms` | int | 3000 | spec's 3 s time box |
| `refresh.check_interval_s` | int | 60 | CLI cheap-check throttle |
| `refresh.max_loops` | int | 3 | coalescing loop bound |
| `refresh.foreground_lock_wait_s` | int | 30 | |
| `refresh.background_nice` | int | 10 | |

Changing `index.*` config invalidates only the caches keyed on that section's fingerprint (§4.2).

---

## 6. Edge cases and failure behaviour

| Case | Behaviour |
|---|---|
| Commit during a running refresh | Hook touches pending; runner loops once more |
| Rebase of 50 commits | Hooks skip while in progress; one refresh after `post-rewrite` |
| `git checkout` to an unrelated branch | Content via caches (mostly hits); `H0` not an ancestor → full co-change rebuild (≤ 15 s, background) |
| `git reset --hard HEAD~3` / amend | `H0` not an ancestor → full co-change; content unaffected |
| `git stash` / `stash pop` | No hook fires; detected by the next freshness check (dirty set changed) |
| Detached HEAD, bisect | Refresh works on the detached commit; skipped during bisect |
| `index_head` commit garbage-collected | Irrelevant for content; co-change full rebuild |
| History rewritten with a backward committer clock | Full co-change (04 §4.2.7) |
| Shallow clone | Co-change from the available history; `cochange_status="shallow"`; doctor suggests `git fetch --unshallow` |
| `git` not on PATH but `.git` exists | Treated as non-git; doctor error |
| Refresh crashes | Lock released by the OS; staged dir left; the previous snapshot keeps serving; the next trigger retries; `refresh.log` has the traceback |
| Refresh raises on a bad file | That extractor records a warning and the file gets a minimal card; the build continues (01/02 rules) |
| Two worktrees | Separate `.surf/cache/`, separate locks |
| Lock dir on NFS without flock | `O_EXCL` fallback with 10-minute takeover |
| Hook runs with a different Python/venv | Hooks call `surf` from PATH only; if missing, no-op |
| GUI git clients with a minimal PATH | Hook no-ops; SessionStart catches up; doctor explains |
| `SessionStart` with a missing index on a huge repo | Background full build; session proceeds with `index-missing` routes until it's done |
| `--check` on Windows with CRLF checkout | `content_id` from blob ids for clean files, so line endings don't matter |
| User edits committed baseline files by hand | `--check` → 1 (digest or content mismatch) |
| `commit_catalog` switched true → false | `surf init --reconfigure` removes the committed files via `git rm --cached` (printed, not auto-committed) |

---

## 7. Performance budget

| Path | Target (5k files) |
|---|---|
| Hook script overhead on commit | ≤ 5 ms (shell guards; background spawn) |
| `check_freshness(deep=True)` | ≤ 150 ms |
| Incremental refresh after a typical commit (≤ 20 files, ≤ 5 commits) | ≤ 3 s total (**Phase 5 exit**); breakdown ≈ 0.1 s discover and ids, 0.3 s extraction, 0.1 s mentions, 0.3 s co-change, 0.3 s assemble, 0.2 s serialise, 1.0 s `index.sqlite` |
| Refresh with no changes | ≤ 400 ms (step 10 short-circuit, meta only) |
| Full build | < 60 s (**Phase 1 exit**) |
| `--check` (committed mode) | full-build time + ≤ 2 s |

---

## 8. Test plan

**Unit**
- Freshness: HEAD moved; dirty file edited twice with the same status letter (caught via stat); config changed; user-level caps config changed; untracked file added.
- Hook skip logic: each in-progress marker; `post-checkout` flags; the missing-`meta.json` guard.
- Lock: trigger while held → pending set, no second runner; race test (1,000 random interleavings with threads and fake processes) → every trigger is followed by a run that started after it.
- Hook installer on fixture repos: plain, existing hook with `exit 0` at the end, python-shebang hook, husky v9, husky v4, lefthook (with and without `post-commit`), pre-commit generated hook (`.legacy` path), `core.hooksPath` custom, worktree. Assert: original content preserved byte for byte outside our block; idempotent; `uninstall` restores the original bytes.

**Equivalence (the key test)**
- Hypothesis-driven: generate a fixture repo and a random sequence of operations (edit, add, delete, rename, commit, branch + checkout, rebase, amend, stash, config change, migration add, table rename). After each op run `refresh(mode="changed")`, then assert `catalog.jsonl` and `edges.jsonl` equal a fresh `build_full` byte for byte. 300 sequences in CI, 5,000 nightly.

**`--check`**
- Default mode: stale local catalog → 1; fresh → 0.
- Committed mode: baseline at B, commit C touching only `.surf/` → 0 at C; a code change after → 1; shallow clone → 2; shallow + `--trust-cochange` → 0; baseline commit missing → 2; hand-edited catalog → 1; `--allow-lag` builds at B via worktree → 0.

**Fail-open**
- Kill the refresh at random points; the concurrent router loop never errors (05 concurrency harness).
- SessionStart with a slow build (fake 10 s extractor) returns within 3.05 s with `state="refreshing"`.

**Performance**
- Phase 5 benchmark: the typical-commit refresh on the 5k-file eval repo, p95 over 20 commits ≤ 3 s.

---

## 9. Acceptance criteria

1. **Phase 5 exit:** refresh after a typical commit in < 3 s (p95) on the larger eval repo. The SessionStart path never blocks longer than `session_start_wait_ms` + 50 ms.
2. The equivalence property test passes: incremental == full after every operation.
3. No git operation by surf ever takes `.git/index.lock` (verified by running `git commit` in a loop concurrently with refreshes: zero lock failures).
4. The hook installer passes all fixture setups; `uninstall` is byte-exact.
5. `--check` behaves per §4.7 on all listed scenarios, including exit 2 for shallow clones.
6. In default mode, `git status` is clean after any commit/merge/checkout with hooks installed.

---

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-06-1 | Commit catalog by default; hooks refresh it (§9.5, §10.1) | Default local-only; opt-in committed **baseline** changed only by `surf index --baseline`; hooks write only `cache/` | §4.1: dirty tree after every commit, self-referential `index_head`, merge conflicts, shallow CI |
| D-06-2 | Changed set from `git diff <index_head>..HEAD` ∪ working-tree changes | Content-addressed caches keyed by blob ids; diffs only for co-change ranges | Robust to rebase/reset/stash/checkout; proves incremental == full |
| D-06-3 | SessionStart: "refresh incrementally (time-boxed to 3 s), else warn" | Background refresh; the hook *waits* up to 3 s; the build is never cut short | A killed build would need partial-write handling; waiting gives the same UX |
| D-06-4 | Steps 6–7: recompute ancestors and edge-derived fields for affected nodes | Re-render all cards every refresh (≈ 200 ms) | Removes a class of roll-up bugs; cost is small |
| D-06-5 | `--check`: "fail if the committed catalog doesn't match a fresh build" | Freshness = no indexed-path change in `index_head..HEAD`, plus a rebuild at `index_head` (§4.7); exit code 2 for "cannot verify" | A catalog can't be built at the commit that contains it; shallow clones can't reproduce co-change |
| D-06-6 | "surf index … installs into husky/lefthook/pre-commit" (unspecified) | Per-manager rules in §4.8; committed manager configs are never edited except husky files (with consent) | Never replace or silently rewrite shared config |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-06-1 | **(User decision)** Default commit policy: (a) local-only, (c′) committed baseline, (b) pre-commit, (d) CI bot | (a) default, (c′) opt-in, (d) as a documented recipe, (b) rejected | User; spec open question 5 (churn on real repos) |
| Q-06-2 | Should the first route in a fresh clone wait for a small repo's cold build (e.g. ≤ 2 s estimated) instead of returning `index-missing`? | No: SessionStart already waits 3 s | Fresh-clone UX feedback |
| Q-06-3 | Should `--check` in committed mode fail on any lag (strict) or only on a lag larger than N commits? | Strict (0 indexed-path changes), with `--allow-lag` | Team feedback on PR friction |
| Q-06-4 | Live MCP listings in a committed baseline can't be verified in CI | Trusted as committed inputs | Security review (14): a committed listing is an injection vector like any committed text |
| Q-06-5 | pre-commit `.legacy` chaining and lefthook `{0}` argument passing: verify against current releases | As specified | Phase 4 integration test against pinned versions |
| Q-06-6 | Stash, `git restore`, editor saves: no hook fires; is SessionStart + the 60 s CLI check enough, or do we need a cheap per-route freshness check? | Enough in v1 (leases re-validate by card hash) | Stale-pointer rate in decision logs |
