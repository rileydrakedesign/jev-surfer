# Jev Surfer (`surf`)

A harness-neutral context router for coding agents. `surf` builds a deterministic index of a project's surfaces (code, docs, database schema, MCP servers, skills, subagents, commands) and, for each new task, uses a Jev judge to pick the smallest useful set. The agent gets a short note of pointers, not file contents.

**Status:** design phase. No code yet.

- Architecture spec: [`docs/spec/architecture-v1.md`](docs/spec/architecture-v1.md)
- Component design docs: [`docs/design/`](docs/design/) (start with `00-foundations.md`)
- Build plan: [`docs/build-plan.md`](docs/build-plan.md)
- Open questions: [`docs/open-questions.md`](docs/open-questions.md)

> Jev Surfer is a community project and is not affiliated with or endorsed by TypeSafe AI. "TypeSafe", "Jev" and "System One" are names of TypeSafe AI.
