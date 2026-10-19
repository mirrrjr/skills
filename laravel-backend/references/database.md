# Database & Eloquent

## Contents
- Schema design
- Migrations on live systems
- Indexes
- N+1 and query shape
- Locking & concurrency
- Pagination & bulk operations

## Schema design
- Use `foreignId('user_id')->constrained()->cascadeOnDelete()` (or `restrictOnDelete()` when children must block deletion). Decide the delete behavior consciously; the default is a bug magnet.
- Put `unique` indexes on natural keys (`email`, `(tenant_id, slug)`, `(user_id, product_id)` for join rows). Validation says "nice error", the index says "never".
- Add `check` constraints (via raw `DB::statement`) for invariants like `quantity >= 0` or `end_at > start_at` where the DB supports it.
- Prefer `string` with an enum cast over DB `enum` columns: adding a value to a native enum is a migration headache.
- `json` columns are for genuinely schemaless payloads (webhook bodies, settings). If you filter or join on it often, it wants real columns.
- Tenant tables: include `tenant_id` in every composite unique/index, leading with `tenant_id` when queries always filter by it.
- Use `timestamps()` and `softDeletes()` deliberately. Soft-deleted rows still occupy unique indexes; use partial indexes (PostgreSQL) or include `deleted_at` in the unique key.

## Migrations on live systems
Assume the old code and new code run side by side during deploy.
1. **Additive first**: add nullable column / new table / new index. Deploy.
2. **Backfill** in a queued job or command with `chunkById()`, not inside the migration for big tables.
3. **Switch code** to read/write the new shape. Deploy.
4. **Tighten** (set NOT NULL, add FK) once data is clean.
5. **Remove** the old column in a later release.

Rules:
- Never rename or drop a column in the same release that stops using it.
- Large-table index creation: PostgreSQL `CREATE INDEX CONCURRENTLY` (set `public $withinTransaction = false;` on the migration). MySQL: use online DDL (`ALGORITHM=INPLACE, LOCK=NONE`) or pt-osc/gh-ost for huge tables.
- Keep `down()` honest or explicitly irreversible; never edit a migration that has run in shared environments. Write a new one.
- Don't use model classes inside migrations (they change over time); use `DB::table()`.

## Indexes
- Index columns used in `WHERE`, `JOIN`, `ORDER BY` of hot queries; put equality columns first, then range/sort columns. `(status, created_at)` serves `where status = ? order by created_at`.
- Check with `EXPLAIN` (`->explain()` or `DB::listen`) on realistic data volume. An index that the planner ignores is just write overhead.
- Foreign keys are not automatically indexed on every engine; MySQL/InnoDB indexes them, PostgreSQL does not. Add the index.
- Avoid `LIKE '%term%'` on big tables; use full-text, trigram (`pg_trgm`), or a search engine (Scout + Meilisearch/Typesense).

## N+1 and query shape
- Turn on `Model::preventLazyLoading()` (or `shouldBeStrict()`) outside production.
- Eager load nested relations (`with('items.product')`), constrained loads (`with(['comments' => fn ($q) => $q->latest()->limit(5)])` — limit per-parent needs a window function or separate query; `->limit()` inside `with` limits globally).
- Aggregates: `withCount`, `withSum`, `withExists` instead of loading collections to count them.
- Don't query in loops, Resources, accessors, or Blade. Resources should use `whenLoaded()` so they never trigger lazy loads.
- `exists()` instead of `count() > 0`; `pluck()` / `value()` instead of hydrating models you don't need; `toBase()` for read-only heavy rows.
- Use `DB::listen` or Telescope/Debugbar/Pulse in dev to count queries per request; add a feature-test assertion (`DB::enableQueryLog()` + count) for critical endpoints.

## Locking & concurrency
Pick the lightest tool that is correct.

| Situation | Tool |
|---|---|
| Counter / balance / stock decrement | Atomic SQL: `->where('stock', '>=', $n)->decrement('stock', $n)` and check affected rows |
| Read-modify-write on one row | `DB::transaction(fn () => Model::lockForUpdate()->findOrFail($id))` |
| Prevent duplicate create | Unique index + catch `UniqueConstraintViolationException` |
| Concurrent edits by users | Optimistic: `version` column, `update ... where version = ?`, 409 on 0 rows |
| Cross-process critical section | `Cache::lock('key', 10)->block(5, fn () => ...)` (Redis/DB driver, not `file`/`array`) |
| Queue job must not overlap | `WithoutOverlapping` middleware, `ShouldBeUnique` |

Keep transactions short: no HTTP calls, no sleeps, no queue waits inside. Acquire locks in a consistent order to avoid deadlocks, and retry deadlocks with `DB::transaction($cb, attempts: 3)`.

## Pagination & bulk operations
- `cursorPaginate()` needs a deterministic, indexed order (unique tiebreaker like `id`).
- Deep `OFFSET` is O(n); don't expose `page=50000`.
- `chunkById(1000)` when mutating rows you iterate; plain `chunk()` skips/duplicates rows if the filter column changes.
- `lazyById()` for streaming reads/exports with low memory; combine with `LazyCollection` and stream responses (`response()->streamDownload`).
- `upsert($rows, uniqueBy: [...], update: [...])` for batch syncs; wrap in transaction when atomic.
- Exports/imports of big files go to a queued job writing to storage, then notify with a signed URL.
