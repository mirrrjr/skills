# Quality, accessibility, SEO, i18n, ops

## Contents
- Testing strategy
- TypeScript practices
- Accessibility
- SEO and metadata
- Internationalization
- Performance budget and Web Vitals
- Observability
- Build and deployment
- Pages Router migration notes

## Testing strategy
Test behavior users (and other systems) depend on; avoid testing implementation details.
- **Unit**: pure functions, schemas, reducers, utilities (Vitest).
- **Component**: Vitest/Jest + React Testing Library; query by role/label/text (`getByRole`), use `userEvent`, assert visible outcomes. Mock network with MSW, not by stubbing `fetch` in each test.
- **Async Server Components** aren't well supported by unit renderers; cover them with e2e/integration tests, or extract logic into plain functions and test those.
- **E2E**: Playwright for critical journeys (sign-in, checkout, create/edit/delete, permissions). Use role/label locators, isolate state (fresh user per test, API/DB seeding), run in CI with traces on failure.
- **Auth matrix**: test guest, wrong-owner, correct-owner on each protected page/action/handler.
- Visual regression (Storybook/Chromatic/Playwright screenshots) for design-system components only.
- Don't chase coverage numbers; do add a regression test with every bug fix. Mock at the edges (network, time), not your own modules.

## TypeScript practices
- `strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` if the team can handle it. Avoid `any`; use `unknown` + narrowing/zod parsing.
- Infer types from schemas (`z.infer`) to keep runtime and compile-time in sync; share schemas between client forms and server actions.
- Model state with discriminated unions; use `satisfies` for config objects; use `as const` for literal tables.
- Typed routes (`experimental.typedRoutes`) and typed `params` (`Promise<{...}>`) help catch navigation bugs.
- Component props: `React.ComponentProps<'button'>`, `React.ReactNode` for children, avoid `React.FC`.
- Don't export giant barrel types; keep types next to their domain.

## Accessibility
Target WCAG 2.2 AA. Most failures are basic.
- Use semantic elements first (`button`, `a`, `nav`, `main`, `ul`, `label`, `table`); ARIA only to fill genuine gaps. A clickable `div` is a bug.
- Every interactive element is keyboard-operable with a visible `:focus-visible` style; logical tab order; skip link; manage focus on route changes, dialogs (trap + restore), and after errors.
- Names: every control has an accessible name; images have meaningful `alt` (or `alt=""` if decorative); icon-only buttons need `aria-label`.
- Color contrast ≥ 4.5:1 text / 3:1 UI; don't use color alone to convey state; respect `prefers-reduced-motion` and `prefers-color-scheme`.
- Announce async changes (`aria-live`, `role="status"`/`alert`) for toasts, form errors, loading results.
- Use proven primitives (Radix, React Aria, Base UI) for dialogs, menus, comboboxes, tabs; test with keyboard, a screen reader smoke test, and axe (`@axe-core/playwright`, `eslint-plugin-jsx-a11y`).
- Touch targets ≥ 24×24 CSS px (44 recommended on mobile); responsive down to 320px; zoom to 200% without loss.

## SEO and metadata
- Use the Metadata API (`export const metadata` / `generateMetadata`) for title, description, canonical, Open Graph, Twitter, robots; `app/sitemap.ts`, `app/robots.ts`, `opengraph-image.tsx`.
- One `<h1>` per page, meaningful heading order, descriptive link text, structured data (JSON-LD via `<script type="application/ld+json">` with escaped output).
- Ensure content that must be indexed is in the initial HTML (Server Components), not behind client-only fetches.
- Canonical URLs for filtered/paginated listings; 404 via `notFound()` for missing content (not a soft-404 200 page).
- `hreflang` alternates for multilingual sites.

## Internationalization
- Use a maintained library (`next-intl` is the common App Router choice) with locale in the route segment (`app/[locale]/...`), ICU message format for plurals/gender, and typed message keys.
- Keep messages out of components' logic: no string concatenation for sentences; pass variables to ICU messages.
- Format dates, numbers, currency, and lists through `Intl`/library with explicit locale and `timeZone`; handle text expansion and RTL (`dir`, logical CSS properties like `margin-inline-start`).
- Ensure the chosen fonts cover required scripts (Latin diacritics, Cyrillic, etc.) and load with `next/font` subsets.
- Add `hreflang`, localized metadata, and a locale switcher that preserves the path.

## Performance budget and Web Vitals
- Targets (75th percentile, field data): LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1.
- LCP: server-render the hero, `priority` on the LCP image with correct `sizes`, preload critical font via `next/font`, avoid client-side-only hero content, avoid render-blocking third-party scripts.
- INP: keep handlers cheap, split long tasks, `startTransition` for heavy updates, virtualize long lists, avoid hydrating huge client trees.
- CLS: reserve space (width/height, `aspect-ratio`), `next/font` for font metrics, skeletons matching final layout.
- JS size: analyze with `@next/bundle-analyzer`; lazy-load heavy, rarely used components (`next/dynamic`, `import()`), replace heavy libs (moment → date-fns/Temporal, lodash → native or per-method imports), avoid shipping server-only deps.
- Use `next build` output (route sizes, first-load JS) in CI as a budget guard; track real-user metrics with `useReportWebVitals`/RUM.

## Observability
- `instrumentation.ts` to register OpenTelemetry/Sentry on the server; capture errors from Server Components, Actions, and Route Handlers (`onRequestError`).
- Structured logs with request ID and user ID (no PII/secrets). Track Web Vitals and key business events.
- Alert on error rate, p95 latency of key routes, failed webhooks/jobs, and Core Web Vitals regression.

## Build and deployment
- Pin Node and package manager versions (`engines`, `packageManager`); commit the lockfile; `npm ci` in CI.
- CI order: install → lint → type-check → unit/component tests → build → e2e against the built app.
- Self-hosting: `output: 'standalone'`, multi-stage Docker (deps → build → minimal runner, non-root user), set `NODE_ENV=production`, health endpoint, graceful shutdown. For multiple instances configure a shared cache handler (Redis) so ISR/`use cache` and tag revalidation stay consistent, and a stable `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` across builds/instances.
- Put a CDN in front; make sure static assets have immutable caching and HTML cache rules match your auth model.
- Build-time `NEXT_PUBLIC_*` values differ per environment → build per environment or load runtime config from a server endpoint.
- Use preview deployments, feature flags, and staged rollouts for risky changes; keep database migrations backward-compatible with the previous deployed frontend.

## Pages Router migration notes
- Migrate incrementally: `app/` and `pages/` can coexist (avoid colliding routes).
- `getServerSideProps`/`getStaticProps` → async Server Components with explicit caching; `getStaticPaths` → `generateStaticParams`; `_app`/`_document` → root `layout.tsx`; `next/head` → Metadata API; `API routes` → Route Handlers; `useRouter` from `next/navigation` in the App Router.
- Move shared providers into a client `Providers` component mounted in the root layout; keep pages as Server Components that pass data down.
- Migrate leaf routes first, verify auth/middleware behavior, and compare Core Web Vitals before and after.
