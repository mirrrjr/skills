---
name: react-nextjs
description: Senior-level guidance for building, reviewing, and refactoring React 19 and Next.js (App Router, v15/16) applications in TypeScript — Server vs Client Components, data fetching and caching, Server Actions, forms, state management, performance, auth and security, accessibility, testing, and deployment. Use this skill whenever the user works with React, Next.js, JSX/TSX, hooks, app/ directory, route handlers, middleware/proxy, Server Actions, SSR/SSG/ISR, TanStack Query, Tailwind/shadcn components, or asks for a component, page, hook, layout, form, or frontend architecture decision — even if they only say "frontend", "UI", or "my page is slow/re-rendering". Also use for React/Next.js code review, hydration errors, caching confusion, bundle-size problems, and migrations from the Pages Router.
---

# React & Next.js (senior discipline)

Write frontend code that is fast by default, correct under async and concurrency, accessible, and easy to delete. The App Router gives several overlapping ways to do everything; this skill says which to pick and why.

## Operating principles

1. **Server first, client by exception.** A component is a Server Component until it needs state, effects, event handlers, or browser APIs. Ship less JavaScript instead of optimizing more JavaScript.
2. **Push `"use client"` to the leaves.** The directive marks a *boundary*: everything imported below it joins the client bundle. Keep interactive islands small and pass server-rendered content in via `children`/props.
3. **Treat the server as the source of truth.** Fetch where the data is needed, on the server, close to the source. Reach for client-side server-state libraries only for genuinely client-driven data (polling, infinite scroll, optimistic collaboration).
4. **Derive, don't sync.** If a value can be computed from props, state, or the URL, compute it. Every `useEffect` that sets state from other state is a bug waiting to happen.
5. **The URL is state.** Filters, tabs, pagination, and search belong in the URL (searchParams / route segments) so they are shareable, back-button-friendly, and server-renderable.
6. **Every Server Action and Route Handler is a public HTTP endpoint.** Authenticate, authorize, and validate inside each one. UI hiding is not security.
7. **Types are the contract.** Strict TypeScript, runtime validation (zod/valibot) at every trust boundary, types inferred from schemas.
8. **Check the installed version before applying advice.** Caching and request APIs changed between Next 14, 15, and 16. Read `package.json` and match the project; if unsure of an API, check the official docs rather than guessing.

## Workflow for non-trivial tasks

1. **Clarify the unit of work**: route, data needed, who may see it, what mutates, what must be interactive, SEO needs.
2. **Decide the rendering model** with the table below; then the data and cache strategy (`references/rsc-and-data.md`).
3. **Draw the component tree** and mark client boundaries. Keep the client islands minimal.
4. **Design state** (`references/react-patterns.md`): URL → server → form → local → shared client, in that order.
5. **Handle the unhappy paths up front**: loading (`loading.tsx`/Suspense), empty, error (`error.tsx`), not-found, unauthorized, offline/slow.
6. **Implement** following conventions; **verify** with the checklist at the bottom.
7. For auth, input handling, or anything user-generated, read `references/security-auth.md`. For tests, a11y, tooling, i18n, and deploy, read `references/quality-and-ops.md`.

## Server or Client Component?

| Need | Choice |
|---|---|
| Fetch data, read env secrets, access DB, render static/mostly-static UI | Server Component |
| `useState`, `useReducer`, `useEffect`, refs, event handlers, `onClick` | Client Component |
| Browser-only APIs (`window`, `localStorage`, IntersectionObserver), most third-party UI libs with hooks | Client Component (wrap the lib once in a small client file) |
| Context provider | Client Component; mount it as low in the tree as practical, pass server children through |
| Interactive widget inside a server-rendered page | Small client island; keep data props serializable |
| Code that must never reach the browser (DB client, secrets) | Mark module with `import 'server-only'` |

Props crossing server → client must be serializable (no functions except Server Actions, no class instances, no Dates you forgot are converted to strings in JSON). Passing a Server Component as `children` into a Client Component is allowed and the standard way to avoid bloating the client.

## Where does this logic live?

| Concern | Home |
|---|---|
| Reading data for a page | `async` Server Component calling a data-access function (`lib/data/*.ts`, `server-only`) |
| Mutation from a form/button in your own app | Server Action (`'use server'`), validated + authorized |
| Public/third-party HTTP API, webhooks, file streaming | Route Handler (`app/api/**/route.ts`) |
| Cross-route redirects/rewrites/headers/geo, cheap checks | `middleware.ts` (Next ≤15) / `proxy.ts` (Next 16); never as the *only* auth gate |
| Authorization of a specific record | Data-access layer, next to the query |
| Reusable stateful logic | Custom hook (`useX`) in the client tree |
| Shared UI primitives | `components/ui/*` (design-system level, no data fetching) |
| Feature-specific UI + logic | Colocate in the route segment or `features/<name>/` |
| Client-driven async data (polling, infinite scroll, optimistic) | TanStack Query in a client island |

## Conventions

### Project structure
- Colocate by feature; share only what is truly shared. Prefer `app/(group)/feature/_components/` and `features/<name>/` over a giant global `components/`.
- Absolute imports via `@/`. No barrel files (`index.ts` re-exports) in hot paths: they defeat tree-shaking and slow dev builds.
- One component per file, named export for components (default export only where the framework requires: `page`, `layout`, `error`, etc.).
- Strict TS (`strict`, `noUncheckedIndexedAccess`), ESLint with `eslint-plugin-react-hooks` (+ React Compiler rules if enabled), Prettier, `tsc --noEmit` in CI.

### Components
- Props: small, typed, explicit; use discriminated unions for variants instead of boolean piles (`variant: 'link' | 'button'` with conditional props).
- Prefer composition (`children`, slots, compound components) over configuration props and prop drilling.
- Accept `className` and spread remaining props (`React.ComponentProps<'button'>`) in UI primitives. React 19: `ref` is a normal prop; `forwardRef` is no longer needed for new code.
- Stable, meaningful `key`s (IDs, never array index for reorderable lists). Changing `key` is the clean way to reset component state.
- No components defined inside other components (they remount every render).
- Keep render pure: no mutations, no `Date.now()`/`Math.random()` in render for SSR content (hydration mismatch), no side effects.

### Data & caching (summary; details in reference)
- `params` and `searchParams` are **async** in Next 15+: `const { id } = await params`. Same for `cookies()` and `headers()`.
- Parallelize independent fetches (`Promise.all`) and stream slow parts with `<Suspense>` instead of blocking the whole page.
- Deduplicate within a request with React `cache()` for non-`fetch` data access.
- Be explicit about caching: tag data and revalidate on mutation. Never rely on remembering the framework default.

### Mutations
- Validate input with zod; return a typed result (`{ ok: true, data } | { ok: false, fieldErrors }`) rather than throwing for expected failures.
- After a mutation: revalidate the affected tags/paths, then `redirect()` (call it outside `try/catch`; it works by throwing).
- Use `useActionState` for form state and errors, `useFormStatus` for pending UI inside forms, `useOptimistic` for instant feedback; progressive enhancement comes free when you use `<form action={...}>`.

### Styling & UI
- Tailwind + a headless primitive layer (Radix/shadcn/Base UI) beats bespoke widgets for accessibility. Don't hand-roll dialogs, comboboxes, menus, tooltips.
- `next/image` for images (set `sizes`, `priority` only for the LCP image), `next/font` for fonts (no layout shift), `next/link` for navigation, `next/script` for third-party scripts with the right `strategy`.

## Review checklist (run before presenting code)
- [ ] `"use client"` only where needed and as deep in the tree as possible; no server-only code or secrets reachable from client modules.
- [ ] Props crossing the boundary are serializable; nothing sensitive is passed to a Client Component (it ends up in the HTML/RSC payload).
- [ ] No data fetching in `useEffect` when a Server Component or Server Action would do; no state that duplicates props/URL.
- [ ] Every Server Action/Route Handler: authenticates, authorizes the specific resource, validates input, handles errors.
- [ ] Loading, empty, error, and not-found states exist for each async segment; Suspense boundaries are placed where they help.
- [ ] Caching strategy is explicit (tags/revalidate/`use cache`/dynamic) and mutations invalidate it.
- [ ] No hydration hazards (browser-only values or time/random in initial render; invalid HTML nesting).
- [ ] Lists have stable keys; no components defined inside components; effects have correct dependencies and cleanup.
- [ ] Forms have labels, errors tied to inputs (`aria-describedby`), keyboard operability, visible focus.
- [ ] Types are strict; no `any`/unchecked `as`; external data validated at the edge.
- [ ] Bundle impact considered: heavy libs lazy-loaded (`next/dynamic`/`import()`), no barrel imports, images/fonts optimized.
- [ ] Tests cover the behavior that matters (user-visible flow, auth rules, validation); lint, type-check, and tests pass.

## How to respond
- Match the project's existing conventions, Next/React versions, and libraries before applying these defaults.
- In code review, lead with correctness/security/data-leak issues, then performance and UX, then style. Give concrete fixes.
- Provide complete, runnable TSX (imports, types, file path as a comment on the first line, e.g. `// app/orders/page.tsx`).
- Explain non-obvious choices in a sentence; when a trade-off is genuine (Server Action vs Route Handler, TanStack Query vs RSC), choose one, justify, and say when you'd switch.
- Don't add memoization (`useMemo`/`useCallback`/`memo`) speculatively. Measure first; with the React Compiler enabled it is mostly unnecessary.

## Reference files
- `references/rsc-and-data.md` — rendering modes, streaming, fetching and caching (Next 15 vs 16), revalidation, Server Actions, route handlers, error/loading boundaries.
- `references/react-patterns.md` — state placement, effects, forms, custom hooks, performance and re-render control, React 19 APIs, React Compiler, TanStack Query.
- `references/security-auth.md` — authN/authZ placement, data-access layer, cookies/sessions, XSS/CSRF/SSRF, env vars, headers/CSP, uploads.
- `references/quality-and-ops.md` — testing, TypeScript, accessibility, SEO/metadata, i18n, observability, build/deploy, migration notes.
