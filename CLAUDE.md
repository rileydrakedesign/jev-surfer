# CLAUDE.md

Jev Surfer (`surf`): a deterministic-index + Jev-judge context router for coding agents. The repo is in the design phase.

## Where things are

- `docs/spec/architecture-v1.md`: the v1 spec (intent, scope, decision log). Cite sections as "spec §N".
- `docs/design/NN-*.md`: per-component design docs (implementation). `00-foundations.md` defines the shared types, id grammar, determinism rules and conventions every other doc and module follows.
- `docs/build-plan.md`: work items `P<phase>.<n>` mapped to docs and modules.
- `docs/open-questions.md`: open questions and proposed spec deviations.

## Rules for working here

- Implement against the design doc for the component. If code has to diverge from it, update the doc in the same change.
- Non-negotiables from the spec: no model is used at index time; the router fails open on every error path; routing never grants permissions; nothing in `.surf/` committed output depends on wall-clock time.
- Python 3.11+, `uv`, `ruff`, `pyright` (strict), `pytest`. Tests are offline and use the fixture judge.
- No thresholds or question wordings change without an evaluation run on the dev set (spec §17).
