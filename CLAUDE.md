# CLAUDE.md

Context for agents working in this repo. Read this first, then the docs it points to.

## What this project is

**Jev Surfer** (CLI `surf`) is a context router for coding agents. You point it at an existing project and it:

1. **Indexes** every surface an agent might use into short, deterministic text **cards**:
   - **content surfaces:** code files and directories, docs, database tables (parsed from migrations);
   - **capability surfaces:** MCP servers, skills, subagents, slash commands.

   It also builds a light **graph** between surfaces: containment, git co-change and schema references. **No model runs at index time.**
2. **Routes** each new task: the **Jev** judge (TypeSafe's System One model, pinned to `jev-1.13.0`) picks the smallest useful set of surfaces for the prompt.
3. **Directs** the agent with a short plain-text **note** of pointers ("likely relevant: …, use: supabase MCP, skip: figma MCP"). It never pastes file contents.
4. **Stays fresh** through git hooks and content hashing.
5. **Works with any harness:** a CLI and an MCP server, plus a Claude Code hook adapter for automatic delivery in v1.

The per-prompt pipeline:

1. Skip rules.
2. Path matching on pasted stack traces.
3. **Call 1** (continuity Choice `same`/`extends`/`new`, `needs_context`, one Noul per capability).
4. Jev **directory walk** (lenient; up to 40 candidates per request).
5. Graph expansion.
6. Strict **final pass**.
7. Selection and budget (≤ 12 pointers), then the note.

A **task lease** reuses the selection while follow-up prompts continue the same task. Every error path **fails open**: the agent runs as if `surf` weren't installed.

**Status: design phase.** There's no code yet. Jev details were verified against the TypeSafe docs on 2026-09-23 (`docs/jev-reference.md`); what remains is live-only (07 §8.4). Next steps: settle the owner decisions (`docs/open-questions.md` §1), then build-plan Phase −1 (scaffolding) and Phase 0 (evaluation foundations).

## Where things are

| Path | Role |
|---|---|
| `docs/spec/architecture-v1.md` | The v1 spec: goals, scope, decision log, targets. Source of truth for **intent**. Cite as "spec §N". Don't edit it without the owner's approval. |
| `docs/design/00-foundations.md` | Shared types (`surf/model.py`), id grammar (`code:`, `doc:`, `db:`, `mig:`, `mcp:`, `skill:`, `agent:`, `cmd:`), determinism rules, commit policy and the local overlay, fail-open model, conventions, module layout. **Every other doc and module follows it.** |
| `docs/design/01`–`16` | One design doc per component. Source of truth for **implementation**. They all use the same section headings; §5 is Configuration and §10 is Deviations and open questions. |
| `docs/design/13-config.md` | The complete `config.toml` schema. Every config key used anywhere must exist here with the same spelling. |
| `docs/build-plan.md` | Work items `P<phase>.<n>`, each mapped to a design doc and a module. Use the ids in branch names, commit subjects and issues. |
| `docs/open-questions.md` | Register of every deviation (`D-NN-k`) and open question (`Q-NN-k`). §1 lists the decisions that need the owner. |
| `docs/jev-reference.md` | Verified Jev/TypeSafe reference: endpoint, request/response shapes, limits, rate limits, latency, pricing, answer semantics, jaggedness, SDK. Read it before touching `judge/`. |
| `docs/architecture-review.md` | Architecture review against the Jev docs (2026-09-24). Proposals Q-09-13…15, Q-07-10/11, Q-12-9 are pending eval or owner decisions; read before changing the router's request shape. |
| `docs/jev-docs-checks.md` | Checklist of every Jev, TypeSafe and harness assumption, with the verification results (§3) and what is left for the Phase 0 live test. |

Component → doc → code:

| Doc | Component | Code |
|---|---|---|
| 01 | Discovery, excludes, agent-config detection | `index/discover.py` |
| 02 | Cards, budgets, hashing | `index/extract_*.py`, `index/cards.py` |
| 03 | Schema extraction (migration replay) | `index/extract_schema.py` |
| 04 | Graph edges, expansion scoring | `graph/*` |
| 05 | Catalog store (JSONL + SQLite cache) | `catalog/*` |
| 06 | Refresh, git hooks, `--check` | `index/build.py`, `adapters/git_hooks.py` |
| 07 | Judge protocol and backends | `judge/*` (wire format only in `judge/jev_wire.py`) |
| 08 | Path matching | `route/pathmatch.py`, `route/pathindex.py` |
| 09 | Router pipeline | `route/*` |
| 10 | Task lease | `lease/*` |
| 11 | Note format | `route/note.py` |
| 12 | CLI, MCP server, Claude Code adapter, init/uninstall | `cli.py`, `adapters/*` |
| 13 | Configuration | `config.py` |
| 14 | Redaction, sanitization, injection defenses | `redact.py` |
| 15 | Decision log, `--explain`, stats | `log/*` |
| 16 | Evaluation | `eval/*`, `bench/` |

## Non-negotiables

- **No model at index time.** Cards and edges come only from files, git history and tool listings. No LLM summaries.
- **Jev decides, code enforces.** Jev only answers bounded, typed questions (Noul = P(yes); Choice = one of N options). Thresholds, math, budgets, lease logic and fallbacks live in code.
- **Never send the judge the whole index.** At most 40 questions per request.
- **Fail open everywhere.** `route()` never raises; adapters never break the host harness.
- **Routing never grants permissions.** "Skip" lines are advisory.
- **Pointers, not payloads.** Notes never contain file contents.
- **Reproducible output.** Nothing written to `.surf/` depends on wall-clock time. Output is sorted, and floats are rounded to 4 decimal places. Co-change decay is measured from the indexed commit's time.
- **Measured, not claimed.** No threshold, question wording or feature changes without a dev-set evaluation run (spec §17). Never tune on the test set.
- **Privacy.** Only redacted prompts and cards (paths, names, headings, table and column names) go to the judge. Never file contents or secrets.

## Working rules

- Implement against the component's design doc. If code must diverge, update the doc **in the same change** and add a `D-NN-k` row to its §10, mirrored in `docs/open-questions.md`.
- When you resolve a question, record the answer and the evidence in `docs/open-questions.md`, then update the affected docs.
- Keep docs consistent with each other:
  - config keys match 13;
  - `.surf/` paths match 05's layout;
  - shared types match 00;
  - the router reads the catalog only through 05's store API, never the raw JSONL.
- Stack: Python 3.11+, `uv`, `ruff`, `pyright` (strict), `pytest`, pydantic v2, typer, httpx (async), sqlglot. The distribution name is proposed as `jev-surfer` (import `surf`, console scripts `surf` and `surf-hook`); this is still an open decision.
- Tests are offline and use the `fixture` judge (record/replay). Live Jev calls happen only in the Phase 0 conformance test and in live eval runs.
- Anything marked **UNVERIFIED** in the docs is an assumption. Check `docs/jev-docs-checks.md` before relying on it.
- Jev's wire format is: Choice options in `criteria`; answers `noul` (Noul) and `choice`/`probabilities`/`confidence` (Choice); `usage.input_tokens`/`output_tokens`; response `model` is the versioned id. Details and sources: `docs/jev-reference.md`.
- Doc style: concise, tables over prose, no emojis; reference the spec as "spec §N" and other docs as "07 §3.1".
