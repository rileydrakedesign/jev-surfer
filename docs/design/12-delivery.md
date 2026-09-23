# 12 · Delivery: CLI, MCP server, instruction snippet, Claude Code adapter, init/uninstall

**Status:** draft for review
**Spec sections:** §15 (all), §11.2 (control commands, disabled routing), §12.5 (session identity), §10.1 (git hooks, SessionStart trigger), §18.2 (`surf stats`), §19.1 (init privacy table)
**Depends on:** 00-foundations, 05-catalog-store, 06-refresh (what `surf refresh` does; this doc owns hook *installation*), 07-judge (breaker state for `status`/`doctor`), 09-router (`route()`), 10-lease, 11-note, 13-config, 14-security-privacy, 15-observability (`--explain`, `stats`), 16-evaluation (`surf eval`)
**Code:** `surf/cli.py`, `surf/runtime.py` (new), `surf/control.py` (new), `surf/adapters/claude_code.py`, `surf/adapters/_cc_transcript.py` (new), `surf/adapters/mcp_server.py`, `surf/adapters/instructions.py`, `surf/adapters/git_hooks.py` (installation logic owned by 06 §4.8; called from here), `surf/adapters/_jsonfile.py` (new; safe JSON settings editing), `surf/install/manifest.py` (new)

---

## 1. Purpose and scope

Everything between the engine (`route()`) and the outside world: the commands people type, the hook Claude Code runs, the MCP server other harnesses call, and the installers that wire these into a project and remove them again.

| In scope (v1) | Out of scope (v1) |
|---|---|
| Full CLI surface, flags, exit codes, `--json` schemas | API proxy delivery (spec §15.1 level 2) |
| Claude Code hooks: `UserPromptSubmit`, `SessionStart`, `SessionEnd`, optional `Stop` | Push adapters for Codex, OpenCode, Cursor (v2) |
| MCP server: `route_context`, `surface_info`, `surf_status`; stdio and local HTTP | Hard MCP tool pruning |
| Instruction snippet management | Writing user-level harness configs by default (`~/.codex`, `~/.claude`) |
| Orchestrating git hook installation (the hook block, manager handling and its manifest are 06 §4.8) | Refresh algorithm and freshness checks (06) |
| `surf init`, `surf uninstall`, `surf doctor`, `surf status/on/off/reroute` | `surf eval` internals (16), `surf stats` internals (15) |

## 2. Interfaces

### 2.1 Runtime facade (`surf/runtime.py`)

All three front doors (CLI, hook, MCP) go through one object, so the load order, fail-open wrapping and enable checks are identical everywhere. The spec layout doesn't have this module; it's added to avoid three copies of the same wiring.

```python
class Runtime:
    @classmethod
    def open(cls, start: Path, *, env: Mapping[str, str] = os.environ,
             overrides: Mapping[str, Any] | None = None, t0: float | None = None) -> "Runtime": ...
    root: Path                         # project root (dir containing .surf/)
    cfg: SurfConfig                    # lazily loaded (13)
    def route(self, req: RouteRequest, *, explain: bool = False) -> RouteResult: ...   # never raises
    def status(self, session_id: str | None) -> StatusReport: ...
    def control(self) -> ControlStore: ...
    def leases(self) -> LeaseManager: ...
```

- `t0` is the process start time (`time.monotonic()` taken first thing in the entrypoint). The route `Deadline` is created from `t0`, so interpreter and import time count against `router.route_deadline_ms` (§7).
- `route()` checks, in order: control state (§4.2) → index presence → then calls `route/pipeline.route()`. Any exception → `RouteResult(status="error", note=None)` + decision record.

### 2.2 Root discovery

`find_root(start)`: walk up from `start` to the first directory containing `.surf/config.toml`; stop at the filesystem root. No `git` subprocess on the hot path. Override: `--root PATH` or `SURF_ROOT`. If nothing is found, `index-missing` (hook/MCP: silent; CLI: exit 3 with "run `surf init`").

### 2.3 Control store (`surf/control.py`)

```python
class ControlStore:
    def effective(self, session_id: str | None) -> tuple[bool, DisabledBy | None]: ...
    def set_project(self, enabled: bool) -> None: ...
    def set_session(self, session_id: str, enabled: bool) -> None: ...

DisabledBy = Literal["env", "project", "session", "judge-null"]
```

### 2.4 Entrypoints

| Entrypoint | Console script / command | Module |
|---|---|---|
| CLI | `surf …` | `surf.cli:app` (typer) |
| Claude Code hooks | `surf-hook <event>` | `surf.adapters.claude_code:main` (no typer, lazy imports) |
| MCP server | `surf mcp [--http]` | `surf.adapters.mcp_server:serve` |
| Git hooks | shell block calling `surf refresh --changed --background --quiet` | 06 |

`surf hook claude <event>` is a hidden alias of `surf-hook <event>` for debugging.

## 3. Data structures

### 3.1 Control files (all under gitignored `.surf/cache/`)

```
.surf/cache/control/project.json            {"v":1,"enabled":false,"since":"<utc>"}      # absent = enabled
.surf/cache/control/sessions/<key>.json     {"v":1,"session_id":"…","enabled":false,"since":"<utc>"}
```

`<key>` uses the lease key rule (10 §3.3). Session control files are garbage-collected with leases (idle > `lease.idle_minutes` × 4, i.e. 3 h by default).

### 3.2 Install manifest (`<git-common-dir>/surf/install.json`)

```python
class InstalledItem(BaseModel):
    kind: Literal["claude_hooks", "mcp_config", "snippet", "git_exclude", "surf_gitignore"]   # git hooks: 06 manifest
    path: str                        # repo-relative (or "<git-dir>/hooks/post-commit")
    created: bool                    # surf created the file
    backup: str | None               # <git-common-dir>/surf/backups/<name>.<utc>.bak
    sha256_after: str                # file hash right after surf wrote it
    detail: dict[str, Any] = {}

class InstallManifest(BaseModel):
    v: Literal[1] = 1
    surf_version: str
    installed_at: datetime
    items: list[InstalledItem]
```

The manifest sits next to 06's git-hook manifest (`<git-common-dir>/surf/manifest.json`), so it survives deletion of `.surf/cache/`. In a non-git project it falls back to `.surf/cache/install.json`. It is per clone and **optional**: uninstall works without it by recognizing surf-owned entries by marker (§4.10). It adds two things: deleting files surf created, and byte-exact restore when a file hasn't changed since install.

### 3.3 `--json` output envelope

Every `--json` output is one JSON object on stdout with a `schema` field `"surf.<command>/<n>"`. JSON Schemas generated from the pydantic models are checked in at `docs/schemas/*.json`, and a test fails if they drift. Field additions keep `n`; removals or type changes bump it.

## 4. Behavior

### 4.1 Session identity (spec §12.5, made concrete)

| Front door | session_id | Namespacing |
|---|---|---|
| Claude Code hook | payload `session_id` (UUID) | as-is |
| MCP, `session_id` argument given | the argument | as-is (an agent that knows its harness session id shares that lease, which is intended) |
| MCP, no argument | `mcp-<ULID>`, one per MCP session (stdio: per process; HTTP: per `Mcp-Session-Id`) | prefix `mcp-` |
| CLI | `--session` → `SURF_SESSION` → none (leases off) | as-is |

### 4.2 Enable / disable (`surf on|off`)

Effective state, first match wins:

| # | Source | Scope | Set by |
|---|---|---|---|
| 1 | `SURF_DISABLE=1` in the environment | process | user shell / harness env |
| 2 | `.surf/cache/control/sessions/<key>.json` `enabled` | one session | `surf off --session ID`, in-prompt `surf off` |
| 3 | `.surf/cache/control/project.json` `enabled` | this clone | `surf off` / `surf on` (no `--session`) |
| 4 | `judge.backend = "null"` | config scope | config |
| — | default | | enabled |

A session-level `on` overrides a project-level `off` for that session (rule 2 before 3). This is deliberate: "off for the repo, but on in this one session" is a real debugging workflow.

The project scope is **per clone and never committed**, since `control/` lives under the gitignored cache. A team-wide kill switch is `judge.backend = "null"` in the committed config. Disabled routing returns `status="skipped"` with `trace.skip_reason = "disabled:<source>"`.

### 4.3 In-prompt control commands (spec §11.2)

The Claude Code adapter matches the **whole** prompt (trimmed, case-insensitive) against:

```
^/?surf\s+(off|on|status|reroute)(\s+--project)?\s*$       # control-only prompt
^/?surf\s+reroute[:\s]+(?P<rest>\S.*)$                     # reroute + a real prompt (DOTALL)
```

| Prompt | Action | Hook output |
|---|---|---|
| `surf off` | session off | `{"decision":"block","reason":"surf: routing off for this session (surf on to resume)"}` |
| `surf off --project` | project off | block, reason names the scope |
| `surf on [--project]` | enable | block, reason confirms |
| `surf status` | none | block, reason = one-line status (enabled, index age, lease generation/pointers) |
| `surf reroute` | expire lease | block, reason "surf: task lease cleared; your next prompt will be routed as a new task" |
| `surf reroute: <text>` | expire lease, route `<text>` as `new` | not blocked; note injected as usual |

Blocking means Claude Code doesn't send the control prompt to the model, which is what the user wants for a control command (D-12-3). Anything that doesn't match exactly is routed normally, so a prompt like "surf off the old API" is unaffected.

### 4.4 Claude Code adapter

#### 4.4.1 Events

| Hook | Matcher | Handler |
|---|---|---|
| `UserPromptSubmit` | — | control commands (§4.3), otherwise route and inject |
| `SessionStart` | none (all sources) | `clear`/`compact` → expire lease; then, for every source, `ensure_fresh` (06) + lease `gc` |
| `SessionEnd` | — | expire the session's lease; delete the session control file |
| `Stop` | — | optional (`delivery.claude_code.stop_hook`, default off): log implicit feedback (spec §15.4) |

`PreCompact` is not used: `SessionStart` with `source="compact"` fires after compaction, which is exactly when the earlier note has been summarized away. Expiring there makes the next prompt get a full note (10 §4.10).

#### 4.4.2 Payloads and outputs

Input (stdin JSON; unknown fields ignored; only `hook_event_name` is required):

| Field | Used for |
|---|---|
| `session_id` | lease / control key |
| `transcript_path` | previous-message fallback (§4.4.4) |
| `cwd` | root discovery start; `RouteRequest.cwd`; note root hint |
| `prompt` (UserPromptSubmit) | the request |
| `source` (SessionStart) | `startup` / `resume` / `clear` / `compact` |

Output for `UserPromptSubmit` with a note (stdout, exit 0):

```json
{"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "additionalContext": "[surf] Likely relevant — …"}}
```

No note → exit 0 with **empty** stdout. Every other event → exit 0, empty stdout. The adapter never exits non-zero and never writes anything except the protocol JSON to stdout. Logging goes to stderr only at `-v` / `SURF_DEBUG=1`, because Claude Code surfaces stderr in some modes.

#### 4.4.3 `UserPromptSubmit` handler

```python
def main() -> None:                              # surf-hook
    t0 = time.monotonic()                        # before any non-stdlib import
    try:
        event = sys.argv[1]; payload = json.loads(sys.stdin.read() or "{}")
        # --- fast path: stdlib only ---
        root = find_root(Path(payload.get("cwd") or os.getcwd()))
        if root is None or os.environ.get("SURF_DISABLE") == "1": return
        if event == "prompt":
            prompt = payload.get("prompt") or ""
            if (cmd := match_control(prompt)): return emit(handle_control(cmd, root, payload))
            if fast_disabled(root, payload.get("session_id")): return         # stat 2 files
            if fast_ack_skip(root, prompt, payload.get("session_id")): return  # ack list + lease peek (default ack list; §7)
            # --- slow path: imports pydantic, catalog, judge ---
            from surf.runtime import Runtime
            rt = Runtime.open(root, t0=t0)
            prev = None if rt.leases().peek_active(sid) else read_prev_user_text(payload, prompt)
            res = rt.route(RouteRequest(request=prompt, session_id=sid, previous_message=prev, cwd=payload.get("cwd")))
            if res.note: emit({"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "additionalContext": res.note}})
        elif event == "session-start": ...
    except BaseException:                         # incl. KeyboardInterrupt, SystemExit from libraries
        log_crash_best_effort()
    finally:
        sys.exit(0)
```

- The fast ack skip reads `router.skip.*` with `config.raw_peek` (stdlib `tomllib` only, 13 §4.8) and falls back to the default ack list when the key is absent or invalid. The full pydantic config load is only on the slow path.
- With a lease, the previous message comes from `lease.last_request` inside the pipeline (10 §4.7). The transcript is read only when there's no active lease, e.g. the first routed prompt after installing surf mid-session, or after a compaction expired the lease.

#### 4.4.4 Transcript reader (`_cc_transcript.py`)

The transcript format is Claude Code-internal and can change. The reader is isolated, tolerant, time-boxed and fail-open.

```python
def read_prev_user_text(payload: Mapping, current_prompt: str, *,
                        max_bytes: int = 1 << 20, budget_ms: int = 30) -> str | None: ...
```

1. `transcript_path` must be an absolute path to a regular file; otherwise `None`.
2. Read backwards in 64 KiB blocks up to `max_bytes` from the end, splitting on `\n`; stop at the budget.
3. For each line from the end: `json.loads`; skip on error. Accept only entries where:
   - `type == "user"` and `message.role == "user"`;
   - not `isMeta`, not `isSidechain` (subagent turns);
   - `message.content` is a string, **or** a list that has ≥ 1 `{"type":"text"}` block and no `tool_result` block (tool results are also `type:"user"` entries).
4. Text = the string, or text blocks joined with `\n`. Skip if it starts with `<command-name>`, `<command-message>`, `<local-command-stdout>` or `Caveat:` (slash-command and local-command artifacts), or if it matches a surf control command.
5. The **first** accepted text equal to `current_prompt` (whitespace-normalized) is skipped once, because Claude Code may or may not have written the current prompt before the hook runs (Q-12-2).
6. Return the next accepted text, truncated to `lease.max_request_chars` (head + tail). It's redacted later by the pipeline like any request text.
7. Any exception → `None`.

#### 4.4.5 `SessionStart`, `SessionEnd`, `Stop`

- **SessionStart**: if `source in {"clear","compact"}`: `leases.expire(sid, reason)` first (cheap, so it can't be lost to a timeout). Then `refresh.ensure_fresh(wait_ms=refresh.session_start_wait_ms)` (06 §4.5): when fresh (the common case) this returns in ≤ 150 ms. When stale it triggers a background refresh and **waits** up to 3 s, and the refresh keeps running after the wait (D-06-3). Then `leases.gc()`. No stdout. Routes that arrive during a refresh read the previous immutable SQLite snapshot (05), and lease validation catches changed pointers (10 §4.8).
- **SessionEnd**: expire the lease and the session control file.
- **Stop** (off by default): read the transcript tail for `Read`/`Grep` tool uses since the last routed prompt and append `{"kind":"feedback","route_id","opened":[ids…]}` to the decision log (15). v1 only records; nothing consumes it.

#### 4.4.6 Installer (`claude_code.install(root, *, shared: bool, stop_hook: bool)`)

Target: `.claude/settings.local.json` (default) or `.claude/settings.json` (`--shared`).

Hook entries written:

```json
{
  "hooks": {
    "UserPromptSubmit": [{ "hooks": [{ "type": "command", "command": "<CMD> prompt", "timeout": 10 }] }],
    "SessionStart":     [{ "hooks": [{ "type": "command", "command": "<CMD> session-start", "timeout": 5 }] }],
    "SessionEnd":       [{ "hooks": [{ "type": "command", "command": "<CMD> session-end", "timeout": 5 }] }]
  }
}
```

| Mode | `<CMD>` | Why |
|---|---|---|
| local | absolute path of the installed `surf-hook` (from `shutil.which`, resolved), shell-quoted | Hook environments often have a different `PATH` than the user's shell (GUI launches, uv tool shims) |
| shared | `command -v surf-hook >/dev/null 2>&1 && surf-hook` … `|| true` wrapper | Committed file: no machine paths; teammates without surf get a no-op instead of a hook error |

Algorithm (via `_jsonfile.py`):

1. Read the file. Missing → start from `{}`. Invalid JSON (comments, trailing commas) → **abort this step** with an actionable error; never rewrite a file surf can't parse.
2. Back up the original bytes to `.surf/cache/backups/<basename>.<utc>.bak` (first install only; re-installs don't create new backups).
3. For each event, remove existing **surf-owned** hook commands, identified by a command whose first word's basename is `surf-hook`, or which contains `surf-hook ` inside the shared wrapper, or which starts with `surf hook claude`. Drop matcher groups left empty by that removal. Don't touch other groups or keys.
4. Append one surf group per event (after existing groups, so user hooks run first).
5. Serialize preserving key order, the detected indent (2/4 spaces or tab) and trailing newline; write atomically.
6. Local mode only: if `git check-ignore -q .claude/settings.local.json` fails, append the path to `.git/info/exclude` (local, never committed) and record it in the manifest.
7. Record `InstalledItem(kind="claude_hooks", created, backup, sha256_after)`.

Idempotency: running install twice yields identical bytes (tested). Switching local ↔ shared removes surf entries from the other file.

### 4.5 MCP server (`surf mcp`)

Built on the official MCP Python SDK (`FastMCP`). Imported only by `surf mcp`.

#### 4.5.1 Tools

**`route_context`**

Description (what agents see; it decides whether they call the tool):

> Find the files, docs, database tables and tools in this repository that are relevant to a task. Call this once at the start of each new task, and again when the task moves to a different area, **before** searching the codebase. Pass the user's request verbatim. Returns a short list of pointers (paths, table names, which MCP servers/skills to use or skip). Nothing is preloaded: open the pointed files yourself, and keep using your normal search if the pointers don't cover the task. Don't call it for follow-ups on the same task, or if a `[surf]` note is already present for this task.

Input schema:

| Field | Type | Required | Notes |
|---|---|---|---|
| `request` | string, 1–20,000 chars | yes | the user's request text |
| `session_id` | string ≤ 200 | no | a stable id for the conversation; defaults to the MCP session |
| `previous_message` | string ≤ 20,000 | no | the previous user message, if the harness has it |

Output: text content = the note, or `"surf: no pointers for this request (<status>). Proceed with your normal search."`. Structured content = `RouteContextOutput` (same fields as `surf route --json`, §4.12.3).

**`surface_info`**

> Explain one item from a surf note: its index card and its strongest relationships (files that usually change together, tables it references, the migration that defines a table). Use it when you want to know why something was pointed to, or what is related to a file you're editing. Read-only.

| Field | Type | Notes |
|---|---|---|
| `id` | string | a surface id (`code:src/a.ts`, `db:orders`, `mcp:supabase`) |
| `path` | string | alternative to `id`; a repo-relative path |
| `max_edges` | int 1–30, default 10 | |

Exactly one of `id` / `path`. Output: `{schema:"surf.surface_info/1", id, type, card, edges:[{kind, other, weight, direction:"out"|"in"}]}`, with edges sorted by weight desc, then id; `alias` edges always included; unknown id → tool error `not_found` with up to 3 path suggestions (same basename).

**`surf_status`**

> Report whether surf is enabled for this project, how fresh its index is, and which judge backend it uses. Use only for troubleshooting.

No input. Output = `StatusReport` (§4.12.4), with the lease section for the calling MCP session.

#### 4.5.2 Server behavior

- **Transport:** stdio by default. `--http` starts streamable HTTP on `127.0.0.1:<delivery.mcp.port>` (default 8765). Non-loopback binds need `--allow-remote` **and** a `SURF_MCP_TOKEN` (Bearer check). DNS-rebinding protection: `Origin`/`Host` must be loopback unless `--allow-remote`.
- **stdout discipline (stdio):** stdout belongs to JSON-RPC. At startup `sys.stdout` is replaced by a guard that raises on any stray write outside the SDK's writer; logging goes to stderr.
- **Hot reload:** before each tool call, `stat` `meta.json` and `config.toml` (mtime_ns + size). If changed, reopen the catalog / reload config between calls. A failed reload keeps the previous state and logs once.
- **Concurrency:** `route_context` runs `Runtime.route` in a worker thread (`anyio.to_thread`) so concurrent calls don't block the event loop; the judge concurrency cap (07) is shared across calls. Lease commits are safe under concurrency (10 §4.9).
- **Freshness:** `ensure_fresh(wait_ms=0)` (06) at server start and whenever a new session id appears; it never blocks a tool call.
- **Never** executes project code, calls other MCP servers, or runs live MCP listing; it only reads `.surf/` and calls the judge.
- **Session end:** when a connection-scoped session closes, `expire(reason="session-end")` its `mcp-…` lease.

#### 4.5.3 Registration

| Harness | File | Written by `surf init --harness mcp` | Entry |
|---|---|---|---|
| Claude Code (project) | `.mcp.json` | yes | `{"mcpServers":{"surf":{"command":"surf","args":["mcp"]}}}` |
| Cursor | `.cursor/mcp.json` | if `.cursor/` exists | same shape |
| VS Code / Copilot | `.vscode/mcp.json` | if `.vscode/` exists | `{"servers":{"surf":{"type":"stdio","command":"surf","args":["mcp"]}}}` |
| Codex CLI, others | user-level files | no; `init` prints the snippet (written only with `--user-level`) | |

Merging uses `_jsonfile.py` (same rules as §4.4.6). An existing `surf` entry whose command isn't `surf` is **not** overwritten (conflict reported). **The index must ignore surf's own server**: capability extraction (02) must skip MCP entries whose command is `surf`/`surf-hook` or whose name is `surf`. Otherwise surf routes to itself.

When Claude Code hooks are installed, `.mcp.json` is still written (for `surface_info`), and the `route_context` description tells the agent not to call it when a `[surf]` note is present. The CLAUDE.md snippet is skipped (§4.6).

### 4.6 Instruction snippet (`adapters/instructions.py`)

Block (v1):

```markdown
<!-- surf:begin v1 -->
## Project navigation (surf)
At the start of each new task, call the `route_context` tool with the user's request
before searching the repository. Treat its pointers as a starting point, not a limit.
<!-- surf:end -->
```

Target files, relative to root:

| File | Condition |
|---|---|
| `AGENTS.md` | if it exists; **created** if none of the files below exist |
| `CLAUDE.md` | if it exists **and** Claude Code hooks are not being installed (else double routing) |
| `GEMINI.md` | if it exists |
| `.github/copilot-instructions.md` | if it exists |
| `.cursorrules`, `.windsurfrules` | if they exist (legacy single-file rules) |
| `.cursor/rules/surf.mdc` | if `.cursor/rules/` exists; surf **owns** this file (frontmatter `alwaysApply: true` + block body) |

Update algorithm (per file):

1. Read bytes; detect and keep BOM and line ending (`\r\n` if that's the majority).
2. Find markers with `^<!-- surf:begin( v\d+)? -->$` … `^<!-- surf:end -->$` (multiline).
3. Exactly one well-formed pair → replace the inner text if it differs (upgrades v1 → v2 wording in place).
4. No markers → append: ensure the file ends with a newline, add one blank line, then the block.
5. Malformed (a begin without an end, nested, or more than one pair) → leave the file untouched; report `snippet.malformed` (shown by `doctor`).
6. Write atomically; record the item in the manifest.

Removal: delete the block and the single blank line that precedes it (only if that line is blank). If the file is now empty or whitespace-only **and** the manifest says surf created it, delete it. An owned `.mdc` file is deleted.

### 4.7 Git hooks (installation owned by 06 §4.8)

06 owns the hook block (`# >>> surf >>>` … `# <<< surf <<<`), the per-manager handling (native hooks, `core.hooksPath`, husky, lefthook via `lefthook-local.yml`, pre-commit via `<hook>.legacy`) and its manifest. This doc only orchestrates it:

| Flow | Call into `adapters/git_hooks.py` (06) |
|---|---|
| `surf init` plan (§4.8 step 3) | `detect_hook_setup(root)` → `plan_install(setup)`; each `HookAction` is listed in the plan, and committed files (husky) are called out |
| `surf init` apply (step 7) | `apply(actions)` unless `--no-git-hooks` or `refresh.git_hooks = false` |
| `surf uninstall` | `uninstall(root)` |
| `surf doctor` (`git.hooks`) | `verify(root)` → one check result per `HookProblem` |

### 4.8 `surf init`

```
surf init [--harness claude,mcp,cli] [--shared] [--live-mcp NAME…] [--no-snippet] [--no-git-hooks]
          [--no-smoke] [--yes] [--dry-run] [--user-level] [--json]
```

1. **Root.** `git rev-parse --show-toplevel`; non-git → cwd with a warning ("no co-change edges"). Refuse `$HOME` or `/` unless `--force`.
2. **Detect** (no writes): languages, migration formats, agent configs, harnesses (`claude`: `.claude/` or `CLAUDE.md` or `claude` on PATH; `cursor`: `.cursor/`; `vscode`: `.vscode/`), hook managers, an existing `.surf/` (re-init = upgrade, idempotent), commit count and shallowness.
3. **Plan.** Print what will be indexed (counts per type, excludes), each file that will be created or modified, and the privacy table (spec §19.1, from 14). `--dry-run` prints the plan (`--json`: `surf.init_plan/1`) and exits 0. Otherwise confirm (`[y/N]`; `--yes` skips). Declining → exit 7.
4. **Write `.surf/`**: `config.toml` only if absent (detected values + commented defaults, 13 §4.6); `.surf/.gitignore` with `cache/`, `logs/`, `config.local.toml`. With the default `index.commit_catalog = false` (05/06, Q-06-1), the catalog lives in `.surf/cache/`, and `config.toml` plus `.surf/.gitignore` are the only files surf adds to the tree. A committed baseline is an explicit, separate step (`surf index --baseline`), which init mentions but never runs.
5. **Build** the full index into `.surf/cache/` (progress on stderr). Failure → exit 1; integrations are not installed. The derived project descriptor (02 §4.10, stored in `meta.json`) is printed so the user can override it with `project.descriptor`.
6. **Live MCP listing and purposes.** Live listing is **off by default** (D-14-5): only servers named with `--live-mcp NAME` (or already in `capabilities.live_mcp`) are spawned, each after a confirmation that shows the exact command, and never under `--yes` unless named explicitly. Then, for each static MCP server with no listing and no `capabilities.describe` entry: prompt for one line (Enter skips). Write them into `config.toml` by targeted text insertion under `[capabilities.describe]`, preserving comments (13 §4.7). Rebuild capability cards only.
7. **Integrations**, each independent (a failure is reported and the others continue): git hooks → Claude Code hooks (if `claude` harness) → MCP registration (if `mcp`) → instruction snippets (unless `--no-snippet`). Write the manifest.
8. **Doctor** (§4.11). Errors are shown; they don't undo the install.
9. **Smoke route** (unless `--no-smoke`, or no judge key): three prompts generated deterministically from the catalog: (a) `"where is <title of the highest-churn doc> described?"`, (b) `"how does <highest-churn code dir name> work?"`, (c) a two-frame synthetic stack trace using the top-churn code file's path (exercises path matching). Routed without a session; notes and latencies printed.
10. **Next steps**: how to label eval queries (`surf eval --help`, 16).

Exit: 0 success (warnings allowed), 1 build failed, 2 usage, 5 config invalid, 7 aborted.

### 4.9 `surf uninstall`

```
surf uninstall [--yes] [--purge] [--keep KIND…] [--json]
```

Order: Claude Code hooks (both settings files) → MCP entries (only `surf` entries whose command is `surf`) → instruction blocks → git hooks (06 `uninstall`) → `.git/info/exclude` line → `.surf/cache/` and `.surf/logs/`. The manifests in `<git-common-dir>/surf/` are deleted last. With `--purge` (asks for confirmation unless `--yes`), delete `.surf/` entirely, including the committed catalog and config.

Per file:

| File state | Action |
|---|---|
| unchanged since install (`sha256 == sha256_after`) and a backup exists | restore the backup byte-exact |
| unchanged, surf created it | delete |
| changed since install, or no manifest | remove surf-owned entries only (markers / command match); keep everything else; if now empty and created by surf → delete |

This refines the spec's "restores it": a blind restore would destroy edits made after install (D-12-1). Backups in `<git-common-dir>/surf/backups/` are kept (not deleted by uninstall), and `uninstall` prints their paths.

### 4.10 Ownership markers (for manifest-less uninstall and `doctor`)

| Integration | Marker |
|---|---|
| Claude hooks | command basename `surf-hook`, shared wrapper containing `surf-hook `, or `surf hook claude` |
| MCP config | key `surf` with command `surf` and first arg `mcp` |
| Snippet | `<!-- surf:begin… -->` … `<!-- surf:end -->` |
| `.cursor/rules/surf.mdc` | file name + block markers inside |
| Git hooks / husky / lefthook | owned by 06 §4.8 (`# >>> surf >>>` … `# <<< surf <<<`; `surf-refresh` in `lefthook-local.yml`) |

### 4.11 `surf doctor`

```
surf doctor [--live] [--strict] [--json]
```

| Id | Check | Severity when failing |
|---|---|---|
| `python` | Python ≥ 3.11 | error |
| `config.parse` | config files parse and validate (13) | error |
| `config.unknown_keys` | unknown keys (typos) | warn |
| `index.present` | `catalog.jsonl`, `edges.jsonl`, `meta.json` exist; `schema_version` supported | error |
| `index.fresh` | 06 `check_freshness(deep=True)` | warn |
| `index.cache` | SQLite cache present and consistent with JSONL digest (else rebuilt) | info |
| `index.size` | catalog > 20 MB → suggest `index.commit_catalog = false` (spec §9.5) | warn |
| `git.history` | < 200 commits or shallow clone → suggest `git fetch --unshallow` (spec §8.2.6) | warn |
| `git.hooks` | 06 `verify(root)`: blocks present and executable, or manager entries present; manual steps pending | warn |
| `claude.hooks` | settings entries present; `<CMD>` resolves and runs `surf-hook --version` in < 1 s | warn (error if harness selected at init) |
| `claude.local_ignored` | `settings.local.json` is ignored by git | warn |
| `mcp.config` | `surf` entry present in registered files; `surf mcp --check` starts and lists 3 tools | warn |
| `mcp.self_indexed` | no `mcp:surf` card in the catalog | error |
| `snippet` | blocks present and well-formed in target files | warn |
| `judge.key` | API key env var for the configured provider is set | error (unless backend `null`/`fixture`) |
| `judge.ping` (`--live`) | one minimal judge request (1 Noul, no repo data) succeeds within `judge.timeout_ms` | error |
| `judge.breaker` | breaker state (07) closed | warn |
| `judge.thresholds` | a threshold set exists for the active backend (13) | warn |
| `rg` | ripgrep found (else Python fallback) | info |
| `dirs.writable` | `.surf/cache/`, `.surf/logs/` writable | error |
| `hook.import_time` | `surf-hook` fast path under budget (§7) on this machine | warn |
| `control` | project or session disabled | info |

Exit: 0 when there are no errors (warnings allowed); 4 when any error; with `--strict`, warnings also give 4. JSON: `surf.doctor/1` = `{schema, ok, checks:[{id, severity, ok, message, fix}]}`.

### 4.12 CLI reference

#### 4.12.1 Global options

`--root PATH` (or `SURF_ROOT`), `--config PATH` (or `SURF_CONFIG`; extra config layer, 13 §4.2), `-v/--verbose` (repeatable), `-q/--quiet`, `--no-color`, `--version`. Human output goes to stdout; progress and logs go to stderr. `--json` writes exactly one JSON object to stdout.

#### 4.12.2 Commands

| Command | Purpose | Key flags | Exit codes |
|---|---|---|---|
| `surf init` | index + install (§4.8) | see §4.8 | 0, 1, 2, 5, 7 |
| `surf index` | full build / verify (06) | `--full`, `--check [--trust-cochange] [--allow-lag]`, `--baseline` (write the committed baseline), `--json` | 0; 3 not initialized; 4 `--check` mismatch or stale; 9 cannot verify (e.g. shallow clone); 5; 1 |
| `surf refresh` | incremental (06) | `--changed` \| `--full`, `--background`, `--reason R`, `--quiet`, `--json`; hook args pass through | 0 (also when another refresh holds the lock: `"skipped":"busy"`); 3; 5; 1 |
| `surf route "<prompt>"` | route one prompt | `--session ID`, `--previous TEXT`, `--stdin` (prompt from stdin; also when the prompt arg is `-`), `--json`, `--explain`, `--judge NAME`, `--no-lease`, `--strict` | **0 for every `RouteStatus`** (fail-open); 2 usage; with `--strict`: 3 for `index-missing`, 8 for `judge-unavailable`, 1 for `error` |
| `surf mcp` | MCP server (§4.5) | `--http`, `--host`, `--port`, `--allow-remote`, `--check` (start, self-list tools, exit) | 0; 1 on bind/startup failure; 3 |
| `surf eval` | evaluation (16) | `--set dev|test`, `--judge`, `--ablate`, `--json`, `--record` | 0; 4 regression gate failed (spec §17.7); 8 judge unavailable |
| `surf stats` | decision-log aggregates (15) | `--since`, `--json` | 0; 3 |
| `surf doctor` | health checks (§4.11) | `--live`, `--strict`, `--json` | 0, 4 |
| `surf status` | runtime state | `--session ID`, `--json` | 0; 3 |
| `surf on` / `surf off` | enable / disable | `--session ID` (else project scope) | 0; 3 |
| `surf reroute` | expire the session's lease | `--session ID` (or `SURF_SESSION`), `--all` | 0; 2 when no session and no `--all` |
| `surf uninstall` | remove integrations (§4.9) | `--purge`, `--keep KIND`, `--yes`, `--json` | 0; 7 aborted; 1 partial failure (the report lists what remains) |
| `surf config` | show / validate config (13) | `show [--origin] [--defaults] [--toml] [--json]`, `validate [--strict]`, `path` | 0; 5 invalid (`--strict`: unknown keys too) |
| `surf hook claude <event>` | hidden alias of `surf-hook` | | always 0 |

Exit code table:

| Code | Meaning |
|---|---|
| 0 | success (includes fail-open route statuses and busy refreshes) |
| 1 | unexpected error |
| 2 | usage error (typer default) |
| 3 | not initialized / index missing |
| 4 | a check failed (`index --check`, `doctor`, eval gate) |
| 5 | configuration invalid |
| 7 | aborted by the user |
| 8 | judge unavailable where it's required (`route --strict`, `eval`; `doctor --live` reports via 4) |
| 9 | cannot verify (`index --check` without enough git history; 06 §4.7) |

#### 4.12.3 `surf route --json` (`surf.route/1`), also the MCP structured output

```python
class RouteOutput(BaseModel):
    schema_: Literal["surf.route/1"] = Field("surf.route/1", alias="schema")
    route_id: str
    status: RouteStatus
    note: str | None
    note_kind: Literal["full", "delta", "caps_only"] | None
    selection: Selection | None          # content in note order; capabilities_use / _not_needed
    continuity: Literal["same", "extends", "new"] | None
    continuity_reason: str | None        # 10 §4.1 ContinuityReason
    low_confidence: bool
    session_id: str | None
    lease: LeaseBrief | None             # {task_id, generation, pointers, stale}
    latency_ms: dict[str, int]           # {"total": …, "call1": …, "walk": …, "final": …}
    trace: dict | None                   # only with --explain; format owned by 15
```

`surf route` without `--json` prints the note to stdout (nothing if none) and a one-line status to stderr (`surf: routed · new · 1240 ms`). `--explain` prints the trace tree (15) after the note.

#### 4.12.4 `surf status --json` (`surf.status/1`)

```json
{
  "schema": "surf.status/1",
  "root": "/abs/path",
  "enabled": {"effective": true, "disabled_by": null, "project": true, "session": null},
  "index": {"present": true, "schema_version": 1, "index_head": "a1b2c3d", "head": "a1b2c3d",
            "stale": false, "built_at": "…", "content_cards": 2189, "walk_mode": "walk",
            "refresh_running": false},
  "judge": {"backend": "jev", "provider": "typesafe", "model": "jev-1.13.0", "key_present": true,
            "breaker": {"state": "closed", "open_until": null}},
  "lease": {"session_id": "…", "task_id": "t_…", "generation": 2, "pointers": 5, "stale": false,
            "idle_s": 312},
  "integrations": {"claude_hooks": "local", "mcp": [".mcp.json"], "snippets": ["AGENTS.md"],
                   "git_hooks": ["post-commit", "post-merge", "post-checkout", "post-rewrite"]}
}
```

`lease` is `null` without a session. Request text is never shown (privacy); the human form shows `generation`, pointer count and age.

#### 4.12.5 Other JSON outputs

| Command | Schema | Fields |
|---|---|---|
| `index` | `surf.index/1` | `counts`, `content_cards`, `walk_mode`, `duration_ms`, `index_head`, `check: {ok, added[], removed[], changed[]} \| null` (lists capped at 50) |
| `refresh` | `surf.refresh/1` | `skipped: null\|"busy"\|"fresh"`, `changed_files`, `cards_updated`, `edges_updated`, `duration_ms` |
| `doctor` | `surf.doctor/1` | §4.11 |
| `init` | `surf.init/1` | `plan`, `installed: [InstalledItem]`, `warnings`, `smoke: [{prompt, status, latency_ms}]` |
| `uninstall` | `surf.uninstall/1` | `removed: [{kind, path, action: "restored"\|"edited"\|"deleted"}]`, `remaining` |
| `config show` | `surf.config/1` | 13 §2 |

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `delivery.claude_code.prompt_timeout_s` | int 3–60 | `10` | written as the hook `timeout` |
| `delivery.claude_code.session_timeout_s` | int 1–60 | `5` | SessionStart / SessionEnd `timeout` |
| `delivery.claude_code.stop_hook` | bool | `false` | spec §15.4 |
| `delivery.claude_code.transcript_max_bytes` | int | `1048576` | §4.4.4 |
| `delivery.mcp.host` | str | `"127.0.0.1"` | |
| `delivery.mcp.port` | int | `8765` | |
| `delivery.instructions.files` | list[str] \| null | `null` (auto, §4.6) | explicit target list |
| `delivery.control_commands` | bool | `true` | in-prompt `surf off` etc. |
| `refresh.session_start_wait_ms` | int | `3000` | SessionStart wait bound (06) |
| `refresh.check_interval_s` | int | `60` | CLI cheap freshness-check throttle (06) |
| `router.skip.ack_words` | list[str] | spec list | used by the hook fast path |
| `router.route_deadline_ms` | int | `3000` | counted from process start (§2.1) |
| `lease.*` | | | 10 §5 |

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| Hook stdin empty / not JSON | exit 0, no output |
| Hook payload without `session_id` | route without lease |
| `cwd` outside any surf project | exit 0, no output (fast path, no heavy import) |
| Index missing / corrupt | `index-missing`, no note; SessionStart does **not** trigger a full build |
| Judge key missing | `judge-unavailable`, no note; `doctor` reports it |
| Hook exceeds Claude Code's `timeout` | Claude Code kills it; the route deadline (3 s from process start) makes this unlikely. The decision log may miss the record (acceptable) |
| Transcript unreadable / format changed | `previous_message=None`; routing continues |
| Two Claude Code windows on one repo | distinct session ids → distinct leases; shared index and breaker state |
| `settings.local.json` has comments | install step aborted with a message; other steps continue |
| User already has a `surf` MCP entry pointing elsewhere | not overwritten; reported as a conflict |
| Existing hook file is a symlink (e.g. to a shared hooks repo) | not edited; warning with manual instructions |
| `AGENTS.md` is a symlink to `CLAUDE.md` | resolve realpath; edit each real file once |
| `surf off` in a session, then SessionStart `clear` | session control file keyed by old id; new id is enabled (documented: `/clear` resets `surf off`) |
| MCP HTTP port in use | exit 1 with message; stdio unaffected |
| MCP client sends a 1 MB request | rejected by input schema (20,000 chars) with a tool error |
| `surf route` in a non-initialized dir | status `index-missing`, exit 0 (3 with `--strict`) |
| Concurrent `surf init` runs | init takes `.surf/cache/init.lock`; the second exits 1 "init already running" |
| Uninstall with no manifest | marker-based removal (§4.10); created files are not deleted (unknowable) |
| Windows | hooks: local mode uses the absolute `surf-hook.exe` path; shared wrapper requires a POSIX shell (Git Bash); flagged Q-12-5 |

## 7. Performance budget

| Path | Budget (p50 / p95, warm disk) |
|---|---|
| `surf-hook prompt` fast exits (no project, disabled, control command, ack skip) | ≤ 60 ms / 100 ms wall, stdlib + `tomllib` only |
| `surf-hook prompt` import overhead before routing (pydantic, httpx, sqlite3, surf core; no typer/rich/mcp) | ≤ 200 ms / 300 ms |
| `surf-hook session-start` | ≤ 150 ms when fresh; ≤ `session_start_wait_ms` + 50 ms when stale (06) |
| `surf route` CLI overhead over engine (typer import) | ≤ 250 ms |
| MCP tool-call overhead over engine | ≤ 10 ms |
| Transcript read | ≤ 30 ms (hard budget) |
| Installer steps | ≤ 1 s total, excluding the index build |

The spec's latency targets (§1.2, §11.9) are **engine** targets. Hook wall time adds process start and imports, and it's reported separately in the decision record (`latency_ms.process`). The deadline counts from process start, so the 3 s hard cap holds for wall time. Enforcement: a CI test runs `python -X importtime -c "import surf.adapters.claude_code"` and fails if the fast path imports any non-stdlib module; a benchmark asserts the fast-exit wall time on the CI runner (with a 2× tolerance).

## 8. Test plan

**Claude Code adapter**
- Payload fixtures for all four events and all `SessionStart` sources; malformed stdin; missing fields.
- Fail-open matrix: judge raising, timing out and returning garbage; index missing; lease dir unwritable. Always exit 0, no stdout (or valid JSON), and a decision record with the right status.
- Transcript fixtures (`tests/adapters/transcripts/*.jsonl`): string content, block content, tool_result entries, meta/sidechain entries, slash-command artifacts, current prompt present/absent, truncated last line, 5 MB file (budget), non-UTF-8 bytes.
- Control commands: each row of §4.3, plus near-misses ("surf off the old API" → routed).
- Import-time guard (§7).

**Installers** (golden before/after files in `tests/adapters/install/`)
- Claude settings: missing file, `{}`, existing unrelated hooks, existing surf hooks from an older version, invalid JSON, 4-space and tab indent. Install twice → identical bytes; local ↔ shared switch; uninstall restore when untouched; uninstall after the user edits another key (only surf removed).
- MCP config: create, merge, conflicting `surf` entry.
- Snippets: each target file, CRLF, BOM, malformed markers, v1 → v2 upgrade, removal, created-file deletion, symlinked AGENTS.md.
- Git hooks: tested in 06; here only the init/uninstall/doctor orchestration against a fake `git_hooks` module.

**MCP server** — SDK in-memory client: tool list and schemas match the checked-in snapshot; `route_context` with and without `session_id` (lease continuity across two calls in one session); `surface_info` by id/path/not-found; hot reload after `meta.json` changes; a stray `print` in stdio mode is caught by the guard test.

**CLI** — exit code table (one test per row); JSON outputs validate against `docs/schemas/*.json`; `route` fail-open statuses exit 0; `--strict` codes.

**End to end (manual checklist, Phase 4 exit)** — Claude Code: install, new-task note appears, follow-up gets no note, `/compact` → next prompt gets a full note, `surf off` blocks and disables, uninstall leaves settings as before. One other MCP harness (Cursor or Codex CLI) via the pull path with the snippet.

## 9. Acceptance criteria

1. Phase 4 exit: end-to-end use in Claude Code (push) and one other MCP-capable harness (pull), per the §8 checklist.
2. Every adapter entrypoint passes the fail-open matrix; the hook never exits non-zero in any test.
3. Install → uninstall on an untouched repo leaves `git status` clean (except the ignored `.surf/cache` if `--purge` isn't given).
4. Install is idempotent (byte-identical on rerun) for every integration.
5. The §7 budgets hold on the CI runner.
6. `mcp.self_indexed` is false after `surf init` on a repo with `.mcp.json`.

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-12-1 | §15.4: `surf uninstall` restores the backed-up settings file | Byte-exact restore only when the file is unchanged since install; otherwise surgical removal of surf-owned entries | A blind restore would destroy user edits made after install |
| D-12-2 | §15.7 lists only `surf` | Separate `surf-hook` console script with a stdlib-only fast path | Typer/rich/pydantic imports would add 150–300 ms to every prompt |
| D-12-3 | §11.2: control commands are "handled directly" | In Claude Code, a control-only prompt is **blocked** (`decision: block`) with a status reason; `surf reroute: <text>` routes `<text>` as `new` | Sending "surf off" to the model wastes a turn |
| D-12-4 | §15.3: snippet goes into whichever instruction files exist | CLAUDE.md is skipped when Claude Code hooks are installed; AGENTS.md is created if no instruction file exists | Avoids double routing; AGENTS.md is the cross-harness default |
| D-12-5 | §15.4 hook table has UserPromptSubmit, SessionStart, Stop | Adds `SessionEnd` (expire lease) | Lease cleanup without waiting for idle expiry |
| D-12-6 | §15.7 command list | Adds `surf stats` (spec §18.2), `surf config`, hidden `surf hook`; adds `--strict`, `--stdin`, `--previous`, `--no-lease`, `--dry-run`, `--purge` | Needed for scripting, debugging and safe uninstall |
| D-12-7 | §15.2: MCP `route_context` output `{status, note, selection, continuity, route_id}` | Same fields plus `note_kind`, `continuity_reason`, `low_confidence`, `session_id`, `lease`, `latency_ms`, and a text content block with the note | The agent reads text; scripts read structure |
| D-12-8 | not specified | `surf off` scopes: project (per clone, gitignored) and session; team-wide off is `judge.backend = "null"` | Nothing in `cache/` is committed |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-12-1 | Does Claude Code keep the same `session_id` after `/clear`? | Expire the payload's id on `source="clear"`; stale ids idle out | Verify against current Claude Code; test fixture |
| Q-12-2 | Is the current prompt already in the transcript when `UserPromptSubmit` runs? | Handle both (skip the first exact match once) | Verify; fixture for each |
| Q-12-3 | Should `surf init` register the MCP server for Claude Code when hooks are installed? | Yes (for `surface_info`); the tool description discourages duplicate `route_context` calls | Count duplicate routes in decision logs (same session, < 5 s apart) |
| Q-12-4 | Hook fast path: accept a slightly stale ack list (default) instead of full config validation? | Yes, unless `ack_words` appears in the raw TOML | Import-time measurements |
| Q-12-5 | Shared-mode hook command on Windows without a POSIX shell | POSIX wrapper; Windows users use local mode | Windows user reports |
| Q-12-6 | Should the snippet also tell agents to call `route_context` again on task change? | Yes, via the tool description only; the snippet stays spec wording | Pull-path sequence eval in a non-hook harness |
| Q-12-7 | `surf-hook` as a second console script name: acceptable, or `surf _hook`? | `surf-hook` | Packaging review (Q-F1) |
