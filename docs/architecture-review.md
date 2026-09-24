# Architecture review against the Jev docs

**Date:** 2026-09-24. **Inputs:** [`jev-reference.md`](jev-reference.md) (verified docs), [`jev-docs-checks.md`](jev-docs-checks.md) §3, prior-art source (8 repos, cloned 2026-09-23), and keyless probes of `api.typesafe.ai`.
**Question:** now that Jev's behavior is documented, does each architecture decision still earn its keep?
**Status:** applied 2026-09-24 at the owner's request. Starting values are still confirmed by the dev-set eval (spec §17) before release.

| Finding | Applied as |
|---|---|
| §3.1 flat-first, principle 5 | Spec D14, D15 (§2, §11, §16, §17.5, §20); 09 §4.5, §4.8, D-09-22; 05 `flat_cards()`/`flat_card_tokens()`; 13 `router.flat_max_tokens`, `router.mode = walk`; 16 A4w, D-16-13. `flat_max_tokens` = 40,000 is set by A0 vs A4w (Phase 2 exit) |
| §3.2 latency | Spec D18, §11.9; 07 D-07-12 (HTTP/2; `h2` import ~14 ms measured). Warm path stays a Phase 0 measurement (12 Q-12-9) |
| §3.3 gate | Flat mode: content answers are the gate (09 §4.6). Walk mode: open (Q-09-14) |
| §3.4 speculation | Kept separate requests in the same wave (spec D16); `task.cancel()` kept, since it costs nothing and the route returns immediately anyway. The review's "drop cancellation" overstated the saving |
| §3.5 TPS | Token budget per route (spec D14); 13 warns above 100k |
| §3.6 providers | Spec D17; 07 D-07-10 (Vercel deferred) |
| §3.7 per-backend caps | Spec D15; 07 §4.2, D-07-11 |
| §3.8 note wording | Candidates in 11 Q-11-4 (eval decides) |
| Also | Spec D19 (concurrency 8), D20 (final pass one request), §4 licenses, §22.1 client, §20 jaggedness rows |

---

## 1. Verdict

| Area | Decision today | Verdict | Proposal |
|---|---|---|---|
| Candidate judgments | One Noul per candidate | **Keep** | – |
| Continuity | One Choice `same`/`extends`/`new` | **Keep** | Tune `continuity_min_conf` on the API's `confidence` (already in 07) |
| Score primitive | Unused | **Keep unused** | – |
| ≤ 40 questions per request (principle 5) | Hard cap, justified by "large irrelevant state" | **Change**: it bounds the wrong variable | Bound *state* tightly and *requests* by tokens (§3.1) |
| Walk as the default candidate generator | Walk above 60 content cards | **Change, gated by A0 vs A4** | Flat-first up to a per-route token budget; walk only above it (§3.1) |
| `needs_context` gate | One abstract Noul | **Weak evidence; measure** | In flat mode the content Nouls are the gate; in walk mode test a composite gate (§3.3) |
| Speculative walk level 1 | Separate cancellable task | **Simplify** | Same wave, no cancellation machinery (§3.4) |
| Final pass | Strict re-judgment of ≤ 40 | **Keep in walk mode; narrow in flat mode** | Flat mode re-judges only expansion candidates (§3.1) |
| Graph expansion | Containment, co-change, schema refs | **Keep** | – |
| Task lease | Continuity Choice every prompt | **Keep** (stability and note dedup, not cost) | – |
| Latency budget | Assumes ~400 ms per Jev round trip | **Change the model**: TLS and process start dominate | HTTP/2 + one connection per route now; measure a warm path for `same` (§3.2) |
| Concurrency and rate limits | 8 in flight | **Keep**, add a per-route token budget | TPS, not RPM or price, is the binding limit (§3.5) |
| Providers | `typesafe`, `openrouter`, `vercel` | **Trim** | Keep `typesafe` + `openrouter`; defer `vercel` (no version pin) (§3.6) |
| `systemone-local` | Same codec, same caps | **Keep for A7**, per-backend caps | Question cap per backend (§3.7) |
| Async + httpx, SDK dev-only | – | **Keep** | – |
| Breaker, retry-once, fixture judge, eval harness, `llm` baseline | – | **Keep** (non-determinism makes fixtures more necessary, not less) | Extend the Phase 0 curves (§4) |
| Note wording | "use:" / "skip:" lines | **Measure** | Factual phrasing + explicit "ignore if it doesn't fit" as candidates (§3.8) |

The two changes with real leverage are §3.1 (flat-first) and §3.2 (latency comes from TLS and process start). Everything else is small.

## 2. What we know now that the design didn't

| Fact | Source | Design assumption it overturns |
|---|---|---|
| Each question sees only the state and itself: "Jev ingests the `state` once and evaluates every question against it in parallel"; the 32k budget is "`state` plus the single longest question"; "Every answer is independent" | Models, Primitives | That many candidates in one request crowd each other. That was the rationale for principle 5 (spec §20 row 1) |
| The jaggedness warning is about **state**: "Accuracy falls as the state grows with content unrelated to the decision" | Jaggedness #5 | Same |
| A directory judgment ("this directory likely contains what's needed") is a property of a property; "Instructions carrying … indirection are answered less reliably" | Jaggedness #4 | That walk questions are as reliable as file questions |
| ~100 ms typical, 70–500 ms end to end; question count "barely changes" latency (cookbook: 16 q 0.32 s, 62 q 0.51 s; jev-skillful: 4 q 0.75–0.86 s vs 16 q 0.69–0.76 s) | Build, blog, cookbooks, jev-skillful | ~400 ms per round trip (09 §4.14) |
| TCP + TLS is 0.21–0.32 s of a 0.7–0.8 s route (jev-skillful, measured); `api.typesafe.ai` negotiates HTTP/2 (Envoy) | jev-skillful docs/routing.md; probe 2026-09-24 | HTTP/2 unknown (Q-07-3); latency attributed to the model |
| $0.042/Mtok input; ~260–300 tokens fixed per request; 1,200 RPM and **250k tokens/s per account** | Models, API examples | Cost as the reason to avoid judging many cards |
| A single abstract "is anything needed?" Noul gate scored 0.39 on a certain case and 0.10 for both "fix a typo" and "thanks!"; jev-skillful removed it. TypeSafe's cookbook uses three action-phrased gate Nouls averaged and warns that subject-matter gates don't separate | jev-skillful docs/routing.md; Skill suggestion | `needs_context` as one Noul |
| Not bit-reproducible (SD ≈ 0.01) | Self-consistency cookbook | Nothing; confirms fixtures |
| Local reproductions differ: Laya ~815 ms median on CPU; openjev-sglang caps at 64 questions | jev-router README; openjev-sglang defaults | One question cap for every backend |
| Vercel exposes only an unversioned `typesafe-ai/jev`; OpenRouter pins the minor version and has no waitlist; TypeSafe direct launched with a waitlist | Gateway docs, blog | Gateways as equivalent providers |
| A keyless request returns **403** with `{"detail": {"error_type": "authentication_error", "message": …}}`, not the documented 401 | Probe 2026-09-24 | 07 maps only 401 to `AUTH` (403 already maps there; body shape noted in 07) |

## 3. Findings

### 3.1 The 40-question cap bounds the wrong variable; route flat-first up to a token budget

**First principles.** Jev's accuracy risk is irrelevant *state* (jaggedness #5), and questions are judged independently against state (Models, Primitives). surf puts candidates in **questions** and keeps state small (09 §3.2). So a request with 300 file Nouls exposes each judgment to the same small state as a request with 40. The cap limits nothing the docs warn about. What the walk actually buys is a smaller total token volume, and it pays for it with 2–3 sequential round trips and directory-level questions that the docs flag as indirection.

**Cost and limits of judging every file card directly** (file card ≤ 60 tokens + wording + JSON ≈ 85 tokens per question; state ≈ 500; overhead ≈ 280 per request; requests ≤ 30k tokens to fit OpenRouter's 32k):

| File cards | Tokens per route | Requests | Cost | Share of 250k TPS |
|---|---|---|---|---|
| 60 (today's `small_repo_cutoff`) | ~6k | 1 | $0.0003 | 2 % |
| 300 | ~26k | 1 | $0.0011 | 10 % |
| 500 | ~44k | 2 | $0.0018 | 18 % |
| 1,000 | ~87k | 3 | $0.0037 | 35 % |
| 2,000 | ~175k | 6 | $0.0074 | 70 % |
| Walk mode today (spec §11.9) | 10–30k | 5–20 | ≤ $0.0013 | 4–12 % |

**Latency (engine, cold process):**

| Mode | Sequential waves | Estimate |
|---|---|---|
| Walk (today) | call 1 ∥ L1 → L2 → final | TLS ~0.25 s + 3 × 0.15–0.5 s ≈ 0.7–1.75 s |
| Flat (≤ budget) | call 1 ∥ flat chunks → (final for expansion-only candidates) | TLS ~0.25 s + 1–2 × 0.2–0.6 s ≈ 0.45–1.45 s |

**Proposal (Q-09-13).**
1. Replace the card-count `small_repo_cutoff` with a per-route token budget. The flat pass asks one Noul per leaf card (file, doc, table) with the *final* wording, in the same wave as call 1. Starting budget: ~40k tokens, about 450 file cards and ≤ 16 % of account TPS. Above the budget the walk runs as designed.
2. In flat mode, the final pass re-judges only candidates the flat pass didn't judge: expansion neighbors and ambiguous path hits. A route becomes one wave, or two when expansion adds candidates.
3. Restate principle 5 and the matching CLAUDE.md non-negotiable as "state stays small (≤ ~2k tokens); every request ≤ 30k tokens; every route ≤ the token budget". This is owner-only, and only after the eval below.

**Decides it:** the A0 (flat) vs A4 (walk) ablation already in 16, run on repos that sit *below* the proposed budget, plus a flat request latency curve at 40/100/300 questions (§4). Risk to watch: precision. Thousands of independent Nouls produce more false positives above τ, so check that the ≤ 12 budget and ranking absorb them (`must_exclude` violations, 16 §4.5).

### 3.2 Latency is dominated by TLS and process start, not by Jev

With Jev at ~100–500 ms, the fixed costs of a per-prompt hook process (Python start and imports, ≤ 200–300 ms in 12 §7, plus a cold TLS handshake of 0.2–0.3 s measured by jev-skillful) are as large as the model call. The **continuing-task target** is where this bites: `same` = TLS + one request ≈ 0.35–0.8 s of engine time, against p50 ≤ 0.5 s.

**Proposals.**
- **Q-07-3 → answer and adopt (07):** the origin negotiates HTTP/2. Use one `httpx.AsyncClient(http2=True)` per route so all parallel chunks share one handshake, if the `h2` import cost stays within 12 §7's hook budget (measure).
- **Q-12-9 (new):** measure `same`-path engine latency in Phase 0 from the developers' location. If p50 > 0.5 s, add a warm path: the hook forwards to an already-running `surf mcp`/daemon process over a local socket that holds a live connection. That would be a v1.1 trigger, not v1 work.

### 3.3 The `needs_context` gate has weak support

Independent evidence says one abstract "is anything needed?" Noul separates poorly. jev-skillful measured it and replaced it. TypeSafe's cookbook averages three *action-phrased* Nouls and warns that subject-matter gates don't separate "explain a monad" from real work. In surf the gate decides `no-context` exits and arms the dead-end guard (09 §4.6, §4.8).

**Proposal (Q-09-14).** In flat mode, drop the separate gate: "no content Noul ≥ τ" is the gate, measured on the same cards. In walk mode, keep a gate but add a composite candidate (2–3 action-phrased Nouls, mean) to `wordings.yaml` and let the dev set pick. Decided by eval (`no_context_gate` attribution bucket, 16 §4.6).

### 3.4 Speculation doesn't need cancellation

The speculative walk was designed as a cancellable `asyncio.Task` to save tokens when call 1 says `same` (09 §4.7, 07 §4.4). At $0.042/Mtok the saving is ~$0.0002 per `same` prompt (one or two root-level chunks). TypeSafe's own guidance is the fan-out pattern: send speculative questions and ignore unused answers.

**Proposal (Q-09-15).** Fire walk L1 (or the flat chunks) in the same wave as call 1 and simply stop awaiting them on `same`; the route closes the client when it returns. Remove the ledger's `cancelled` outcome and the cancellation tests. Keep separate requests rather than merging walk questions into call 1: call 1's state carries `previous_task`, which would be an irrelevant-state distractor for walk judgments on `new` tasks (jaggedness #5).

### 3.5 TPS, not price or RPM, is the per-route constraint

A route's token volume competes for one account-wide 250k tokens/s budget ("adjusting dynamically"), shared by every session and by eval runs. jev-skillful reaches the same conclusion: "The real constraint is the rate limit, not the price."

**Proposal (part of Q-09-13).** Make the flat/walk switch a token budget, and log `input_tokens` per route (already in the ledger) so the 429 share can be correlated with route size. The concurrency cap of 8 (D-07-8) stays.

### 3.6 Providers: keep TypeSafe and OpenRouter, defer Vercel

Model-qualified thresholds (D-07-6) only work if the served version is known. TypeSafe pins exactly. OpenRouter pins the minor version, echoes a dated snapshot, and has no waitlist, which matters for onboarding while TypeSafe direct is early-access. Vercel serves one unversioned id and echoes it, so a silent model change can't even be detected.

**Proposal (Q-07-10).** v1 providers: `typesafe`, `openrouter`. Defer `vercel` until it exposes versioned ids. If it's kept, mark it always `uncalibrated`, so `surf doctor` warns and decision records flag it.

### 3.7 Question caps and latency are per backend

`openjev-sglang` rejects more than 64 questions per request, Laya degrades on large Choices, and CPU reproductions take ~0.8 s per request, so three sequential walk waves approach the 3 s deadline.

**Proposal (Q-07-11).** Make `max_questions_per_request` and the request-token cap per-backend defaults: `jev` bounded by tokens only; `systemone-local` at 64. Flat-first (§3.1) also helps local backends most, because it cuts the number of round trips.

### 3.8 Note wording

Claude Code's hook docs advise writing injected context "as factual statements rather than imperative system instructions" to avoid prompt-injection defenses. TypeSafe's skill-suggestion cookbook found that advisory phrasing ("Ignore this if it does not fit") and an explicit line when nothing fits beat silence, and that confident wrong suggestions still break some turns (7 of 315).

**Proposal (11 Q-11-4, extended).** Add candidate note wordings: factual labels ("probably relevant", "probably not needed") in place of `use:`/`skip:`, plus an explicit "ignore if it doesn't fit". Choose by the sequence eval.

### 3.9 What holds up

| Decision | Why it still earns its keep |
|---|---|
| Nouls for candidates | Absolute, can all be low (surf must be able to select nothing); multiple selections allowed; Choice is relative and "a large kind drowns out a small one" (jev-skillful quotas) |
| Choice for continuity | Three exhaustive, mutually exclusive options with a documented confidence; keep options aligned with the instruction (jaggedness #7) |
| Deterministic index, no summaries (D1) | Docs: "Do not rely on knowledge stored in model weights"; Jev can't generate. Cards are the only practical input |
| Graph expansion (D7) | Catches files whose cards are too thin to be judged relevant; flat mode doesn't change that |
| Lease (D10) | Latency gain is smaller than assumed, but note stability and no repeated notes remain |
| Fixture judge with question-level keys | Independence is documented; non-determinism makes record/replay the only way to get reproducible CI |
| `llm` baseline (A6) | Still the only way to show Jev beats a general LLM on *this* task |
| Async + httpx at runtime | Parallel waves need it; the SDK costs ~115 ms more import time and brings a second HTTP stack |
| Persisted breaker, retry-once | The per-process hook still can't see failure streaks otherwise; retries now include 408/529 |

## 4. Phase 0 measurements this review adds

| Measurement | Decides |
|---|---|
| Latency vs question count at 1, 40, 100, 300 questions, warm and cold, HTTP/1.1 vs HTTP/2 | Q-09-13 budget, Q-07-3 |
| Route-level token volume and 429 share at flat budgets of 20k/40k/80k tokens | Q-09-13 budget, D-07-8 |
| `same`-path engine latency from the developers' location | Q-12-9 |
| A0 (flat, final wording) vs A4 (walk) on repos under the budget: recall, `must_exclude`, precision | Q-09-13 |
| Gate variants: single Noul / composite / "max content Noul" | Q-09-14 |

## 5. Registered proposals

| Id | Doc | Proposal | Decided by |
|---|---|---|---|
| Q-09-13 | 09 | Flat-first by per-route token budget; walk above it; restate principle 5 | A0 vs A4 eval, then owner (spec §2, CLAUDE.md) |
| Q-09-14 | 09 | Gate from content Nouls in flat mode; composite gate candidate in walk mode | Eval |
| Q-09-15 | 09 | Speculation without cancellation; separate requests, same wave | Design review |
| Q-07-3 | 07 | Origin supports HTTP/2: one h2 client per route if import cost fits | Phase 0 latency curve |
| Q-07-10 | 07 | Defer `vercel` (no version pin) | Owner |
| Q-07-11 | 07 | Per-backend question and token caps | Design review |
| Q-12-9 | 12 | Warm path for the `same` route if p50 > 0.5 s | Phase 0 measurement |
