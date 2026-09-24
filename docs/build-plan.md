# Build plan: phases → work items

**Spec section:** §23 (phases and exit criteria). This file breaks each phase into work items that map to a design doc and a module, so work can be picked up and tracked one item at a time. Estimates and exit criteria are the spec's. The phase exits matter more than the dates.

Item ids (`P<phase>.<n>`) are stable and meant to be used in branch names, commit subjects and issues.

---

## Phase −1: Repository scaffolding (½ day)

Not in the spec. It has to exist before Phase 0 can start.

| Id | Item | Design doc | Output |
|---|---|---|---|
| P-1.1 | `pyproject.toml` (dist `jev-surfer`, script `surf`), `uv.lock`, src layout, `ruff`, `pyright`, `pytest` config | 00 §6 | installable empty package |
| P-1.2 | CI workflow: lint, typecheck, unit tests | 00 §6 | green on empty package |
| P-1.3 | `surf/model.py`, `surf/ids.py`, `surf/deadline.py`, `surf/proc.py` with tests | 00 §2–5 | shared types |
| P-1.4 | Synthetic fixture-repo builder: YAML (files + commit history) → temp git repo | 00 §6.1 | `tests/fixtures/repos/{feature,layered,monorepo}.yaml` |

---

## Phase 0: Evaluation foundations (3–4 days)

Evaluation comes first. Phase 0 must **not** depend on the Phase 1 indexer; it uses a minimal bootstrap card builder (see `16-evaluation.md`).

| Id | Item | Design doc | Module |
|---|---|---|---|
| P0.1 | Choose the 2 target repos (one feature-organized, one layered) and pin them by commit SHA | 16 | `bench/manifest.yaml` |
| P0.2 | Label 60–100 queries per repo following the protocol; 60/40 dev/test split; second labeler on 20 % | 16 | `.surf/eval/*.yaml` in each target |
| P0.3 | Dataset models and loader with validation | 16 | `eval/dataset.py` |
| P0.4 | Judge protocol + `jev` backend + `fixture` (record/replay) + `null` | 07 | `judge/base.py`, `jev.py`, `fixture.py`, `null.py` |
| P0.5 | Implement the documented wire format (verified 2026-09-23) in one adapter; live conformance test with the latency curves (1–300 questions, HTTP/1.1 vs HTTP/2), noise and 429 measurements (07 §8.4) | 07, `jev-reference.md` | `judge/jev_wire.py`, `judge/jev.py` |
| P0.6 | Bootstrap file-card builder (path + lang + line bucket only) | 16 | `eval/bootstrap_cards.py` |
| P0.7 | A0 flat brute-force router | 16, 09 | `eval/baselines.py` |
| P0.8 | Metrics incl. bootstrap and paired-bootstrap CIs; report skeleton (md + json) | 16 | `eval/metrics.py`, `report.py` |
| P0.9 | `surf eval` CLI entrypoint (dev set only) | 12, 16 | `cli.py` |

**Exit:** a baseline recall/precision/latency report for A0 on both repos' dev sets.

---

## Phase 1: Indexer (4–5 days)

| Id | Item | Design doc | Module |
|---|---|---|---|
| P1.1 | Discovery: enumeration, default and user excludes, binary/secret detection | 01 | `index/discover.py` |
| P1.2 | Agent-config detection (MCP, skills, subagents, commands; per harness) | 01 | `index/discover.py` |
| P1.3 | Code and doc extractors | 02 | `index/extract_code.py`, `extract_docs.py` |
| P1.4 | Card renderer: budgets, deterministic truncation, sanitization, hashing | 02, 14 | `index/cards.py`, `redact.py` |
| P1.5 | Directory cards (bottom-up roll-up) | 02 | `index/cards.py` |
| P1.6 | Schema extraction: SQL/Supabase via sqlglot, migration replay | 03 | `index/extract_schema.py` |
| P1.7 | Schema extraction: Prisma, then Rails / Alembic as needed | 03 | `index/extract_schema.py` |
| P1.8 | Capability extraction: MCP static, skills, subagents, commands | 02 | `index/extract_caps.py` |
| P1.9 | MCP live listing (opt-in, never calls tools) | 02, 14 | `index/extract_caps.py` |
| P1.10 | Catalog store: JSONL writer/reader, SQLite cache, meta | 05 | `catalog/store.py`, `meta.py` |
| P1.11 | Config models (subset needed so far) | 13 | `config.py` |
| P1.12 | `surf index` (full) and flat-mode `surf route --explain` | 12 | `cli.py`, `index/build.py` |

**Exit:** complete catalogs for both repos; card budgets respected; full index in < 60 s for 5k files.

---

## Phase 2: Routing core (4–5 days)

| Id | Item | Design doc | Module |
|---|---|---|---|
| P2.1 | Pipeline skeleton: `Deadline`, fail-open guard, `RouteTrace` | 09 | `route/pipeline.py` |
| P2.2 | Skip rules | 09 | `route/skip.py` |
| P2.3 | Path matching + suffix index + table-driven corpus | 08 | `route/pathmatch.py` |
| P2.4 | Mode choice and call 1 (needs_context, capabilities; continuity stubbed until Phase 4) | 09 | `route/call1.py` |
| P2.4a | Flat pass: flat set, token-balanced requests, speculative with call 1 (spec D14) | 09 §4.8 | `route/flat.py` |
| P2.5 | Directory walk (above the flat budget): chunking, flattening, beam, dead-end guard, walk budget, speculative level 1 | 09 | `route/walk.py` |
| P2.6 | Final pass | 09 | `route/final.py` |
| P2.7 | Selection and budget (collapse, type diversity) | 09 | `route/select.py` |
| P2.8 | Note builder (full note) | 11 | `route/note.py` |
| P2.9 | Wordings file + first sweeps: A0 vs A4w by index size (sets `flat_max_tokens`), A1, partial A8 | 16 | `eval/` |

**Exit:** A0 vs A4w sets `router.flat_max_tokens` (spec D14), and A1 beats A0 on precision at comparable recall on repos above it, **or** there's a documented reason to change approach.

---

## Phase 3: Graph (3–4 days)

| Id | Item | Design doc | Module |
|---|---|---|---|
| P3.1 | Containment edges | 04 | `graph/containment.py` |
| P3.2 | Co-change: git log parsing, filters, rename following, recency-weighted coupling, pruning | 04 | `graph/cochange.py` |
| P3.3 | Directory coupling → `dir_coupling` edges → `coupled_dirs` on dir cards | 04, 02 | `graph/cochange.py`, `index/cards.py` |
| P3.4 | Schema refs: variants, matcher (rg / Aho-Corasick parity), specificity weighting, guards | 04 | `graph/schema_refs.py` |
| P3.5 | Expansion scoring wired into the pipeline | 04, 09 | `graph/expand.py` |
| P3.6 | Failure attribution report; ablations A2–A5 | 16 | `eval/attribution.py` |

**Exit:** a measured contribution for each edge type, with attribution showing fewer "walk" losses on cross_layer queries.

---

## Phase 4: Lease and delivery (3–4 days)

| Id | Item | Design doc | Module |
|---|---|---|---|
| P4.1 | Lease manager: store, locking, transitions, idle expiry, staleness | 10 | `lease/manager.py` |
| P4.2 | Continuity Choice in call 1; decision rules; delta notes | 09, 10, 11 | `route/call1.py`, `route/note.py` |
| P4.3 | CLI polish: full command set, `--json` schemas, exit codes | 12 | `cli.py` |
| P4.4 | MCP server (`route_context`, `surface_info`, `surf_status`) | 12 | `adapters/mcp_server.py` |
| P4.5 | Instruction snippet management | 12 | `adapters/instructions.py` |
| P4.6 | Claude Code adapter: hooks, transcript reader, installer / uninstaller | 12 | `adapters/claude_code.py` |
| P4.7 | Git hook install (incl. hook managers) | 06, 12 | `adapters/git_hooks.py` |
| P4.8 | `surf init` / `surf uninstall` | 12 | `cli.py` |
| P4.9 | Sequence evaluation (continuity accuracy) | 16 | `eval/runner.py` |

**Exit:** end-to-end use in Claude Code and one other MCP-capable harness via the pull path; continuity ≥ 0.9 on dev sequences.

---

## Phase 5: Refresh and hardening (3 days)

| Id | Item | Design doc | Module |
|---|---|---|---|
| P5.1 | Incremental refresh (proven equal to a full build) | 06 | `index/build.py` |
| P5.2 | SessionStart freshness check (time-boxed) | 06, 12 | `adapters/claude_code.py` |
| P5.3 | `surf index --check` for CI | 06 | `index/build.py` |
| P5.4 | Redaction, adversarial fixtures, privacy traffic verification | 14 | `redact.py`, `tests/` |
| P5.5 | Judge circuit breaker (persisted across hook processes), deadlines audit | 07 | `judge/` |
| P5.6 | `surf doctor`, decisions log, `surf stats`, rotation | 12, 15 | `log/decisions.py`, `cli.py` |
| P5.7 | Ablations A6 (LLM judge) and A7 (local backend) | 07, 16 | `judge/llm.py`, `systemone_local.py` |

**Exit:** refresh after a typical commit in < 3 s; all fail-open paths tested; privacy table verified against actual traffic.

---

## Phase 6: Tuning and release (2–3 days)

| Id | Item | Design doc |
|---|---|---|
| P6.1 | Final wording and threshold selection on dev | 16 |
| P6.2 | Single test-set run; report with CIs; write the release baseline | 16 |
| P6.3 | README, privacy statement, non-affiliation note, user evaluation guide | 14, 16 |
| P6.4 | Trademark / naming check before public release (spec §0.1) | — |

**Exit:** the v1 targets in spec §1.2 are met on test, or the gaps are documented with a v1.1 plan.
