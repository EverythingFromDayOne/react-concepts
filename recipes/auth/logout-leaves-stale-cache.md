---
recipe_id: logout-leaves-stale-cache
track: auth
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - ecosystem/routing-react-router
  - ecosystem/state-management-landscape
  - recipes/auth/refresh-storm-on-401
  - recipes/state-management/zustand-goes-stale
status:
  drafted: true
  reviewed: false
---

# Logout clears the token but not the cache

> **What you'll build:** a logout that actually *ends* the session — cancels in-flight requests, wipes the TanStack Query cache in memory **and** on disk, resets user-scoped client state, and broadcasts to other tabs — so the next person on a shared device never sees the last person's data. One shared `endSession` teardown, called by both the logout button and the auth-failure path.

## The scenario

An internal support console. Agents share workstations — call center, clinic front desk, warehouse floor. Agent A works a shift; the app caches `['tickets', 'queue']`, `['customer', id]` (names, emails, order history), `['agent', 'me']`. A clicks **Log out** — the token is cleared and the app redirects to `/login`. Agent B badges in and signs in. The protected dashboard mounts, `useQuery(['tickets', 'queue'])` runs, and TanStack Query returns **A's queue from cache instantly** — a cache hit, still inside `staleTime`. B is now looking at A's tickets and the customer PII inside them. For a beat, or — if `staleTime` is long, the network slow, or the query is `staleTime: Infinity` — indefinitely.

Why it happens: the `QueryClient` is created once at the app root and lives for the life of the tab. A client-side `navigate('/login')` changes the route; it does not touch the cache. Logout cleared the *credential* (the access token) and left the *data that credential unlocked* sitting in memory, keyed by nothing user-specific.

Why it escaped QA:

- QA logs in as one user, confirms logout redirects and the token is gone — green. Nobody switches accounts on the same tab, so nobody triggers the leak.
- The bug needs a **second identity on the same `QueryClient` instance**: shared workstations, kiosks, a support agent switching accounts, a family tablet.
- With a `persistQueryClient` (localStorage/IndexedDB) persister it is worse — A's cache is on **disk**, survives a full reload and a browser restart, and re-hydrates *before* the auth check runs. Closing the tab doesn't clear it.
- A 300ms flash of another customer's PII is a reportable data-exposure incident, not a cosmetic flicker.

## Walkthrough

### Stage 1 — Name it: logout is a teardown, and the cache is session-scoped

The token is the credential; the Query cache is the data that credential unlocked. Both are server state, and the cache is where server state *belongs* ([the placement funnel](../../ecosystem/state-management-landscape.md#the-placement-funnel)) — so this is not a misplacement bug. It's an **incomplete teardown**: you destroyed the key and left the vault open. A session owns a token **and** a cache (and usually some user-scoped client state); logout has to end all of them. The rule: *anything derived from the identity dies with the identity.*

### Stage 2 — Clear the cache on logout, with the right verb and the right order

Use `queryClient.clear()` — not `invalidateQueries` or `resetQueries`. Those two **refetch** (invalidate marks active queries stale and refetches; reset returns them to initial state and refetches). On logout you want the data *gone*, not reloaded. `clear()` empties the whole cache.

Order matters. A protected component still mounted when you clear sees empty data and refetches — with a dying token. So cancel in-flight work first, kill the credential, then clear.

```tsx
// auth/session.ts — ONE teardown, reused by the button and the 401 path
import { QueryClient } from "@tanstack/react-query";
import { tokenStore } from "./tokenStore";

export async function endSession(queryClient: QueryClient): Promise<void> {
  await queryClient.cancelQueries(); // in-flight fetches won't resolve into a fresh cache
  tokenStore.clear();                // the credential dies first
  queryClient.clear();               // every cached query and mutation — all of it, gone
}
```

```tsx
// the logout button
import { useQueryClient } from "@tanstack/react-query";
import { useNavigate } from "react-router";
import { endSession } from "../auth/session";

export function useLogout() {
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  return async function logout() {
    await endSession(queryClient);
    navigate("/login", { replace: true });
  };
}
```

`useQueryClient()` returns the one root client; `clear()` removes every query and mutation and empties the cache in a single call — but it does *not* abort requests already in flight, which is why `cancelQueries()` runs first (a straggler that resolves after `clear()` would repopulate the cache with the old user's data). The protected-route guard ([routing-react-router](../../ecosystem/routing-react-router.md#real-world-patterns)) redirects the now-token-less user, so its components unmount before any refetch matters.

### Stage 3 — If you persist the cache, clear the persisted copy too

`clear()` empties *memory*. With a persister, A's data is also on disk and will re-hydrate on the next load:

```tsx
const persister = createSyncStoragePersister({ storage: window.localStorage });
persistQueryClient({ queryClient, persister, buster: userId });
```

Two defenses, used together:

- **`removeClient()` in the teardown** — delete the on-disk snapshot.
- **`buster: userId`** — tag the persisted cache with the user. A different user's `buster` won't match, so their app refuses to hydrate the previous user's snapshot — cover for the reload-before-teardown-ran race.

```tsx
export async function endSession(queryClient: QueryClient): Promise<void> {
  await queryClient.cancelQueries();
  tokenStore.clear();
  queryClient.clear();
  await persister.removeClient(); // the on-disk copy
}
```

The honest default for sensitive apps: **don't persist authenticated queries at all** — persist only public/anonymous data, or set `gcTime: 0` on PII queries so they never linger. A persister is a warm-start optimization; it is not worth a cross-user disk leak.

### Stage 4 — Propagate logout across tabs, and share the teardown with the 401 path

Two holes remain:

- **Other tabs.** The user has the app open in three tabs; logging out in one leaves the others holding the in-memory token and cache. Broadcast it:

```tsx
const authChannel = new BroadcastChannel("auth");
// inside endSession(): authChannel.postMessage("logout");

authChannel.onmessage = (event) => {
  if (event.data === "logout") {
    queryClient.clear();
    window.location.assign("/login");
  }
};
```

- **The 401 path.** When a token refresh finally fails, the [`refresh-storm-on-401`](./refresh-storm-on-401.md#stage-4--fail-once-and-integrate-with-the-app) recipe's `onAuthFailure` must run the **same** `endSession` — otherwise a session that dies from expiry leaves the cache resident even though the button path is spotless. One teardown, two callers.

**Verify the loop.** Reproduce the incident: sign in as A, load the ticket queue, log out; sign in as B. Assert that B's queue query has no data on first render (cache empty), that `localStorage` holds no A-keyed snapshot, and that a second open tab has redirected to `/login`. The PII flash is gone because there is nothing left to flash.

## Variations

1. **`clear` vs the refetching verbs.** `clear()` (or a targeted `removeQueries`) for logout; `invalidateQueries`/`resetQueries` are for when you're *staying* logged in and want fresh data.
2. **Scope client state too.** A Zustand store holding the agent's filters, selection, or a draft is user-derived client state and leaks the same way — reset it inside `endSession`. ([`zustand-goes-stale`](../state-management/zustand-goes-stale.md#stage-4--keep-the-client-half-in-the-store) covers why server state was never in a store to begin with; this is the residue that legitimately was.)
3. **Server-side session (httpOnly cookie).** If the refresh token lives in an httpOnly cookie, client teardown isn't enough — `POST /logout` so the server invalidates the session and clears the cookie; otherwise it's alive server-side and a restored cookie silently signs the user back in.
4. **Per-user key isolation.** If you genuinely must keep some cache across users (you shouldn't, for authed data), namespace query keys by identity (`['user', userId, 'tickets']`) so a cache hit can never cross users. Clearing is still simpler and safer.
5. **`gcTime: 0` for PII.** Queries with `gcTime: 0` are garbage-collected the moment their last observer unmounts, so sensitive data never sits in an idle cache waiting for the next person — even before logout runs.

## Trade-offs and common pitfalls

1. **Clearing the token but not the cache** — the whole bug. Credential gone, data resident; the next login reads cache-first and renders it.
2. **`invalidateQueries`/`resetQueries` on logout** — both refetch. Either they 401 noisily (token dead) or, worse, reload *more* of the previous user's data if the token isn't dead yet. Use `clear()`.
3. **Clearing while protected components are mounted** — they see empty and refetch. Cancel in-flight, kill the token, navigate away behind the guard, then clear.
4. **Forgetting the persister** — `clear()` is memory-only; the localStorage/IndexedDB copy survives reload and browser restart and re-hydrates before auth. `removeClient()` plus a user-scoped `buster`.
5. **Persisting authed data with a global key** — no user scoping means any user hydrates any user's snapshot on load, before auth runs. Scope it, or don't persist authed queries.
6. **Only clearing on the button, not the 401 path** — a session that dies from token expiry leaves the cache full. Share one `endSession` between the button and `onAuthFailure`.
7. **Leaving Zustand or other client stores populated** — user-derived client state (filters, drafts, selection) leaks like the cache. Reset it in the same teardown.
8. **Treating `staleTime`/refetch as the fix** — a background refetch is a *window*, not a fix. `staleTime: Infinity`, offline, or a slow network leaves the previous user's data on screen, and any flash of PII is an incident.
9. **Not ending the server session** — client teardown without `POST /logout` leaves an httpOnly session alive; a cached or restored cookie logs the user back in.
10. **Storing the access token or user object in `localStorage`** — it survives logout if you forget the `removeItem`, and it's XSS-readable. Keep the access token in memory and the refresh token in an httpOnly cookie ([`refresh-storm-on-401`](./refresh-storm-on-401.md#stage-4--fail-once-and-integrate-with-the-app)).
11. **Multi-tab left behind** — logout in one tab leaves the others rendering data until they happen to refetch. Broadcast logout to every tab.
12. **`clear()` racing an in-flight fetch** — a request that resolves *after* `clear()` repopulates the cache with the old user's data. `cancelQueries()` before `clear()`.

### When NOT to nuke the whole cache

If your app is guaranteed **single-identity per device** and "logout" is really "lock" for the *same* user — a personal PWA on your own phone, a single-account kiosk that only ever serves one login — then a full clear buys nothing but a cold re-login, and keeping the non-sensitive cache is a legitimate warm-start feature; just require re-auth. The moment a *second* identity can appear in front of that `QueryClient` — shared workstations, family devices, account switching, support impersonation — a full teardown is mandatory. The test is one question: *could a different person end up looking at this cache?* If yes, clear it.

## See also

- [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns) — the `QueryClient`, `clear()`, and the cache lifecycle this teardown operates on.
- [`refresh-storm-on-401`](./refresh-storm-on-401.md#stage-4--fail-once-and-integrate-with-the-app) — the auth-failure path that must share this teardown, and the in-memory-token / httpOnly-cookie security stance.
- [`zustand-goes-stale`](../state-management/zustand-goes-stale.md#when-not-to-move-it-to-query) — why server state lives in the cache, and how the residue client state that *is* in a store also needs clearing.
- [`state-management-landscape`](../../ecosystem/state-management-landscape.md#the-placement-funnel) — the funnel that sorts what a session owns.
- [`routing-react-router`](../../ecosystem/routing-react-router.md#real-world-patterns) — protected routes and the redirect that unmounts the authenticated tree.

## References

- TanStack Query v5 — `QueryClient.clear()`, `cancelQueries`, `removeQueries`, `persistQueryClient` / `persister.removeClient()`, `buster`, `gcTime`.
- React Router v7 — protected routes, `redirect` / `Navigate`.
- MDN — `BroadcastChannel`.
- OWASP — Session Management Cheat Sheet (logout and session invalidation).

## Demo source

- `demos/auth/logout-leaves-stale-cache/` — the shared-workstation repro: sign in as A, log out, sign in as B; the leak before, a clean cache after. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*