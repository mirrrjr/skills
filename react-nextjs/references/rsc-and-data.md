# Rendering, data fetching, caching, mutations

## Contents
- Rendering modes
- Fetching patterns
- Caching: Next 15 vs Next 16
- Revalidation
- Server Actions
- Route Handlers
- Loading, error, not-found

## Rendering modes
- **Static** (prerendered at build) is the default when a route uses no request-time APIs. Fastest; use for marketing, docs, mostly-static content.
- **Dynamic** (rendered per request) happens when the route reads `cookies()`, `headers()`, `searchParams`, or uncached data. Fine and often necessary; stream it.
- **ISR/revalidated**: static output refreshed by time or tag.
- **Partial prerendering / Cache Components** (Next 15 experimental → Next 16 via `cacheComponents`): static shell served instantly, dynamic holes streamed in via `<Suspense>`. Prefer this mental model: *static shell + dynamic islands*.
- `export const dynamic = 'force-dynamic'` / `revalidate = N` are blunt route-level tools; prefer data-level caching with tags.

## Fetching patterns
```tsx
// app/orders/[id]/page.tsx
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { getOrder } from '@/lib/data/orders';

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const order = await getOrder(id);     // authz inside the DAL
  if (!order) notFound();
  return (
    <>
      <OrderHeader order={order} />
      <Suspense fallback={<TimelineSkeleton />}>
        <OrderTimeline orderId={id} />   {/* slow part streams in */}
      </Suspense>
    </>
  );
}
```
- Avoid **waterfalls**: start independent requests together (`Promise.all`) or give each its own Suspense boundary so they resolve in parallel. Sequential `await`s are only for true dependencies.
- Wrap non-`fetch` data access in React `cache()` so the same call in layout, page, and children runs once per request.
- Pass promises to Client Components and read with `use()` when the client island needs streamed data.
- Put data access in `lib/data/*.ts` marked `import 'server-only'`; components call functions, not raw ORM queries.
- Select only needed columns and map to DTOs before returning to components; never pass full DB rows (they may carry sensitive fields) to Client Components.

## Caching: Next 15 vs Next 16
Check the installed version first.

**Next 15 (App Router)**
- `fetch` is **not cached by default**; opt in with `fetch(url, { next: { revalidate: 3600, tags: ['orders'] } })` or `cache: 'force-cache'`.
- GET Route Handlers are not cached by default.
- For ORM/DB calls use `unstable_cache(fn, keys, { tags, revalidate })` (or the experimental `use cache` flag).
- Client Router Cache for dynamic pages is not reused by default (`staleTimes` = 0); tune in config if needed.

**Next 16**
- Opt in to **Cache Components** (`cacheComponents: true`); then caching is explicit via the `'use cache'` directive on a function, component, or file. Everything is dynamic unless you cache it.
```ts
import { cacheLife, cacheTag } from 'next/cache';

export async function getProducts() {
  'use cache';
  cacheLife('hours');
  cacheTag('products');
  return db.product.findMany();
}
```
- `middleware.ts` becomes `proxy.ts` (Node runtime). Turbopack is the default bundler. React Compiler support is stable (`reactCompiler: true`).
- Arguments to a `'use cache'` function become part of the cache key; don't read `cookies()`/`headers()` inside (read outside and pass values in).

Rule for both: **tag what you cache, and invalidate by tag on mutation.** Don't cache per-user data in shared caches.

## Revalidation
- `revalidateTag('orders')` / `revalidatePath('/orders')` in Server Actions or Route Handlers after writes. In Next 16, `revalidateTag` takes a cache-life profile (stale-while-revalidate semantics); use `updateTag` in Server Actions when the user must immediately see their own write (read-your-writes), and `refresh()` to refresh uncached dynamic data.
- Prefer tags over paths: one mutation can affect many pages.
- For external changes (CMS, webhooks), expose a secured Route Handler that calls `revalidateTag`; verify a signature or secret.

## Server Actions
```ts
// features/orders/actions.ts
'use server';
import { z } from 'zod';
import { revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';
import { requireUser } from '@/lib/auth';

const Input = z.object({ productId: z.string().uuid(), qty: z.coerce.number().int().min(1).max(20) });

export async function placeOrder(_prev: State, formData: FormData): Promise<State> {
  const user = await requireUser();                       // authN
  const parsed = Input.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors };

  const order = await createOrder(user.id, parsed.data);  // authZ + business rules inside
  revalidateTag('orders');
  redirect(`/orders/${order.id}`);                        // outside try/catch
}
```
- Treat arguments as untrusted input: they are plain HTTP POST bodies. Never trust hidden fields for IDs/prices/roles.
- Return serializable results for expected failures; throw only for unexpected ones (caught by `error.tsx`).
- Actions run sequentially per client; they are for mutations, not for data reads. Use Server Components for reads.
- Make mutations idempotent where retries/double-submits can happen (disable the button with `useFormStatus`, and add server-side uniqueness/idempotency keys for payments/orders).
- Built-in CSRF protection compares `Origin` with `Host`; configure `serverActions.allowedOrigins` if behind proxies/multiple domains.
- Large payloads/uploads: prefer direct-to-storage presigned uploads; adjust `serverActions.bodySizeLimit` only deliberately.

## Route Handlers
- Use for webhooks, third-party consumers, file/stream responses, health checks, and OAuth callbacks. Export named functions (`GET`, `POST`, ...) using Web `Request`/`Response`.
- Validate input (zod), authenticate, set explicit status codes and `Cache-Control`.
- Verify webhook signatures on the raw body (`await req.text()`) before parsing; make processing idempotent (dedupe by event ID) and respond quickly.
- Stream long responses with `ReadableStream`; do not buffer large files in memory.

## Loading, error, not-found
- `loading.tsx` creates an automatic Suspense boundary for a segment; for finer control place `<Suspense>` yourself around slow children.
- `error.tsx` must be a Client Component (`'use client'`) with `error` and `reset`; it catches render/data errors of its segment. `global-error.tsx` replaces the root layout on catastrophic errors.
- Call `notFound()` for missing resources (renders `not-found.tsx`, returns 404); `redirect()`/`permanentRedirect()` for moves; `forbidden()`/`unauthorized()` where enabled.
- Don't swallow framework control-flow errors (`redirect`, `notFound`) in `try/catch`; rethrow with `unstable_rethrow(error)` if you must catch broadly.
- Log server errors with context (request ID, user ID); show users a safe message with a retry path. Error digests in production hide details intentionally.
