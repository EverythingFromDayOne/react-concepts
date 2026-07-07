---
recipe_id: refresh-storm-on-401
track: auth
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: advanced
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - ecosystem/routing-react-router
  - effects/effects-and-synchronization
  - recipes/data-fetching/search-race-condition
  - recipes/data-fetching/strictmode-double-mount
status:
  drafted: true
  reviewed: false
---

# A refresh storm on 401 logs the user out

> **What you'll build:** you'll take a dashboard that randomly logs users out after an idle tab, trace it to six parallel requests each independently refreshing an expired token, and fix it with a **single-flight refresh** — one refresh the whole app waits on, then retries. This is the auth-track equivalent of [the search race](../data-fetching/search-race-condition.md): the failure is concurrency, and the fix is deduplicating the in-flight work.

## The scenario

A SaaS dashboard loads six widgets in parallel on mount, each hitting the API with a short-lived access token. A user leaves the tab backgrounded for 20 minutes; the access token expires. They refocus, the dashboard mounts, and all six requests fire — and all six come back **401** at almost the same instant.

Each request's 401 handler does the reasonable thing: call `/auth/refresh` to get a new token. So six requests trigger **six concurrent refresh calls**. On its own that's wasteful. But the backend uses **refresh-token rotation** — a security best practice where each refresh consumes the old refresh token and issues a new one. The first refresh call succeeds and rotates the token; the other five arrive presenting the now-consumed old token, get rejected, and the app reads that rejection as *"refresh failed — the session is dead."* It logs the user out.

```ts
// api.ts — the handler that storms
async function apiGet(path: string) {
  let res = await fetch(path, { credentials: "include" });
  if (res.status === 401) {
    // Every 401 independently refreshes. Six requests → six refreshes.
    const refresh = await fetch("/auth/refresh", { method: "POST", credentials: "include" });
    if (!refresh.ok) {
      logout(); // ← five of six land here after rotation consumes the token
      throw new Error("Session expired");
    }
    res = await fetch(path, { credentials: "include" });
  }
  return res.json();
}
```

The user had a perfectly valid session; the app destroyed it. It happens on roughly one in three post-idle loads. Support tickets: *"it keeps logging me out."*

**Why it escaped QA:** tests issue requests one at a time, so the refreshes never collide. The storm needs *concurrency* (a multi-request page) *and* an expired token at the exact moment of load (a refocused idle tab) *and* rotation enabled — and rotation is often only turned on in staging/prod, not the dev backend. Every ingredient is normal; only their intersection fails.

## Walkthrough

### Stage 1 — Name it

The refresh is happening **per request** when it should happen **per app**. Six requests each believe they're responsible for refreshing, so six refreshes race. Rotation turns that race from wasteful into destructive: concurrent refreshes invalidate each other, and the losers look identical to a real expiry. The fix is to make the refresh **single-flight** — at most one in flight, with everyone else waiting on it and then retrying.

### Stage 2 — The naive fix drops requests

A boolean guard is the first instinct:

```ts
let isRefreshing = false;
// ...
if (res.status === 401) {
  if (!isRefreshing) {
    isRefreshing = true;
    await fetch("/auth/refresh", { method: "POST", credentials: "include" });
    isRefreshing = false;
  }
  // ❌ the other five requests see isRefreshing === true and… do what?
}
```

The flag stops the extra refreshes, but the five requests that skipped it have nothing to await and no way to retry — they either fall through with a 401 or silently drop. A flag tells you a refresh is happening; it doesn't let you *wait for it and resume*. You need a shared promise, not a boolean.

### Stage 3 — Single-flight refresh with a shared promise

Store the refresh **promise**, not a flag. The first 401 starts it; every concurrent 401 awaits the *same* promise; when it resolves, each retries its own request once:

```ts
// authFetch.ts
let refreshPromise: Promise<void> | null = null;

async function refreshToken(): Promise<void> {
  const res = await fetch("/auth/refresh", { method: "POST", credentials: "include" });
  if (!res.ok) throw new Error("Session expired");
  // The server sets the rotated tokens (httpOnly cookies); nothing to store here.
}

export async function authFetch(input: RequestInfo, init?: RequestInit): Promise<Response> {
  const res = await fetch(input, { ...init, credentials: "include" });
  if (res.status !== 401) return res;

  // Single-flight: first 401 creates the promise; the rest await the SAME one.
  refreshPromise ??= refreshToken().finally(() => {
    refreshPromise = null; // reset so the next real expiry can refresh again
  });

  try {
    await refreshPromise;
  } catch {
    onAuthFailure(); // idempotent: clears cache + redirects ONCE
    throw new Error("Session expired");
  }

  // Retry the original request exactly once with the refreshed session.
  return fetch(input, { ...init, credentials: "include" });
}
```

The `refreshPromise ??= …` is the whole trick: assign only if null, so six simultaneous 401s produce **one** refresh call and five awaiters. The `.finally(() => refreshPromise = null)` resets it so a *later* expiry can refresh again. Now rotation is safe — there's only ever one refresh consuming one token.

### Stage 4 — Fail once, and integrate with the app

When the refresh genuinely fails (the session really is dead), every waiting request lands in the `catch` — so `onAuthFailure` must be **idempotent**: clear the [TanStack Query](../../ecosystem/data-fetching-tanstack-query.md) cache and redirect to login *once*, not six times:

```ts
// onAuthFailure.ts
let handling = false;

export function onAuthFailure() {
  if (handling) return; // six failed requests, one logout
  handling = true;
  queryClient.clear(); // don't leave the next user this user's cached data
  window.location.assign("/login");
}
```

Wire `authFetch` as the transport under everything: Query's `queryFn`/`mutationFn` call it, and [React Router](../../ecosystem/routing-react-router.md) loaders call it too — the single-flight refresh lives *below* Query, so every consumer inherits it for free:

```ts
// the widgets from the scenario, now storm-proof:
function useWidget(id: string) {
  return useQuery({
    queryKey: ["widget", id],
    queryFn: () => authFetch(`/api/widgets/${id}`).then((r) => r.json()),
  });
}
```

Critically, tell Query **not to retry a 401** — otherwise its retry logic re-fires the request three times and multiplies the very storm you just fixed:

```ts
new QueryClient({
  defaultOptions: { queries: { retry: (count, err) => !isAuthError(err) && count < 2 } },
});
```

Clearing the cache on logout is its own recurring bug — the sibling [logout-leaves-stale-cache](./logout-leaves-stale-cache.md) recipe owns it; here `queryClient.clear()` is the one line that prevents the next user from seeing the last user's data.

**Verify against the incident.** Reproduce it: expire the access token (or mock a 401), fire six parallel `authFetch` calls, and assert exactly **one** POST to `/auth/refresh` and **zero** logouts. Before: six refreshes, ~one success, a logout. After: one refresh, six retries, session intact.

## Variations

- **Access token in a header instead of a cookie.** If you carry the access token in an `Authorization` header (in memory — never localStorage), `refreshToken` updates the in-memory token and the retry re-reads it. The single-flight structure is identical; only the token plumbing changes.
- **Axios interceptors.** The same pattern as a response interceptor: on 401, await the shared `refreshPromise`, then replay the original `config`. Axios makes the "queue and replay" explicit, but it's the same single-flight promise.
- **Proactive refresh.** Refresh *before* expiry — a timer set to fire a minute before the token dies, or a refresh on window-focus — so the 401 dance rarely runs at all. You still need single-flight, because a focus event and an in-flight request can collide.
- **Mutations mid-refresh.** A `POST` that 401s should also queue behind the refresh and retry — but retrying a non-idempotent write needs the same care as [the double-mount recipe's write classification](../data-fetching/strictmode-double-mount.md#stage-3--the-write-path-is-a-different-bug): only auto-retry writes that are safe to repeat, or gate them with an idempotency key.

## Trade-offs and common pitfalls

1. **Refreshing per request instead of per app** is the root storm. One shared in-flight refresh, always.
2. **A boolean `isRefreshing` with no queue** stops extra refreshes but drops the requests that were supposed to wait. Store the promise and await it.
3. **Forgetting to reset `refreshPromise` in `finally`** leaves every future 401 awaiting a stale, already-settled promise — a permanent broken-auth state. Reset it when the refresh settles.
4. **Letting the data layer retry a 401** (Query's default retry, an axios-retry plugin) multiplies the storm 3×. Treat auth errors as non-retryable at that layer; `authFetch` owns the single retry.
5. **Redirecting to login per failed request** produces N redirects and a flicker. Make `onAuthFailure` idempotent — one logout for the whole batch.
6. **Retrying the original request more than once** risks an infinite loop if the retry also 401s (e.g., the refresh "succeeded" but the new token is also rejected). Retry exactly once, then fail.
7. **Storing tokens in `localStorage`** exposes them to XSS exfiltration. Prefer httpOnly, Secure, SameSite cookies for the refresh token; keep the access token in memory. Cookie-based refresh also needs **CSRF** protection (SameSite plus a token).
8. **Not clearing the Query cache on logout** leaks one user's data into the next session on a shared device. `queryClient.clear()` on auth failure; see the sibling recipe.
9. **Auto-retrying non-idempotent mutations** after refresh can double-charge or double-post. Classify writes before replaying them.
10. **A refresh endpoint that itself requires the access token** deadlocks — refresh must authenticate with the refresh credential only.
11. **Racing a proactive refresh against a reactive one** reintroduces the storm at the seam. Route both through the same `refreshPromise`.

### When NOT to build this

If your backend uses **opaque server-side sessions** (a session cookie the server validates on each request, no client-visible token to refresh), none of this applies — the server renews the session and you'd be building machinery for a problem you don't have. Likewise, a genuinely **single-request** screen can't storm, though routing `authFetch` through the shared refresh is cheap insurance for when that screen grows a second request. Build the single-flight refresh when you have client-managed short-lived tokens *and* pages that fire more than one request — which is most token-based SPAs, but not all apps.

## See also

- [Data fetching with TanStack Query](../../ecosystem/data-fetching-tanstack-query.md) — the cache to clear on logout and the retry policy to tame.
- [Routing with React Router](../../ecosystem/routing-react-router.md) — where loaders call `authFetch` and route guards handle the redirect.
- [Recipe: search race condition](../data-fetching/search-race-condition.md) — the same "deduplicate concurrent in-flight work" shape, for reads.
- [Recipe: logout leaves stale query cache](./logout-leaves-stale-cache.md) *(planned)* — the cache-clearing half of auth teardown.
- [Recipe: flash of protected content before redirect](./flash-of-protected-content.md) *(planned)* — the auth-track sibling on guard timing.

## References

- MDN — [`credentials` and CORS with cookies](https://developer.mozilla.org/en-US/docs/Web/API/Request/credentials)
- OWASP — [Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- TanStack Query — [Query Retries](https://tanstack.com/query/latest/docs/framework/react/guides/query-retries)

## Demo source

*Demo pending — see the roadmap's demo-hosting decision (StackBlitz vs. CodeSandbox vs. local `demos/`).*