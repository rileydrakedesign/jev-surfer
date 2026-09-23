# Jev docs checks

**Purpose:** a working checklist for verifying every assumption the design docs make about Jev, TypeSafe's API, the providers, and the agent harnesses. The docs were written in a sandbox that couldn't reach `docs.typesafe.ai`, so these items are marked **UNVERIFIED** or were written from memory.

**Who uses it:** the agent (or person) doing the cross-referencing pass. It is a starting point, not a complete list. Add a row for anything else you find.

---

## How to work through this

1. **Read first:** `CLAUDE.md`, `docs/design/00-foundations.md`, then `docs/design/07-judge.md` in full. 07 owns every Jev detail. 09, 10, 14 and 16 build on it.
2. **Handle each check the same way:**
   - Find the answer in a primary source (list below). Record the URL and the date you checked, and quote the relevant line.
   - Set the status: `confirmed`, `corrected` or `unanswerable`. `Unanswerable` means the docs don't say; it becomes a Phase 0 live test item.
   - If it's corrected, edit **only** the places listed in "Where it lives", and keep the change minimal:
     - wire and provider details go in 07 §3.1–3.2 (the future `judge/jev_wire.py`);
     - semantics go in the doc that relies on them.
   - Update `docs/open-questions.md`: mark the related `Q-` row resolved with the evidence, or add a `D-` row if the answer changes a design decision.
3. **Don't change thresholds, wordings or the pipeline** because of a docs finding alone. Record it; evaluation decides (spec §17). The exception is a hard API limit, which is a constraint, not a tuning choice.
4. **Keep the docs consistent:** a changed config key or default must also change in `13-config.md`.
5. **Fill in the results table** (§3) as you go. Close by updating the "Not verified" notes in 07 §3.1 and §3.2.

### Primary sources

| Source | Use for |
|---|---|
| https://docs.typesafe.ai/introduction and https://docs.typesafe.ai/primitives | Request/response shapes, primitives (Noul, Choice, Score), limits |
| https://docs.typesafe.ai/model-jaggedness/jev-1.13 | Known weaknesses (spec §20 table) |
| https://typesafe.ai/blog/introducing-system-one-models-and-jev | Model positioning, pricing, availability |
| TypeSafe API reference and pricing pages (find from the docs nav) | Endpoints, auth, rate limits, prices, model ids, version pinning |
| `typesafe-sdk-python` source (PyPI / GitHub) | Real field names, async support, retries, import cost |
| OpenRouter, Vercel AI Gateway and Cloudflare AI Gateway docs | Whether and how they proxy Jev; model id strings; envelope |
| https://www.langchain.com/blog/building-a-harness-with-jev | Harness patterns, batching practice |
| Prior-art repos listed in spec §4 / §27 | How others call Jev (ideas only; check each license before copying anything) |
| Claude Code docs (hooks, settings, plugins, MCP) | Part C checks |

---

## 1. Jev and TypeSafe checks

### A. Wire format (owner: 07 §3.1, `judge/jev_wire.py`)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-A1 | Endpoint `POST https://api.typesafe.ai/v1/systemone` | 07 §3.2 provider table | Fix the table; `judge.base_url`/`judge.path` defaults in 13 |
| J-A2 | Auth `Authorization: Bearer $TYPESAFE_API_KEY` | 07 §3.2; 13 env-var list; 14 (secrets handling) | Fix header/env var names in 07, 13, 14 |
| J-A3 | Request body `{"model", "state": {…}, "questions": {key: {"type": "noul"\|"choice", "instructions", "options"?}}}` (spec §11.4 illustration) | 07 §3.1 | Rewrite `encode` description; golden bodies in 07 §8.2 |
| J-A4 | `state` is a flat object of string fields (`request`, `project`, `location`, `previous_task`, `last_message`); any key names allowed | 07 §3.1; 09 §3.2 (state keys) | If state must be a string or has a fixed schema, update 09 §3.2 and 07 `encode` |
| J-A5 | Question keys are free-form strings. The design sends opaque `q000`…`q039` anyway. | 07 §3.1 "Key mapping" | Probably no change; confirm key grammar and max length |
| J-A6 | Response `{"answers": {key: {"p": float}}}` for Noul and `{"choice", "probs", "confidence"}` for Choice; `decode` tolerates aliases (`probability`/`yes`, `answer`, `probabilities`/`distribution`) | 07 §3.1, §4.3 | Fix field names; drop unneeded aliases |
| J-A7 | Response carries usage (input tokens) | 07 §3.3 ledger, §4.8 | If absent, mark usage as estimated everywhere (07 §4.8, 15 decision record `judge.input_tokens`) |
| J-A8 | Errors are HTTP status codes (4xx/5xx, 429 with `Retry-After`); a malformed body is detectable | 07 §4.6 retry table, §6 | Update the error-class mapping |
| J-A9 | Model is pinned by the string `jev-1.13.0` and pinning is honored (no silent upgrades) | spec header; 07 §5 `judge.model`; 07 D-07-6 (model-qualified threshold profiles) | Fix the model id format; if pinning isn't possible, flag it: evaluation reproducibility depends on it (16) |

### B. Primitive semantics (owner: 07 §2.2; used by 09, 10, 16)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-B1 | **Noul** returns a calibrated-ish probability that the statement is true, in [0, 1] | 07 §2.2, §4.3; all thresholds in 13 `router.thresholds.*` | If it's a score or logit, define the mapping in `jev_wire.decode` |
| J-B2 | **Choice** returns the chosen key, a probability per option, and a `confidence`; if `confidence` is missing we use `max(probs)` | 07 §2.2, §4.3; 09 §4.6 (continuity decision); 10 | Fix the fallback; check what "confidence" means relative to `probs` |
| J-B3 | Choice option descriptions are passed as `options: {key: description}` | 07 §2.2 `ChoiceQ`; 09 §3.3 continuity wording | Adjust `ChoiceQ` and the continuity question |
| J-B4 | Noul and Choice outputs aren't comparable, so there are separate thresholds per question type (spec §20) | 07 §4.9; 13 thresholds | Confirm; cite the source |
| J-B5 | **Score** primitive exists but v1 doesn't use it | spec glossary; 07 §1 | Confirm; note anything that makes Score better for walk ranking as a v2 idea (don't change v1) |
| J-B6 | All questions in one request are judged **independently** over the same state (co-batched questions don't affect each other) | 07 Q-07-2; 07 §3.5 per-question fixture fallback; 16 §4.10 | If there are cross-question effects, disable the question-level fixture fallback (07 §3.5) and note it in 16 |
| J-B7 | The `instructions` field is the statement being judged; there's no separate system prompt | 07 §2.2; 09 §3.3; 16 `wordings.yaml` | If the API has a separate instructions/statement split, restructure the wording templates in 09 §3.3 and 16 |
| J-B8 | State is treated as data, but Jev doesn't treat it as hostile by default (spec §19.4) | 14 §4.9 threat model, §4.4 injection hardening | Update the threat model with anything TypeSafe documents (mitigations, delimiters) |

### C. Limits and batching (owner: 07 §4.2)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-C1 | There's no hard API limit below **40 questions per request** (40 is our own cap, spec principle 5) | 07 §4.2; 13 `judge.max_questions_per_request`; 09 `chunk_size` | If the API cap is lower, lower the default and note the latency impact in 09 §4.14 |
| J-C2 | Choice supports up to **255 options** (spec §20) | 07 §2.2 | Fix the number (v1 uses 3) |
| J-C3 | Request size: our estimate caps requests at 8,000 tokens (`judge.max_request_tokens`) and the prompt is truncated to ~1,500 tokens head+tail | 07 §4.2; 09 (request truncation); 14 §4.2 redaction pipeline | Set the cap from the documented context limit |
| J-C4 | Token estimate is `ceil(utf8_bytes / 4)` | 02 D-02-1 / Q-02-1; 07 §4.8 | If TypeSafe documents a tokenizer or counting endpoint, note it in Q-02-1 (the divisor changes only with a format-version bump) |
| J-C5 | Instruction and option-description length limits | 07 §4.3; 02 card budgets | Add validation if limits exist |

### D. Latency, rate limits and cost (owner: 07 §4.4–4.8; targets spec §1.2, §11.9)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-D1 | A round trip is fast enough for the budgets: per-request timeout 1,200 ms; 2–4 sequential round trips reach p50 ≤ 1.5 s on a new task | 07 §4.5, §7; 09 §4.2, §4.14 | Record the documented or published latency; flag it in `open-questions.md` §1 row 6 if the targets look unreachable |
| J-D2 | Latency grows little with questions per request, so the final pass isn't split by count | 09 Q-09-3 | If latency scales with count, the proposal is to split the final pass at about 20 questions (09 Q-09-3) |
| J-D3 | Rate limits allow 16 concurrent requests per process | 07 §4.4; 13 `judge.max_concurrency`; Q-07-7 (no cross-process cap) | Lower the default; revisit Q-07-7 |
| J-D4 | 429 responses carry `Retry-After` | 07 §4.6 | Adjust the retry rule |
| J-D5 | Price is **$0.042 per million input tokens; output is free** | spec §11.9; 07 §4.8; 13 `judge.price_per_mtok_input` / `_output` | Fix the defaults; recheck the "well under $0.01 per route" goal (spec §1.2) |
| J-D6 | HTTP/2 support (optional multiplexing) | 07 Q-07-3 | Record the answer |
| J-D7 | Early-access status, quotas and SLA | spec §20 row "Hosted, early access"; 07 §4.7 breaker | Record them; they affect the breaker defaults only if quotas are tight |

### E. Providers and gateways (owner: 07 §3.2)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-E1 | OpenRouter serves Jev, model id is a guess (`typesafe/jev-1.13.0`), native envelope or chat envelope unknown | 07 §3.2; Q-07-6 | Fix id and envelope. A chat envelope is added only inside `jev_wire.py`. |
| J-E2 | Vercel AI Gateway serves Jev (same unknowns), key `AI_GATEWAY_API_KEY` | 07 §3.2; 13 | Same |
| J-E3 | Cloudflare AI Gateway proxies TypeSafe at `/v1/{account_id}/{gateway_id}/typesafe/...`, with optional `cf-aig-authorization` | 07 §3.2; 13 `judge.cloudflare.*` | Fix the path; if Cloudflare doesn't support TypeSafe as a provider, drop it from v1 and add a `D-07-k` row |
| J-E4 | Gateways return usage and probabilities unchanged | 07 §3.1, §4.8 | Note any per-provider decode differences |
| J-E5 | Pinning a version works through gateways | 07 §3.2 | If not, recommend `typesafe` direct for eval runs (16) |

### F. SDK (owner: 07 D-07-2, Q-07-1)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-F1 | `typesafe-sdk-python` exists under that name | spec §22.1; 07 D-07-2 | Fix the name everywhere |
| J-F2 | The design uses httpx at runtime and the SDK only as a dev dependency to cross-check encoding. Deciding facts: does the SDK support async, what's its import time, does it support gateways? | 07 D-07-2, Q-07-1, §8.4 | If the SDK is async, light and gateway-aware, reopen Q-07-1 with the evidence (owner decision, `open-questions.md` §1 row 3) |

### G. Model behavior and jaggedness (owner: spec §20; 07; 14)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-G1 | The spec §20 table matches TypeSafe's jev-1.13 jaggedness notes: accuracy drops with large, irrelevant state; instructions are read literally; weak at numbers and dates; susceptible to adversarial state | spec §20; 09 (≤ 40 candidates, bounded state); 14 | Add any missing limitation to `open-questions.md` with a proposed design response. Don't edit the spec. |
| J-G2 | Behavior with many near-duplicate candidates (40 file cards from one directory) | 09 §4.8 walk chunking; 02 cards | Note it; it informs the chunk ordering and flatten rules (eval decides) |
| J-G3 | Recommended question phrasing (positive statements, "likely" vs "is") | 09 §3.3 wordings; 16 `wordings.yaml`; spec §25 Q1 | Add documented guidance as candidate wordings in 16, not as shipped wordings |
| J-G4 | Language support (non-English prompts, identifiers) | 14 redaction; 09 | Record it |

### H. Local System One backend (owner: 07 §4.10 `systemone-local`)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| J-H1 | Community reproductions expose a `/v1/systemone`-compatible endpoint with the same wire format | 07 §3.2 `local` row; 13 `judge.local.base_url` | Find at least one and record its differences |
| J-H2 | Open weights or reproductions are usable offline and license-compatible | spec §13.2; 14 D-14-3 | Record the licenses; A7 depends on this (16) |

### I. Prior-art projects (ideas only, spec §4)

| Id | Check | Where it lives |
|---|---|---|
| J-I1 | Re-check the licenses: jev-router (no LICENSE seen during design), jev-code-context-router and JevRouter (MIT per spec), blink, jev-codex-router, jev-knowledge-base, langchain-skill-router, jev-skillful | spec §4; nothing is copied yet |
| J-I2 | Look at how they call Jev: field names, batching, error handling. This is independent evidence for Parts A–D. | 07 |

---

## 2. Harness checks (not Jev, but also written without docs access)

### Claude Code (owner: 12 §4.4; 01 §4.8)

| Id | What the docs assume | Where it lives | If wrong |
|---|---|---|---|
| H-C1 | `UserPromptSubmit` stdin carries `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `prompt` | 12 §4.4 (input table) | Fix the reader |
| H-C2 | Output `{"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "additionalContext": "…"}}` injects context | 12 §4.4 | Fix the output shape |
| H-C3 | `SessionStart` has `source` ∈ `startup`/`resume`/`clear`/`compact`, and fires after compaction | 12 §4.4.5 | Fix the lease-expiry triggers (10) |
| H-C4 | A `SessionEnd` hook exists | 12 D-12-5 | Drop it; rely on idle expiry |
| H-C5 | `/clear` keeps or changes `session_id` | 12 Q-12-1 | Resolve Q-12-1 |
| H-C6 | Whether the current prompt is already in the transcript when the hook runs | 12 Q-12-2 | Resolve Q-12-2 |
| H-C7 | Transcript is JSONL and user turns are identifiable | 12 §4.4.4 | Fix the parser description |
| H-C8 | Settings hook format (`hooks.<Event>[].hooks[].{type, command, timeout}`), `settings.local.json` vs `settings.json`, timeout units in seconds | 12 §4.4.6 | Fix the installer |
| H-C9 | MCP servers in `.mcp.json` and `.claude/settings*.json`; skills at `.claude/skills/*/SKILL.md` with `description` frontmatter; agents at `.claude/agents/**/*.md`; commands at `.claude/commands/**/*.md` | 01 §4.8 | Fix the discovery table |
| H-C10 | Plugin layout: `~/.claude/plugins/installed_plugins.json` plus `enabledPlugins` in settings | 01 Q-01-2 | Resolve Q-01-2 |

### Other harnesses (owner: 01 §4.8)

| Id | What the docs assume | Where it lives |
|---|---|---|
| H-O1 | Codex: `~/.codex/config.toml` (+ `$CODEX_HOME`), project `.codex/config.toml`, `~/.codex/prompts/` | 01 D-01-6 |
| H-O2 | Cursor: `.cursor/mcp.json` | 01 §4.8 |
| H-O3 | OpenCode config and `.opencode/agent(s)/`, `.opencode/command/` paths | 01 §4.8 |

### MCP protocol (owner: 02 live listing; 12 MCP server)

| Id | What the docs assume | Where it lives |
|---|---|---|
| H-M1 | Live listing needs only `initialize` + `notifications/initialized` + `tools/list`, and the server's `instructions` come back in the `initialize` result | 02 (live MCP listing); 14 §4.5 |
| H-M2 | The official MCP Python SDK supports a stdio server with the three tools as specified | 12 §4.5 |

---

## 3. Results

Fill in one row per check. Keep "Evidence" to a URL plus a short quote.

| Id | Status (confirmed / corrected / unanswerable) | Evidence (URL, date, quote) | Docs changed |
|---|---|---|---|
| J-A1 | confirmed | https://docs.typesafe.ai/api, 2026-09-23: "POST https://api.typesafe.ai/v1/systemone" | 07 §3.1 (no longer UNVERIFIED) |
| J-A2 | confirmed | https://docs.typesafe.ai/api: "Authorization: Bearer <API_KEY>"; https://docs.typesafe.ai/sdk/python/api/constants: `API_KEY_ENV = 'TYPESAFE_API_KEY'` | none |
| J-A3 | corrected | https://docs.typesafe.ai/api: Choice takes "`criteria` … A map of option to rubric description"; Noul takes optional `criteria` {`true`,`false`}; `instructions` "string \| object \| array" | 07 §2.2, §3.1 (encode `options` → `criteria`); jev-reference §3 |
| J-A4 | confirmed (wider) | https://docs.typesafe.ai/api: `state` "string \| object \| array"; https://docs.typesafe.ai/concepts/state: "Use an object for most requests so each part of the state has a descriptive name" | none (surf sends an object of strings); backticked state paths noted as a wording candidate (16 Q-16-9) |
| J-A5 | confirmed | https://docs.typesafe.ai/api: "You choose each key … The key is not sent to the underlying model and is not used in inference." Grammar/length undocumented | 07 §3.1 (keep opaque keys; option keys *are* seen by the model) |
| J-A6 | corrected | https://docs.typesafe.ai/api: Noul answer `{"type":"noul","noul":0.95}`; Choice `{"type":"choice","choice",…,"probabilities",…,"confidence"}`; top level also has `model` | 07 §2.2, §3.1, §4.3 (aliases dropped; `type` checked; `model` echo checked) |
| J-A7 | confirmed | https://docs.typesafe.ai/api: `usage` (required) with `input_tokens`, `output_tokens` | 07 §4.8; 02 Q-02-1 |
| J-A8 | corrected | https://docs.typesafe.ai/api errors table: 401, 422 ("failed validation … the body details the offending field"), 429, **529 Overloaded**; https://docs.typesafe.ai/sdk/python/api/retries: retries `{408, 429, *range(500, 600)}`, honors `Retry-After` and `retry-after-ms` | 07 §4.6 (408, 529, `retry-after-ms`), §6; `request_id` from `x-typesafe-request-id` (07 §3.3) |
| J-A9 | confirmed | https://docs.typesafe.ai/models: "`jev-1.13.0`"; aliases `jev-latest`, `jev-preview`; "The response's `model` field reports the versioned ID that answered … pin that version's ID instead of the alias" | 07 §4.3 (model echo), 13 (alias warning covers `*preview*`) |
| J-B1 | confirmed | https://docs.typesafe.ai/primitives/noul: "the probability that the answer is yes where 0 means no and 1 means yes"; https://docs.typesafe.ai/introduction/machine-learning-primer: calibrated (RLCD) | none |
| J-B2 | corrected | https://docs.typesafe.ai/api: `confidence` required, "derived from probabilities"; https://docs.typesafe.ai/primitives/choice examples: probabilities 0.61/0.35/0.04 → confidence 0.42, so not `max(probs)` | 07 §2.2, §4.3 (missing `confidence` from `jev` → key invalid; `max(probs)` fallback kept only for `llm`/`systemone-local`) |
| J-B3 | corrected | https://docs.typesafe.ai/api: options go in `criteria`; values may be string, object, array or `null`; https://docs.typesafe.ai/primitives/choice: "The option names and their descriptions are both sent to the model" | 07 §3.1 (wire mapping only; internal `ChoiceQ.options` unchanged) |
| J-B4 | confirmed | https://docs.typesafe.ai/model-jaggedness/jev-1.13: "Don't carry a threshold tuned on a Noul over to a Choice" | none |
| J-B5 | confirmed | https://docs.typesafe.ai/primitives/score: Score returns `score`, `legend`, `probabilities`, `confidence`; 2–10 levels (https://docs.typesafe.ai/api). v2 idea: Choice-based ranking, not Score (16 Q-16-10) | 16 Q-16-10 |
| J-B6 | confirmed | https://docs.typesafe.ai/primitives: "Every answer is independent. One question's answer is not hidden context for another. You can add or remove questions without changing the others' results." | 07 Q-07-2 narrowed to a sanity check |
| J-B7 | confirmed (plus optional criteria) | https://docs.typesafe.ai/primitives: `instructions` is "the question you are asking … or … a statement for the model to judge"; no system prompt field. Noul `criteria` and structured instructions are optional extras | 07 Q-07-8; 16 Q-16-9 |
| J-B8 | confirmed | https://docs.typesafe.ai/model-jaggedness/jev-1.13: "State is data, and `jev-1.13` does not treat it as hostile by default … can move the answer." No server-side mitigation documented | 14 §4.9 T4 |
| J-C1 | confirmed | No question cap in https://docs.typesafe.ai/api or https://docs.typesafe.ai/primitives; limits are context-based (https://docs.typesafe.ai/models). Cookbooks send 182–218 options in one request | 07 §4.2 wording |
| J-C2 | confirmed | https://docs.typesafe.ai/api: "You can have a maximum of 255 options per Choice." | none |
| J-C3 | confirmed (cap kept) | https://docs.typesafe.ai/models: "64k tokens per request; 32k tokens for `state` plus the longest question" | 07 §4.2; 13 range 1,000–64,000 for `judge.max_request_tokens` (default 8,000 kept for accuracy) |
| J-C4 | unanswerable → Phase 0 | No tokenizer or counting endpoint documented; responses report `usage.input_tokens`. Examples show ~296 tokens for a 1-question request (fixed overhead) | 02 Q-02-1, 07 §4.8 |
| J-C5 | unanswerable | No length limit for instructions or criteria beyond the context budget | none |
| J-D1 | confirmed | https://docs.typesafe.ai/concepts/how-to-build-with-system-one: "Most queries complete in about 100 ms"; consistency cookbooks: 111 ms / 114 ms mean round trip | 07 §4.5, 09 §4.14 (note); targets look reachable, `open-questions.md` §1 row 6 unaffected |
| J-D2 | confirmed | https://docs.typesafe.ai/primitives: "System One models evaluate every question in a request in parallel. Adding questions barely changes the response time" | 09 Q-09-3 (resolved pending the Phase 0 curve) |
| J-D3 | corrected | https://docs.typesafe.ai/models: "250,000 tokens per second / 1,200 requests per minute", "can change without notice"; https://docs.typesafe.ai/cookbooks/entity_alignment: "the public endpoint rate-limits above roughly eight" | 07 §4.4, §5, D-07-8; 13 default 8; 16 §7 |
| J-D4 | corrected (not guaranteed) | https://docs.typesafe.ai/models: SDKs "honor the `retry-after` header when the response carries one"; SDK also reads `retry-after-ms` | 07 §4.6 |
| J-D5 | confirmed | https://docs.typesafe.ai/models: "\$42 / \$0.042" per Btok/Mtok; "Charged per input token. Output tokens are free." | 07 §4.8 (quote) |
| J-D6 | unanswerable → Phase 0 | HTTP/2 not mentioned | 07 Q-07-3 |
| J-D7 | corrected | No "early access" wording; https://docs.typesafe.ai/models warning: "Rate limits are adjusting dynamically … Higher limits are available on custom and enterprise plans." No SLA in docs; MCA has a generic service warranty | spec §20 row "Hosted, early access" is stale (Q-09-12 lists spec §20 updates); breaker defaults unchanged |
| J-G1 | corrected (gaps) | https://docs.typesafe.ai/model-jaggedness/jev-1.13 lists 9 modes; spec §20 misses indirection, contradictory instructions/criteria, structural invariants, generation, language | 09 Q-09-12; `open-questions.md`; jev-reference §10 (spec not edited) |
| J-G2 | unanswerable | Not documented. Related advice: structured option descriptions with `not_for` separate lookalikes (https://docs.typesafe.ai/primitives/advanced) | none; eval decides (09 §4.8) |
| J-G3 | confirmed (guidance found) | https://docs.typesafe.ai/primitives/noul: "Phrase the question so that a high value means yes … A statement works as well as a question"; backticked state paths; structured instructions; Noul `criteria` | 16 Q-16-9 (candidates only) |
| J-G4 | confirmed | https://docs.typesafe.ai/models: "English is the primary training language … Other languages, including CJK scripts, are handled but not equally well" | 09 Q-09-12 |
<!-- rows J-E*, J-F*, J-H*, J-I*, H-* appended below -->

## 4. Done when

- Every row in §1 has a status and evidence; §2 is covered as far as public docs allow.
- 07 §3.1–3.2 no longer say UNVERIFIED for anything that was confirmed or corrected. Anything left is listed as a Phase 0 live conformance item (07 §8.4).
- `docs/open-questions.md` is updated: Q-07-1, Q-07-2, Q-07-3, Q-07-6, Q-07-7, Q-09-3, Q-02-1, Q-12-1, Q-12-2 and Q-01-2 are resolved or narrowed, and owner decision row 3 has the evidence it needs.
- Any config key or default that changed also changed in `13-config.md`.
