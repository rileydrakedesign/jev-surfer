# 03 · Schema extraction: migration replay, table and migration cards

**Status:** draft for review
**Spec sections:** §7.5, §6.2, §6.3, §8.3.5 (record shapes), §10.1–10.2 (schema triggers), §21 ("no migrations")
**Depends on:** 00-foundations (F1, F3), 01-discovery (file list, `GlobSet`), 02-cards (`fit_card`, `card_hash`, budgets, sanitization)
**Consumed by:** 04-graph-edges (table names/variants for `schema_ref`; `fk`, `defined_in`, `alias` edge intents), 06-refresh, 09-router (schema root as a walk node)
**Code:** `src/surf/index/extract_schema.py` (+ private submodules `schema_sql.py`, `schema_prisma.py`, `schema_rails.py`, `schema_alembic.py` if the file grows past ~800 lines)

---

## 1. Purpose and scope

Compute the **current** table set of the project from files in the repo (no database connection), and emit:

- `db:*` (schema root), `db:<table>` cards, and `mig:<path>` cards;
- edge intents for 04: `fk` (table → table), `defined_in` (table → migration or snapshot file), `alias` (`code:<path>` ↔ `mig:<path>`, F3);
- per-table search names for 04's schema-reference scan.

| In scope (v1) | Out of scope (v1) |
|---|---|
| Plain SQL / Supabase / Flyway / golang-migrate / dbmate / goose / Drizzle SQL migrations via `sqlglot` | Live introspection (spec §1.4) |
| Prisma schema (+ `prisma/migrations` SQL for provenance) | Django migrations (detected, reported unsupported) |
| Rails `db/schema.rb`, `db/structure.sql`, `db/migrate/*.rb` (provenance) | Knex, Sequelize, TypeORM, Drizzle TS schema |
| Alembic `versions/*.py` (regex) | Functions, triggers, types, enums, indexes, grants as surfaces |
| Views and materialized views, RLS policies | Column comments (`COMMENT ON`) in cards (prose; injection vector) |

## 2. Interfaces

```python
def detect_schema_sources(files: Sequence[FileEntry], cfg: SchemaConfig) -> list[SchemaSource]: ...
def extract_schema(root: RootInfo, sources: Sequence[SchemaSource], cfg: SchemaConfig,
                   cache: ParseCache | None = None) -> SchemaFacts: ...
# rendering lives in cards.py (one renderer module), specified here:
def render_table_card(t: TableFacts, ctx: SchemaRenderCtx) -> Card: ...
def render_migration_card(m: MigrationFacts, ctx: SchemaRenderCtx) -> Card: ...
def render_schema_root(facts: SchemaFacts, ctx: SchemaRenderCtx) -> Card | None: ...
```

Called by `index/build.py` in step 2 of the build order (02 §4.1). `extract_schema` is pure given file contents and the parse cache; it never raises on bad input (only on programmer error).

## 3. Data structures

```python
class SourceKind(StrEnum):
    SQL = "sql"; PRISMA = "prisma"; RAILS_SCHEMA = "rails_schema"; RAILS_MIGRATE = "rails_migrate"
    ALEMBIC = "alembic"; DJANGO = "django"

class SourceRole(StrEnum):
    SNAPSHOT = "snapshot"      # declares the current state (schema.prisma, schema.rb, structure.sql)
    MIGRATIONS = "migrations"  # ordered deltas

class SchemaSource(BaseModel, frozen=True):
    kind: SourceKind; role: SourceRole
    label: str                 # "supabase/migrations", "prisma/schema.prisma" …
    files: list[str]           # repo-relative, in replay order (§4.3)
    dialect: str               # sqlglot dialect name
    precedence: int            # lower wins (§4.2)

class QName(BaseModel, frozen=True):
    schema: str | None         # None = the dialect's default schema
    name: str
    def display(self) -> str: ...          # "orders" | "billing.invoices" | '"a.b"'
    def surface_id(self) -> SurfaceId: ... # "db:" + display()

class Column(BaseModel):
    name: str; type: str | None; pk: bool = False

class ForeignKey(BaseModel):
    name: str | None; columns: list[str]; ref: QName; ref_columns: list[str]

class TableKind(StrEnum):
    TABLE = "table"; VIEW = "view"; MATVIEW = "materialized_view"

class TableFacts(BaseModel):
    qname: QName; kind: TableKind
    columns: list[Column]            # definition order
    columns_known: bool              # False for `SELECT *` views, CTAS we couldn't resolve
    fks: list[ForeignKey]
    policies: list[str]              # sorted
    rls: bool
    depends_on: list[QName]          # views only, sorted
    partitions: int                  # folded partition children
    created_in: str | None           # path of creating migration / snapshot
    changed_in: list[str]            # ordered paths of later migrations that touched it
    renamed_from: list[str]          # older display names, oldest first
    status: Literal["defined", "partial", "external"]
    model_name: str | None           # Prisma model name when different from table name
    source_label: str
    also_defined_by: list[str]       # labels of lower-precedence sources that also define it

class MigrationOp(BaseModel, frozen=True):
    op: Literal["create","alter","drop","rename","policy","view"]
    table: QName; detail: str | None      # e.g. "+shipped_at", "→ order_lines"

class MigrationFacts(BaseModel):
    path: str; source_label: str; order: int; version_key: tuple
    ops: list[MigrationOp]                # replay order; deduped per (op, table, detail)
    statements: int; unparsed: int

class SchemaFacts(BaseModel):
    tables: dict[QName, TableFacts]       # current state after merge
    migrations: list[MigrationFacts]
    sources: list[SchemaSource]
    edge_intents: list[Edge]              # fk, defined_in, alias (04 validates and stores)
    search_names: dict[SurfaceId, list[str]]   # handed to 04 §8.3 variant generation
    report: SchemaReport                  # unparsed statements, warnings (never committed)
```

`ParseCache`: `.surf/cache/index.sqlite` table `schema_ops(path TEXT, content_sha TEXT, extractor_version INT, ops_json TEXT, PRIMARY KEY(path))`. Per-file parsing is cached by content hash; replay itself is always recomputed (cheap).

## 4. Behavior / algorithm

### 4.1 Source detection

If `index.schema.sources` is set, only those globs are used; kind is inferred per glob (`.sql` → SQL; `*.prisma` → Prisma; `schema.rb` → Rails schema; `db/migrate/*.rb` → Rails migrate; `.py` with `alembic` in path or an `alembic.ini` at the glob's ancestor → Alembic). Otherwise auto-detect over `Discovery.files` (excluded files are never sources):

| Precedence | Kind / role | Auto pattern | Dialect |
|---|---|---|---|
| 10 | SQL migrations | `supabase/migrations/*.sql` | postgres |
| 20 | Prisma snapshot | `prisma/schema.prisma`, `prisma/schema/*.prisma`, or `schema.prisma` anywhere at depth ≤ 3 | from `datasource provider` |
| 25 | SQL migrations (Prisma provenance) | `prisma/migrations/*/migration.sql` | same |
| 30 | Rails snapshot | `db/schema.rb`, else `db/structure.sql` (SQL snapshot) | postgres |
| 35 | Rails migrate (provenance) | `db/migrate/*.rb` | — |
| 40 | SQL migrations | `migrations/*.sql`, `db/migrations/*.sql`, `db/migrate/*.sql`, `sql/migrations/*.sql` | cfg |
| 45 | Drizzle SQL | `drizzle/*.sql` (and `drizzle/meta/_journal.json` for order) | from `drizzle.config.*` `dialect` if a literal, else cfg |
| 50 | SQL (loose) | `db/*.sql`, `schema.sql` at root | cfg |
| 60 | Alembic | `alembic/versions/*.py`, `migrations/versions/*.py` with `alembic.ini` present | cfg |
| — | Django | `*/migrations/0*.py` importing `django.db` | reported as unsupported; no tables (spec: model files remain code cards) |

Each matching directory is its own source (two `supabase/migrations` in a monorepo → two sources, ordered by path). `index.schema.dialect` (default `postgres`) is the fallback dialect.

### 4.2 Multiple sources: snapshot vs. migrations

1. **Table set:** if any SNAPSHOT source exists, the table set is the union of snapshot sources (Prisma, `schema.rb`, `structure.sql`). Otherwise, it's the replay of MIGRATIONS sources.
2. **Provenance:** MIGRATIONS sources are always replayed, even when a snapshot defines the table set. Their `created_in`/`changed_in` are attached to snapshot tables by `QName` match (with Prisma `@@map`-resolved names). Snapshot-only tables get `created_in = <snapshot path>`.
3. **Conflicts:** when two sources of the same role define the same `QName`, the lower `precedence` value wins entirely (columns, FKs, policies); the loser's label goes to `also_defined_by`. Policies found by a migration replay are merged into a snapshot table (Prisma and `schema.rb` don't carry RLS).
4. Tables present in a migration replay but absent from the snapshot: dropped from the table set, counted in the report ("migrations define N tables the snapshot doesn't").

### 4.3 Migration ordering

Order is by **file name**, never by mtime or git history (both non-deterministic across clones).

| Pattern (first match on file name, or directory name for `*/migration.sql`) | `version_key` |
|---|---|
| Flyway `V<ver>__desc.sql` | `(0, ints(ver split on . or _))`; `R__` repeatables → `(2, name)`; `U…` undo → skipped |
| leading digits `^(\d+)[_-]` (Supabase timestamps, golang-migrate, dbmate, goose, Drizzle `0001_`) | `(0, (int(digits),))` |
| anything else | `(1, name)` |

Ties → full path. Drizzle: if `drizzle/meta/_journal.json` exists and parses, its `entries[].idx` order wins (files not in the journal go last). Alembic: §4.8. Files named `*.down.sql` or `*_down.sql` are skipped; `*.up.sql` kept.

### 4.4 SQL: splitting and classification

Per file (UTF-8, `errors="replace"`):

1. **Up-section filter.** dbmate `-- migrate:up` / `-- migrate:down` and goose `-- +goose Up` / `-- +goose Down`: keep only up sections. goose `StatementBegin/End` blocks are kept as one statement.
2. **Split** with our own scanner (sqlglot's whole-file parse fails the file on one bad statement): `;` outside single/double quotes, `--` and nested `/* */` comments, `$tag$…$tag$` dollar quotes, and `BEGIN ATOMIC … END`. Comments are removed from the statement text.
3. **Classify** by leading keywords (case-insensitive, after stripping comments):

| Class | Leading tokens | Handling |
|---|---|---|
| DDL of interest | `CREATE [OR REPLACE] [UNLOGGED\|TEMP] TABLE`, `ALTER TABLE`, `DROP TABLE`, `CREATE [OR REPLACE] [MATERIALIZED] VIEW`, `ALTER VIEW … RENAME`, `DROP [MATERIALIZED] VIEW`, `CREATE/ALTER/DROP POLICY`, `SET search_path`, `ALTER TABLE … ENABLE ROW LEVEL SECURITY` | parse (§4.5) |
| Temp tables | `CREATE TEMP[ORARY] TABLE` | ignored |
| Everything else | `INSERT`, `UPDATE`, `CREATE FUNCTION/TRIGGER/INDEX/TYPE/EXTENSION/SCHEMA`, `GRANT`, `DO`, `COMMENT`, `BEGIN`, `COMMIT` … | ignored, counted `skipped_non_ddl` |

### 4.5 SQL: parsing a statement

- `sqlglot.parse_one(stmt, read=dialect)` inside `try`. Accept `exp.Create`, `exp.Alter` (`AlterTable` in older sqlglot), `exp.Drop`; `exp.Command` (sqlglot's fallback for unsupported syntax) goes to the regex fallback.
- **Regex fallback** (always used for `POLICY` statements, which sqlglot parses only as `Command`, and on `ParseError`):
  - `CREATE POLICY <name> ON <qname>`; `DROP POLICY [IF EXISTS] <name> ON <qname>`; `ALTER POLICY <name> ON <qname> RENAME TO <new>`;
  - `CREATE TABLE [IF NOT EXISTS] <qname> (` → table with `columns_known=False` if the body can't be split;
  - `ALTER TABLE [IF EXISTS] [ONLY] <qname> ADD [COLUMN] [IF NOT EXISTS] <col>`, `RENAME TO <new>`, `RENAME [COLUMN] <a> TO <b>`, `DROP [COLUMN] [IF EXISTS] <col>`;
  - otherwise: **unparsed** — logged to the report with `(path, statement index, first 120 chars sanitized and redacted)`, counted on the migration (`unparsed`), and skipped. Never fatal (spec §7.5).
- **Name resolution:** identifiers are normalized per dialect (`sqlglot.optimizer.normalize_identifiers`: Postgres folds unquoted to lower case, quoted kept verbatim). Schema: explicit qualifier, else the file's current `search_path` head (reset to default at the start of each file), else default. Default schema per dialect: `postgres → public`, `mysql → none` (a `db.table` qualifier is treated as schema), `sqlite → main`, `tsql → dbo`, others → `public`. A `QName` whose schema equals the default is stored with `schema=None` (so `public.orders` and `orders` are the same table, id `db:orders`). Three-part names (`db.schema.table`) drop the catalog part.
- Tables in `index.schema.ignore_schemas` are dropped at every operation (§5 default list: Supabase and Postgres internals), except as FK targets (§4.6 external).

### 4.6 Replay semantics

State: `dict[QName, TableFacts]`, plus the current file path for provenance. `touch(t)` appends the current path to `t.changed_in` unless it is `created_in` or already last.

| Statement | Effect |
|---|---|
| `CREATE TABLE q (…)` | New `TableFacts` (columns in order; inline `PRIMARY KEY`, table-level `PRIMARY KEY (…)` mark `pk`; inline `REFERENCES r(c)` and table `FOREIGN KEY (…) REFERENCES r(…)` → `ForeignKey`; missing `ref_columns` → `[]`). If `q` exists: with `IF NOT EXISTS` → no-op + `touch`; without → replace, keep `created_in` of the new file, warn. |
| `CREATE TABLE q AS SELECT …` | Columns from `named_selects` when all projections are named; else `columns_known=False` |
| `CREATE TABLE q (LIKE r …)` | Copy `r`'s columns (not FKs) |
| `CREATE TABLE q PARTITION OF p` | No new table; `p.partitions += 1`; `touch(p)` |
| `ALTER TABLE q ADD COLUMN c t` | Append (no-op if exists and `IF NOT EXISTS`); inline `REFERENCES` → FK; op detail `+c` |
| `ALTER TABLE q DROP COLUMN c` | Remove column; remove FKs whose `columns` contain `c`; detail `-c` |
| `ALTER TABLE q RENAME COLUMN a TO b` | Rename in `columns`, own FKs' `columns`, and **every** table's FKs with `ref == q` in `ref_columns`; detail `a→b` |
| `ALTER TABLE q ALTER COLUMN c TYPE t` | Update type |
| `ALTER TABLE q RENAME TO n` | Move key `q → QName(q.schema, n)` (Postgres semantics: a rename never changes schema); `renamed_from.append(q.display())`; rewrite `ref` in every FK pointing at `q`; policies move with the table; provenance moves with the table (earlier migrations stay linked to the renamed table) |
| `ALTER TABLE q SET SCHEMA s` | Move key to `QName(s, q.name)`; same rewrites as rename |
| `ALTER TABLE q ADD [CONSTRAINT n] FOREIGN KEY …` / `PRIMARY KEY …` | Add FK (named) / mark pk |
| `ALTER TABLE q DROP CONSTRAINT n` | Remove FK named `n`; unnamed FKs get Postgres default names `{table}_{col}_fkey` for matching |
| `ALTER TABLE q ENABLE ROW LEVEL SECURITY` | `rls=True` |
| `ALTER TABLE` on unknown `q` | Create `status="partial"` table with the known effects (e.g. added columns); `created_in = None`; report warning. Covers baselines not in the repo. Not created for ignored schemas. |
| `DROP TABLE [IF EXISTS] a, b [CASCADE]` | Delete; delete FKs in other tables referencing them (Postgres drops them with CASCADE, and refuses without it — either way the resulting state has no dangling FK); unknown table → no-op |
| `CREATE [OR REPLACE] [MATERIALIZED] VIEW v AS …` | `kind=view/matview`; columns from `named_selects` (`*` → `columns_known=False`); `depends_on` = `exp.Table` nodes in the query minus CTE names, resolved to `QName`; replace if exists |
| `ALTER VIEW v RENAME TO n` / `DROP VIEW` | As table rename / drop |
| `CREATE POLICY p ON q` | Add `p` to `q.policies` (set); `touch(q)`; unknown `q` → partial table |
| `DROP POLICY p ON q` / `ALTER POLICY p ON q RENAME TO n` | Remove / rename |
| `SET search_path TO s, …` | Current file's default schema = `s` |

After the last file:
- **External tables:** every FK `ref` or view `depends_on` not in the state becomes a stub `TableFacts(status="external", columns=[], columns_known=False)` — e.g. Supabase `auth.users`, `storage.objects` — including refs into ignored schemas (a stub is the only form those take). Stubs are only created when referenced, so they carry a real signal. `index.schema.external_stubs = false` disables them; FK lines still show the target name.
- `MigrationFacts.ops` records every effect per file (for `mig:` cards and `defined_in` edges).

### 4.7 Prisma (`schema.prisma`)

A small line-oriented parser (spec: "small grammar"), comments `//`/`///` stripped, blocks balanced by braces:

| Construct | Handling |
|---|---|
| `datasource db { provider = "postgresql" }` | dialect (`postgresql→postgres`, `mysql`, `sqlite`, `sqlserver→tsql`, `cockroachdb→postgres`) |
| `model Name { … }` / `view Name { … }` | table / view |
| field line `name Type[?\|[]] @attrs` | scalar type (built-ins and `enum` names declared in the file) → column; type is another model → relation field, not a column |
| `@map("col")` | column name |
| `@id`, `@@id([a,b])` | pk |
| `@relation(fields: [a], references: [b])` on a relation field | FK from this table's `a` to the related model's table |
| `@@map("table")` | table name; else the model name verbatim (Prisma's default; case preserved → id `db:Order`) |
| `@@schema("s")` | schema (multiSchema) |
| `@ignore` / `@@ignore` | column / table skipped |
| `enum`, `type` (composite), `generator` | ignored |

`model_name` is set when `@@map` differs from the model name, and both are in `search_names` so 04 matches `prisma.order.findMany` and raw `"orders"` SQL. Multi-file schemas (`prisma/schema/*.prisma`) are concatenated in path order.

### 4.8 Rails, Alembic

**`db/schema.rb`** (regex over the DSL, block-scoped by `create_table … do |t|` … `end` at the same indent):
- `create_table "name"[, id: false][, primary_key: "x"]` → table; implicit `id` pk column unless `id: false`; `"schema.name"` qualifies.
- `t.<type> "col"` → column (type = DSL type); `t.references/belongs_to "x"` → column `x_id` (+ `x_type` if `polymorphic: true`); FK only via `foreign_key: true` (target = pluralized `x` by a simple `+s`/`y→ies` rule, `to_table:` if present); `t.timestamps` → `created_at`, `updated_at`; `t.index` ignored.
- `add_foreign_key "from", "to"[, column: "c"]` → FK (default column `singular(to) + "_id"`).
- `db/migrate/*.rb` (provenance only): per file, regex for `create_table :x`, `change_table :x`, `add_column :x`, `remove_column :x`, `rename_column :x`, `add_reference :x`, `rename_table :a, :b`, `drop_table :x` → `MigrationOp`s. Order by the 14-digit timestamp prefix.

**Alembic** (`versions/*.py`, regex, `upgrade()` body only — from `def upgrade` to the next top-level `def`):
- `op.create_table('t', sa.Column('c', …, sa.ForeignKey('r.c2')), …, schema='s')` → table; the call's argument list is found by a paren-balancing scan that respects string literals.
- `op.add_column('t', sa.Column('c', …))`, `op.drop_column('t', 'c')`, `op.alter_column('t', 'c', new_column_name='d')`, `op.rename_table('a', 'b')`, `op.drop_table('t')`, `op.create_foreign_key(name, 'src', 'ref', ['a'], ['b'])`, `op.execute("…")` with a string literal → fed to the SQL path.
- **Order:** module-level `revision = '…'` and `down_revision = '…' | ('…','…') | None` form a DAG; topological order with ties broken by file path. Missing parents, cycles or unparseable ids → filename order + report warning.
- Anything else (dynamic names, loops, `batch_alter_table`) → unparsed count; `batch_alter_table('t')` at least `touch`es `t`.

### 4.9 Cards (rendered by `cards.py`)

**Table card** (budget 120, drop ranks: columns 1, policies 2, fk 3; minimums: columns 6, others 2):

```
table orders — 14 columns
columns: id, customer_id, status, total_cents, created_at, shipped_at, +8
fk: customer_id → customers, shipment_id → shipments
policies: orders_select_own, orders_update_admin
defined in: 20250302_init.sql · last changed: 20260611_add_shipments.sql
```

| Variant | Header / extra line |
|---|---|
| view / matview | `view order_totals — 5 columns` / `materialized view …`; extra `from: orders, order_items` |
| `columns_known=False` | `table x — columns unknown` |
| partial | header suffix ` (partial: created outside migrations)` |
| external | `table auth.users — external (referenced, not defined in repo)`; no other lines |
| renamed | extra `renamed from: order_items_old` (last 2) |
| Prisma `model_name` | header `table orders (model Order) — …` |

Columns in definition order; FK line `col → table` (multi-column: `(a, b) → t`); `defined in`/`last changed` use 02 `short_path` of `created_in` / last of `changed_in` (omitted when absent or equal). All names pass control-char stripping only (02 §4.11).

**Migration card** `mig:<path>` (budget 60; `parent = None`; not a walk node):

```
migration supabase/migrations/20260611_add_shipments.sql
creates: shipments · alters: orders (+shipped_at) · policies: shipments
```

Groups in fixed order `creates`, `alters`, `renames`, `drops`, `views`, `policies`; tables sorted within a group; `alters` details capped at 2 per table. Zero ops → `migration <path> (no schema changes)`.

**Schema root** `db:*` (budget 150; `parent = "root:"`; emitted only if ≥ 1 table that isn't external):

```
database schema — 41 tables, 3 views: customers, orders, shipments, products, … +33
```

Tables ordered by table churn (02 §4.7: count of migrations touching it) desc, then FK degree desc, then display name. Tables and views have `parent = "db:*"`; external stubs too.

**Hash inputs** (02 §4.9 function):

| Card | Inputs |
|---|---|
| db_table | `qname, kind, status, columns (name, type, pk), columns_known, fks, policies, rls, depends_on, partitions, created_in, changed_in[-1], renamed_from, model_name` |
| db_migration | `path, ops` |
| schema_root | sorted `[(table_id, table_hash)]` |

**Fields** mirror the facts (`kind`, `status`, `columns` (names), `column_count`, `fks`, `policies`, `rls`, `depends_on`, `created_in`, `last_changed_in`, `churn`, `source`).

### 4.10 Edge intents for 04

| Kind | From → to | Weight | Evidence | Rule |
|---|---|---|---|---|
| `fk` | `db:<table>` → `db:<ref>` | 0.7 (spec §8.3.5) | `{"columns": [...]}` | one per (from, to) pair, columns merged; self-references dropped; target may be an external stub |
| `defined_in` | `db:<table>` → `mig:<path>` | 1.0 for `created_in` and the last `changed_in`; 0.5 for others | `{"op": "create"\|"alter"\|…}` | at most 10 per table: creator + 9 most recent; snapshot sources target `code:<snapshot path>` instead (no `mig:` card for snapshots) |
| `alias` | `code:<path>` → `mig:<path>` | 1.0 | `{}` | one per migration file that is also in the content tree (F3) |

Views additionally get `defined_in` to the view-defining migration. View → base-table dependencies are shown on the card only (Q-03-3). 04 owns validation (dropping intents whose endpoints don't exist) and writing.

`search_names[db:<t>]` = `[name]` plus `model_name` if set; never old names (stale names in code would attach to the wrong table) and never the schema qualifier.

### 4.11 Refresh (handoff to 06)

Any added, removed or modified file in any schema source, or a change to `index.schema.*` config, triggers a full re-detect + replay. Per-file ops come from the parse cache when `content_sha` and `extractor_version` match, so a one-migration commit replays in ≪ 1 s. Table-set changes (added, removed, renamed) drive 04's targeted schema-ref rescan (spec §10.2 step 4).

## 5. Configuration

| Key | Type | Default | Notes |
|---|---|---|---|
| `index.schema.sources` | list[str] \| None | None (auto) | spec §16 |
| `index.schema.dialect` | str | `"postgres"` | spec §16; fallback only |
| `index.schema.enabled` | bool | true | new |
| `index.schema.ignore_schemas` | list[str] | `["pg_catalog","information_schema","supabase_migrations","supabase_functions","extensions","graphql","graphql_public","realtime","_realtime","vault","pgsodium","net","cron","pgbouncer"]` | new |
| `index.schema.external_stubs` | bool | true | new |
| `index.schema.max_statements_per_file` | int | 20000 | new; beyond → rest of file counted unparsed |

## 6. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| No schema sources | No `db:*`, no tables; `doctor` notes "no migrations found" (spec §21) |
| Only Django migrations | Report "Django migrations not supported in v1"; no tables |
| File not valid UTF-8 | Decoded with replacement; parse continues |
| Statement sqlglot can't parse | Regex fallback, else unparsed + report; never fatal |
| Huge `pg_dump` in `structure.sql` | Only DDL of interest parsed; `COPY … FROM stdin` data blocks skipped by the splitter (lines until `\.`) |
| `CREATE TABLE` twice without `IF NOT EXISTS` | Second wins, warning |
| Rename to an existing name | Target overwritten, warning |
| Drop then recreate same name | New table; provenance restarts at the recreating file |
| Circular FKs | Fine; two `fk` edges |
| FK to `auth.users` | External stub `db:auth.users`; FK line `user_id → auth.users` |
| Mixed-case quoted `"Orders"` and unquoted `orders` (Postgres) | Different tables (`db:Orders`, `db:orders`), per Postgres semantics |
| Table name containing `.` | Display quotes it: `db:"a.b"`; `billing."a.b"` if qualified |
| `public.orders` vs `orders` | Same table (`db:orders`) |
| `SET search_path` inside a function body | Inside dollar quotes → not a statement → ignored |
| Prisma `@@map` to a name also defined by SQL migrations | Snapshot wins; migrations supply provenance by the mapped name |
| Prisma relation to a model with `@@ignore` | FK dropped |
| `schema.rb` and `structure.sql` both present | `schema.rb` (precedence 30) wins; `structure.sql` listed in `also_defined_by` |
| Two SQL sources define `orders` differently | Lower precedence value wins; `also_defined_by` set; warning |
| Alembic revision graph with two heads | Topological order, ties by path; warning "multiple heads" |
| Migration file excluded by user `index.exclude` | Not a source (excludes apply first) |
| `mig:` path also a content file | Always (unless the dir is excluded); `alias` edge |
| Partition children (`orders_2024_01`) | Folded into parent; no cards |
| Views on views | `depends_on` lists the view; fine |
| Different sqlglot version | Output may differ; version pinned in lockfile and recorded in `meta.json` (05); `--check` compares under the lockfile version |

## 7. Performance budget

| Workload | Budget |
|---|---|
| Split + classify | ≥ 50k statements/s |
| sqlglot parse of DDL-of-interest | ≈ 1–2 ms/statement → ≤ 5 s for 3k DDL statements (cold) |
| Replay from cached ops | ≤ 200 ms for 1k migrations |
| Refresh after one new migration | ≤ 300 ms total |
| Prisma / schema.rb / Alembic parse | ≤ 100 ms for 300 models |

Cold parses are parallelized per file with a process pool (`min(4, cpu)`) when there are > 50 files; results merged in replay order, so parallelism can't change output.

## 8. Test plan

**Unit (SQL)** — table-driven `(migrations…) → expected tables` cases, each asserting columns, FKs, policies, provenance:
- create / add / drop / rename column (incl. FK `ref_columns` rewrite in *other* tables);
- rename table (FKs retargeted, policies and provenance moved, `renamed_from`);
- drop with and without CASCADE (no dangling FKs), drop-then-recreate;
- `SET SCHEMA`, `search_path`, `public.` normalization, quoted identifiers, dotted names;
- views: named selects, `SELECT *`, CTE exclusion, view rename/drop;
- policies: create/drop/rename, on unknown table (partial), RLS flag;
- partitions, CTAS, `LIKE`;
- external stubs for `auth.users`; `external_stubs=false`;
- splitter: dollar quotes with `;`, nested comments, `BEGIN ATOMIC`, `COPY … \.`, dbmate/goose markers;
- unparsed statements are reported with redacted snippets and never raise.

**Ordering:** Flyway `V2` < `V10`, `R__` last; Supabase timestamps; Drizzle journal overriding names; Alembic DAG with a merge revision and a missing parent.

**Prisma / Rails / Alembic:** grammar fixtures covering every row of §4.7 / §4.8.

**Fixture repos:** `schema-supabase` (20 migrations incl. the spec's worked example: `orders`, `shipments`, `shipped_at` added later, `auth.users` FK, RLS policies), `schema-prisma` (schema + `prisma/migrations`), `schema-rails`, `schema-alembic`, `schema-multi` (Supabase + a stray `db/schema.sql`).

**Golden:** `db:*`, `db:*` children, `mig:*` cards and edge intents for each fixture.

**Property:** random sequences of create/rename/drop/add-column ops on a toy model, applied both to the replayer and to a reference in-memory model → equal final state; replay is independent of parse-cache hits.

## 9. Acceptance criteria

1. Phase 1 exit: complete table sets for both target repos, spot-checked against the live schema by hand (every table present, no dropped tables, renamed tables under their current name).
2. On `schema-supabase`, the worked example holds: `db:orders` card lists `shipped_at`; `defined_in` edges link `db:orders` to the migration that added it; `alias` links its `code:` and `mig:` ids.
3. No schema input can make `surf index` fail; unparsed statements appear in the report and `surf doctor`, with redacted snippets only.
4. Replay output is byte-identical with a cold and a warm parse cache.
5. §7 budgets hold on a 1k-migration synthetic fixture.

## 10. Deviations from the spec and open questions

### Deviations

| Id | Spec says | We do | Why |
|---|---|---|---|
| D-03-1 | "Migrations are replayed in order" | When a snapshot (Prisma, `schema.rb`, `structure.sql`) exists, it defines the table set; migrations only supply provenance | Snapshots are the tool's own statement of current state; replaying Rails/Prisma migrations through regex would be less accurate |
| D-03-2 | `CREATE VIEW` is parsed; no view surface type | Views are `db_table` cards with `fields.kind = "view" \| "materialized_view"` | Agents query views like tables; no new surface type or prefix needed |
| D-03-3 | Silent on unresolved FK targets | External stub cards (`status="external"`) for referenced but undefined tables such as `auth.users` | Supabase apps FK to `auth.users` constantly; tasks about "users" should reach it |
| D-03-4 | `defined_in` weight 1.0 | 1.0 for creator and latest change, 0.5 for others; ≤ 10 per table | Heavily altered tables would otherwise flood expansion with old migrations |
| D-03-5 | Migration order unspecified | File-name version keys (Flyway, timestamps, numeric), Drizzle journal, Alembic revision DAG; never mtime or git time | Deterministic across clones; matches each tool's own ordering |
| D-03-6 | `mig:` parent per F3 is `db:*` "for bookkeeping" | `Card.parent = None` for `mig:` (as 00 §3's `Card` comment says) and no `contains` edge | A `contains` edge would make migrations walk children of `db:*` and crowd the table chunks |
| D-03-7 | Django: "fall back to model file names" | Detection + report only; model files are ordinary code cards | Nothing to extract without a Python-AST pass |

### Open questions

| Id | Question | Proposed default | Decided by |
|---|---|---|---|
| Q-03-1 | Column order on truncated table cards: definition order (spec example) or pk/fk first, then recently added? | Definition order | Eval: final-pass losses on table cards where the relevant column was cut |
| Q-03-2 | Should `COMMENT ON TABLE` text appear on table cards? | No (prose, injection vector) | Adversarial eval + recall on generic table names |
| Q-03-3 | Should views emit edges to base tables (e.g. `fk`-like, weight 0.5)? | No edge in v1; `from:` line on the card | Expansion ablation with vs without |
| Q-03-4 | Monorepo with several independent databases: one `db:*` or one root per source? | One root; conflicts by precedence | Monorepo users (spec §24) |
| Q-03-5 | Should the external stubs list be seeded for Supabase (`auth.users`, `storage.objects`) even when unreferenced? | No, referenced only | Recall on auth-related queries |
