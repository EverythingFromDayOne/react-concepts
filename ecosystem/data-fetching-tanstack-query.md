---
article_id: data-fetching-tanstack-query
concept_folder: ecosystem
wave: 4
related:
  - state/context
  - concurrent/use-and-promises
  - concurrent/suspense
  - rendering/error-boundaries
  - server/server-components
  - effects/effects-and-synchronization
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this:** Server state is not application state. It lives on a machine you don't control, it goes stale the instant you read it, and two components that need it should never fetch it twice. TanStack Query is the cache that owns that category — and once it does, an entire class of `useEffect` fetching bugs (races, dedup, stale reads, manual loading flags) stops being your problem. This article is where the state-placement rule's "server state never in a client store" clause finally cashes out.

## What it is

TanStack Query is an **async-state cache**. You give it a key and a function that returns a promise; it gives you back the data plus everything around the data — loading and error status, background refetching, deduplication, invalidation, and mutation lifecycle — without a single `useEffect` or `useState` in your components.

The reframing it forces is the whole point, and it's the same distinction [`context`](../state/context.md) drew when it insisted context is transport, not a store:

> **Server state** is data owned by a server: products, orders, the current user. You don't own it, it can change without you, and your copy is always a cache. **Client state** is data owned by the browser session: which tab is open, what's typed in a field, whether the sidebar is collapsed.

Conflating the two is the root of most React data bugs. The moment you copy server data into a client store — `useState`, Zustand, Redux — you've promised to keep it fresh, and you will fail, because you have no idea when the server changed. TanStack Query removes that promise: the cache knows when data is stale and refetches on its own. Server state → Query. Client state → `useState`/`useReducer`/Zustand/context. That table is the spine of this entire wave.

This is *not* a state manager, and it's not competing with Zustand or Redux — they own different halves. It's also not made redundant by React Server Components: [`server-components`](../server/server-components.md) can render your initial data on the server and run mutations there, but it does **not** manage a client-side cache — background refetch on window focus, optimistic updates, `staleTime` windows, invalidation after a mutation. That's client-side server-state management, and it stays Query's job even in an RSC app (more in *How this evolved*).

> **When NOT to reach for it.** Query earns its keep when data is shared across components, cached across navigations, refetched, or mutated. A screen that fires exactly one request that never changes and nobody else reads doesn't need a cache — a plain `use()` on a promise ([`use-and-promises`](../concurrent/use-and-promises.md)) or a route loader ([`routing-react-router`](./routing-react-router.md), *planned*) is lighter. And it is never the tool for **client** state: a modal's open flag or a form's draft has no server to be a cache *of*, and forcing it into Query is the mirror image of mistake 1.

## How it works under the hood

Three objects, one subscription model. Trace them and the behavior stops being magic.

**`QueryClient` → `QueryCache` → `Query`.** The `QueryClient` you create once holds a `QueryCache`, which is a map from a **query hash** to a `Query` object. The hash is a deterministic serialization of your `queryKey` — `['product', 42]` and `['product', 42]` hash identically no matter where in the tree they're called, which is *why* two components asking for the same thing share one cache entry.

Each `Query` holds the cached `data`, its `state`, a list of **observers**, and — while a request is in flight — the `promise` for that request:

```
QueryClient
  └─ QueryCache (Map<queryHash, Query>)
       └─ Query  ['product', 42]  →  { data, state, observers[], promise? }
            ├─ QueryObserver  (from <ProductPage>'s useQuery)
            └─ QueryObserver  (from <MiniCart>'s useQuery)
```

Calling `useQuery` creates a `QueryObserver` that subscribes to the `Query` for its hash. **Dedup falls straight out of this:** when an observer wants to fetch and the `Query` already has an in-flight `promise`, it reuses that promise instead of starting a second request. This is the exact mechanism behind the [`strictmode-double-mount`](../recipes/data-fetching/strictmode-double-mount.md) recipe's final stage — StrictMode's double-mount produces two observers on one hash, and they share one network call.

**The two-axis state machine.** This is the single most important mental model in the library, and it's why the v5 status names matter. A query has two *orthogonal* status fields:

- `status`: does it have data? — `'pending'` (no data yet) · `'error'` · `'success'`
- `fetchStatus`: is a request happening right now? — `'fetching'` · `'paused'` (offline) · `'idle'`

They combine freely. `success` + `fetching` is a **background refetch** — you have data on screen *and* a request in flight. `pending` + `paused` is **offline with no cached data**. Collapsing these two axes into one boolean is the source of the "why is my whole page flashing a spinner on every refetch" bug (mistake 3). The derived flags map onto them: `isPending === status === 'pending'`, `isFetching === fetchStatus === 'fetching'`, and the convenience `isLoading === isPending && isFetching` (the *first* load specifically).

> **v5 rename, so you read old tutorials correctly:** the old `loading` status is now `pending`; `isLoading` became `isPending`; `cacheTime` became `gcTime`; `useErrorBoundary` became `throwOnError`; `keepPreviousData` became `placeholderData: keepPreviousData`. `useQuery` also lost its `onSuccess`/`onError`/`onSettled` callbacks and now takes a single options object (no more overloads).

**The staleness lifecycle.** On a successful fetch, data is written to the cache and marked **fresh**. After `staleTime` elapses (default `0` — stale immediately), it's **stale**: still shown, but eligible for a background refetch on any trigger — a new mount, window refocus, network reconnect, or a refetch interval. When the *last* observer for a `Query` unmounts, the query goes **inactive** and a `gcTime` timer starts (default 5 minutes); if no new observer subscribes before it fires, the entry is garbage-collected. So `staleTime` controls *when we refetch*; `gcTime` controls *how long we keep data nobody's watching*. They are not the same dial and tuning the wrong one is mistake 8.

**Structural sharing.** On refetch, if the new response is deeply equal to the cached data, Query keeps the *old reference*. Consumers comparing by identity don't re-render on a no-op refetch. This is also why you never hand-memoize a query result — it's already referentially stable across background updates, and the [Compiler](../rendering/memoization-and-the-compiler.md) has nothing to do here.

## Basic usage

Two pieces: a client at the root, and `useQuery` where you need data.

```tsx
// main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { App } from "./App";

// Module scope: one client for the app's lifetime. NEVER new it inside a component.
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 30_000, retry: 2 },
  },
});

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </StrictMode>,
);
```

```tsx
// ProductPage.tsx
import { useQuery } from "@tanstack/react-query";

async function fetchProduct(id: string, signal: AbortSignal): Promise<Product> {
  const res = await fetch(`/api/products/${id}`, { signal });
  if (!res.ok) throw new Error(`Product ${id}: ${res.status}`); // fetch does NOT reject on 4xx/5xx
  return res.json();
}

export function ProductPage({ id }: { id: string }) {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ["product", id],
    queryFn: ({ signal }) => fetchProduct(id, signal), // signal wires abort in for free
  });

  if (isPending) return <Spinner />;
  if (isError) return <ErrorNote message={error.message} />;
  return <ProductCard product={data} />; // data is Product here — TS narrowed it
}
```

No effect, no cleanup, no ignore flag, no local loading/error state. The `signal` handed to `queryFn` is the same `AbortController` discipline [`effects-and-synchronization`](../effects/effects-and-synchronization.md) locked — Query owns the controller now, and aborts the request if the observer unmounts or the key changes mid-flight.

Add `@tanstack/react-query-devtools` and drop `<ReactQueryDevtools />` under the provider in development; it renders a live inspector of the cache — every `Query`, its `status`/`fetchStatus`, staleness, and observer count. It's the fastest way to *see* the state machine traced below instead of guessing at it, and it's the Query-specific companion to [`performance-profiling`](./performance-profiling.md)'s broader workflow *(planned)*.

## Walkthrough — from a raw fetch to a cache

We'll take the failure mode the whole roadmap points at — `fetch` inside `useEffect` — and dismantle it one guarantee at a time. The [`search-race-condition`](../recipes/data-fetching/search-race-condition.md) recipe owns the *race* symptom; here we focus on what's left: dedup, caching, and staying in sync after a write.

### Stage 1 — The starting point and its four holes

```tsx
// The failure mode. Do not ship this.
function ProductPage({ id }: { id: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setLoading(true);
    fetch(`/api/products/${id}`)
      .then((r) => r.json())
      .then((p) => { setProduct(p); setLoading(false); })
      .catch((e) => { setError(e); setLoading(false); });
  }, [id]);
  // ...
}
```

Four holes, none of which more `useState` fixes: **no dedup** (a sibling that needs the same product fetches it again), **no cache** (navigate away and back, full reload), **a race** (fast `id` changes can land out of order — the recipe's whole subject), and **no refetch** (the product changes on the server and this screen never knows). This is one component's worth of bugs that every data-fetching component re-implements slightly differently.

### Stage 2 — `useQuery` closes all four

The basic-usage version above already closes them. Trace what the cache did the first time `<ProductPage id="42" />` mounted:

1. `useQuery` creates a `QueryObserver` for hash `["product","42"]`. No `Query` exists → cache creates one in `pending`/`fetching`, calls `queryFn`, stores the promise.
2. Response arrives → `Query` moves to `success`, `data` cached, `staleTime` timer (30s) starts. The observer re-renders with `data`.
3. Navigate away → observer unmounts → `Query` goes inactive, `gcTime` timer (5 min) starts.
4. Navigate back within 5 min → new observer subscribes, **reads cached data instantly** (`success` immediately, no spinner), and because 30s hasn't passed it doesn't even refetch. Instant screen.

### Stage 3 — A second reader proves dedup

Drop a mini-cart badge that also needs the product, mounted at the same time:

```tsx
function MiniCartLine({ id }: { id: string }) {
  const { data } = useQuery({
    queryKey: ["product", id],
    queryFn: ({ signal }) => fetchProduct(id, signal),
  });
  return <span>{data?.name ?? "…"}</span>;
}
```

Same key → same `Query` → **one network request**, two observers, both updated together. There is no coordination code to write; the hash *is* the coordination. (This is precisely why the StrictMode double-mount stops mattering — see the recipe.)

### Stage 4 — A mutation, with the server as source of truth

Add-to-cart is a write. Writes are `useMutation`, and the correctness move is to **invalidate** the affected queries so the cache refetches authoritative state instead of guessing:

```tsx
// useAddToCart.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";

export function useAddToCart() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (productId: string) =>
      fetch("/api/cart/items", {
        method: "POST",
        body: JSON.stringify({ productId }),
        headers: { "Content-Type": "application/json" },
      }).then((r) => {
        if (!r.ok) throw new Error("Add to cart failed");
        return r.json();
      }),
    onSettled: () => queryClient.invalidateQueries({ queryKey: ["cart"] }),
  });
}
```

```tsx
function AddToCartButton({ productId }: { productId: string }) {
  const { mutate, isPending } = useAddToCart();
  return (
    <button onClick={() => mutate(productId)} disabled={isPending}>
      {isPending ? "Adding…" : "Add to cart"}
    </button>
  );
}
```

`invalidateQueries({ queryKey: ["cart"] })` marks every query whose key *starts with* `["cart"]` as stale and refetches the active ones. The cart badge updates itself — no prop threading, no manual `setState`. The `isPending` here is the *mutation's* pending flag, which is also your double-submit guard (the button disables while the write is in flight — the client-UX half of the [`double-submit`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) recipe).

### Stage 5 — Make it optimistic, with rollback

For an instant-feeling cart, update the cache *before* the server answers, and roll back if it fails. The three-handler contract is exact and the order is load-bearing:

```tsx
export function useAddToCart() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (productId: string) => addToCartRequest(productId),

    onMutate: async (productId) => {
      // 1. Stop in-flight cart refetches so they can't clobber our optimistic write.
      await queryClient.cancelQueries({ queryKey: ["cart"] });
      // 2. Snapshot for rollback.
      const previousCart = queryClient.getQueryData<Cart>(["cart"]);
      // 3. Apply the optimistic change.
      queryClient.setQueryData<Cart>(["cart"], (old) =>
        old ? { ...old, items: [...old.items, { productId, qty: 1 }] } : old,
      );
      return { previousCart }; // becomes `context` below
    },

    onError: (_err, _productId, context) => {
      // Roll back to the snapshot.
      if (context?.previousCart) {
        queryClient.setQueryData(["cart"], context.previousCart);
      }
    },

    // Always resync — runs after BOTH success and rollback, which is why it's
    // onSettled and not onSuccess (mistake 7).
    onSettled: () => queryClient.invalidateQueries({ queryKey: ["cart"] }),
  });
}
```

`cancelQueries` in step 1 is not optional — without it, a background cart refetch that resolves after your optimistic write overwrites it, and your item flickers away. This is the **cache-level** optimistic pattern; contrast it with [`actions`](../concurrent/actions.md)' `useOptimistic`, which is a *local render-time overlay* for form-shaped state. Rule of thumb: form submission with a transition → `useOptimistic`; a mutation whose result many components read from cache → this. The [`state-management` recipe](../recipes/state-management/context-rerenders-the-whole-tree.md)'s server-persisted-cart variation is exactly this shape.

## Real-world patterns

**Query key factories.** Ad-hoc string keys drift and make invalidation guesswork. Centralize them:

```tsx
export const productKeys = {
  all: ["products"] as const,
  detail: (id: string) => ["products", "detail", id] as const,
  list: (filters: ProductFilters) => ["products", "list", filters] as const,
};
```

Now `invalidateQueries({ queryKey: productKeys.all })` nukes everything product-shaped, while `productKeys.detail(id)` targets one. Pair with `queryOptions` for a type-safe, reusable definition shared between `useQuery` and prefetching:

```tsx
import { queryOptions } from "@tanstack/react-query";

export const productDetailOptions = (id: string) =>
  queryOptions({
    queryKey: productKeys.detail(id),
    queryFn: ({ signal }) => fetchProduct(id, signal),
    staleTime: 60_000,
  });
```

**`select` for derived, render-optimized reads.** `select` narrows what a component subscribes to, so it only re-renders when *that slice* changes:

```tsx
const productName = useQuery({
  ...productDetailOptions(id),
  select: (p) => p.name, // re-renders only when the name changes, not the whole product
});
```

**Dependent queries with `enabled`.** When one query needs another's result, gate it with `enabled` so it stays idle (not `pending`) until its input exists:

```tsx
const { data: user } = useQuery({ queryKey: ["user", token], queryFn: fetchMe });
const { data: orders } = useQuery({
  queryKey: ["orders", user?.id],
  queryFn: () => fetchOrders(user!.id),
  enabled: !!user?.id, // won't run until the id is real
});
```

Note this is a deliberate serial fetch — a two-request waterfall by design. If the second query *doesn't* actually depend on the first's data, don't chain it; run both independently so they parallelize. (In suspense mode there's no `enabled`; you express the same dependency by splitting into separate components, since suspended queries already run in order.)

**Suspense mode — the production stable-promise source.** [`use-and-promises`](../concurrent/use-and-promises.md) taught `use()` on a promise and was explicit that its hand-rolled cache was *not production* — because a real cache needs stable promise identity, dedup, and lifecycle. `useSuspenseQuery` is that cache. It never returns `undefined` (loading and errors leave via the boundaries, not the return value), so `data` is typed as present:

```tsx
function ProductPanel({ id }: { id: string }) {
  const { data } = useSuspenseQuery(productDetailOptions(id));
  return <ProductCard product={data} />; // data: Product, guaranteed
}
```

This composes into the **loading/error/success triad** from [`suspense`](../concurrent/suspense.md) — ErrorBoundary *outside*, Suspense *inside* — with Query supplying the error-reset wiring via `QueryErrorResetBoundary`, which pays off the [`error-boundaries`](../rendering/error-boundaries.md) debt:

```tsx
import { QueryErrorResetBoundary } from "@tanstack/react-query";
import { ErrorBoundary } from "react-error-boundary";

function ProductSection({ id }: { id: string }) {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ resetErrorBoundary }) => (
            <RetryPanel onRetry={resetErrorBoundary} />
          )}
        >
          <Suspense fallback={<ProductSkeleton />}>
            <ProductPanel id={id} />
          </Suspense>
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

`reset` clears the errored queries so that when `react-error-boundary` remounts the subtree, the query retries instead of instantly re-throwing the cached error.

**Waterfall prevention** — the [`suspense`](../concurrent/suspense.md#common-mistakes) debt ("TanStack owns the prevention"). Two `useSuspenseQuery` calls in *one* component run **in serial**: each suspends, so the second doesn't start until the first resolves. That's a self-inflicted waterfall. The fix is one hook:

```tsx
// Serial — waterfall: teams waits for users, projects waits for teams.
const users = useSuspenseQuery(userOptions);
const teams = useSuspenseQuery(teamOptions);

// Parallel — one suspend, all three fetch at once.
const [users, teams, projects] = useSuspenseQueries({
  queries: [userOptions, teamOptions, projectOptions],
});
```

When the queries live in *different* components down the tree, hoist the fetch with `usePrefetchQuery` *before* the Suspense boundary — it fires the request during render so the child's `useSuspenseQuery` reads a warm (or in-flight) cache instead of starting cold.

**Keep-previous-data for filters and pagination.** Switching a filter shouldn't drop to a skeleton. `placeholderData: keepPreviousData` holds the old list on screen (flagged `isPlaceholderData`) while the new one loads:

```tsx
import { keepPreviousData } from "@tanstack/react-query";

const { data, isPlaceholderData } = useQuery({
  queryKey: productKeys.list(filters),
  queryFn: ({ signal }) => fetchProducts(filters, signal),
  placeholderData: keepPreviousData,
});
```

### Refetch triggers and dials

`staleTime` only marks data **eligible** to refetch. Nothing network happens until a *trigger* fires while the data is stale (or you force one). The default triggers:

| Trigger | Option | Default | Real-world use |
| --- | --- | --- | --- |
| Window / tab focus | `refetchOnWindowFocus` | `true` | Multi-tab dashboards that must converge when the user returns |
| Observer remount | `refetchOnMount` | `true` | Navigating back to a page while data may be stale |
| Network reconnect | `refetchOnReconnect` | `true` | Mobile / flaky networks |
| Wall-clock polling | `refetchInterval` | `false` | Live scores, delivery ETA — only when *watching* needs continuous refresh |
| Poll while hidden | `refetchIntervalInBackground` | `false` | Leave `false` unless the tab must keep polling when backgrounded (battery/server cost) |

An idle user staring at a stale screen with no focus/mount/reconnect/`refetchInterval` will **never** see a network call — by design. If that surprises you, you wanted polling (or a push channel into `setQueryData`), not a shorter `staleTime`. The [`dashboard-never-updates-while-watching`](../recipes/data-fetching/dashboard-never-updates-while-watching.md) recipe owns the "stale but idle" symptom.

Tier `staleTime` by data class rather than one global number: static catalogs → large / `Infinity`; product lists → minutes; wallet / stock / flash-sale → `0`. Pair with a modest global `retry` (1–2, not the aggressive default 3 on every query) so a downed API isn't DDoSed by every open tab.

### `refetchOnMount`: `true` vs `'always'`

Both run on mount. They differ on **fresh** cache:

- **`true` (default):** refetch only if the cached data is **stale**. Fresh cache → instant paint, **zero** network.
- **`'always'`:** refetch on every mount **even when fresh**. Cache still paints immediately (if `gcTime` hasn't GC'd it); a background fetch updates afterward.

`gcTime` is independent: it decides whether *any* cache survives unmount to paint as a preview. `'always'` does **not** nullify `gcTime` — without a surviving entry you get a first-load spinner, then the forced fetch. Use `'always'` on checkout / wallet / coupon pages where a global 5-minute `staleTime` is too trusting; leave lists on `true`. Symptom: [`refetch-on-mount-always-spams`](../recipes/data-fetching/refetch-on-mount-always-spams.md).

### Cancel vs invalidate

Two verbs, opposite intents:

- **`cancelQueries`** — *stop* an in-flight request (or mark observers cancelled). No new fetch. Used in `onMutate` so a late refetch can't overwrite an optimistic write, and when a user hits "Cancel upload."
- **`invalidateQueries`** — mark matching queries **stale** and **refetch active ones**. Used after a successful (or settled) mutation so the server stays source of truth.

Cancel without invalidate leaves you on whatever was last written (including an optimistic guess). Invalidate without cancelling an overlapping refetch can race your optimistic `setQueryData`. The optimistic contract needs **both**, in order: cancel → snapshot → set → (rollback) → invalidate on settle. Symptom confusion: [`cancel-vs-invalidate-confusion`](../recipes/data-fetching/cancel-vs-invalidate-confusion.md).

### `mutate` vs `mutateAsync`

`mutate` is fire-and-forget: lifecycle via `onSuccess` / `onError` / `onSettled` (or per-call callbacks). Query **swallows** the rejection — safe default for "click → toast → invalidate."

`mutateAsync` returns a **Promise**. Use it when the caller must `await` a result: chained dependent mutations, `Promise.all` of uploads, or React Hook Form's `handleSubmit` needing to `setError` from a 422. **You** own the rejection — wrap in `try/catch` or an unhandled rejection can surface. Prefer `mutate` unless you need the Promise. Symptom: [`mutate-async-unhandled-rejection`](../recipes/data-fetching/mutate-async-unhandled-rejection.md).

`mutationFn` takes the **variables** you pass to `mutate`/`mutateAsync` (one argument — pack multiples into an object). It does **not** receive Query's auto `signal` the way `queryFn` does; aborting a write is opt-in (you thread an `AbortController` yourself). See [`upload-cant-be-cancelled`](../recipes/data-fetching/upload-cant-be-cancelled.md).

### Dynamic parallel queries with `useQueries`

Independent fixed queries → multiple `useQuery` (alias the returns) or `useSuspenseQueries`. A **runtime-length** list of keys (cart line ids, selected user ids) cannot call `useQuery` inside `.map` — that violates the Rules of Hooks. Use **`useQueries`**:

```tsx
const results = useQueries({
  queries: ids.map((id) => ({
    queryKey: productKeys.detail(id),
    queryFn: ({ signal }) => fetchProduct(id, signal),
  })),
});
```

Each entry fails/succeeds independently. Prefer **batching on the server** (`?ids=1,2,3`) when you control the API — one request beats N. Symptom: [`n-plus-one-usequery-in-map`](../recipes/data-fetching/n-plus-one-usequery-in-map.md). Wrap the options factory in a custom hook ([`custom-hooks`](../effects/custom-hooks.md)) so UI stays dumb.

### Prefetch on intent

`queryClient.prefetchQuery` / `usePrefetchQuery` warm the cache before navigation. Prefer **intent** (hover/focus after a short dwell, likely-next page) over prefetch-everything. Hover without debounce **hover-bombs** the server as the pointer skims a list — delay ~150–200ms and cancel on `mouseleave`. Always set a non-zero `staleTime` on the prefetch or the destination `useQuery` treats the warm entry as immediately stale and refetches anyway. Symptom: [`hover-prefetch-request-storm`](../recipes/data-fetching/hover-prefetch-request-storm.md).

### Persisting the cache

`persistQueryClient` + a storage persister (sync localStorage or IndexedDB) survives reload for instant paint / offline shell. **Defaults are dangerous:** plain-text localStorage of every successful query, including PII. Filter with `shouldDehydrateQuery` (allowlist public keys; deny `user` / `wallet` / tokens), scope with `buster: userId`, and on logout call `clear()` **and** `persister.removeClient()` ([`logout-leaves-stale-cache`](../recipes/auth/logout-leaves-stale-cache.md)). Prefer IndexedDB when payloads can exceed ~5MB. Symptom for the filter half: [`public-only-persist-filter`](../recipes/data-fetching/public-only-persist-filter.md).

### Infinite queries

`useInfiniteQuery` caches **pages** (`{ pages, pageParams }`), not a flat array. You supply `initialPageParam`, `getNextPageParam` (return `undefined` when exhausted), and call `fetchNextPage` from a sentinel (`IntersectionObserver` / `useInView`). **`maxPages`** caps retained pages so a deep scroll doesn't keep every page in RAM — pair with virtualization ([`virtualization-long-list`](../recipes/performance/virtualization-long-list.md)) so the DOM window stays small too. Bidirectional feeds add `getPreviousPageParam` + `fetchPreviousPage`. Suspense variant: `useSuspenseInfiniteQuery` (initial load via boundaries; `isFetchingNextPage` still local). Refresh without blanking: wrap `refetchQueries` in `startTransition` ([`suspense`](../concurrent/suspense.md#the-transition-interplay--holding-content-instead-of-flashing-a-spinner)). Symptoms: [`infinite-scroll-rams-the-tab`](../recipes/data-fetching/infinite-scroll-rams-the-tab.md), [`suspense-infinite-refresh-blanks-feed`](../recipes/data-fetching/suspense-infinite-refresh-blanks-feed.md).

### Production defaults and cost

Aggressive defaults (`staleTime: 0`, `retry: 3`, focus refetch everywhere) optimize for freshness at the expense of your bill. A production-shaped client often looks like:

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      gcTime: 5 * 60_000,
      retry: 1,
      refetchOnWindowFocus: true,
      networkMode: "online", // or "offlineFirst" for flaky mobile — pause when offline
    },
  },
});
```

Then **override per query**: checkout `'always'` / `staleTime: 0`; static config `staleTime: Infinity`; live widgets `refetchInterval`. Batch ID lists server-side instead of N `useQueries`. Debounce intent-prefetch. These dials are the cost layer; the mechanism stays the same.

### Feature-shaped Query modules

Don't sprinkle raw `queryKey` strings through JSX. Per feature (`features/products/`):

1. **`queryKeys` / key factory** — hierarchical arrays for partial invalidate.
2. **`*Api.ts`** — pure `fetch`/`axios` functions (accept `signal`).
3. **`hooks/use*.ts`** — `useQuery` / `useMutation` / `queryOptions` wrappers (`staleTime`, `select`, …).
4. **Components** — call the hook; no URLs, no key literals.

Share one `queryOptions(...)` between route-loader prefetch and the component read. Cross-feature imports stay at the API/key layer, not into another feature's components. This is organization, not a new runtime concept — the cache still keys on the arrays you defined.

## API / type reference

| Symbol | Shape | Notes |
| --- | --- | --- |
| `useQuery(options)` | `{ queryKey, queryFn, staleTime?, gcTime?, enabled?, select?, placeholderData?, retry?, refetchOnMount?, refetchOnWindowFocus?, refetchOnReconnect?, refetchInterval?, throwOnError? }` | Returns `{ data, error, status, isPending, isError, isSuccess, isFetching, isPlaceholderData, refetch }`. Single object — no overloads in v5 |
| `status` / `fetchStatus` | `'pending'\|'error'\|'success'` / `'fetching'\|'paused'\|'idle'` | Two orthogonal axes; don't collapse them |
| `staleTime` / `gcTime` | ms; defaults `0` / `300_000` | *When to refetch* vs *how long to keep unwatched data* |
| `refetchOnMount` | `boolean \| 'always'` | `true` = if stale; `'always'` = even if fresh |
| `useMutation(options)` | `{ mutationFn, onMutate?, onError?, onSuccess?, onSettled? }` | Returns `{ mutate, mutateAsync, isPending, isError, data, variables, reset }` |
| optimistic contract | `onMutate` → `onError` → `onSettled` | cancel → snapshot → set → (rollback) → invalidate |
| `useSuspenseQuery(options)` | like `useQuery`, **no** `enabled`/`placeholderData`/`throwOnError` | `data` never `undefined`; loading/error via boundaries |
| `useSuspenseQueries({ queries })` | array in, array out | Parallel suspense — the anti-waterfall hook |
| `useQueries({ queries })` | dynamic-length parallel queries | Runtime id lists — not `useQuery` inside `.map` |
| `useInfiniteQuery` / `useSuspenseInfiniteQuery` | `{ pages, pageParams }` + `fetchNextPage` / `maxPages?` | Paginated / infinite scroll cache |
| `usePrefetchQuery(options)` | returns nothing | Fire during render, before a Suspense boundary |
| `useQueryClient()` | `QueryClient` | `.invalidateQueries` / `.setQueryData` / `.getQueryData` / `.cancelQueries` / `.prefetchQuery` / `.clear` |
| `queryOptions(options)` | typed options object | Share one definition across `useQuery`, prefetch, `setQueryData` |
| `QueryErrorResetBoundary` | render-prop `{ reset }` | Bridge to `react-error-boundary`'s `onReset` |
| `persistQueryClient` | `{ queryClient, persister, maxAge?, buster?, shouldDehydrateQuery? }` | Disk-backed cache; filter sensitive keys |

## Common mistakes

1. **Caching server state in a client store.** Copying query data into Zustand/Redux/`useState` re-signs the freshness promise Query exists to cancel — now *you* own invalidation, and stale carts and ghost data follow. Server state → Query, full stop. (The [`state-management` recipe](../recipes/state-management/context-rerenders-the-whole-tree.md) and [`context`](../state/context.md) both point here.)
2. **A key that omits an input.** `queryKey: ["product"]` for a per-id fetch dedupes across *different* products and serves whichever landed first. Every value the `queryFn` reads belongs in the key — same discipline as effect deps.
3. **Treating `isFetching` as `isPending`.** Gating a full-page spinner on `isFetching` flashes the skeleton on every background refetch and window refocus. Spinner on `isPending` (first load, no data); a subtle inline indicator on `isFetching`.
4. **Not throwing in `queryFn` on a bad response.** `fetch` resolves on 404/500 — it only rejects on network failure. Without `if (!res.ok) throw`, the query lands in `success` holding an error body, and your error UI never shows.
5. **Mirroring query data into `useState` (or syncing it in an effect).** `const [p, setP] = useState(); useEffect(() => setP(query.data), ...)` re-introduces the exact staleness/duplication bug the cache eliminates. Read `query.data` directly; derive during render.
6. **Optimistic update without `cancelQueries`.** An in-flight refetch that resolves after your `setQueryData` overwrites the optimistic value and the UI flickers. `cancelQueries` in `onMutate` is mandatory, not decorative.
7. **Invalidating in `onSuccess` instead of `onSettled`.** On a failed mutation you roll back but skip the resync, leaving the cache on a *guessed* rolled-back value instead of authoritative server state. `onSettled` runs after both paths.
8. **Turning the wrong staleness dial.** `staleTime: 0` everywhere → refetch storms on every focus; never invalidating → permanently stale screens. `staleTime` is the read-freshness knob; `gcTime` is memory retention; invalidation is the write-triggered resync. They're three different tools.
9. **Serial suspense waterfall.** Multiple `useSuspenseQuery` in one component fetch one-after-another. Reach for `useSuspenseQueries` (parallel) or hoist with `usePrefetchQuery`.
10. **`new QueryClient()` inside a component.** A fresh client every render throws away the whole cache on every update. Create it at module scope, or in `useState(() => new QueryClient())` if it must be per-app-instance (SSR).

## How this evolved

The arc is a slow separation of two things React originally smeared together. Early React fetched in `componentDidMount`, then in `useEffect` — and every app re-derived races, dedup, caching, and retry by hand, badly. SWR and React Query emerged to name the missing abstraction: **server state is a distinct category with its own cache**, not a special case of `useState`. That reframing — not any single API — is the contribution.

v5 (the baseline here) sanded the API: one options object instead of overloads, the `loading→pending` / `isLoading→isPending` rename to free `isLoading` for "first fetch only," `cacheTime→gcTime`, `useErrorBoundary→throwOnError`, and — most relevant to this project — **stable Suspense hooks** (`useSuspenseQuery` and friends), retiring the old experimental `suspense: boolean` flag. That's what let [`use-and-promises`](../concurrent/use-and-promises.md) defer "the production version" here.

And RSC doesn't retire any of it. [`server-components`](../server/server-components.md) render initial data on the server and run mutations there — but a Server Component can't refetch on window focus, hold an optimistic overlay, or manage a `staleTime` window, because it never runs on the client after the first paint. In an RSC app the division is: server components for the initial, static-ish read; Query for everything interactive and cache-shaped after that (there's even a streaming bridge for calling `useSuspenseQuery` inside client components under the App Router). Production RSC wiring is [`nextjs-and-rsc-in-practice`](../server/nextjs-and-rsc-in-practice.md)'s job; the boundary is the point here.

## Exercises

1. **Prove the dedup.** Render two components that both call `useQuery` with `queryKey: ["me"]`. Open the Network tab and confirm one request. Then change one key to `["me", "v2"]` and watch a second request appear. *Hint: the hash is the coordination — identical keys share a `Query`, distinct keys don't.*
2. **Optimistic toggle with rollback.** Build a "favorite" button backed by `useMutation` that flips a boolean optimistically and rolls back on a forced server error. *Hint: `cancelQueries` → `getQueryData` snapshot → `setQueryData` → return context → `setQueryData(context.previous)` in `onError` → `invalidateQueries` in `onSettled`.*
3. **Flatten a waterfall.** Given a component with three sequential `useSuspenseQuery` calls, cut total load time to the slowest single request. *Hint: one hook, plural name.*

## Summary

- Server state is a distinct category with its own cache. Copying it into a client store is the root bug; TanStack Query owning it is the fix. That's the state-placement spine.
- The cache is `QueryClient → QueryCache → Query`, keyed by a hash of `queryKey`; multiple observers on one hash give you dedup and shared updates for free.
- `status` (has data?) and `fetchStatus` (fetching now?) are two axes. Read them separately or flash spinners forever.
- `staleTime` decides when to refetch; `gcTime` decides how long to keep unwatched data; invalidation is the write-triggered resync. Three dials, not one.
- Mutations write; then `invalidateQueries` lets the server stay source of truth. Optimistic updates follow the exact `onMutate`/`onError`/`onSettled` contract — and `cancelQueries` is mandatory.
- `useSuspenseQuery` is the production stable-promise cache `use()` gestured at; `useSuspenseQueries` prevents the serial waterfall; `QueryErrorResetBoundary` wires the triad's reset.

## See also

- [`context`](../state/context.md) — transport-not-store; the "server state never in a client store" clause this article cashes out
- [`use-and-promises`](../concurrent/use-and-promises.md) — `use()` and the stable-promise rule; `useSuspenseQuery` is its production form
- [`suspense`](../concurrent/suspense.md) — the loading/error/success triad and the waterfall this article's hooks prevent
- [`error-boundaries`](../rendering/error-boundaries.md) — the reset design `QueryErrorResetBoundary` bridges into
- [`server-components`](../server/server-components.md) — why RSC doesn't replace client-side cache management
- [`strictmode-double-mount` recipe](../recipes/data-fetching/strictmode-double-mount.md) — the dedup mechanism, seen from the bug side
- [`double-submit-and-optimistic-like` recipe](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) — the write-idempotency and optimistic-UI companion
- [`state-management-landscape`](./state-management-landscape.md) — where Query sits in the full state-placement table *(planned)*

## References

- TanStack Query — [Overview](https://tanstack.com/query/latest) and [Queries guide](https://tanstack.com/query/v5/docs/react/guides/queries)
- TanStack Query — [Caching](https://tanstack.com/query/v5/docs/react/guides/caching), [Optimistic Updates](https://tanstack.com/query/v5/docs/framework/react/guides/optimistic-updates), [Suspense](https://tanstack.com/query/v5/docs/react/guides/suspense)
- TanStack Query — [Migrating to v5](https://tanstack.com/query/v5/docs/react/guides/migrating-to-v5) (the rename table)
- TanStack Query — [`useQuery`](https://tanstack.com/query/v5/docs/framework/react/reference/useQuery) and [`useMutation`](https://tanstack.com/query/v5/docs/react/reference/useMutation) reference

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).