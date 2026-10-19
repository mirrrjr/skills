# Testing

## Philosophy
Test behavior at the boundary users and other systems rely on. Coverage percentage is not the goal; a bug fix without a regression test is unfinished.

Worth testing: authorization rules, validation edge cases, state transitions, money/stock math, idempotency/retries, jobs' effects, query counts on hot endpoints, regressions.
Not worth testing: framework internals, getters/setters, that a mock was called when the mock is the only thing under test, snapshot dumps of whole JSON responses nobody reviews.

## Structure (Pest preferred; PHPUnit equivalent)
- **Feature tests** (HTTP → DB): the default. Hit real routes with `actingAs()`, assert status, JSON structure, and database state (`assertDatabaseHas`, `assertDatabaseCount`).
- **Unit tests**: pure logic (value objects, pricing calculators, enums, DTO mappers) with no framework boot.
- **Integration tests** for external adapters using `Http::fake()` sequences or contract tests against sandbox environments, kept out of the default CI path if slow.

```php
it('rejects placing an order for someone else\'s cart', function () {
    $user  = User::factory()->create();
    $cart  = Cart::factory()->for(User::factory())->create();

    $this->actingAs($user)
        ->postJson("/api/v1/carts/{$cart->ulid}/checkout")
        ->assertForbidden();

    expect(Order::count())->toBe(0);
});

it('is idempotent for the same key', function () {
    $user = User::factory()->hasCart()->create();
    $headers = ['Idempotency-Key' => 'abc-123'];

    $first  = $this->actingAs($user)->postJson('/api/v1/orders', $payload, $headers)->assertCreated();
    $second = $this->actingAs($user)->postJson('/api/v1/orders', $payload, $headers)->assertCreated();

    expect($second->json('data.id'))->toBe($first->json('data.id'))
        ->and(Order::count())->toBe(1);
});
```

## Practices
- `LazilyRefreshDatabase` (or `RefreshDatabase`) with a real engine matching production (SQLite in-memory is fast but hides PostgreSQL/MySQL differences: JSON operators, locking, collation, constraint behavior). Run CI against the production engine.
- Factories: sensible defaults, named states (`->suspended()`), relationships via `for()`/`has()`. Keep tests explicit about the fields that matter to the assertion.
- Fakes: `Queue::fake()`, `Event::fake([...])`, `Mail::fake()`, `Notification::fake()`, `Http::fake()`, `Storage::fake()`, `Bus::fake()`. Fake at the edge; assert the *contract* (what was dispatched with which data), then separately test the job's `handle()` directly.
- Time: `$this->travelTo()` / `Carbon::setTestNow()`; never `sleep()`.
- Query count guard for hot endpoints:
```php
DB::enableQueryLog();
$this->getJson('/api/v1/orders')->assertOk();
expect(count(DB::getQueryLog()))->toBeLessThanOrEqual(6);
```
- Authorization matrix: for each protected route test guest (401), wrong user/tenant (403/404), correct user (2xx).
- Validation: data providers/datasets for each invalid field; assert `assertJsonValidationErrors`.
- Concurrency-sensitive code: test the atomic SQL path (affected rows) and the unique-violation path directly; true parallel tests are rare, so make the logic *deterministic* by design.
- Parallel runs: `php artisan test --parallel`; ensure tests don't share files, caches, or fixed IDs.
- Static analysis: Larastan at level 6+ (aim higher on new code), Pint for style, Rector optional. They catch more bugs per minute than most unit tests.
- TDD where it pays: bug fixes (write failing test first), pricing/rules engines, state machines.
