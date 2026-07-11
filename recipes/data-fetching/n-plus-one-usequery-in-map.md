---
recipe_id: n-plus-one-usequery-in-map
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - effects/custom-hooks
  - foundations/rules-of-react
  - recipes/data-fetching/request-waterfall
status:
  drafted: true
  reviewed: false
---

# Cart rows each call `useQuery` in a `.map` — hooks explode or the API melts

> **What you'll build:** a cart (or any dynamic id list) that loads line details safely — `useQueries` for a runtime-length list, or better a single batched endpoint — without violating the Rules of Hooks or opening N parallel GETs you didn't mean to.

## The scenario

Checkout receives `productIds: string[]` from the cart API (often 1–30 ids). A dev writes:

```tsx
// ❌ Rules of Hooks + N+1
{productIds.map((id) => {
  const { data } = useQuery({ queryKey: ["product", id], queryFn: () => fetchProduct(id) });
  return <Line key={id} product={data} />;
})}
```

Strict mode / lint: **hooks called from a callback**. Someone "fixes" it by making a `<Line id={id} />` child that calls `useQuery` — lint passes. Prod cart with 24 lines fires **24 GETs** on every open (connection pool saturation, rate limits, waterfall-ish delay on slow mobiles).

**Why it escaped QA:** fixtures use 1–2 ids; lint green after the child-component move; nobody counted Network waterfalls on a fat cart.

## Walkthrough

### Stage 1 — Name both bugs

1. **Hooks in `.map`** — conditional/variable hook count breaks the [Rules of Hooks](../../foundations/rules-of-react.md#how-it-works-under-the-hood).
2. **N independent detail fetches** — even with legal hooks, you may want one batch ([UI N+1](./request-waterfall.md)).

[Dynamic parallel queries](../../ecosystem/data-fetching-tanstack-query.md#dynamic-parallel-queries-with-usequeries) exist for the legal parallel case.

### Stage 2 — Reject fake fixes

- **Disable the lint** — hides a real rules violation.
- **Always 30 hardcoded `useQuery` calls** — absurd; still N requests.
- **Serial `enabled` chain** — turns N into a staircase.

### Stage 3 — `useQueries` or batch

**Client-side parallel (when no batch API):**

```tsx
// features/cart/hooks/useCartProducts.ts
import { useQueries } from "@tanstack/react-query";
import { productKeys } from "@/features/products/api/productKeys";
import { fetchProduct } from "@/features/products/api/productApi";

export function useCartProducts(ids: string[]) {
  return useQueries({
    queries: ids.map((id) => ({
      queryKey: productKeys.detail(id),
      queryFn: ({ signal }: { signal: AbortSignal }) => fetchProduct(id, signal),
      staleTime: 60_000,
    })),
  });
}
```

```tsx
export function CartLines({ ids }: { ids: string[] }) {
  const queries = useCartProducts(ids);
  if (queries.some((q) => q.isPending)) return <CartSkeleton />;
  return (
    <ul>
      {queries.map((q, i) =>
        q.data ? <Line key={ids[i]} product={q.data} /> : null,
      )}
    </ul>
  );
}
```

**Better — one request:**

```tsx
useQuery({
  queryKey: productKeys.batch(ids),
  queryFn: ({ signal }) => fetchProductsByIds(ids, signal),
  enabled: ids.length > 0,
});
```

### Stage 4 — Harden + verify the loop

- Wrap in a [custom hook](../../effects/custom-hooks.md) so components don't own key factories.
- Partial failure: `useQueries` lets one id 404 without failing siblings; batch APIs should return per-id errors in-band.
- Stable `ids` array reference or sorted copy in the key to avoid refetch thrash.

**Verify the loop.** Cart with 12 ids: either **1** batch GET or **12** parallel GETs from `useQueries` — never hooks-in-map errors. Remove one id: query count follows. Lint clean.

## Variations

1. **`combine` in useQueries (v5)** — derive a single `{ data, isPending }` view.
2. **Suspense** — not `useSuspenseQuery` in a map; prefer batched `useSuspenseQuery` or parallel fixed `useSuspenseQueries`.
3. **Prefetch batch in route loader** — warm before paint.
4. **Select/transform per row** — `select` on each query options entry.
5. **Empty ids** — `enabled: ids.length > 0` / empty `queries: []`.

## Trade-offs and common pitfalls

1. **Hooks inside `.map` / conditions** — illegal.
2. **Child `useQuery` without noticing N** — lint ≠ performance.
3. **Unsorted ids in the batch key** — same set, different order → duplicate cache entries.
4. **No `signal` forwarding** — cancels don't abort HTTP.
5. **Over-fetching details the list endpoint already returned** — don't.
6. **One failure fails the whole batch UX** — design partial UI.
7. **Recreating `queries` array with new options every render carelessly** — usually OK (Query hashes options); don't put unstable functions inline without need.
8. **Using `useQueries` when the server offers batch** — extra complexity.
9. **Ignoring max browser connections** — 24 GETs queue; batch wins.
10. **Testing only 1-id carts** — miss the storm.

### When NOT to use `useQueries`

If you control the API, **batch**. If the list is fixed length of 2–3 known keys, separate `useQuery` hooks are clearer. `useQueries` is for **runtime-length** parallel reads you cannot collapse.

## See also

- [Dynamic parallel queries](../../ecosystem/data-fetching-tanstack-query.md#dynamic-parallel-queries-with-usequeries)
- [Custom hooks](../../effects/custom-hooks.md)
- [Request waterfall](./request-waterfall.md) — UI N+1 note
- [Feature-shaped Query modules](../../ecosystem/data-fetching-tanstack-query.md#feature-shaped-query-modules)

## References

- TanStack Query — [useQueries](https://tanstack.com/query/latest/docs/framework/react/reference/useQueries)
- React — [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)

## Demo source

- `demos/data-fetching/n-plus-one-usequery-in-map/` — illegal map vs useQueries vs batch. *(Demo host TBD)*
