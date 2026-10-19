# Production readiness

## Contents
- Caching
- Performance
- Observability
- Deployment
- Octane / long-running processes

## Caching
- Cache computed, expensive, read-heavy data; not rows you can fetch by primary key in 1 ms.
- Use `Cache::remember()` with TTL + jitter; for stampede protection use `Cache::flexible()` (stale-while-revalidate, Laravel 11.23+) or a lock around the rebuild.
- Key naming: `domain:entity:id:version`. Invalidate on write via explicit `Cache::forget` in the Action, or use tags (Redis/Memcached only). Prefer short TTLs over complex invalidation.
- Never cache per-user data under a shared key; never cache authorization decisions longer than the permission change window.
- Use Redis for cache, queues, locks, and sessions in multi-node setups; the `file` and `array` drivers don't give correct locks across servers.
- HTTP caching: ETags/`Cache-Control` for public GETs; `ConditionalGet`-style 304 responses save bandwidth.

## Performance
- Order of attack: (1) query count/N+1, (2) missing/wrong indexes and slow queries, (3) payload size (select columns, paginate, compress), (4) move slow work to queues, (5) cache, (6) scale out, (7) Octane.
- Autoload and cache in prod: `composer install --no-dev --optimize-autoloader`, `php artisan optimize` (config, route, event, view caches), OPcache enabled with `validate_timestamps=0` and preloading when appropriate.
- Avoid heavy work in service providers' `boot()`, global middleware, and Resource constructors; they run on every request.
- Use `DB::listen` with a slow-query threshold in logs; `Model::preventLazyLoading()` outside production; Laravel Pulse/Telescope (non-prod) to find hotspots.
- Streaming and chunking for large responses/exports; set memory/time limits consciously.

## Observability
- Structured JSON logs with a request/correlation ID added by middleware and attached via `Log::withContext()`; include user/tenant IDs but never secrets or full PII.
- Errors to Sentry/Flare/Bugsnag; configure `dontReport` for expected exceptions; make sure queued job failures report with job context.
- Metrics that matter (RED): request rate, error rate, latency p50/p95/p99 per route; queue depth + age of oldest job; failed jobs count; DB connection saturation; cache hit rate.
- Health endpoints: Laravel's `/up` for liveness; a separate readiness check that verifies DB/Redis (not exposed publicly without protection).
- Alert on symptoms (error rate, latency, queue age), not on every exception.
- Audit logging for sensitive actions (who, what, when, from where) in an append-only table.

## Deployment
- Zero-downtime flow: build artifact → run migrations (backward-compatible) → swap release → `queue:restart` → `horizon:terminate` → warm caches.
- Run migrations with `--force` from a single place (one job), never from every node simultaneously.
- Keep `.env` out of images; inject secrets at runtime. `APP_KEY` rotation requires a plan (`APP_PREVIOUS_KEYS` in Laravel 11+).
- Maintenance mode with a secret/bypass for smoke tests; feature flags (Pennant) for risky rollouts so you can ship dark and enable gradually.
- Backups: tested restores, point-in-time recovery for the primary DB, storage versioning for uploads.
- Set trusted proxies/hosts correctly behind load balancers (`trustProxies` in `bootstrap/app.php`), or `request()->ip()` and HTTPS detection will be wrong.

## Octane / long-running processes
The app stays in memory between requests.
- Never store request-specific state in static properties or singletons; avoid injecting `Request`/auth user into singleton constructors (inject resolvers or use `request()` lazily).
- Watch for memory leaks: growing static arrays, listeners registered per request, unclosed resources.
- Reset state in tests with Octane's test helpers; verify third-party packages are Octane-safe.
- Use Octane only after measuring; a well-indexed, N+1-free app on PHP-FPM is often fast enough.
