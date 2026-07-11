---
recipe_id: public-only-persist-filter
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: advanced
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - recipes/auth/logout-leaves-stale-cache
  - recipes/state-management/zustand-goes-stale
status:
  drafted: true
  reviewed: false
---

# Persisted Query cache writes PII into `localStorage` — and survives logout

> **What you'll build:** disk persistence that speeds reloads for **public** data only — `shouldDehydrateQuery` allowlisting, user-scoped `buster`, IndexedDB when payloads are big — wired so logout clears memory *and* disk ([teardown recipe](../auth/logout-leaves-stale-cache.md)).

## The scenario

Dev enables `persistQueryClient` + `createSyncStoragePersister({ storage: localStorage })` for "instant F5." It works — catalogs paint immediately. Security review opens Application → Local Storage and finds `['user','me']`, `['wallet', userId]`, order history — **plaintext PII**. Worse: after logout, `queryClient.clear()` runs but the persister key remains; the next visitor on a shared PC hydrates the previous user's wallet before auth ([logout-leaves-stale-cache](../auth/logout-leaves-stale-cache.md)).

Separately, an infinite product scroll persists megabytes → **QuotaExceededError** on Safari's ~5MB localStorage cap → app crash on hydrate.

**Why it escaped QA:** single-user laptops; logout tests check the URL not Storage; catalogs alone never hit quota.

## Walkthrough

### Stage 1 — Name it: persist is a security boundary

[Persisting the cache](../../ecosystem/data-fetching-tanstack-query.md#persisting-the-cache) copies Query's successful entries to disk. Disk is readable by XSS and by the next OS user. Default "persist everything successful" is hostile to auth apps.

### Stage 2 — Reject unsafe defaults

- **Persist all + clear memory only on logout** — disk leak.
- **`gcTime: Infinity` + persist authed keys** — maximizes retention of secrets.
- **Ignore quota** — crash on large lists.

### Stage 3 — Filter, scope, storage

```tsx
import { PersistQueryClientProvider } from "@tanstack/react-query-persist-client";
import { createSyncStoragePersister } from "@tanstack/query-sync-storage-persister";
import { QueryClient } from "@tanstack/react-query";
import { useState } from "react";

const PERSIST_KEY = "app-query-cache-v1";

const persister = createSyncStoragePersister({
  storage: window.localStorage,
  key: PERSIST_KEY,
});

function shouldDehydrateQuery(query: { queryKey: readonly unknown[]; state: { status: string } }) {
  const root = String(query.queryKey[0] ?? "");
  if (root.startsWith("user") || root === "wallet" || root === "session") return false;
  if (!root.startsWith("public") && root !== "catalog" && root !== "products") return false;
  return query.state.status === "success";
}

export function AppProviders({ children, userId }: { children: React.ReactNode; userId?: string }) {
  const [client] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: { staleTime: 60_000, gcTime: 24 * 60 * 60_000 },
        },
      }),
  );

  return (
    <PersistQueryClientProvider
      client={client}
      persistOptions={{
        persister,
        maxAge: 24 * 60 * 60_000,
        buster: userId ?? "anon",
        dehydrateOptions: { shouldDehydrateQuery },
      }}
    >
      {children}
    </PersistQueryClientProvider>
  );
}
```

Convention: public keys start with `public` / `catalog` / `products`. Authed hooks use `user` / `wallet` roots and never pass the filter.

Logout (mandatory companion):

```ts
await queryClient.cancelQueries();
queryClient.clear();
await persister.removeClient(); // or localStorage.removeItem(PERSIST_KEY)
```

Prefer **IndexedDB** persisters when lists are large.

### Stage 4 — Harden + verify the loop

- Non-zero `staleTime` or hydrate immediately refetches (persist pointless).
- Version `buster` or `PERSIST_KEY` when the dehydrated shape changes.
- Never persist tokens — tokens aren't Query data; keep access token in memory.

**Verify the loop.** Log in, load wallet + catalog, inspect Storage: catalog present, wallet **absent**. Log out, inspect: persister key gone / empty. Log in as B: no A's catalog under A's buster. Persist a huge infinite query to localStorage → migrate to IDB before quota errors in prod.

## Variations

1. **Allowlist by `meta: { persist: true }`** on query options instead of key prefixes.
2. **Anon vs authed persisters** — two keys; only anon survives logout.
3. **`createAsyncStoragePersister` + IDB** — mobile / large caches.
4. **Encrypt at rest** — rare; prefer not writing secrets.
5. **No persist in sensitive enterprises** — warm memory cache only.

## Trade-offs and common pitfalls

1. **Persisting authed queries** — PII on disk.
2. **Logout without `removeClient`** — cross-user hydrate.
3. **Global buster** — no user isolation.
4. **localStorage + infinite scroll** — quota crash.
5. **`staleTime: 0`** — hydrate then instant refetch.
6. **Persisting errors** — usually exclude non-success (default filter helps).
7. **SSR** — only persist in the browser; guard `window`.
8. **Assuming DevTools "clear site data" in QA** — write an automated assert on Storage.
9. **Key typos breaking the allowlist** — centralize key factories.
10. **Treating persist as offline-write queue** — it's a read cache; mutations need a real outbox (`pwa-offline` track).

### When NOT to persist

Regulated PII, shared kiosks, or tiny SPAs where reload cost is fine. Persist is a **UX acceleration for public/read-mostly data**, not a default for every query.

## See also

- [Persisting the cache](../../ecosystem/data-fetching-tanstack-query.md#persisting-the-cache)
- [Logout leaves stale cache](../auth/logout-leaves-stale-cache.md) — teardown half
- [Production defaults and cost](../../ecosystem/data-fetching-tanstack-query.md#production-defaults-and-cost)

## References

- TanStack Query — [Persist client](https://tanstack.com/query/v5/docs/framework/react/plugins/persistQueryClient)

## Demo source

- `demos/data-fetching/public-only-persist-filter/` — allowlist + logout disk clear. *(Demo host TBD)*
