# Jev reference for surf

**Status:** verified against the TypeSafe docs on 2026-09-23 (docs page "Jev 1.13 jaggedness" last reviewed 2026-09-17).
**Purpose:** everything surf needs to know about Jev, TypeSafe's API and its SDK, with a source for every fact. Design docs cite this file instead of repeating it. [`jev-docs-checks.md`](jev-docs-checks.md) §3 records the check-by-check verdicts.
**Owner of the wire code:** 07 (`judge/jev_wire.py`). If this file and 07 §3.1 disagree, the docs changed: re-check the source and fix both.

Sources (all fetched 2026-09-23; the docs publish an index at `https://docs.typesafe.ai/llms.txt` and a Markdown copy of each page at `<page>.md`):

| Short name | URL |
|---|---|
| API | https://docs.typesafe.ai/api |
| Models | https://docs.typesafe.ai/models |
| Primitives | https://docs.typesafe.ai/primitives (+ `/noul`, `/choice`, `/score`, `/advanced`) |
| State | https://docs.typesafe.ai/concepts/state |
| Confidence | https://docs.typesafe.ai/confidence |
| Build | https://docs.typesafe.ai/concepts/how-to-build-with-system-one |
| Jaggedness | https://docs.typesafe.ai/model-jaggedness/jev-1.13 |
| SDK | https://docs.typesafe.ai/sdk/python (+ `/api/constants`, `/api/retries`, `/api/exceptions`, `/api/types/*`) |
| Cookbooks | https://docs.typesafe.ai/cookbooks/* (named where cited) |

---

## 1. What Jev is, in surf's terms

Jev is a "System One" model: it takes a `state` and a map of typed questions and returns one typed answer per question. It doesn't generate text ("Jev is **not** a drop-in replacement for the LLM behind Claude Code"; Introduction → Jev with coding agents). That is exactly how surf uses it: code builds bounded questions, Jev answers, code decides.

| TypeSafe concept | surf usage |
|---|---|
| `state` | Redacted prompt and context strings (09 §3.2) |
| Noul question | Every candidate judgment: capability use, walk children, final pass, `needs_context` |
| Choice question | `continuity` only (`same` / `extends` / `new`) |
| Score question | Not used in v1 |
| Speculative fan-out (Patterns) | Call 1 asks every capability at once; walk level 1 runs speculatively |
| Keep math in code (Build, Jaggedness) | Thresholds, cumulative scores, budgets, lease logic live in code |

## 2. Endpoint and authentication

| Item | Value | Source |
|---|---|---|
| Endpoint | `POST https://api.typesafe.ai/v1/systemone` | API |
| Auth | `Authorization: Bearer <API_KEY>` | API |
| Content type | `application/json` | API |
| Key env var (SDK convention) | `TYPESAFE_API_KEY` | SDK constants |
| Base URL env var (SDK only) | `TYPESAFE_BASE_URL` (default `https://api.typesafe.ai`) | SDK constants |
| Model listing | `GET https://api.typesafe.ai/v1/models` → `{"models": [{"name", "description", "release_date"}]}`; lists aliases; versioned ids are accepted even if unlisted | Models |
| Request id | `x-typesafe-request-id` response header | SDK exceptions (`request_id`) |

## 3. Request body

```json
{
  "model": "jev-1.13.0",
  "state": {"request": "why do some orders never get a shipped date?",
            "project": "TypeScript + Supabase e-commerce backend",
            "previous_task": "add a shipped_at column to orders"},
  "questions": {
    "q000": {"type": "choice",
             "instructions": "How does the current request relate to the previous task?",
             "criteria": {"same": "continues the same work with no new area of the project",
                          "extends": "same overall goal but involves a new area, file type, or capability",
                          "new": "a different task"}},
    "q001": {"type": "noul",
             "instructions": "Answering the request requires information about this specific project's files, documentation, or database."},
    "q002": {"type": "noul",
             "instructions": "Completing the request likely requires using this capability: mcp supabase — tools: execute_sql, list_tables …"}
  }
}
```

This corrects the spec §11.4 illustration: Choice options go in **`criteria`**, not `options`.

| Field | Type | Rules | Source |
|---|---|---|---|
| `model` | string | Required. Versioned id (`jev-1.13.0`) or alias (`jev-latest`, `jev-preview`) | API, Models |
| `state` | string \| object \| array | Required. Text only; nested JSON allowed. Questions can point at parts with backticked paths (`` `ticket.messages[0].text` ``) | API, State, Primitives |
| `questions` | map id → question | Required. Ids are yours: "The key is not sent to the underlying model and is not used in inference." Grammar and max length undocumented | API |
| `questions.*.type` | `"noul"` \| `"choice"` \| `"score"` | Required | API |
| `questions.*.instructions` | string \| object \| array | Required. A question or a statement to judge. An object can hold the question in one field and data it refers to in others | API, Advanced |
| Noul `criteria` | `{"true": …, "false": …}` | Optional descriptions of yes and no | API, Noul |
| Choice `criteria` | map option → description \| object \| array \| `null` | Required. **Max 255 options.** Option names *and* descriptions are shown to the model | API, Choice |
| Score `criteria` | ordered array of levels | Required. 2–10 levels | API |

## 4. Response body

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "q000": {"type": "choice", "choice": "extends",
             "probabilities": {"same": 0.12, "extends": 0.81, "new": 0.07}, "confidence": 0.7},
    "q001": {"type": "noul", "noul": 0.93},
    "q002": {"type": "noul", "noul": 0.88}
  },
  "usage": {"input_tokens": 612, "output_tokens": 45}
}
```

(Values illustrative; shapes from the API reference.)

| Field | Type | Meaning | Source |
|---|---|---|---|
| `model` | string | The **versioned** id that answered, even when an alias was requested | API, Models |
| `answers.<id>.type` | string | Matches the question type | API |
| Noul `noul` | number 0–1 | Probability that the answer is yes. No `confidence` field | API, Noul |
| Choice `choice` | string | The highest-probability option | API |
| Choice `probabilities` | map option → number | Sums to 1 (values are rounded to 2 decimals in examples) | API |
| Choice `confidence` | number 0–1 | Required. Derived from the whole distribution; **not** `max(probabilities)` (e.g. 0.88/0.12/0 → 0.81; 0.61/0.35/0.04 → 0.42) | API, Choice, Confidence |
| Score `score`, `legend`, `probabilities`, `confidence` | – | Probability-weighted level position; not used in v1 | API |
| `usage.input_tokens`, `usage.output_tokens` | integer | Required | API |

## 5. Errors and retries

| Status | Meaning (API) | surf (07 §4.6) |
|---|---|---|
| 401 | Missing or invalid API key | `AUTH`, no retry, breaker opens 600 s |
| 422 | Body failed validation; JSON body names the field | `HTTP_4XX`, no retry (our bug) |
| 429 | Rate limit exceeded; "back off and retry after a short delay" | Retry once if the server wait fits the deadline |
| 529 | "TypeSafe is temporarily overloaded" | 5xx class, retry once |

The SDK adds what the API page doesn't list: 400, 403, 404 and generic 5xx exception classes; default retries on 408, 429 and all 5xx, plus connection errors and timeouts; `max_retries=2`, backoff 0.5 s doubling to 5 s with 25 % jitter, a 30 s total retry budget; it honors both `Retry-After` and `retry-after-ms` (SDK retries, exceptions). "Honor the `retry-after` header when the response carries one" (Models): the header is not guaranteed.

## 6. Models, pinning and aliases

| Name | Points to | Meaning (Models) |
|---|---|---|
| `jev-1.13.0` | itself | Current model |
| `jev-latest` | `jev-1.13.0` | "most recent stable, official release"; SDK default |
| `jev-preview` | `jev-1.13.0` | most recent release, official or not; no preview build exists now |

- "An alias moves when a new release ships, so the answers behind it can change without a change on your side … If you have tuned confidence thresholds against a specific version, pin that version's ID." surf pins `jev-1.13.0` (13 warns on aliases) and model-qualifies thresholds (07 D-07-6).
- Older versions stay callable: cookbooks published in 2026-07/08 still run `jev-1.12`; examples also use the short form `jev-1.13`. The docs don't state a deprecation policy (unanswerable; ask TypeSafe before relying on a pinned id for more than a release cycle).
- Same weights for every account; no fine-tuning ("Customizing Jev"). Domain knowledge goes into state, instructions and criteria.

## 7. Limits

| Limit | `jev-1.13` value | surf setting | Source |
|---|---|---|---|
| Context | 64k tokens per request (state + all questions); 32k for state + the longest single question | `judge.max_request_tokens` 8,000 (accuracy, not capacity) | Models |
| Questions per request | No documented limit | 40 (spec principle 5) | API, Primitives |
| Choice options | 255 | 3 | API, Choice |
| Score levels | 2–10 | – | API |
| Rate limits | 250,000 tokens/s and 1,200 requests/min, "adjusting dynamically … can change without notice"; over either → 429 | `judge.max_concurrency` 8 (D-07-8) | Models |
| Concurrency | Not documented. Cookbooks use 6–8 workers: "the public endpoint rate-limits above roughly eight" (Entity alignment) | 8 | Cookbooks |
| Input | Text only (string, JSON object, array of text) | strings only | Models, State |
| Language | "English is the primary training language and where accuracy is currently best"; others, incl. CJK, "handled but not equally well" | Q-09-12 | Models |

## 8. Latency and cost

| Fact | Value | Source |
|---|---|---|
| Typical latency | "Most queries complete in about 100 ms" (Build); "End-to-end response time is 70ms-500ms", service on the US West Coast (launch blog) | Build, blog |
| Measured round trips | 111 ms (14 Nouls) and 114 ms (Choice) mean over 15 runs; 0.09–0.31 s for a 182-option Choice + 3 Nouls. Third-party: jev-router reports a 511 ms median (location unknown) | Self-consistency cookbooks, Skill suggestion, jev-router README |
| Question count vs latency | Questions run in parallel; "adding questions barely changes the response time" | Primitives |
| Batching vs single calls | One 13-question call ≈ 12x cheaper and 10x faster than 13 calls with "no change in answers" | Parallel questions cookbook |
| Price | $0.042 per million input tokens ($42 per billion); output free | Models |
| Per-request overhead | ~260–300 input tokens (a 1-question request on a 12-word state reports 296; jev-fanout-bench measured ~261 via OpenRouter) | API examples, awesome-jev |
| SDK default timeout | 10 s per HTTP operation | SDK constants |

Implication for surf: spec §11.9 cost (≈ $0.001 per route) holds. Latency targets look reachable from the US; the published range (70–500 ms) is wide enough that distance to the West Coast and cold TLS matter more than question count (07 §4.5, 12). Phase 0 measures from where the developers are. Row 6 of `open-questions.md` §1 is unaffected.

## 9. Answer semantics

| Property | What the docs say | surf consequence |
|---|---|---|
| Noul is a probability | "the probability that the answer is yes"; trained with RLCD for "calibrated probabilities" (Noul, AI primer) | Thresholds apply directly to `noul` |
| Noul is not a degree | "A Noul value runs from 0 to 1, but it's not a scale of the thing you asked about" | Wordings must be yes/no propositions, not "how relevant" |
| Choice confidence | Summarizes how peaked `probabilities` is; "you are never locked into our definition" | `continuity_min_conf` is tuned on the API's `confidence`, never on `max(probs)` (07 §4.3) |
| Independence | "Every answer is independent … add or remove questions without changing the others' results" | Question-level fixture fallback is sound (07 Q-07-2) |
| Not comparable across types | "Don't carry a threshold tuned on a Noul over to a Choice"; a Noul and a yes/no Choice on the same question gave 0.22 vs 0.01 | Separate thresholds per question type (spec §20) |
| Negation isn't symmetric | A question and its negation as two Nouls: 0.72 + 0.47 = 1.19 | Never ask a negated question to get "not needed" (Q-09-12) |
| Choice is relative, Noul absolute | "the Choice is relative, settling *which* option, while each Noul is absolute and can be low for all of them" | Nouls for candidates is right for surf, which must be able to select nothing |
| Run-to-run stability | Not bit-reproducible: mean per-question SD 0.0102 over 15 repeats; one answer spanned 0.43–0.53 (runs varied a `uid` field in state) | Fixture replay for tests; live eval uses CIs (16 Q-16-2) |

## 10. Jaggedness (known failure modes) and surf's response

From Jaggedness (9 modes). Spec §20 covers the first group; the second group is proposed for spec §20 in Q-09-12.

| # | Failure mode | TypeSafe advice | surf response | In spec §20? |
|---|---|---|---|---|
| 1 | Literal reading | "state the exact condition"; put boundary cases in criteria | Wordings chosen by eval (spec §17.6); candidates per Q-16-9 | yes |
| 2 | Math, numbers, counting | "Keep the arithmetic in code" | All scoring math in code; churn not rendered in cards (Q-02-3) | yes |
| 3 | Date and time comparison | "Extract components; compare in code" | Recency is computed in code (04) | yes |
| 5 | Large state full of irrelevant detail | "Filter first; send only what the question needs" | ≤ 40 candidates, walk narrows, bounded state keys, 8k token cap | yes |
| 6 | Adversarial content | "State is data, and `jev-1.13` does not treat it as hostile by default" | 14 §4.4 filter, structural cards, routing grants nothing | yes |
| 4 | Indirection ("a property of a property", double negatives) | "Reduce hops; point to the relevant state" | Every question is one hop about one card; no negated wordings | no (Q-09-12) |
| 7 | Contradictory instructions and criteria | "Align the criteria and instruction" | Continuity option texts are reviewed with the instruction as one wording unit (16 §3.6) | no (Q-09-12) |
| 8 | Common-sense structural invariants | "Ask each decision one way; enforce identities in code" | Per-type thresholds; `not_needed` derived from the `use` Noul | partly ("not comparable") |
| 9 | Generation | "Use a generative model" | surf never asks Jev to produce text | no (not applicable) |

## 11. Question-writing guidance (candidates, not shipped)

Collected from Primitives, Noul, Choice, Advanced and Build. Per spec §17.6 none of these change a shipped wording without a dev-set run; 16 Q-16-9 adds them as candidates.

| Guidance | Quote | Candidate for surf |
|---|---|---|
| One snap judgment per question | "Ask for a judgment a knowledgeable person makes in a second" | Already the design |
| High value means yes | "Phrase the question so that a high value means yes" | Already the design |
| Statement or question both work | "A statement works as well as a question … Try both phrasings" | Question-form variants of w1/f1/c1 |
| Write the whole question | "Question IDs are for your code. They are not sent to the model." | Already the design (card text is in the instruction) |
| Point at state by path | "name it in the `instructions` with a dot-and-index path … including the backticks" | "Is this item needed to complete `request`? {card}" |
| Structured instructions | Put the question in one field and the data in others | `{"item": "<card>", "question": "Is \`item\` needed to complete \`request\`?"}` (needs 07 Q-07-8) |
| Noul criteria for subtle boundaries | "add `criteria` with `true` and `false` descriptions … try your questions with and without" | `true`: "the agent would open or query this item"; `false`: "only loosely related" |
| Separate lookalike options | Object criteria with `what` / `not_for` / `examples` | Continuity options (`same` vs `extends` is the known hard boundary) |
| Add a none-fits option to Choices | "add an `other` or `none of the above` option" | Not needed for continuity (the three options are exhaustive) |

## 12. Patterns in the TypeSafe docs that match surf

| TypeSafe pattern / cookbook | What it does | Relevance |
|---|---|---|
| Speculative fan-out | Ask every question you might need in one call; ignore irrelevant answers | Validates call 1 and the speculative walk |
| Skill suggestion cookbook | Picks ≤ 1 of 182 skills per agent turn: request 1 = Choice over all skills + 3 gate Nouls ("does the request need a skill at all?", mean < 0.30 → nothing); request 2 = Choice over the top 3 with fuller text + one "fits" Noul per finalist (all < 0.30 → nothing). Wrong loads 16.8 % → 7.3 %, needless loads 9.8 % → 4.0 % (Haiku 4.5, 488 requests) | Closest published analogue to surf's capability routing. Confirms: gate question ≈ `needs_context`; suggestion text worded as advisory ("Ignore this if it does not fit") and still sent when nothing fits ("No skill in the roster appears relevant"); a confident wrong suggestion hurts (7 of 315 broken). Informs 11 (note wording) and Q-16-10 |
| Re-ranking cookbook | BM25 top 30, then one Noul per query–candidate pair, sorted by value | Same shape as the final pass |
| Hierarchical classification cookbook | One Choice per tree node, beam search K = 3, paths compared by geometric-mean probability; includes a source-file hierarchy | Alternative walk design (Q-16-10); note it length-normalizes, while surf's `p_cum` deliberately ranks deep candidates lower |
| Confidence-gated routing | Act, confirm or escalate by confidence band | Same idea as `continuity_min_conf` |

## 13. Python SDK

| Fact | Value | Source |
|---|---|---|
| Install / import | `pip install typesafe-sdk` (0.7.1) / `import typesafe_sdk`. Spec §22.1's `typesafe-sdk-python` is the GitHub repo name, not a PyPI package | SDK, PyPI |
| Clients | `TypeSafeClient` (sync) and `AsyncTypeSafeClient` (async); `client.system_one(state=…, questions=…, model=…)`; typed `Noul`, `Choice`, `Score`, `NoulCriteria`; answers via `response.answers[...]` or `response.nouls` / `.choices` / `.scores` | SDK |
| Config | `api_key`, `base_url` (so any `/v1/systemone`-compatible endpoint works), `timeout`, `retry=RetryPolicy(...)`, custom `http_client` | SDK clients |
| Errors | `TypeSafeAPIError` subclasses per status, `TypeSafeAPIConnectionError`, `TypeSafeAPITimeoutError`, `TypeSafeAPIResponseValidationError` (with `field_path`) | SDK exceptions |

Version, dependencies, import cost and the runtime decision are in §14.

## 14. Providers, gateways and the SDK decision

All checked 2026-09-23. Every provider below uses the native envelope of §3–4; none needs a chat codec.

| Provider | Endpoint | Model id | Pin | Response `model` | Notes | Source |
|---|---|---|---|---|---|---|
| TypeSafe | `https://api.typesafe.ai/v1/systemone` | `jev-1.13.0` | exact | `jev-1.13.0` | Early access with a waitlist at launch (blog, 2026-09-15) | Models; https://typesafe.ai/blog/introducing-system-one-models-and-jev |
| OpenRouter | `https://openrouter.ai/api/v1/systemone` | `typesafe/jev-1.13` (`~typesafe/jev-latest` for the alias) | minor version, dated snapshot | `typesafe/jev-1.13-20260917` | "no waitlist or separate TypeSafe account"; not in the default chat-only `/api/v1/models` list (use `?output_modalities=all`); extra `id`, `provider`, `usage.cost`; errors `{"error":{"code","message"}}`, adds 402; 32k context listed; data policy `training: false, retainsPrompts: false`. An alpha `/api/alpha/decisions` surface exists (used by some prior art); don't use it | https://openrouter.ai/docs/guides/community/typesafe-sdk |
| Vercel AI Gateway | `https://ai-gateway.vercel.sh/typesafe/v1/systemone` | `typesafe-ai/jev` | none | `typesafe-ai/jev` | AI Gateway key or OIDC token; "The response uses TypeSafe's field names"; errors `{message, error_type}`; `zdr: all`; free until 2026-09-25. Its separate `/v1/evaluate` API renames Noul to `boolean`; don't use it | https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe |
| Cloudflare | Workers AI `POST https://api.cloudflare.com/client/v4/accounts/{id}/ai/run`, body `{"model":"typesafe/jev","input":{state, questions}}` | `typesafe/jev` | none | `jev-1.13.0` (example) | Not an AI Gateway provider; different envelope. Dropped from v1 (07 D-07-9) | https://developers.cloudflare.com/ai/models/typesafe/jev/ |

Local open reproductions ("independent efforts, not official TypeSafe releases", awesome-jev list) expose the same `/v1/systemone` path. Reference for surf: **Laya** (`pip install "laya[serve]"`, `laya-serve`, `127.0.0.1:8321`, Apache-2.0 code and weights, CPU, "degrades past about 20 options"). Others: LitJev (Apache-2.0 code, Qwen weights, "Probabilities are not calibrated by default"), kev (Apache-2.0), ruling (MIT, Apple MLX), open-jev (Gemma weights, non-Apache terms), openjev-sglang (no LICENSE). There are no published Jev weights.

**SDK decision (07 D-07-2, Q-07-1 resolved):** `typesafe-sdk` 0.7.1 (MIT, Python ≥ 3.10, first public release 2026-09-14) is async and works with OpenRouter and Vercel through `base_url`, but its path is hardcoded to `/v1/systemone`, it depends on `httpx2` (not httpx), `import typesafe_sdk` measured ~243 ms vs ~129 ms for httpx + pydantic, it has had two breaking minor releases, and debug logging writes request bodies unredacted. surf keeps httpx at runtime and uses the SDK only in the conformance test. If it is ever used at runtime, pass `RetryPolicy(max_retries=0)` so 07's retry and breaker logic governs.

**Prior art (spec §4), licenses checked 2026-09-23:** MIT: jev-code-context-router, JevRouter, jev-codex-router, langchain-skill-router, jev-skillful. No LICENSE: jev-router, blink, jev-knowledge-base (spec §4 lists only jev-router as unlicensed; see `open-questions.md` Q-07-9). All of them use the field names in §3–4. Useful practice: jev-codex-router caps at 40 questions per request; jev-skillful retries {429, 502, 503, 504, 529}; one project defaults a missing answer to `noul = 0.0`, which surf must not do (07 §4.3 treats it as missing).

## 15. Still unverified: Phase 0 live checks

Everything below is either undocumented or documented only by example. 07 §8.4 runs them with a real key.

| Check | Why it matters | Linked |
|---|---|---|
| Live responses match §3–4 (shapes, `model` echo, `usage`) | Docs can drift from the service | 07 §8.4 |
| Identical-request spread (×20) | Separates noise from the cookbook's `uid` confound | Q-16-2 |
| Same question in two batches (×20) | Confirms documented independence | Q-07-2 |
| Latency vs question count, cold and warm | Confirms "barely changes"; sizes TLS cost | Q-09-3, Q-07-3 |
| 16-way burst: 429 share, `Retry-After` presence | Concurrency default | D-07-8, Q-07-7 |
| Reported `input_tokens` vs `ceil(bytes/4)` | Token heuristic, fixed overhead | Q-02-1 |
| Question-id grammar and length | We send opaque `q000` anyway; only matters if that changes | 07 §3.1 |
| Deprecation policy for versioned ids | Pinning lifetime | §6 |
| OpenRouter accepts `jev-1.13.0`? Probabilities identical to TypeSafe direct on a fixed request? | Gateway use outside eval | 07 Q-07-6 |
