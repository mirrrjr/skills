---
name: laravel-backend
description: Senior-level guidance for designing, writing, reviewing, and refactoring Laravel backend code (Laravel 11/12, PHP 8.2+) — HTTP APIs, Eloquent models, migrations, queues and jobs, authorization, validation, transactions, caching, testing, and production readiness. Use this skill whenever the user works on a Laravel project, mentions Eloquent, artisan, migrations, Form Requests, policies, Sanctum, queues, Horizon, Pest/PHPUnit, or asks for a controller, model, service, job, endpoint, or schema in PHP/Laravel — even if they don't say "backend" or "best practices". Also use it for Laravel code review, N+1 hunting, slow-query fixes, and architecture decisions (where should this logic live?).
---

# Laravel Backend (senior discipline)

Write Laravel the way a senior engineer who will be paged for it would: boring, explicit, testable, and safe under retries and concurrency. The framework gives you many ways to do everything; this skill tells you which to pick and why.

## Operating principles

1. **Use the framework's grain.** Prefer built-ins (Form Requests, Policies, API Resources, queues, scheduler, rate limiters, cache locks) over custom machinery. Custom code is a liability you maintain forever.
2. **Invariants live in the database, rules live in the app.** Anything that must *never* be violated (uniqueness, foreign keys, non-null, check constraints) is enforced by the schema. Application validation gives friendly errors; it is not a guarantee, because races exist.
3. **Every write path must survive a retry and a concurrent twin.** Clients retry, queues redeliver, users double-click. Ask "what happens if this runs twice, at the same time?" before shipping.
4. **Thin edges, explicit core.** Controllers translate HTTP to a use case and back. Business rules live in plain PHP classes that you can call from a controller, a job, a command, or a test.
5. **Make the failure path part of the design.** Decide what the caller sees, what gets logged, and what gets retried before writing the happy path.
6. **Measure before optimizing, but never ship an N+1.** Indexes and eager loading are not premature optimization; caching and sharding are.

## Workflow for any non-trivial task

1. **Clarify the contract.** Inputs, outputs, auth, who calls it (HTTP, job, CLI), expected volume. If unknown, state assumptions in one line and proceed.
2. **Sketch the data.** Tables, keys, constraints, indexes, and which queries will run against them. Read `references/database.md` when touching schema or queries.
3. **Place the logic** using the decision table below.
4. **Write the failure and concurrency story** (transactions, locks, idempotency, retries). Read `references/reliability.md` for jobs, events, external calls, and money/stock/counter-style writes.
5. **Implement**, following the conventions below.
6. **Test** the behavior, not the implementation. Read `references/testing.md`.
7. **Run the review checklist** at the bottom before presenting code.

## Where does this logic live?

| Concern | Home |
|---|---|
| Input shape, type coercion, field rules | `FormRequest` (`rules()`, `prepareForValidation()`, `after()` for cross-field checks) |
| "May this user do this to this record?" | `Policy` (via `$this->authorize()`, `Gate`, or route `can:` middleware) — never ad-hoc `if ($user->id === ...)` in controllers |
| Orchestrating one use case (create order, cancel booking) | Action class: `final class PlaceOrder { public function handle(PlaceOrderData $data): Order }` |
| Reusable domain rule on one model | Method on the model, scope, or enum — keep it pure and DB-light |
| Rule spanning several models or external systems | Action/Service class, called from controller, job, or command |
| Output shape | `JsonResource` / `ResourceCollection` — never return models directly |
| Side effects after commit (email, webhook, search index) | Event + queued listener, or dispatched job with `afterCommit` |
| Cross-cutting HTTP concern (locale, tenant, idempotency key) | Middleware |
| Query reuse | Eloquent local scope or query-object; avoid repositories that merely wrap Eloquent |

Do not introduce a "Service" layer by reflex. A controller that calls one Eloquent method and returns a Resource is fine. Add an Action when there is real orchestration, a transaction, multiple side effects, or reuse from more than one entry point.

## Conventions

### HTTP layer
- Controllers: single-purpose, resourceful (`index/show/store/update/destroy`); use invokable controllers for one-off actions. Return Resources with explicit status codes (`201` on create with `Location`, `204` on delete, `422` for validation, `409` for state conflicts, `429` for throttling).
- Validate with Form Requests and use `$request->validated()` (or `safe()`), never `$request->all()` into `create()/update()`. Convert validated data to a DTO (`readonly` class) at the boundary when an Action needs it.
- Version APIs in the route prefix (`/api/v1`) from day one; evolve Resources additively.
- Apply `throttle` limiters via `RateLimiter::for()` in a service provider; key by user ID when authenticated, IP otherwise; use stricter limits on auth, OTP, and password reset.
- Paginate every list endpoint. Prefer `cursorPaginate()` for large or append-heavy tables; use `paginate()` only when the client needs page numbers. Cap `per_page`.
- Consistent error shape. Customize in `bootstrap/app.php` via `withExceptions()` (Laravel 11+): render API errors as JSON with a stable `code`, `message`, and `errors`; report unexpected exceptions, never leak stack traces or SQL.

### Eloquent
- Declare `$fillable` explicitly (or `$guarded = []` only with Form-Request-validated input). Use `casts()` for enums, dates, JSON, encrypted attributes, and money-as-integer.
- Use **backed enums** for finite states; cast them on the model; put transition rules next to the enum or in an Action.
- In `AppServiceProvider::boot()` enable `Model::shouldBeStrict(! app()->isProduction())` (lazy-loading prevention, silently-discarded attributes, missing attributes) so N+1 and typo bugs fail loudly in dev/test.
- Eager load deliberately: `with()`, `withCount()`, `withExists()`, `loadMissing()`. Select only needed columns for heavy lists, but always include keys used by relations.
- Prefer `whereBelongsTo()`, scopes, and `when()` over string-built queries. Never interpolate user input into `whereRaw/orderByRaw`; bind parameters, and whitelist sortable columns.
- Bulk work: `chunkById()` / `lazyById()` for iteration with updates, `upsert()` / `insertOrIgnore()` for bulk writes, `update()` on a query for set-based changes. Remember bulk methods skip model events.
- Use `firstOrCreate/updateOrCreate` only when a unique index backs the lookup columns; otherwise they race.
- Soft deletes only when the business needs restore/audit; remember they interact with unique indexes and cascades.

### Money, time, IDs
- Store money as integer minor units (or `decimal` with a fixed scale), never `float`. Keep currency alongside the amount.
- Store timestamps in UTC; convert at the edges. Use `CarbonImmutable` (`Date::use(CarbonImmutable::class)`).
- Expose ULIDs/UUIDs (or hashids) publicly; keep auto-increment IDs internal to avoid enumeration. Use `HasUlids`/`HasUuids`.

### Configuration & structure
- `env()` only inside `config/*.php`; everywhere else `config()`. Otherwise `config:cache` silently breaks things.
- Bind interfaces to implementations only at real seams (payment gateway, SMS, storage). Wrap third-party SDKs behind a small interface so tests can fake them.
- Keep `routes/` files thin; group by auth and version. Name routes. Use route model binding (`scopeBindings()` for nested resources so `/users/{u}/posts/{p}` cannot cross owners).
- Laravel 11+ skeleton: middleware, exceptions, and routing are configured in `bootstrap/app.php`; there is no `Http/Kernel.php`.

## Security baseline (every endpoint)
- Authentication: Sanctum for first-party SPA/mobile tokens, Passport only if you truly need OAuth2 server flows. Scope tokens with abilities; set expirations; hash nothing yourself.
- Authorization on **every** route that touches a record: policy check or scoped query (`$request->user()->orders()->findOrFail($id)`). IDOR/BOLA is the most common real-world API bug.
- Mass assignment: validated data only. Never let clients set `role`, `is_admin`, `user_id`, `tenant_id`, `status`.
- File uploads: validate mimetypes and size, store outside `public` by default, serve through signed/temporary URLs, never trust the client filename.
- Secrets in env/secret manager, never in the repo; `APP_DEBUG=false` in production; rotate on leak.
- Webhooks from third parties: verify signatures, reject replays (timestamp + stored event ID), respond fast and process in a queue.
- Multi-tenancy: enforce tenant scope in one place (global scope or tenant-aware relation) *and* test cross-tenant access explicitly.

## Review checklist (run before presenting code)
- [ ] Every query on a list path is eager-loaded, bounded (pagination/limit), and has a supporting index.
- [ ] Constraints (unique, FK, not null) back every validation that is about integrity.
- [ ] Multi-write operations are inside `DB::transaction()`; side effects are dispatched after commit.
- [ ] Jobs/listeners are idempotent, have `tries`/`backoff`/`timeout`, and a `failed()` path.
- [ ] Authorization is checked for the specific record, not just "is logged in".
- [ ] No `$request->all()` into models; no raw SQL with interpolated input; no model returned directly from a controller.
- [ ] Money is integer or decimal; times are UTC; enums replace magic strings.
- [ ] Errors: expected failures map to the right HTTP status; unexpected ones are logged with context and not leaked.
- [ ] Tests cover the happy path, validation failure, unauthorized/forbidden, and one concurrency/retry case where relevant.
- [ ] No `env()` outside config; no `dd()/dump()` left behind; code passes Pint (and Larastan/PHPStan at the project's level).

## How to respond
- Match the project's existing conventions and Laravel version before applying these defaults; check `composer.json` and existing code when available.
- When reviewing code, lead with the highest-risk issues (security, data corruption, N+1/perf), then design, then style. Give a concrete fix for each.
- Explain the *why* of non-obvious choices in one sentence; do not lecture on basics.
- When trade-offs are real (Action class vs fat controller, cursor vs offset pagination), pick one, say why, and note when you would choose differently.
- Produce complete, runnable code (imports, `declare(strict_types=1);`, return types) rather than fragments, and include the migration and test when you add a feature.

## Reference files
- `references/database.md` — schema design, migrations on live tables, indexes, N+1, locking, pagination, bulk operations.
- `references/reliability.md` — transactions, queues and jobs, idempotency, outbox/events, external API calls, scheduling.
- `references/testing.md` — Pest/PHPUnit structure, feature vs unit, factories, fakes, what is and isn't worth testing.
- `references/production.md` — caching, performance, observability, deployment, Octane caveats.
