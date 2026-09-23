# 10 · Task lease

**Status:** draft for review
**Spec sections:** §12 (all), §10.3, §11.2 (ack skip), §11.4 (continuity decision rules), §17.1 (sequence labels)
**Depends on:** 00-foundations, 05-catalog-store (card lookup by id/path, card hash), 09-router (calls the lease), 11-note (renders the delta), 13-config (`lease.*`), 14-security-privacy (redaction of stored request text), 15-observability (lease fields in decision records)
**Code:** `surf/lease/manager.py` (I/O + locking), `surf/lease/logic.py` (pure merge/validate/continuity functions), `surf/filelock.py` (small cross-platform lock helper)

---

## 1. Purpose and scope

The lease caches the selection for the current **task** in a session so follow-up prompts reuse it (spec §12.1). This doc makes the lease safe under the conditions the spec doesn't discuss: one process per prompt (Claude Code hook), concurrent prompts in one session, index refreshes between prompts, and cleanup.

| In scope (v1) | Out of scope (v1) |
|---|---|
| Lease record format, storage, atomic writes, per-session file locks | Cross-machine or shared leases |
| Effective-continuity rules (combining Jev's Choice, confidence, staleness, path hits) | A running "task description" (spec §25 Q6; would be generated text) |
| `extends` union with ordering and eviction; delta computation | Retracting pointers the agent already saw |
| `new` replacement; `same` reuse | Leases across repositories (spec §25 Q7) |
| Staleness via card-hash validation at load time | Learning continuity from feedback |
| Expiry: idle, compaction, clear, session end, reroute; garbage collection | |

## 2. Interfaces

### 2.1 Pure logic (`surf/lease/logic.py`)

No I/O, no clock reads (times are passed in). Everything here is unit-testable with plain values.

```python
Continuity = Literal["same", "extends", "new"]

def effective_continuity(
    *, lease: Lease | None, answer: ChoiceA | None, min_conf: float,
    path_hits: Sequence[SurfaceId], catalog: CardLookup,
) -> tuple[Continuity, ContinuityReason]: ...

def covers(holder: SurfaceId, item: SurfaceId, catalog: CardLookup) -> bool: ...

def merge_extends(
    lease: Lease, routed: Selection, *, routed_hashes: Mapping[SurfaceId, str],
    now: datetime, max_pointers: int, catalog: CardLookup,
) -> tuple[Lease, LeaseDelta]: ...

def replace_new(
    prior: Lease | None, routed: Selection, *, routed_hashes: Mapping[SurfaceId, str],
    session_id: str, task_request: str, now: datetime, index_head: str | None,
) -> Lease: ...

def validate(lease: Lease, catalog: CardLookup) -> tuple[Lease, ValidationReport]: ...
```

`CardLookup` is the read-only protocol from 05-catalog-store: `get(id) -> Card | None`, `by_path(path) -> Card | None`. `ChoiceA` is the judge answer type from spec §13.1 / 07-judge.

### 2.2 Manager (`surf/lease/manager.py`)

```python
class LeaseManager:
    def __init__(self, surf_dir: Path, cfg: LeaseConfig, clock: Clock) -> None: ...

    def load(self, session_id: str | None, catalog: CardLookup) -> LeaseSnapshot | None: ...
    def peek_active(self, session_id: str | None) -> bool: ...          # cheap; used by the ack skip rule
    def commit(self, snap: LeaseSnapshot | None, outcome: LeaseOutcome) -> CommitResult: ...
    def expire(self, session_id: str, reason: ExpireReason) -> bool: ...
    def gc(self, *, force: bool = False) -> int: ...                     # returns files deleted
    def list_active(self) -> list[LeaseSummary]: ...                     # surf status
```

Callers:

| Caller | Calls |
|---|---|
| `route/pipeline.py` (09) | `load` at route start, `effective_continuity` after call 1, `merge_extends` / `replace_new` via `commit` at the end |
| `route/skip.py` (09) | `peek_active` for the ack rule (spec §11.2) |
| `adapters/claude_code.py` (12) | `expire` on SessionStart `compact` / `clear` and on SessionEnd; `gc` on SessionStart |
| `cli.py` (12) | `expire(reason="reroute")` for `surf reroute`; `list_active` for `surf status` |
| `adapters/mcp_server.py` (12) | same as pipeline; connection-scoped session ids |

`load` returns `None` when leases are disabled (`lease.enabled = false`), `session_id is None`, or no live lease exists. Every method is fail-open: I/O errors are logged and the method behaves as "no lease" (`load` → `None`, `commit` → `CommitResult(written=False, reason="io-error")`).

## 3. Data structures

### 3.1 Lease record (file format, version 1)

Stored at `.surf/cache/leases/<key>.json`. Compact JSON, UTF-8, written atomically (tmp + `os.replace`).

```python
class LeaseItemMeta(BaseModel):
    gen: int                  # generation in which the item was (last) selected
    rank: int                 # 0-based priority within that generation's selection (09 §selection order)
    hash: str | None          # card hash at selection time; None for capabilities

class Lease(BaseModel):
    v: Literal[1] = 1
    session_id: str                           # original id, unsanitized
    task_id: str                              # "t_" + ULID; new on every `new`
    task_request: str                         # redacted, ≤ 2,000 chars (head+tail kept)
    last_request: str | None                  # redacted previous prompt in this session (see §4.7)
    started_at: datetime                      # UTC, when the task (not the session) started
    last_used_at: datetime                    # any prompt that consulted the lease
    last_route_started_at: datetime           # start time of the route that last wrote the selection
    index_head: str | None                    # meta.index_head at last write (logging only; see D-10-2)
    index_fp: str | None                      # meta content_fingerprint + config_fingerprint (05) at last validation
    selection: Selection                      # content ordered newest-first (§4.3); caps sorted by id
    items: dict[SurfaceId, LeaseItemMeta]     # one entry per id in selection.content
    stale: bool = False
    stale_reason: Literal["changed", "removed", "refresh"] | None = None
    generation: int                           # 1 on `new`; +1 per `extends` that changed the selection
    rev: int                                  # +1 on every write, including touches
```

Spec record fields are all present; added fields are `v`, `task_id`, `last_request`, `last_route_started_at`, `index_fp`, `items`, `stale_reason`, `rev`. `task_request` and `last_request` are always the **redacted** text (14 D-14-7); the raw prompt is never stored.

### 3.2 Other types

```python
class LeaseSnapshot(BaseModel):          # what `load` returns
    key: str; lease: Lease; rev: int; validation: ValidationReport

class ValidationReport(BaseModel):
    dropped: list[SurfaceId]             # ids no longer in the catalog
    remapped: dict[SurfaceId, SurfaceId] # dir prefix flip (00 §2.3 F2): old id → new id, same path
    changed: list[SurfaceId]             # hash differs from lease.items[id].hash

class LeaseDelta(BaseModel):             # input to 11-note render_delta
    content_added: list[SurfaceId]       # rank order of the routed selection
    caps_use_added: list[SurfaceId]
    evicted: list[SurfaceId]             # logged only; never rendered
    subsumed: list[SurfaceId]            # leased files replaced by a newly added directory

class LeaseOutcome(BaseModel):
    continuity: Continuity
    routed: Selection | None             # None for `same` / skip (touch only)
    routed_hashes: dict[SurfaceId, str]
    request_text: str                    # redacted current prompt
    route_started_at: datetime

class CommitResult(BaseModel):
    written: bool
    reason: Literal["ok", "disabled", "no-session", "lock-timeout", "superseded", "io-error"]
    delta: LeaseDelta | None             # set for `extends`
    lease: Lease | None

ExpireReason = Literal["idle", "compaction", "clear", "session-end", "reroute", "corrupt", "version"]
ContinuityReason = Literal["no_lease", "judge", "low_conf", "stale", "path_hits_outside_lease",
                           "missing_answer"]          # same strings as 09's `override` field
```

### 3.3 File layout

```
.surf/cache/leases/
├── <key>.json        # the lease
├── <key>.lock        # lock file (empty; never deleted while a lease exists)
└── .gc               # empty marker; mtime = last gc run
```

`key` = the session id if it matches `^[A-Za-z0-9._-]{1,80}$` and is not `.`/`..`, else `h_` + first 32 hex chars of `sha256(session_id)`. Claude Code session ids (UUIDs) are used as-is, which keeps them greppable.

## 4. Behavior

### 4.1 Effective continuity

Called by the pipeline after call 1. Rules are evaluated in order; the first match wins.

| # | Condition | Result | Reason |
|---|---|---|---|
| 1 | no lease | `new` | `no_lease` |
| 2 | lease exists but `continuity` answer missing (that key failed or was dropped) | `extends` | `missing_answer` |
| 3 | choice `new` | `new` | `judge` |
| 4 | choice `extends` | `extends` | `judge` |
| 5 | choice `same` and `answer.confidence < min_conf` | `extends` | `low_conf` |
| 6 | choice `same` and `lease.stale` | `extends` | `stale` |
| 7 | choice `same` and some path hit is not covered by the lease (§4.2) | `extends` | `path_hits_outside_lease` |
| 8 | choice `same` | `same` | `judge` |

Rules 3–8 are exactly 09 §4.6 step 1; the function lives here so 09 and the eval runner (16) share one implementation. Rule 2 is an addition: 09 doesn't say what happens when the continuity key alone fails, and `extends` is the safe middle. Low confidence only downgrades `same` (spec §11.4). Whether a low-confidence `new` should also become `extends`, as the general wording of spec §12.4 suggests, is Q-10-7.

Rule 7 is the alignment with 09-router: a pasted stack trace that points outside the leased area is strong evidence of a new area, whatever Jev says about continuity. Path hits that are covered by the lease don't force anything.

### 4.2 Coverage

`covers(holder, item)` is true when:

- `holder == item`; or
- both are path-bearing, `holder` is a directory, and `path_of(item)` starts with `path_of(holder)` (compared by **path**, per 00 §2.3 F2); or
- the two ids are the `code:` / `mig:` alias pair of the same migration file (00 §2.3 F3).

Tables are covered only by themselves. `db:*` never appears in a selection. Capabilities are compared by id only.

### 4.3 Ordering

`selection.content` is always ordered **newest-first**: by `(gen desc, rank asc, id asc)`. `rank` is the item's position in the routed selection that (last) added it, which is 09's priority order (path hits → final score desc → id asc). The ordering is total and deterministic.

### 4.4 `extends`: union, eviction, delta

```python
def merge_extends(lease, routed, *, routed_hashes, now, max_pointers, catalog):
    g = lease.generation + 1
    old = lease.selection.content                        # newest-first
    added, subsumed, refreshed = [], [], []
    for rank, sid in enumerate(routed.content):
        if any(covers(h, sid, catalog) for h in old):
            refreshed.append((sid, rank))                # already covered: not "new" to the agent
            continue
        added.append((sid, rank))
        subsumed += [o for o in old if covers(sid, o, catalog)]   # new dir swallows leased files
    # Build the new list: this generation's items first (added and re-selected), then older items.
    fresh = sorted(added + [(s, r) for s, r in refreshed if s in old], key=lambda x: x[1])
    keep_old = [o for o in old if o not in {s for s, _ in fresh} and o not in subsumed]
    merged = [s for s, _ in fresh] + keep_old            # newest-first by construction
    evicted = merged[max_pointers:]
    merged = merged[:max_pointers]
    ...
```

Precisely:

1. **Generation.** `g = lease.generation + 1`.
2. **Classify** each routed content id in rank order:
   - covered by a leased item → **re-selected**. If the id itself is in the lease, its meta becomes `(gen=g, rank=r, hash=new)`, which moves it to the front. If it is covered only through a leased directory, the directory keeps its place and the file isn't added.
   - not covered → **added**, meta `(g, r, hash)`.
   - an added directory that covers leased files → those files are **subsumed** (removed from the lease; the directory stands in for them).
3. **Order** the result by §4.3.
4. **Evict** from the tail until `len ≤ max_pointers` (`router.max_pointers`, default 12). Items added in this generation are never evicted, because the routed selection is itself ≤ `max_pointers`.
5. **Capabilities.** `use' = lease.use ∪ routed.use`. `not_needed' = (lease.not_needed ∪ routed.not_needed) − use'`. A capability the lease marked `use` stays `use` even if call 1 now scores it ≤ `cap_skip` (same asymmetry as spec §11.4). Delta: `caps_use_added = routed.use − lease.use`, which includes a capability that moves from `not_needed` to `use`.
6. **Delta** = `LeaseDelta(content_added=[s for s, _ in added], caps_use_added, evicted, subsumed)`. New `not_needed` capabilities are **not** announced mid-task (see 11-note: delta notes carry no skip line).
7. If `content_added` and `caps_use_added` are both empty, the delta is **empty**: `generation` is **not** incremented and the selection and item metadata are left unchanged (only the step 8 fields are written). The note is `None` (11 renders nothing for an empty delta). The status is whatever 09 decided: usually `no-candidates`, because 09 excludes leased items from the final-pass pool on `extends` (D-09-15), so "nothing new" means nothing eligible. The decision record gets `lease.delta_empty = true`. This answers "extends that yields nothing new": inject nothing (D-10-3).
8. `stale = False`, `stale_reason = None`, `index_head = current`, `last_route_started_at = route start`, `last_used_at = now`, `last_request = request_text`, `task_request` unchanged.

When `needs_context < thresholds.needs_context` and there are no path hits, the routed content is empty by 09's rules; the merge then only processes capabilities.

### 4.5 `new`: replace

`replace_new` creates a fresh lease: new `task_id`, `task_request = request_text`, `generation = 1`, `started_at = now`, all items `gen=1` with their routed ranks, `stale=False`. `last_request` is taken over from the prior lease if any (§4.7). `rev` continues from the prior file's `rev + 1` so concurrent readers can detect replacement. A `new` whose routed selection has no content and no capability lines still writes a lease (it records `task_request` so the next prompt's continuity question has context).

### 4.6 `same` and skip: touch

`same` and the ack skip rule write only `last_used_at`, `last_request` and `rev`. Selection is unchanged. If the lock can't be taken within the timeout, the touch is skipped (not critical). Capability drift under `same` (call 1 now scores a new capability ≥ `cap_use`) is logged as `lease.caps_drift` and not acted on (Q-10-2).

### 4.7 `last_request` and the previous message

Call 1's state has `last_message` (spec §11.4). The lease provides it for all delivery paths: `last_request` is the redacted text of the previous prompt that reached surf in this session, including skipped acks. Adapters that have a better source may pass `previous_message` explicitly (the Claude Code adapter falls back to the transcript only when no lease exists; see 12-delivery §4.4). Precedence: explicit `previous_message` → `lease.last_request` → none.

### 4.8 Load and validation (staleness)

```
load(session_id, catalog):
  if not enabled or session_id is None: return None
  path = leases/<key>.json
  read (no lock); on missing → None
  on JSON/validation error → expire(reason="corrupt"); return None
  if lease.v > 1 → return None (leave file; a newer surf wrote it)          # reason "version" logged
  if now − last_used_at > idle_minutes → expire(reason="idle"); return None
  lease', report = validate(lease, catalog)
  return LeaseSnapshot(key, lease', rev=lease.rev, report)
```

`validate` runs on **every** load, not only when `index_head` changed, because working-tree refreshes don't move `index_head` (D-10-2). Short-circuit: if `lease.index_fp` equals the current `meta.content_fingerprint + config_fingerprint` (05), nothing can have changed and the lookups are skipped. Otherwise it's ≤ 12 lookups plus hash comparisons (≤ 1 ms against the SQLite snapshot), after which `index_fp` is updated in memory.

| Finding for a leased content id | Effect |
|---|---|
| id present, hash equal | nothing |
| id present, hash differs | `stale=True`, `stale_reason="changed"` |
| id missing, a card with the same path exists (dir prefix flip, F2) | remap to the new id; hash compare on the new card |
| id missing, no card with that path | drop from selection; `stale=True`, `stale_reason="removed"` |
| capability id missing | drop silently (capability sets change on config edits; not a content signal) |

The validated lease is returned in memory. It's persisted at the next commit, so a read-only path (`surf status`) never writes.

**Refresh-driven staleness (spec §10.3).** Because validation compares card hashes at load time, `surf refresh` doesn't need to find and rewrite lease files. `LeaseManager` still exposes nothing for refresh to call. This removes a cross-process write path (D-10-2). Directory cards change hash when any descendant's card changes, so a leased directory goes stale after most commits under it. The cost is one `extends` walk, which is acceptable.

Stale + `extends`: changed-but-present items **stay** in the lease. 09 excludes leased items from the final-pass pool (D-09-15), so they aren't re-judged, but they still act as expansion anchors. Removed items are already gone. The walk then finds renamed or new areas as additions. Re-judging changed items was considered and rejected: it would spend final-pass slots on items the agent already has (Q-10-6).

### 4.9 Commit and concurrency

Which route outcomes write the lease (aligned with 09 §4.1 and D-09-17):

| Route status | Lease action |
|---|---|
| `routed`, `no-context`, `no-candidates` | `commit` with the effective continuity (`new` replace / `extends` merge; empty content allowed) |
| `lease-reuse` (`same`), `skipped` by the ack rule | `commit` as a touch |
| `deadline`, `judge-unavailable`, `error`, `index-missing`, other `skipped` | none (a half route must not be reused by a later `same`) |

09 calls `commit(session_id, continuity, request=…, selection, index_head)`. That's a thin wrapper that builds the `LeaseOutcome` and returns `CommitResult` (full lease selection + delta).


Routing is not serialized: two prompts in the same session can route concurrently (MCP clients can do this; Claude Code normally can't). Only the final read-modify-write is locked.

```
commit(snap, outcome):
  if disabled / no session → CommitResult(False, "disabled"/"no-session")
  with filelock(leases/<key>.lock, timeout=lease.lock_timeout_ms):   # else "lock-timeout"
      disk = read leases/<key>.json (may be None or expired)
      base = disk if disk is live else None
      if base and snap and base.rev != snap.rev:          # someone wrote since we loaded
          if base.last_route_started_at > outcome.route_started_at and outcome.continuity != "same":
              return CommitResult(False, "superseded")    # a newer route already won
          if outcome.continuity == "extends" and base.task_id != snap.lease.task_id:
              return CommitResult(False, "superseded")    # our extends targets a replaced task
      match outcome.continuity:
          "same"    → new = touch(base or snap.lease)
          "extends" → new, delta = merge_extends(validate(base or snap.lease), outcome.routed, ...)
          "new"     → new = replace_new(base, outcome.routed, ...)
      new.rev = (base.rev if base else 0) + 1
      atomic_write(leases/<key>.json, new)
```

- The note returned to the caller was already computed from the snapshot. When the commit is `superseded`, the note is still returned (the route's content is valid for the prompt); only the lease isn't updated. Decision record: `lease.commit = "superseded"`.
- `extends` re-applies the merge onto the **on-disk** lease when it's the same task, so two concurrent extends both land.
- `filelock` uses `fcntl.flock(LOCK_EX | LOCK_NB)` with 10 ms polling on POSIX and `msvcrt.locking` on Windows. Lock files are never deleted while their lease exists (deleting a lock file races with a waiter holding a descriptor to the unlinked inode).
- Atomic write: `leases/.<key>.<pid>.tmp` → `os.replace`. No `fsync` (cache data; a lost write degrades to "no lease").

### 4.10 Expiry and cleanup

| Event | Source | Action |
|---|---|---|
| Idle > `lease.idle_minutes` | checked on `load` and `peek_active` | delete lease file |
| Compaction | Claude Code `SessionStart` with `source="compact"` | `expire(reason="compaction")` |
| `/clear` | `SessionStart` with `source="clear"` | `expire(reason="clear")`. Claude Code may assign a new session id on clear; expire both the payload id and, if the adapter can't tell, leave the old one to idle-expire |
| Session end | Claude Code `SessionEnd` | `expire(reason="session-end")` |
| `surf reroute` | CLI or in-prompt control command (12-delivery §4.3) | `expire(reason="reroute")` |
| Session resume | `SessionStart` with `source="resume"` | keep the lease if not idle |
| MCP connection closed | 12-delivery | connection-scoped leases (`mcp-…` keys) expire; explicit `session_id` leases are kept |

`expire` takes the lock, deletes `<key>.json`, and leaves `<key>.lock`. `gc()` runs when forced (SessionStart hook, `surf status`) or opportunistically after a commit if `.gc` is older than 1 hour. It deletes lease files idle beyond `idle_minutes`, then, if more than `lease.max_files` remain, the oldest by `last_used_at`; then lock files with no lease older than 1 day; then `*.tmp` files older than 1 hour.

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `lease.enabled` | bool | `true` | `false` → every prompt routes as `new`, nothing is written |
| `lease.idle_minutes` | int ≥ 1 | `45` | spec §12.4 |
| `lease.lock_timeout_ms` | int 10–2000 | `250` | commit lock wait |
| `lease.max_files` | int ≥ 10 | `500` | gc cap on lease files |
| `lease.max_request_chars` | int 200–8000 | `2000` | stored `task_request` / `last_request` length (head + tail) |
| `router.max_pointers` | int | `12` | eviction cap (owned by 09/13) |
| `router.thresholds.<profile>.continuity_min_conf` | float | `0.60` | rule 5 in §4.1 |

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| No session id (CLI without `--session`/`SURF_SESSION`) | Leases disabled for that call; continuity question not asked; route as `new` (spec §12.5) |
| Lease file corrupt / truncated JSON | Delete, route as `new`, log `lease.corrupt` |
| Lease written by a newer surf (`v > 1`) | Ignore without deleting; route as `new`; no commit (would downgrade the file) |
| `leases/` not writable | `load` works if readable; `commit` returns `io-error`; route result unaffected; one stderr warning per process |
| Clock moved backwards (`last_used_at` in the future) | Treat as fresh; clamp `last_used_at = now` on next write |
| Concurrent `new` and `new` | Later `route_started_at` wins; the other is `superseded` |
| Concurrent `extends` and `new` | If `new` committed first, the `extends` is `superseded` (different `task_id`); if `extends` first, `new` replaces it |
| Lock timeout | Skip the write; decision record `lease.commit="lock-timeout"`; the note is still delivered |
| Extends adds a directory that covers 5 leased files | Dir added (delta shows it), 5 files subsumed; lease shrinks |
| Extends routes a file inside a leased directory | Not added, not in delta |
| Extends yields nothing new | `note=None`, `delta_empty=true`, generation unchanged |
| Extends with all 12 routed items new | All 12 kept, every older item evicted |
| Path hit outside lease and Jev says `same` | Forced `extends` (rule 7) |
| Ack prompt ("ok") with an idle-expired lease | `peek_active` false → not skipped → routes normally (no lease → `new`) |
| Judge unavailable during call 1 with a lease | Status `judge-unavailable`, no note, lease untouched (09 D-09-17); it idles out normally |
| Route deadline hit before selection | No commit (a partial selection must not replace a lease) |
| Leased table dropped by a migration | Dropped on load, lease stale; next prompt is at least `extends` |
| Session id with `/`, `..`, unicode | Hashed key (`h_…`) |
| Two repos, same Claude session id | Separate `.surf/` dirs → separate leases (one repo boundary per route, spec §25 Q7) |

## 7. Performance budget

| Operation | Budget (p95, local SSD) |
|---|---|
| `peek_active` (stat + small JSON parse) | ≤ 2 ms |
| `load` incl. validation (≤ 12 SQLite lookups) | ≤ 5 ms |
| `commit` incl. lock, re-read, merge, atomic write | ≤ 8 ms uncontended |
| `gc` with 500 files | ≤ 50 ms (only on SessionStart / status / hourly) |

Imports: `lease/` depends on stdlib, `pydantic` and `surf.model` only.

## 8. Test plan

**Unit (pure logic, `tests/lease/test_logic.py`)**

- `effective_continuity`: table-driven over all 8 rules, including a low-confidence `new` staying `new`, stale + `same` → `extends`, a covered path hit keeping `same`; parity test against 09's trace `override` values.
- `covers`: file-in-dir, dir-in-dir, prefix-flipped dir (`doc:docs/` vs `code:docs/`), migration alias pair, `src/a` must not cover `src/ab/x.ts` (dirs end in `/`).
- `merge_extends`: ordering after 3 generations; eviction from the tail; re-selected item moves to front; directory subsumption; empty delta leaves `generation` unchanged; capability rules (use sticks, not_needed → use announced, new not_needed not announced).
- `validate`: unchanged, changed hash, removed, remapped dir, removed capability.
- Property test (`hypothesis`): for any sequence of routed selections, `len(content) ≤ max_pointers`, ordering invariant holds, and no item appears in both `content` and as covered by another item.

**Manager (`tests/lease/test_manager.py`, tmp dirs, fake clock)**

- Round trip; atomic write leaves no partial file under a simulated crash (monkeypatched `os.replace` raising).
- Idle expiry at exactly `idle_minutes` (boundary: `>` expires, `=` doesn't).
- Concurrency: two processes (`multiprocessing`) commit `extends` against the same snapshot → both additions present; `new` vs `extends` race → `superseded` per §4.9; lock timeout path.
- Corrupt file, `v=2` file, unwritable dir, hashed keys.
- `gc` caps and tmp cleanup.

**Integration / sequence (with 09 and the fixture judge)**

- Spec §17.1 `s004` replayed: `new` → `same` (no note) → `extends` (delta contains `code:src/notifications/email/`) → `new` (full note).
- Stale path: commit modifies a leased file between turns 1 and 2; turn 2 judged `same` → routed as `extends`; changed item stays leased and isn't re-announced; a deleted leased file is dropped.
- Claude Code adapter test (12): SessionStart `compact` deletes the lease; next prompt gets a full note.

## 9. Acceptance criteria

1. Phase 4 exit (spec §23): continuity accuracy ≥ 0.9 on dev sequences, where the measured value is the **effective** continuity from §4.1 compared to `expect_continuity`. Also report raw Jev accuracy separately so wording and rule effects can be separated.
2. `len(selection.content) ≤ router.max_pointers` after any sequence (property test).
3. No lease write ever happens from a route that hit the route deadline before selection, or from a read-only command.
4. With `leases/` read-only, deleted, or corrupt, every route still returns a valid `RouteResult` (fail-open test).
5. Budgets in §7 hold in the CI micro-benchmark on a 12-item lease.

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-10-1 | not specified | A missing continuity answer with a lease → `extends` | Safe middle; the spec only covers low confidence |
| D-10-2 | §10.3: refresh marks leases stale | Staleness is detected at **load** by comparing stored card hashes against the catalog; refresh doesn't touch leases. `index_head` in the lease is informational | Removes a cross-process write path; also catches working-tree refreshes, which don't change `index_head` |
| D-10-3 | §12.4: `extends` injects a delta note | A delta with no new content and no new `use` capability injects nothing (`note=None`, `lease.delta_empty`; status per 09, usually `no-candidates`) | An empty "Also relevant" note is noise |
| D-10-4 | §12.2 record | Adds `v`, `task_id`, `last_request`, `last_route_started_at`, `index_fp`, `items` (gen/rank/hash per id), `stale_reason`, `rev` | Needed for ordering/eviction, hash-based staleness, concurrency and the previous-message fallback |
| D-10-5 | §12.3: `path hits` not mentioned for `same` | Path hits not covered by the lease force `extends` | Pasted traces outside the leased area are strong new-area evidence (same as 09 D-09-5) |
| D-10-6 | §12.4 "newest first" | Defined as `(generation desc, rank asc, id asc)`; directories subsume covered files; tail eviction | The spec gives no ordering or tie-break |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-10-1 | Should the delta note announce leased pointers that were deleted/renamed (a `gone:` line)? | No in v1; the walk re-finds renamed files as additions | Sequence eval: count of turns where the agent opens a deleted pointer |
| Q-10-2 | Under `same`, should a newly `use`-scored capability produce a one-line capability delta? | No (log `caps_drift` only) | Capability accuracy on sequence turns; frequency of `caps_drift` in logs |
| Q-10-3 | Is 45 min idle right for agent sessions with long tool runs? | 45 | Decision logs: distribution of gaps between prompts that Jev judged `same` |
| Q-10-4 | Should the lease store redacted request text when `privacy.log_prompt_text = false`? It's needed for continuity. | Yes (cache is gitignored, redacted per 14 D-14-7, ≤ 2,000 chars); document in the privacy statement | Privacy review (14) |
| Q-10-5 | On `extends`, should the stored `task_request` be updated (e.g. appended) so continuity compares against the widened task? | No; keep the originating request (spec §11.4) plus `last_request` | Continuity accuracy on 3+ turn sequences (spec §25 Q6) |
| Q-10-7 | Spec §12.4 says "low-confidence continuity → extends" in general; §11.4 and 09 apply it only to `same`. Should a low-confidence `new` also become `extends`? | No (follow §11.4 / 09) | Sequence eval: accuracy of `new` answers by confidence bucket |
| Q-10-6 | Should changed-but-present leased items be re-judged in the final pass on a stale `extends` (at the cost of pool slots)? | No: they stay leased and act as anchors | Sequence eval with a commit between turns: pointer precision on the next turn |
