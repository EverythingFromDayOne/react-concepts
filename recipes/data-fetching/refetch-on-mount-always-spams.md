---
recipe_id: refetch-on-mount-always-spams
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - recipes/data-fetching/dashboard-never-updates-while-watching
  - recipes/data-fetching/request-waterfall
status:
  drafted: true
  reviewed: false
---

# Checkout refetches on every visit — even 3 seconds later

> **What you'll build:** a checkout page that always pulls fresh price/stock on enter *without* turning every other screen into a refetch storm — by using `refetchOnMount: 'always'` only where correctness demands it, and keeping `gcTime` so remounts still paint cached data while the forced fetch runs.

## The scenario

An e-commerce SPA sets a global `staleTime: 5 * 60_000` so product browsing feels instant. Users bounce list → detail → list within seconds and almost never wait on a spinner. Then checkout ships with the same defaults.

A shopper opens `/checkout` (stock check + price), backs to the cart for 3 seconds, returns to checkout. Under `refetchOnMount: true`, the 5-minute-fresh cache **skips** the network call. Two seconds earlier another buyer took the last unit; this shopper still sees "In stock" and submits → **409 / oversell**. Support: *"checkout showed available."*

A panicked fix sets `refetchOnMount: 'always'` on the **root** `QueryClient`. Oversell stops. Now every list remount (tab switches, nested routes) fires background GETs even for catalogs that haven't changed in hours — Network tab lights up, mobile data spikes, INP dips on low-end phones.

**Why it escaped QA:** local stock never races another buyer; `staleTime` makes remounts look "correct" in single-session tests; the global `'always'` "fix" is validated only on checkout, not on a 40-query dashboard.

## Walkthrough

### Stage 1 — Name it: fresh cache trusts `staleTime`; checkout can't

[`refetchOnMount: true` vs `'always'`](../../ecosystem/data-fetching-tanstack-query.md#refetchonmount-true-vs-always) diverge only when the entry is still **fresh**. Global `staleTime: 5m` means "trust this cache for five minutes after a successful fetch." That is right for a product grid. It is wrong for **price, stock, wallet, coupon** at the moment of commitment — those need a mount-time fetch even if the browser thinks the data is fresh.

### Stage 2 — Reject the two wrong fixes

**Global `'always'`.** Correctness for one route becomes cost for every route. You wanted a per-query override.

**`staleTime: 0` everywhere.** Every focus and remount refetches ([staleness lifecycle](../../ecosystem/data-fetching-tanstack-query.md#how-it-works-under-the-hood)) — same storm, different dial. Don't punish the whole app for checkout's needs.

### Stage 3 — Scope `'always'` (and keep `gcTime`)

Keep the global browsing defaults. Override on the commitment queries:

```tsx
// features/checkout/hooks/useCheckoutQuote.ts
import { useQuery } from "@tanstack/react-query";
import { checkoutKeys } from "../api/checkoutKeys";
import { fetchCheckoutQuote } from "../api/checkoutApi";

export function useCheckoutQuote(cartId: string) {
  return useQuery({
    queryKey: checkoutKeys.quote(cartId),
    queryFn: ({ signal }) => fetchCheckoutQuote(cartId, signal),
    staleTime: 30_000, // still useful for *within*-checkout soft nav
    refetchOnMount: "always", // even if fresh — stock/price at enter
    gcTime: 5 * 60_000, // keep preview data across a brief leave
  });
}
```

Remount within `gcTime`: UI paints the previous quote immediately; `'always'` starts a background fetch; when it lands, price/stock update. Remount after GC: first-load spinner — correct, there is nothing safe to show.

Lists stay on defaults (`refetchOnMount: true`, multi-minute `staleTime`).

### Stage 4 — Harden + verify the loop

- Pair with mutation invalidation after cart changes (`invalidateQueries` on `checkoutKeys.quote`) so you don't rely on remount alone.
- Optional: `refetchOnWindowFocus: true` on the quote so a tab left open during a flash sale still converges.

**Verify the loop.** Seed a quote, wait 3s (still fresh under 5m global), remount checkout: Network shows a quote GET. Remount a product list within `staleTime`: **no** list GET. Drop stock on the server between remounts: UI updates after the background fetch; submit no longer uses the stale "In stock" claim.

## Variations

1. **Wallet / gift-card balance** — same `'always'` (or `staleTime: 0`) on the balance query only.
2. **`staleTime: 0` on the quote** — equivalent "always eligible"; `'always'` is clearer when a parent default sets a long `staleTime`.
3. **Loader prefetch** — route loader `prefetchQuery` the quote at nav time; component still uses `'always'` so a client remount without nav still refreshes.
4. **WebSocket stock** — push into `setQueryData`; `'always'` becomes backup, not the only freshness path.
5. **SSR hydrate then `'always'`** — warm first paint from [`HydrationBoundary`](../../server/nextjs-and-rsc-in-practice.md#step-3--the-rsc--tanstack-query-bridge); remounts still force a client check.

## Trade-offs and common pitfalls

1. **Global `'always'`** — correctness leak becomes a refetch storm.
2. **Assuming `'always'` disables `gcTime`** — it doesn't; GC still drops preview data.
3. **Spinner on every remount with `'always'`** — you set `gcTime: 0` or gated the UI on `isFetching` instead of `isPending`.
4. **Forgetting invalidate after cart edits** — remount-only freshness misses same-page mutations.
5. **Putting `'always'` on infinite lists** — burns quota; page with `true` + sensible `staleTime`.
6. **Confusing `refetchOnMount` with `refetchOnWindowFocus`** — different triggers; tune separately.
7. **Checkout in a keep-alive tab** — no remount → no mount refetch; add focus refetch or polling.
8. **Shared `queryOptions` without the override** — list and checkout accidentally share the soft options.
9. **Treating 409 as a UI bug** — server must still enforce stock; client freshness only reduces races.
10. **QA only on cold load** — the bug is *warm* remount inside `staleTime`.
11. **Mobile data surprise** — measure Network after the global `'always'` "fix."

### When NOT to use `'always'`

If the worst case of slightly stale data is a refresh away and not a money/stock/auth decision — product descriptions, blog posts, settings copy — keep `refetchOnMount: true` and a real `staleTime`. `'always'` is for **commitment surfaces**, not every GET.

## See also

- [Refetch triggers and dials](../../ecosystem/data-fetching-tanstack-query.md#refetch-triggers-and-dials) — the full trigger table.
- [`refetchOnMount` true vs always](../../ecosystem/data-fetching-tanstack-query.md#refetchonmount-true-vs-always) — mechanism this recipe applies.
- [Dashboard never updates while watching](./dashboard-never-updates-while-watching.md) — idle/stale without a trigger (polling).
- [Request waterfall](./request-waterfall.md) — don't confuse remount refetch with serial waterfalls.

## References

- TanStack Query — [Important Defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults)
- TanStack Query — [Window Focus Refetching](https://tanstack.com/query/latest/docs/framework/react/guides/window-focus-refetching)

## Demo source

- `demos/data-fetching/refetch-on-mount-always-spams/` — warm remount skips fetch under `true`, forces under `'always'`, list unaffected. *(Demo host TBD)*
