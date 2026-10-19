# Reliability: transactions, jobs, idempotency, external calls

## Contents
- Transactions and after-commit side effects
- Queue jobs
- Idempotency
- Events, listeners, outbox
- Calling external services
- Scheduling

## Transactions and after-commit side effects
- Wrap multi-table writes in `DB::transaction()`. Return values from the closure; don't swallow exceptions inside it.
- Side effects that cannot be rolled back (emails, webhooks, search indexing, queue dispatch) must happen **after commit**, or a rolled-back transaction will still trigger them (or a job will run before the row is visible).
  - Set `'after_commit' => true` on the queue connection, or use `->afterCommit()` on dispatch, or implement `ShouldDispatchAfterCommit` on events/jobs.
  - Listeners: `public bool $afterCommit = true;`
- Don't nest `DB::transaction()` expecting independent rollbacks; nested calls use savepoints. Keep the transaction boundary in the Action, not in repositories/models.
- In tests with `RefreshDatabase`, after-commit hooks behave correctly in Laravel 8.x+ (the test transaction is treated as a root) but verify for your version when asserting dispatches.

## Queue jobs
Defaults every job should have:
```php
final class SyncInvoiceToErp implements ShouldQueue
{
    use Queueable;

    public int $tries = 5;
    public int $timeout = 60;          // must be < queue retry_after
    public bool $failOnTimeout = true;

    public function backoff(): array { return [10, 60, 300, 900]; }

    public function __construct(public readonly int $invoiceId) {}   // IDs, not models/payloads

    public function middleware(): array
    {
        return [(new WithoutOverlapping("invoice:{$this->invoiceId}"))->releaseAfter(30)->expireAfter(300)];
    }

    public function handle(ErpClient $erp): void
    {
        $invoice = Invoice::find($this->invoiceId);
        if (! $invoice || $invoice->synced_at) { return; }   // idempotent guard
        // ...
    }

    public function failed(Throwable $e): void { /* alert, mark state */ }
}
```
- Pass identifiers and re-fetch; serialized models go stale (or `SerializesModels` throws when deleted — handle `deleteWhenMissingModels`).
- `retry_after` (queue config) must exceed job `timeout`, otherwise a job is picked up twice while still running.
- Distinguish retryable from permanent failures: `$this->release($seconds)` / throw for retry; `$this->fail($e)` or `return` for permanent. Use `ThrottlesExceptions` / `RateLimited` middleware for flaky upstreams.
- Separate queues by priority and workload (`critical`, `default`, `low`, `emails`); run Horizon (Redis) with per-queue supervisors and balancing.
- Use `Bus::batch()` / `Bus::chain()` for multi-step workflows; handle `catch` and `finally` callbacks. For long workflows with human or time gaps, store state in the DB rather than relying on chain memory.
- `ShouldBeUnique` + `uniqueId()` prevents duplicate queueing; `ShouldBeUniqueUntilProcessing` when you want re-queue allowed once started. Needs a cache driver that supports locks.
- Restart workers on deploy (`php artisan queue:restart`) and use a process manager (Supervisor/systemd/Horizon).

## Idempotency
Assume at-least-once delivery everywhere (HTTP retries, queue redelivery, webhooks).
- **Natural idempotency**: design operations as "set state to X" rather than "add 1".
- **Idempotency keys for POSTs** that create money/orders: client sends `Idempotency-Key`; middleware stores `(key, user_id, request_hash, response)` with a unique index; replay returns the stored response, same key with different body returns 422/409.
- **Dedup table for consumers**: insert `(source, external_event_id)` with a unique index in the same transaction as the work; on `UniqueConstraintViolationException`, skip.
- Use a unique constraint as the final arbiter, never "check then insert".

## Events, listeners, outbox
- Domain events (`OrderPlaced`) describe facts; listeners react. Queue listeners that do I/O.
- When "write to DB" and "publish message to broker/other system" must both happen, use a transactional **outbox**: write an `outbox_messages` row in the same transaction; a scheduled/queued dispatcher publishes and marks sent (at-least-once; consumers dedupe).
- Avoid chains of events triggering events in unpredictable order; for core flows prefer an explicit Action that calls steps in sequence.
- Model observers hide behavior; use sparingly for truly model-intrinsic concerns (e.g., slug generation). Bulk queries bypass them.

## Calling external services
```php
$response = Http::baseUrl(config('services.erp.url'))
    ->withToken(config('services.erp.token'))
    ->acceptJson()
    ->timeout(5)->connectTimeout(2)
    ->retry([200, 1000, 3000], when: fn ($e) => $e instanceof ConnectionException || $e->response?->serverError(), throw: false)
    ->post('/invoices', $payload);
```
- Always set timeouts. Retry only idempotent calls or calls with an idempotency key; retry only on transient errors (network, 5xx, 429 honoring `Retry-After`).
- Wrap each integration in its own client class returning typed results/DTOs; translate vendor errors into your own exceptions.
- Do external calls from jobs, not request cycles, unless the user must wait for the answer; then keep a strict timeout and a graceful degradation path.
- Circuit-break noisy dependencies (e.g., cache a "down" flag with `Cache::remember` for 30–60 s) to avoid thread pile-ups.
- Log request IDs from vendors; redact secrets and PII.

## Scheduling
- Define schedules in `routes/console.php` (Laravel 11+) using `Schedule::command(...)`.
- Add `->onOneServer()` (cache lock driver required) and `->withoutOverlapping()` for multi-node setups; use `->runInBackground()` for long tasks.
- Scheduled tasks should dispatch jobs rather than doing heavy work inline; each run must be safe to repeat or skip.
- Monitor with a heartbeat (Cronitor/Healthchecks/`->pingOnSuccess()`), otherwise a dead scheduler goes unnoticed.
