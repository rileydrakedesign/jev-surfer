# Jev Surfer documentation

| Path | What it is |
|---|---|
| [`spec/architecture-v1.md`](spec/architecture-v1.md) | The v1 architecture and development spec. The source of truth for **intent**: goals, scope, decisions and their revisit triggers. |
| [`design/`](design/) | One design doc per component. The source of truth for **implementation**: interfaces, algorithms, edge cases, tests, acceptance criteria. |
| [`build-plan.md`](build-plan.md) | Spec §23 phases broken into work items, each mapped to a design doc and module. |
| [`open-questions.md`](open-questions.md) | Every open question and proposed spec deviation raised in the design docs, with its proposed default. |
| [`jev-reference.md`](jev-reference.md) | Verified reference for the Jev API, models, limits, semantics and SDK, with a source for every fact. Cite it instead of restating Jev details. |
| [`architecture-review.md`](architecture-review.md) | 2026-09-24 review of the architecture against the verified Jev docs: what earns its keep, and proposals (flat-first routing, latency model, gate, providers) awaiting eval or owner decisions. |
| [`jev-docs-checks.md`](jev-docs-checks.md) | Checklist (and results) for verifying every Jev, TypeSafe and harness assumption against primary docs. |

## Design docs

Read `00-foundations.md` first; the others build on it.

| Doc | Component |
|---|---|
| [00 · Foundations](design/00-foundations.md) | Shared types, id grammar, determinism, fail-open model, conventions |
| [01 · Discovery](design/01-discovery.md) | File enumeration, excludes, agent-config detection |
| [02 · Cards](design/02-cards.md) | Code, doc, directory and capability cards; budgets; hashing |
| [03 · Schema extraction](design/03-schema-extraction.md) | Migration replay, table cards |
| [04 · Graph edges](design/04-graph-edges.md) | Containment, co-change, schema refs, expansion scoring |
| [05 · Catalog store](design/05-catalog-store.md) | JSONL + SQLite cache, meta, commit policy |
| [06 · Refresh](design/06-refresh.md) | Incremental rebuild, git hooks, `--check` |
| [07 · Judge](design/07-judge.md) | Judge protocol, backends, resilience |
| [08 · Path matching](design/08-path-matching.md) | Paths from stack traces and logs |
| [09 · Router](design/09-router.md) | Per-prompt pipeline: skip, call 1, walk, expansion, final pass, selection |
| [10 · Lease](design/10-lease.md) | Task lease |
| [11 · Note](design/11-note.md) | Pointer note format |
| [12 · Delivery](design/12-delivery.md) | CLI, MCP server, instruction snippet, Claude Code adapter, init/uninstall |
| [13 · Configuration](design/13-config.md) | Full `config.toml` schema |
| [14 · Security and privacy](design/14-security-privacy.md) | Redaction, sanitization, injection defenses |
| [15 · Observability](design/15-observability.md) | Decision log, `--explain`, `surf stats` |
| [16 · Evaluation](design/16-evaluation.md) | Datasets, metrics, attribution, ablations |

## How to change things

- **Changing intent** (a goal, a scope line, a decision in spec §3): edit the spec and add a row to its decision log.
- **Changing implementation**: edit the design doc. If it deviates from the spec, add a `D-NN-k` row in that doc's §10 and mirror it in `open-questions.md` until it's accepted.
- **Resolving a question**: record the answer in `open-questions.md` (status → resolved, with the date and the evidence), then update the affected docs.
