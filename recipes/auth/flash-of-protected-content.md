---
recipe_id: flash-of-protected-content
track: auth
primary_concept: ecosystem/routing-react-router
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/routing-react-router
  - ecosystem/data-fetching-tanstack-query
  - server/ssr-and-hydration
  - recipes/auth/refresh-storm-on-401
  - recipes/auth/logout-leaves-stale-cache
status:
  drafted: true
  reviewed: false
---

# The protected page flashes before the redirect

> **What you'll build:** a route guard that never flashes — no frame of protected content before a redirect, no bouncing an authenticated user through `/login` on reload — by modeling auth as three states (loading / authenticated / unauthenticated) and, in React Router data mode, deciding in a loader *before* the protected component ever mounts. Plus preserving the intended destination so the post-login redirect lands where the user meant to go.

## The scenario

A billing dashboard. The user has a valid session — access token in memory, refresh token in an httpOnly cookie. They hard-reload `/billing/invoices`, or open a bookmarked deep link. On first paint the app doesn't yet know who they are: the in-memory access token was wiped by the reload and a silent refresh is in flight. The route guard runs, sees `user == null`, and does one of two ugly things:

- **Flash of protected content.** The guard renders `<Invoices>` optimistically while auth resolves, so a frame — or 300ms on a slow refresh — of real invoice data and account PII paints. Then, if auth resolves to *unauthenticated*, it yanks it away and redirects. Sensitive data was shown to someone who may not be allowed to see it.
- **Wrongful redirect + bounce.** The guard treats `user == null` as *unauthenticated* and redirects to `/login`. The silent refresh then succeeds, the app learns the user *is* authenticated, and bounces them back to `/billing` — a login-page flash and a lost deep link for someone who was never logged out.

Root cause: the guard makes a **binary** decision — authed or not — when auth actually has **three** states: `loading` (we don't know yet), `authenticated`, `unauthenticated`. Collapsing `loading` into either decided branch *is* the flash. A `null` user on cold start isn't "logged out"; it's "not yet known."

Why it escaped QA: in dev, auth resolves in ~2ms (a local `/me`, no network), so the `loading` window is a single frame nobody sees; tests log in through the UI and navigate forward (client-side, token already in memory) so the guard never encounters the `loading` state. The flash needs a **cold start on a protected URL** — hard reload, bookmark, new tab, shared link — over real network latency, which forward-driving QA never exercises.

## Walkthrough

### Stage 1 — Name it: auth is three states, not a boolean

Model it as a discriminated union, not `user | null` ([thinking-in-react: model states, not booleans](../../foundations/thinking-in-react.md#model-states-not-booleans)):

```ts
type AuthState =
  | { status: "loading" }
  | { status: "authenticated"; user: User }
  | { status: "unauthenticated" };
```

While `loading`, the guard renders neither the content nor a redirect — a neutral pending UI. Only once `authenticated` or `unauthenticated` does it decide. A `null`-check guard has no way to express "don't know yet," so it's forced to guess — and every guess is a flash.

### Stage 2 — The SPA guard: gate on the three states

```tsx
import { Navigate, useLocation } from "react-router";
import type { ReactNode } from "react";

export function RequireAuth({ children }: { children: ReactNode }) {
  const auth = useAuth(); // returns AuthState
  const location = useLocation();

  if (auth.status === "loading") {
    return <FullPageSpinner />; // neither content nor redirect
  }
  if (auth.status === "unauthenticated") {
    return <Navigate to="/login" replace state={{ from: location }} />;
  }
  return <>{children}</>; // authenticated — safe to render
}
```

Three states, three branches, zero flash. `<Navigate>` redirects *during render* (not in an effect — see the pitfalls), and `state={{ from: location }}` preserves the intended destination so login can send the user back.

Do **not** redirect in a `useEffect`: `useEffect(() => { if (unauth) navigate("/login"); })` renders the component once *before* the effect runs — that render is the flash. Decide in render.

### Stage 3 — The flash-free default in React Router data mode: decide in a loader

A component guard still mounts the route tree and renders a spinner. In [React Router data mode](../../ecosystem/routing-react-router.md#real-world-patterns) you can decide *before* the protected component exists — the loader runs during navigation, and a `redirect` thrown there means the component never mounts:

```tsx
// routes/billing.tsx
import { redirect, type LoaderFunctionArgs } from "react-router";

export async function loader({ request }: LoaderFunctionArgs) {
  const auth = await resolveAuth(); // awaits the in-flight refresh / /me, once
  if (auth.status !== "authenticated") {
    const next = new URL(request.url).pathname;
    throw redirect(`/login?next=${encodeURIComponent(next)}`);
  }
  return { user: auth.user };
}
```

No protected component renders until auth resolves, and an unauthenticated user is redirected *before* the route commits — the flash is gone by construction, not patched over. The loader blocks the navigation, so there's no half-rendered intermediate state either. Put the check on a parent **layout** route so every child inherits it — one loader, not one per page.

### Stage 4 — Bootstrap once, and land where they meant to go

The whole thing depends on `resolveAuth()` answering exactly once per cold start:

- **httpOnly cookie + SSR.** The server reads the cookie and renders the correct authenticated shell — the client hydrates with auth *already known*, so there's no client-side `loading` window at all (and no [hydration mismatch](../../server/ssr-and-hydration.md#the-agreement-contract-and-why-mismatches-are-expensive) from server-says-out / client-says-in).
- **SPA.** Gate the *whole app* behind a one-time bootstrap (silent refresh or `/me`) that resolves `AuthState` out of `loading` before any route renders — a single splash, not a per-route spinner.
- **Return trip.** Login reads `?next` (or `location.state.from`) and redirects there on success, so the deep link isn't lost.

**Verify the loop.** Reproduce the cold start three ways: (1) hard-reload `/billing/invoices` while authenticated → a brief splash, then the page — no `/login` flash, no bounce; (2) hard-reload it while logged out → straight to `/login`, zero frames of invoice data; (3) open the deep link logged out → `/login`, then land on `/billing/invoices` after signing in. The flash is gone in all three because nothing decides until auth is known.

## Variations

1. **Component guard vs loader guard.** The loader (data mode) is flash-free by construction and the default; the `RequireAuth` component is the fallback when you're not on data-mode routes.
2. **Preserve the destination.** `?next=` in the URL (survives a full reload) or `location.state.from` (SPA nav); login redirects there on success.
3. **SSR/httpOnly for zero client `loading`.** The server reads the cookie and renders authenticated — no client flash and no hydration mismatch.
4. **Authorized ≠ authenticated.** A logged-in user hitting a page they lack permission for is a *403*, not a login redirect — add a fourth `forbidden` branch that renders a 403 page; don't bounce them to `/login`.
5. **Refresh-in-progress isn't logged-out.** During a silent refresh ([refresh-storm's single-flight](./refresh-storm-on-401.md#stage-3--single-flight-refresh-with-a-shared-promise)), `resolveAuth()` should *await* the in-flight refresh, not read the momentarily-absent token as `unauthenticated` — otherwise a mid-session token expiry flashes a redirect.

## Trade-offs and common pitfalls

1. **Boolean auth guard** — the core bug. `user | null` can't express "don't know yet," so `loading` collapses into a decided branch and flashes.
2. **Treating `null` user as unauthenticated** — redirects authed users on every reload, then bounces them back. `null` on cold start means *loading*, not *logged out*.
3. **Rendering children optimistically while loading** — a frame of protected PII paints before auth resolves. Render a pending UI, never the content, until `authenticated`.
4. **Redirecting in `useEffect`** — the component renders once before the effect fires; that first render is the flash. Redirect in render (`<Navigate>`) or in a loader.
5. **Losing the intended destination** — no `?next`/`from`, so login dumps everyone on the dashboard and the deep link is gone.
6. **Per-route guards instead of a layout/loader** — duplicated `loading` flashes and auth waterfalls; check once on a parent route.
7. **SSR renders unauth, client is authed** — a hydration mismatch *and* a flash. Read the cookie server-side so the shell is correct on first paint.
8. **Auth state not ready on first render** — a token read from async storage, or a store hydrated after mount, guarantees a `loading` first render; make sure `loading` is handled, not silently treated as unauth.
9. **Redirect loop** — `/login` itself sits behind the guard, or the guard sends authed users away from `/login` incorrectly; exempt the auth routes.
10. **Flashing a redirect during a token refresh** — a mid-session expiry triggers a silent refresh; `resolveAuth` must await it, not read the gap as logged-out.
11. **Sending permission errors to `/login`** — a 403 (authenticated, not allowed) redirected to login makes an authed user re-authenticate pointlessly and can loop. 403 ≠ 401.
12. **A pending UI that shifts layout** — a spinner a different size than the content causes a CLS jump when it resolves; use a stable skeleton.

### When NOT to build a guard

A fully public route with no protected content and no personalization needs no auth gating — don't wrap public pages in `RequireAuth` ceremony. And if auth is *always* resolved before the client's first render — classic SSR where the server reads an httpOnly cookie and renders the correct shell every time — the client-side three-state machinery is redundant, because the client never has a `loading` window to flash. The test: *can the client's first paint happen before auth is known?* If the server always decides first, it can't, and you don't need the pending-state guard.

## See also

- [`routing-react-router`](../../ecosystem/routing-react-router.md#real-world-patterns) — loaders, `redirect`, and where the flash-free guard lives.
- [`refresh-storm-on-401`](./refresh-storm-on-401.md#stage-3--single-flight-refresh-with-a-shared-promise) — the in-flight refresh that `resolveAuth` must await instead of reading as logged-out.
- [`logout-leaves-stale-cache`](./logout-leaves-stale-cache.md) — the teardown side of the same session lifecycle.
- [`ssr-and-hydration`](../../server/ssr-and-hydration.md#the-agreement-contract-and-why-mismatches-are-expensive) — why a server-unauth / client-authed split both flashes and mismatches.
- [`thinking-in-react`](../../foundations/thinking-in-react.md#model-states-not-booleans) — model states, not booleans: the three-state discipline behind the fix.

## References

- React Router v7 — `loader`, `redirect`, `<Navigate>`, `LoaderFunctionArgs`.
- MDN — `Set-Cookie` `HttpOnly` / `SameSite`.
- OWASP — Authentication and Session Management Cheat Sheets.

## Demo source

- `demos/auth/flash-of-protected-content/` — the cold-start repro: hard-reload a protected deep link both authenticated and logged-out; the flash before, the flash-free loader guard after. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*