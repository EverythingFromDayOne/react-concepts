---
recipe_id: zustand-goes-stale
track: state-management
primary_concept: ecosystem/state-management-landscape
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/state-management-landscape
  - ecosystem/data-fetching-tanstack-query
  - state/context
status:
  drafted: true
  reviewed: false
---

# Server data in a Zustand store goes stale

> **What you'll build:** you'll take an admin dashboard that caches fetched inventory in a Zustand store, watch it serve five-hour-old stock counts and oversell a product, and fix it by moving the server data to its actual home — a TanStack Query cache — while keeping the genuinely-client state (selections, filters) in Zustand. The lesson is the [state-placement funnel](../../ecosystem/state-management-landscape.md#the-placement-funnel) made concrete: **server state never lives in a client store.**

## The scenario

An ops team runs an inventory dashboard. On mount it fetches the product list once and drops it into a Zustand store; every panel reads from the store:

```ts
// stores/inventory.ts — the bug, in six lines
import { create } from "zustand";
import type { Product } from "../types";

interface InventoryState {
  products: Product[];
  load: () => Promise<void>;
}

export const useInventoryStore = create<InventoryState>((set) => ({
  products: [],
  load: async () => {
    const res = await fetch("/api/inventory");
    set({ products: await res.json() });
  },
}));
```

```tsx
// InventoryDashboard.tsx
import { useEffect } from "react";
import { useInventoryStore } from "./stores/inventory";

export function InventoryDashboard() {
  const products = useInventoryStore((s) => s.products);
  const load = useInventoryStore((s) => s.load);

  useEffect(() => {
    load(); // fetch once, into the store
  }, [load]);

  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name} — {p.stock} in stock</li>
      ))}
    </ul>
  );
}
```

It demos perfectly. Then production: an operator opens the dashboard at **09:00** and it reads `stock: 8`. A warehouse sync drops that SKU to `0` at **11:30**. The operator's tab still shows `8` at **14:00** — five hours stale — accepts an order against it, and the company oversells a product that isn't there. Refund, apology, incident review.

**Why it escaped QA:** the store is a plain in-memory snapshot with no concept of staleness, refetch, or invalidation — it only changes when *this* session calls `set()`. In QA one tester works in one tab in one session, and their own edits write through to the store, so it always looks fresh *relative to their own writes*. Staleness only appears with a **second writer** (a colleague, a warehouse job, another tab) or a **long-lived session** — neither of which single-user QA reproduces. The store faithfully reflects the writer's own writes and silently misses everyone else's.

## Walkthrough

### Stage 1 — Name the category error

This isn't a Zustand bug; Zustand is doing exactly what a store does. The mistake is putting *server state* in it. Server state has an owner elsewhere (the database), changes without your app's involvement, and can be stale the instant you read it. A client store has none of the machinery that fact requires: no staleness clock, no background refetch, no request dedup, no invalidation. Every property you'd need to make the store safe is a property of a **cache**, not a store. That's the tell.

### Stage 2 — The tempting wrong fixes (and why they rot)

The instinct is to bolt freshness onto the store. Two common attempts, both worse than they look:

```ts
// ❌ Anti-fix A: poll the store fresh on an interval
useEffect(() => {
  const id = setInterval(() => useInventoryStore.getState().load(), 15_000);
  return () => clearInterval(id);
}, []);
```

You've now hand-rolled a cache — badly. There's no request dedup (two components mounting fire two fetches), no cancellation (an in-flight `load()` can land *after* a newer one and overwrite it — the exact stale-write race the [search-race-condition recipe](../data-fetching/search-race-condition.md) exists to kill), and no shared loading/error state. Anti-fix B — "add a Refresh button" — is worse: it makes staleness the *user's* job, and they won't press it.

```ts
// ❌ Anti-fix C: write through to the store after every mutation
async function setStock(id: string, stock: number) {
  await fetch(`/api/inventory/${id}`, { method: "PATCH", body: `${stock}` });
  useInventoryStore.setState((s) => ({
    products: s.products.map((p) => (p.id === id ? { ...p, stock } : p)),
  }));
}
```

This is the trap that *passed QA*: it reflects the writer's own change immediately, so single-session testing looks correct — while still missing every other writer's changes. Manual write-through is not cache invalidation; it's a mirror of your own hand.

### Stage 3 — Move server state to its home: TanStack Query

Delete the server data from the store. Read inventory through [TanStack Query](../../ecosystem/data-fetching-tanstack-query.md), which *is* the machinery Stage 1 said we needed:

```tsx
// useInventory.ts
import { useQuery } from "@tanstack/react-query";
import type { Product } from "./types";

async function fetchInventory(): Promise<Product[]> {
  const res = await fetch("/api/inventory");
  if (!res.ok) throw new Error("Failed to load inventory");
  return res.json();
}

export function useInventory() {
  return useQuery({
    queryKey: ["inventory"],
    queryFn: fetchInventory,
    staleTime: 30_000, // treat as fresh for 30s, then revalidate in the background
  });
}
```

```tsx
// InventoryDashboard.tsx — now reads the cache, not the store
import { useInventory } from "./useInventory";

export function InventoryDashboard() {
  const { data: products, isPending, isError } = useInventory();

  if (isPending) return <p>Loading inventory…</p>;
  if (isError) return <p role="alert">Couldn't load inventory.</p>;

  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name} — {p.stock} in stock</li>
      ))}
    </ul>
  );
}
```

Two defaults now do what the store never could. `refetchOnWindowFocus` (on by default) refetches when the operator returns to the tab — so the second-writer and multi-tab staleness that caused the incident is bounded to the moment of focus, not the whole session. And after any mutation, you invalidate instead of hand-syncing:

```ts
// useSetStock.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";

export function useSetStock() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (v: { id: string; stock: number }) =>
      fetch(`/api/inventory/${v.id}`, { method: "PATCH", body: `${v.stock}` }),
    onSettled: () => {
      // Mark inventory stale; Query refetches the source of truth — not a guess.
      queryClient.invalidateQueries({ queryKey: ["inventory"] });
    },
  });
}
```

`invalidateQueries` pulls the *server's* truth, so it reflects every writer, not just this one. That single line replaces all of Anti-fix C's fragile write-through.

### Stage 4 — Keep the client half in the store

The funnel leaves a residue, and *that* is what Zustand is for. After moving inventory out, the store holds only state the server has no opinion about — which rows are selected, whether the filter drawer is open, the current sort:

```ts
// stores/inventory-ui.ts — legitimately client state
import { create } from "zustand";

interface InventoryUiState {
  selectedIds: Set<string>;
  filtersOpen: boolean;
  toggleSelected: (id: string) => void;
  setFiltersOpen: (open: boolean) => void;
}

export const useInventoryUi = create<InventoryUiState>((set) => ({
  selectedIds: new Set(),
  filtersOpen: false,
  toggleSelected: (id) =>
    set((s) => {
      const next = new Set(s.selectedIds);
      next.has(id) ? next.delete(id) : next.add(id);
      return { selectedIds: next };
    }),
  setFiltersOpen: (filtersOpen) => set({ filtersOpen }),
}));
```

Query owns the remote truth; Zustand owns the local intent. Neither mirrors the other — the moment one copies from the other, the staleness bug is back.

**Verify against the incident.** Re-run the original repro: open the dashboard in two tabs, drop a SKU's stock to `0` from tab A (or a warehouse job), then focus tab B. With `refetchOnWindowFocus`, tab B pulls fresh data on focus and shows `0` — the five-hour divergence is now bounded to a tab-focus, and the mutation path's `invalidateQueries` guarantees any in-app edit is reflected from the server, not guessed. The oversell can't recur, because the number on screen is never older than the last focus.

## Variations

- **"But I need the current user everywhere."** Still Query — expose it as a `useCurrentUser()` hook backed by `useQuery`, not a store you hydrate at login. If a handful of deeply-nested reads need it without prop-drilling, put the *query hook* behind a thin [context](../../state/context.md), never a hand-synced copy.
- **You were using `persist` to survive reloads.** Zustand's `persist` middleware over server data is a staleness trap by construction — it restores yesterday's data on next load. Use Query's persistence plugin (`persistQueryClient`) with a `maxAge` and a build `buster` instead, so the restored cache has an expiry.
- **You cached in the store for optimistic updates.** Query does optimistic natively via `onMutate`/`onError` rollback — see the [double-submit recipe](../forms-and-ux/double-submit-and-optimistic-like.md). You don't need a store to get an instant UI.
- **The data arrives over a WebSocket.** Push live updates into the Query cache with `queryClient.setQueryData(["inventory"], next)`, not a separate store — one cache, one source of truth. (Real-time track territory.)
- **The data comes from the server render (RSC/SSR).** Hydrate the Query cache from a server prefetch — the [RSC→Query bridge](../../server/nextjs-and-rsc-in-practice.md#step-3--the-rsc--tanstack-query-bridge) — rather than seeding a store.

## Trade-offs and common pitfalls

1. **`staleTime: 0` (the default) refetches eagerly.** Tune it to the data's volatility: inventory maybe 30s, a user profile maybe 5 min. Too low wastes requests; too high recreates the staleness you're fixing.
2. **Don't disable `refetchOnWindowFocus` globally without a reason.** It's the specific default that fixes multi-tab and second-writer staleness. Turn it off per-query only for data that genuinely can't change out from under you.
3. **Don't `setQueryData` *into* a store "for convenience."** Copying the cache into Zustand re-creates the exact bug with extra steps. Read from the query hook at the point of use.
4. **Manual write-through after a mutation looks right and is wrong.** It reflects only the writer's own change (the QA-passing bug). Use `invalidateQueries`; let the server be the source of truth.
5. **Persisting server data anywhere — `persist` middleware, `localStorage`, a store — reintroduces staleness on reload.** A persisted server cache needs an expiry (`maxAge`) and a version buster.
6. **Optimistic updates in a plain store have no rollback.** On failure the store diverges permanently. Query's `onMutate`/`onError` gives you the snapshot-and-restore for free.
7. **Deriving from stale store data compounds the staleness.** A "total value" computed off a stale list is stale twice over. Derive from the query result during render, not from a mirrored copy.
8. **Don't run two caches for the same data.** If you adopt Query, don't also keep RTK-Query or a hand-store for inventory — [same category, one cache](../../ecosystem/state-management-landscape.md#the-decision-table).
9. **Loading and error state belong to Query, not the store.** Don't mirror `isLoading` into Zustand; read Query's discriminated `status` where you render.
10. **Same-tab live freshness needs polling or push, not a store.** If users stare at one screen expecting live numbers, add `refetchInterval` or WebSocket-into-cache — a store solves neither.
11. **Cross-tab sync of client state is a different problem.** If you *do* need the client half (selections) shared across tabs, that's `storage`-event or BroadcastChannel territory — don't conflate it with the server-state fix.

### When NOT to move it to Query

If the fetched value is a **one-time snapshot that must not change mid-session** — a frozen price quote, a signed config blob delivered once at boot, a wizard's captured inputs — then it isn't live server state at all; it's client state you happened to fetch. A store (or `useState`/context) is correct, and Query would *over*-fetch and could replace a value you deliberately froze. The test: **would a fresher value from the server improve correctness?** If yes, it's server state → Query. If a change would be wrong or confusing, it's client state → keep it local. Inventory fails the test (fresher is always more correct); a signed one-time quote passes it.

## See also

- [State management landscape](../../ecosystem/state-management-landscape.md#the-placement-funnel) — the placement funnel; this recipe is its mistake #1 made concrete.
- [Data fetching with TanStack Query](../../ecosystem/data-fetching-tanstack-query.md) — the server-state cache that is inventory's real home.
- [Recipe: search race condition](../data-fetching/search-race-condition.md) — why the hand-rolled refetch-into-store races.
- [Recipe: context re-renders the whole tree](./context-rerenders-the-whole-tree.md) — the sibling state-management recipe (client state placed wrong in the *other* direction).
- [Recipe: prop drilling six levels deep](./prop-drilling-6-levels.md) *(planned)* — the remaining state-management-track recipe.

## References

- TanStack Query — [Important Defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults) (staleTime, `refetchOnWindowFocus`)
- TanStack Query — [Query Invalidation](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation)
- TanStack Query — [Persistence (`persistQueryClient`)](https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient)
- Zustand — [docs](https://zustand.docs.pmnd.rs/)

## Demo source

*Demo pending — see the roadmap's demo-hosting decision (StackBlitz vs. CodeSandbox vs. local `demos/`).*