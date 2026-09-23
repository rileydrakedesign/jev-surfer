# 11 · Routing note

**Status:** draft for review
**Spec sections:** §14 (all), §11.4 (capability-only outcome), §11.8 (selection feeds the note), §12.4 (delta note)
**Depends on:** 00-foundations (ids, `Selection`, `RouteResult`), 05-catalog-store (`CardLookup`, alias lookup), 09-router (selection order, `low_confidence`), 10-lease (`LeaseDelta`), 13-config (`note.*`)
**Code:** `surf/route/note.py`

---

## 1. Purpose and scope

Turn a selection (or a lease delta) into the short plain-text note that the adapter injects or returns. The note is the only thing the agent sees from surf, so its format is part of the product and is covered by golden tests and evaluation.

| In scope (v1) | Out of scope (v1) |
|---|---|
| Full note, delta note, capability-only note | File contents or excerpts (spec principle 4) |
| Line grammar, ordering, display names, path escaping | Localization |
| Line cap with deterministic truncation | Harness-specific markup (Markdown, XML tags); adapters get plain text |
| Format versioning | Hard MCP pruning (spec §14.3) |

The note builder is pure: no I/O, no clock, no judge. Given the same inputs it returns the same bytes.

## 2. Interfaces

```python
NOTE_FORMAT_VERSION: Final = 1        # bump on any change to rendered text; logged in decision records

class NoteKind(StrEnum):
    FULL = "full"; DELTA = "delta"; CAPS_ONLY = "caps_only"

class NoteInput(BaseModel):
    kind: NoteKind
    content: list[SurfaceId]            # priority order (09 selection order, or LeaseDelta.content_added)
    use: list[SurfaceId]                # capabilities to use
    skip: list[SurfaceId]               # capabilities not needed (FULL and CAPS_ONLY only)
    low_confidence: bool
    cwd_rel: str | None = None          # cwd relative to repo root; None or "" when at root

class RenderedNote(BaseModel):
    text: str | None                    # None → inject nothing
    kind: NoteKind
    lines: int
    pointers_shown: int
    pointers_dropped: int               # dropped by the line cap
    format_version: int = NOTE_FORMAT_VERSION

def render(inp: NoteInput, *, catalog: CardLookup, cfg: NoteConfig) -> RenderedNote: ...

# helpers (public for tests and for `surface_info` / `surf status` display)
def display_path(path: str) -> str: ...
def display_capability(card: Card) -> str: ...
def display_table(sid: SurfaceId) -> str: ...
```

Who calls it:

| Caller | Kind |
|---|---|
| `route/pipeline.py`, continuity `new` or no lease, content non-empty | `FULL` |
| pipeline, continuity `extends`, non-empty `LeaseDelta` | `DELTA` |
| pipeline, `no-context` / `no-candidates`, or routed content empty | `CAPS_ONLY` |
| pipeline, `same`, skip, empty delta, `judge-unavailable`, `index-missing`, `error` | not called; `note=None` |

The pipeline puts `RenderedNote.text` into `RouteResult.note` and `format_version`, `lines`, `pointers_dropped` into the decision record (15).

## 3. Data structures: the line grammar

```
note        := header NL body [lowconf NL] [fallback NL-less]
header      := "[surf] " header_text [root_hint]
body        := line+                           (at least one line, else no note)
line        := "  " label items                (continuation: 10 spaces + items)
label       := "code:   " | "schema: " | "docs:   " | "use:    " | "skip:   "
items       := item (" · " item)*
lowconf     := "(low confidence — verify before relying on these)"
fallback    := "(surf routed this task; use your normal search if these don't cover it)"
root_hint   := " (paths from repo root)"
```

- Labels are padded to 8 characters so items align at column 10. The spec's delta example uses unpadded labels; we pad in every kind for one grammar (D-11-1).
- Lines end with `\n`; the note has no trailing newline.
- The only non-ASCII characters surf itself emits are `—` and `·`. With `note.ascii_only = true` they become `-` and `|`.

| Kind | Header text | Lines allowed | lowconf | fallback |
|---|---|---|---|---|
| `FULL` | `Likely relevant — open as needed, nothing is preloaded:` | code, schema, docs, use, skip | if `low_confidence` **and** ≥ 1 content pointer is rendered | always |
| `DELTA` | `Also relevant for this part of the task:` | code, schema, docs, use | if `low_confidence` **and** ≥ 1 content pointer is rendered | never (spec §14.2) |
| `CAPS_ONLY` | `No specific project files flagged; capabilities for this task:` | use, skip | never | always |

## 4. Behavior

### 4.1 Classifying pointers into lines

| Surface type | Line | Rendered as |
|---|---|---|
| `code_file`, `code_dir` | code | `display_path(path)`; dirs keep the trailing `/` |
| `doc_file`, `doc_dir` | docs | `display_path(path)` |
| `db_table` | schema | `display_table(id)`: locator of `db:` (`orders`, `billing.invoices`) |
| `db_migration` | schema | inside the migration group (§4.2) |
| `code_file` that has an `alias` edge to a `mig:` id (00 §2.3 F3) | schema | rendered as its `mig:` id; if both ids are selected, rendered once |
| `schema_root`, `root:` | never selected; dropped defensively with a debug log | |
| capability in `use` / `skip` | use / skip | `display_capability(card)` |

Items are grouped per line **in the input order** (selection priority from 09: path hits → final score desc → id asc; for deltas, `LeaseDelta.content_added` order). Input order is deterministic, so the note is deterministic; we don't re-sort alphabetically because order carries importance. Capability lines are ordered by the call-1 score desc, then id asc, and are already capped by 09 at `router.max_caps_use` (6) and `router.max_caps_skip` (10) (D-09-16). The note builder doesn't re-cap; the line cap in §4.5 still applies.

Lines appear in the fixed order code, schema, docs, use, skip. Empty lines are omitted.

### 4.2 Schema line with migrations

The router doesn't select migrations for tables (09 §4.9); the note builder attaches them. The migration group is built as follows:

1. For each rendered table, in line order, take the table card's `last_changed_in` migration, or `created_in` if it has never been altered (03 table card fields). This is the "migration that added `shipped_at`" in the spec §1.1 example: the latest schema change to a table is the most likely one to matter. Only `mig:` targets count; snapshot sources (`schema.prisma`, `schema.rb`) have no `mig:` card and are skipped.
2. Add any `mig:` ids (or aliased `code:` migration ids) that are in the selection itself, in input order.
3. Dedupe, keep the first `note.max_migrations` (default 2), and drop the rest silently (they're discoverable through `surface_info`).

```
schema: orders · shipments (migration: supabase/migrations/20260611_add_shipments.sql)
schema: orders (migrations: db/001_init.sql · db/014_orders_status.sql)
schema: (migration: supabase/migrations/20260611_add_shipments.sql)       # migrations only
```

Tables first in input order, then one parenthesized group with the migrations from the rule above. `migration:` is singular for one item and plural otherwise. In a `DELTA` note, a migration already shown for the same table in this task isn't known to the note builder (it's stateless), so it may be repeated. That's accepted: it's one short parenthetical.

### 4.3 Display names

| Surface | Id | Display |
|---|---|---|
| MCP server | `mcp:supabase` | `supabase MCP` |
| Skill | `skill:billing-reports` | `billing-reports skill` |
| Subagent | `agent:code-reviewer` | `code-reviewer subagent` |
| Command | `cmd:deploy` | `/deploy command` |
| Table | `db:billing.invoices` | `billing.invoices` |

The name is the id's locator (already sanitized at index time by 02/14). It is passed through `display_path`'s escaping rules to guard against odd names.

### 4.4 Path escaping

Paths are repo-relative POSIX (00 §2.2) and printed as-is when they're "plain":

```python
PLAIN = re.compile(r"^[\w.\-/@+\[\]()~,=#%&!$]+$", re.UNICODE)   # \w includes unicode letters/digits

def display_path(p: str) -> str:
    p = unicodedata.normalize("NFC", p)
    p = "".join(ESCAPES.get(ch, ch) if is_control(ch) else ch for ch in p)   # \n → "\\n", \t → "\\t", other Cc/Cf → "\\xNN"/"\\uNNNN"
    if PLAIN.match(p) and " · " not in p:
        return p
    fence = "``" if "`" in p else "`"
    pad = " " if p.startswith("`") or p.endswith("`") else ""
    return f"{fence}{pad}{p}{pad}{fence}"
```

| Path | Rendered |
|---|---|
| `src/api/orders/[id].ts` | `src/api/orders/[id].ts` |
| `docs/Guía de envíos.md` | `` `docs/Guía de envíos.md` `` |
| `src/ünïcode/ß.ts` | `src/ünïcode/ß.ts` (unicode letters are plain) |
| ``a`b.ts`` | ``` ``a`b.ts`` ``` |
| `weird\nname.ts` (newline in name) | `` `weird\nname.ts` `` (escaped) |
| `a · b.md` | `` `a · b.md` `` |

Paths are never made absolute and never shortened (no basename-only). The root hint (§3) is added when `cwd_rel` is non-empty, because the agent's working directory then differs from the paths' base.

### 4.5 Wrapping and the line cap

Constants from config: `note.max_lines` (default 15, spec §14.3), `note.max_line_chars` (default 160).

```
1. Build lines per §4.1–4.3. A line whose length exceeds max_line_chars is wrapped at item
   boundaries onto continuation lines (10-space indent). An item longer than the limit
   gets its own line and is never split.
2. Count: header + body lines (incl. continuations) + lowconf + fallback.
3. While count > max_lines:
     drop the lowest-priority remaining item, in this order of preference:
       a. skip items, last first
       b. content items, last in input order first (path hits are dropped last)
       c. use items, last first
     rebuild; ensure the marker line "  (+N more not shown)" is present (it counts as a line).
4. If no body line remains, return text=None.
```

The skip line goes first because it's advisory and the cheapest to lose. With ≤ 12 content pointers and one line per category, the cap is only reached through wrapping, so truncation is rare. It's still enforced because the cap is a hard format rule.

### 4.6 Note size

The note should stay under about 250 tokens. `render` asserts `len(text) ≤ note.max_chars` (default 2,000); if it's exceeded after the line cap (e.g. extremely long paths), items are dropped with the same order as §4.5 until it fits. This is a guard, not a normal path.

### 4.7 Examples (golden fixtures)

Full note, spec §1.1 example, `cwd` at root:

```
[surf] Likely relevant — open as needed, nothing is preloaded:
  code:   src/fulfillment/ship.ts · src/api/orders/[id].ts
  schema: orders · shipments (migration: supabase/migrations/20260611_add_shipments.sql)
  docs:   docs/fulfillment/shipping-lifecycle.md
  use:    supabase MCP
  skip:   figma MCP · billing-reports skill
(surf routed this task; use your normal search if these don't cover it)
```

Delta note, low confidence:

```
[surf] Also relevant for this part of the task:
  code:   src/notifications/email/shipment-confirmation.ts
  docs:   docs/notifications/templates.md
(low confidence — verify before relying on these)
```

Capability-only note:

```
[surf] No specific project files flagged; capabilities for this task:
  use:    supabase MCP
  skip:   figma MCP
(surf routed this task; use your normal search if these don't cover it)
```

### 4.8 Format stability

Any change to the header texts, labels, separators, fallback or low-confidence wording changes what agents read, so it bumps `NOTE_FORMAT_VERSION`, updates the golden files, and requires a dev-set eval run (spec §17.6 applies to note wording too). The header and fallback strings live in `route/note_text.py` constants so the eval wording experiments (16) can override them via `eval/wordings.yaml` key `note`.

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `note.max_lines` | int 4–30 | `15` | spec §14.3 |
| `note.max_line_chars` | int 60–400 | `160` | wrap width |
| `note.max_chars` | int 200–8000 | `2000` | size guard |
| `note.show_skip` | bool | `true` | `false` removes skip lines entirely (for users who distrust advisory skips) |
| `note.ascii_only` | bool | `false` | `—` → `-`, `·` → `|` |
| `note.root_hint` | bool | `true` | add "(paths from repo root)" when cwd ≠ root |
| `note.max_migrations` | int 0–5 | `2` | §4.2; `0` disables the migration group |
| `router.max_caps_use` / `router.max_caps_skip` | int | `6` / `10` | applied by 09 before rendering |

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| Empty content, empty use and skip | `text=None` for every kind |
| `FULL` with empty content but capabilities present | Pipeline sends `CAPS_ONLY` instead; if it doesn't, `render` downgrades to `CAPS_ONLY` |
| `DELTA` with only `use` additions | Delta note with a single `use:` line |
| Selection id missing from catalog (race with refresh) | Item dropped, debug log; the rest renders |
| Both `code:X` and `mig:X` selected | Rendered once as a migration |
| Directory and a file inside it both selected (09 should prevent this) | Both rendered; no dedupe in the note layer (dedupe belongs to selection/lease) |
| Path with spaces, backticks, control chars, ` · ` | §4.4 escaping |
| Very long path (> `max_line_chars`) | Own line, never split |
| Capability both `use` and `skip` (caller bug) | `use` wins; `skip` entry dropped; warning logged |
| Table in non-default schema | `billing.invoices` |
| Selected table with no `mig:` history (Prisma/Rails snapshot source) | Table rendered, no migration group |
| `low_confidence` but only capability lines render | No low-confidence line (09 §4.13) |
| Caller passes >12 content items | Rendered subject to the line cap; no silent re-budgeting (budget is 09's job) |
| Exception inside `render` | Pipeline's guard turns it into `status=error`, `note=None` (00 §5) |

## 7. Performance budget

`render` ≤ 1 ms p95 for 12 pointers + 40 capabilities (pure string work, ≤ 60 catalog lookups served from the in-memory card map).

## 8. Test plan

- **Golden tests** (`tests/route/golden/notes/*.txt`): the three examples in §4.7; wrapping; line-cap truncation with marker; root hint; `ascii_only`; migrations-only schema line; escaped paths.
- **Unit**: `display_path` table from §4.4 plus fuzzed `hypothesis` strings (the output never contains a raw newline, and plain paths round-trip unchanged); `display_capability` for all four types; classification of every `SurfaceType`.
- **Property**: for random selections, `lines ≤ max_lines`, `len(text) ≤ max_chars`, `pointers_shown + pointers_dropped == input pointers`, same input → identical bytes, fallback present in every `FULL`/`CAPS_ONLY` note and absent from every `DELTA` note.
- **Integration** with 09 + 10 on the fixture judge: `s004` sequence produces a full note, no note, a delta note, then a full note.

## 9. Acceptance criteria

1. All golden files match byte for byte; the spec §14.1 example renders exactly as shown there.
2. Property tests in §8 pass with ≥ 1,000 examples.
3. Median note has ≤ 12 pointers on the dev set (spec §1.2), measured from `pointers_shown`.
4. Phase 2 exit contributes a full-note renderer; Phase 4 exit adds delta and capability-only notes.

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-11-1 | §14.2 delta example uses `code: ` without padding | Labels padded to 8 chars in every note kind | One grammar; simpler golden tests |
| D-11-2 | §14 defines full and delta notes only | Adds a capability-only note with its own header | §11.4 / §11.8 require "capability lines only" but give no format; a full-note header ("Likely relevant…") with no files would mislead |
| D-11-3 | §14.3 "≤ 15 lines" with no overflow rule | Wrapping at 160 chars, deterministic drop order (skip → content → use) and a `(+N more not shown)` marker | Needed to enforce the cap |
| D-11-4 | §14.2 delta lists "only additions" | Delta can include newly `use` capabilities but never `skip` lines | Mid-task skip advice is risky and costs more than it saves (spec §11.4 asymmetry) |
| D-11-5 | not specified | Root hint when the agent's cwd is a subdirectory | Relative paths are otherwise ambiguous |
| D-11-6 | §1.1 / §14.1: the note shows "the migration that added `shipped_at`" | The note shows each selected table's latest migration (`last_changed_in`, else `created_in`), at most 2 | Column-level intent isn't knowable without a model; the latest change is a deterministic proxy |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-11-1 | Should items within a line be ordered by score (conveys priority) or alphabetically (easier to scan)? | Score order | Agent-behavior eval: first-opened file matches first pointer? |
| Q-11-2 | Should the delta note re-issue the full list instead of additions (spec §25 Q4)? | Additions only | Sequence eval: recall of files opened on `extends` turns |
| Q-11-3 | Should the note show directory pointers with a file count (`src/fulfillment/ (6 files)`)? | No | Precision/agent-behavior eval |
| Q-11-4 | Header wording ("nothing is preloaded") — does it reduce agents over-trusting pointers? | Keep spec wording | Wording experiment in `eval/wordings.yaml` key `note` |
