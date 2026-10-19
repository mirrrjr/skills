# Security & authentication

## Contents
- Auth architecture
- Authorization
- Sessions and cookies
- Injection and XSS
- CSRF, SSRF, open redirects
- Environment variables and secrets
- Headers and CSP
- Uploads and third parties

## Auth architecture
- Prefer an established library (Auth.js, Better Auth, Clerk, Supabase Auth, Lucia-style patterns) over hand-rolled crypto. Never store passwords unhashed; use Argon2id/bcrypt via the library.
- Session strategy: **httpOnly, Secure, SameSite=Lax (or Strict) cookies** holding an opaque session ID or a signed/encrypted token. Avoid `localStorage` for tokens (XSS-readable).
- **Do not make `middleware`/`proxy` your only gate.** It is useful for optimistic redirects (cheap cookie check) but authorization must be verified where data is read or written. A middleware bypass vulnerability (CVE-2025-29927) showed why: keep the check in the data layer too.
- Centralize in a **Data Access Layer**:
```ts
// lib/auth.ts
import 'server-only';
import { cache } from 'react';
export const getSession = cache(async () => { /* read cookie, verify, load user */ });
export async function requireUser() {
  const s = await getSession();
  if (!s) redirect('/login');
  return s.user;
}
```
```ts
// lib/data/orders.ts
export async function getOrder(id: string) {
  const user = await requireUser();
  const order = await db.order.findFirst({ where: { id, userId: user.id } }); // scoped query = authZ
  return order ? toOrderDTO(order) : null;
}
```

## Authorization
- Check **per resource**, not just "logged in". Scope queries by owner/tenant (`where: { id, userId }`) so IDOR is structurally impossible.
- Role/permission checks in one module (`can(user, 'order:refund', order)`), called from actions, handlers, and data functions. UI hiding is a UX nicety only.
- Return 404 instead of 403 when revealing existence is itself a leak.
- Re-check authorization in Server Actions even if the page already checked: actions are directly callable.
- Rate-limit login, OTP, password reset, and expensive endpoints (Upstash/Redis, Vercel Firewall, reverse proxy), keyed by user and IP.

## Sessions and cookies
- Rotate session ID on login/privilege change; expire and allow revocation (server-side session store or token version).
- `cookies().set` options: `httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge`. Use `__Host-` prefix when possible.
- Sensitive operations (change email/password, payments) require recent re-authentication.
- OAuth/OIDC: validate `state` and PKCE; allow-list redirect URIs; never trust the `next`/`returnTo` param without checking it is a relative path on your origin (open redirect).

## Injection and XSS
- React escapes by default. Dangerous spots: `dangerouslySetInnerHTML`, `href`/`src` with user-supplied `javascript:` URLs, `innerHTML` in effects, Markdown/HTML renderers, SVG uploads, `eval`/`new Function`.
- If you must render user HTML, sanitize with DOMPurify (or a server-side allow-list sanitizer) at render time, and keep a strict CSP as defense in depth.
- Validate URLs against allowed protocols (`https:`, `mailto:`) before putting them in `href`.
- Database access via parameterized queries/ORM only; never build SQL/shell strings from input. Validate every Route Handler/Action input with a schema.

## CSRF, SSRF, open redirects
- Server Actions have built-in Origin/Host checks; for Route Handlers using cookie auth, verify `Origin`/custom header or use SameSite + CSRF tokens for state-changing requests.
- **SSRF**: when the server fetches a user-provided URL (link previews, webhooks, image proxy), allow-list schemes/hosts, block private/link-local IP ranges after DNS resolution, set timeouts and size limits. `next/image` `remotePatterns` should be as narrow as possible.
- Redirect targets must be validated (relative paths or allow-listed hosts).

## Environment variables and secrets
- Only variables prefixed `NEXT_PUBLIC_` are inlined into the browser bundle: treat them as public. They're fixed at **build time**, so don't use them for per-environment runtime config in a single built image.
- Read secrets only in server modules marked `import 'server-only'`; the build then fails if a client component imports them.
- Validate env at startup with zod (`env.ts`) so misconfiguration fails fast.
- Never log tokens, cookies, full request bodies, or PII. Don't pass secrets or full DB rows as props to Client Components or through `generateStaticParams`.
- Use `experimental_taintObjectReference`/`taintUniqueValue` style guards cautiously as a safety net, not a design.

## Headers and CSP
- Set via `next.config` `headers()` or proxy: `Content-Security-Policy` (nonce-based `script-src`, `object-src 'none'`, `base-uri 'self'`, `frame-ancestors`), `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`.
- Nonce-based CSP forces dynamic rendering of pages that include the nonce; weigh against static caching, or use hash-based/`'strict-dynamic'` approaches.
- Cache-Control: `private, no-store` for authenticated/sensitive responses.

## Uploads and third parties
- Validate type by content (magic bytes), size, and count on the server; generate filenames; store outside the web root/in object storage; scan if risk warrants; serve with correct `Content-Type` and `Content-Disposition`.
- Prefer presigned direct uploads with constrained policy (size, type, key prefix).
- Load third-party scripts with `next/script` (`afterInteractive`/`lazyOnload`), pin versions, use Subresource Integrity where static, and audit what they can read (tag managers are an XSS surface).
- Keep dependencies patched (`npm audit`, Dependabot/Renovate), pin lockfile, review new packages before adding (supply-chain risk), and run `npm ci` in CI.
