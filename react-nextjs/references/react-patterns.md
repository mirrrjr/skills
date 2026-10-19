# React patterns: state, effects, forms, performance

## Contents
- Where state lives
- Effects
- React 19 APIs
- Forms
- Custom hooks
- Re-render control and the React Compiler
- TanStack Query
- Hydration pitfalls

## Where state lives (in this order)
1. **URL** (route params, `searchParams`): filters, sort, page, tab, selected item. Update with `<Link>`/`router.replace` (use a helper like `nuqs` for typed params). Server can render from it.
2. **Server** (DB/cache): anything persistent. Re-fetched by Server Components after revalidation.
3. **Form state**: owned by the `<form>`/`useActionState`/react-hook-form until submitted.
4. **Local component state** (`useState`/`useReducer`): ephemeral UI (open/closed, hover, draft text).
5. **Shared client state**: only if two distant client components truly need it. Context for low-frequency values (theme, current user), Zustand/Jotai for frequent updates. Don't put server data in a global store; that's a cache you must keep in sync.

Heuristics: lift state only as high as needed; colocate state with its consumer; use `useReducer` when transitions have rules; model "status" as a union (`'idle' | 'loading' | 'error' | 'success'`) not several booleans.

## Effects
`useEffect` synchronizes React with an **external system** (DOM subscriptions, timers, analytics, websockets). It's not for data flow.

Don't use an effect to:
- compute derived values (compute during render; `useMemo` only if measured),
- reset state when props change (use `key`, or compute from props),
- respond to user events (put logic in the event handler),
- fetch initial page data (Server Component / loader / TanStack Query),
- chain state updates.

When you do use one: complete dependency array (lint rule on), a cleanup function for every subscription/timer/listener, handle race conditions (ignore stale responses via a flag or `AbortController`), and remember Strict Mode runs effects twice in dev to expose missing cleanup. Use `useSyncExternalStore` for subscribing to external stores, `useEffectEvent` (React 19.2) for non-reactive callbacks used inside effects.

## React 19 APIs worth using
- `use(promise | context)`: read a promise (with Suspense) or context conditionally.
- `useActionState(action, initial)`: form state + pending for Server/Client actions.
- `useFormStatus()`: pending state from inside a child of `<form>`.
- `useOptimistic(state, reducer)`: instant UI while the action is in flight; reverts on failure.
- `useTransition` with async functions: mark non-urgent updates and get `isPending`; keeps UI responsive during navigation or heavy updates.
- `ref` as a prop (no `forwardRef`), ref callbacks can return cleanup, `<Context value>` replaces `<Context.Provider>`.
- Document metadata (`<title>`, `<meta>`) and resource hints (`preload`, `preconnect`) can render from components; in Next, prefer the Metadata API.

## Forms
- Start with native `<form action={serverAction}>` + `useActionState`. You get progressive enhancement, no client state, and server validation.
- Need rich client behavior (field arrays, instant per-field validation, wizard steps)? Use react-hook-form with a zod resolver, and **reuse the same zod schema on the server**. Server validation is mandatory regardless.
- UX rules: label every input (`<label htmlFor>`), show errors adjacent and link with `aria-describedby`, `aria-invalid`, disable submit while pending (and guard server-side), preserve user input on error, set `autoComplete`, `inputMode`, `type` correctly, move focus to the first error.
- Don't store form values in global state; don't re-render the whole form on each keystroke (uncontrolled inputs or RHF).

## Custom hooks
- Extract when logic is stateful and reused or when it hides an effect pattern (`useDebouncedValue`, `useMediaQuery`, `useLocalStorage` implemented with `useSyncExternalStore`). Name starts with `use`; returns a stable, minimal API.
- Don't wrap one-liners; don't create "god hooks" that fetch, transform, and manage UI state together.
- Hooks can't run in Server Components; keep server-side helpers as plain async functions.

## Re-render control and the React Compiler
- First make it correct, then measure with React DevTools Profiler / `why-did-you-render` / browser performance panel.
- Real causes of slow UIs: huge lists (virtualize with `@tanstack/react-virtual`), expensive work in render, state held too high (moving it down fixes most re-render storms), context with fast-changing values, unstable keys, layout thrash.
- Fixes in order: move state down / split components → pass `children` through → virtualize/paginate → defer with `useDeferredValue`/`startTransition` → memoize (`memo`, `useMemo`, `useCallback`) at measured hotspots.
- **React Compiler** (Next 16: `reactCompiler: true`; 15: babel plugin) auto-memoizes components and hooks that follow the Rules of React. With it on, delete reflexive memoization in new code, keep the ESLint rules enabled, and don't mutate props/state/refs during render. Keep manual `useMemo`/`useCallback` only where referential stability is semantically required (effect dependencies, third-party APIs).

## TanStack Query (client-side server state)
Use inside client islands when you need polling, infinite scroll, background refetch, optimistic collaboration, or heavy client-driven filtering.
- One `QueryClient` per request on the server, a singleton in the browser; hydrate prefetched data (`HydrationBoundary`) when prefetching in a Server Component.
- Query keys are arrays that fully describe the request (`['orders', { status, page }]`); centralize key factories.
- Set sensible `staleTime` (default 0 refetches aggressively), handle `isPending`/`isError` states, and invalidate by key after mutations.
- Don't duplicate the same data in Query *and* RSC props without a reason.

## Hydration pitfalls
Mismatch errors come from different HTML on server vs first client render.
- Don't read `window`, `localStorage`, `Date.now()`, `Math.random()`, or locale/timezone-dependent formatting during the initial render. Render a placeholder, then update in an effect or use `useSyncExternalStore` with a server snapshot.
- Format dates/numbers with explicit `timeZone` and `locale` (or on the server only).
- Invalid nesting (`<div>` in `<p>`, `<a>` in `<a>`, `<tr>` outside `<tbody>`) causes mismatches.
- Browser extensions can modify the DOM; `suppressHydrationWarning` is a last resort for one element, not a fix.
- Theme flicker: set the theme class via a tiny inline script/`next-themes`, not after hydration.
