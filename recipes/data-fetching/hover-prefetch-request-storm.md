---
recipe_id: hover-prefetch-request-storm
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - recipes/data-fetching/request-waterfall
  - recipes/performance/route-splitting-bundle
  - recipes/routing/lazy-route-flashes-blank
status:
  drafted: true
  reviewed: false
---

# Skimming the product grid fires dozens of prefetch GETs

> **What you'll build:** intent-based prefetch that makes clicks feel instant — without hover-bombing your API — by debouncing `prefetchQuery` (~150–200ms), cancelling on mouse leave, and setting a real `staleTime` so the warm cache isn't ignored.

## The scenario

A listing uses `onMouseEnter={() => queryClient.prefetchQuery(productDetailOptions(id))}` on every card. Power users skim 40 cards in two seconds → **40 GETs** (many aborted mid-flight, still costly on the edge). Mobile has no hover; the optimization does nothing there, but desktop QA looks "instant." After launch, API dashboards show a prefetch-shaped traffic spike correlated with scroll/hover, not purchases.

A second bug: prefetch omits `staleTime` (default `0`). Click lands on a detail `useQuery` that treats the warm entry as already stale → **immediate refetch**, prefetch wasted ([prefetch on intent](../../ecosystem/data-fetching-tanstack-query.md#prefetch-on-intent)).

**Why it escaped QA:** slow deliberate hovers in demos; localhost cheap; nobody scrolled quickly across a dense grid with the Network panel open.

## Walkthrough

### Stage 1 — Name it: prefetch is a bet on intent

Prefetch pays when the user **likely** navigates. Continuous hover events are not intent — dwell is. [Request waterfall](./request-waterfall.md) already warns against speculative prefetch everywhere; this recipe is the hover-shaped failure.

### Stage 2 — Reject raw `onMouseEnter` prefetch

No debounce → storm. No `staleTime` → useless warm cache. Prefetching every row in `useEffect` on mount → worse.

### Stage 3 — Debounced intent prefetch

```tsx
import { useEffect, useRef } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { productDetailOptions } from "../api/productOptions";
import { Link } from "react-router";

export function ProductCardLink({ id, children }: { id: string; children: React.ReactNode }) {
  const qc = useQueryClient();
  const timer = useRef<ReturnType<typeof setTimeout> | null>(null);

  function clear() {
    if (timer.current) clearTimeout(timer.current);
    timer.current = null;
  }

  useEffect(() => () => clear(), []);

  return (
    <Link
      to={`/products/${id}`}
      onMouseEnter={() => {
        clear();
        timer.current = setTimeout(() => {
          void qc.prefetchQuery({
            ...productDetailOptions(id),
            staleTime: 60_000, // must be > 0 or click refetches anyway
          });
        }, 175);
      }}
      onMouseLeave={clear}
      onFocus={() => {
        // keyboard users: prefetch on focus without delay storms
        void qc.prefetchQuery({ ...productDetailOptions(id), staleTime: 60_000 });
      }}
    >
      {children}
    </Link>
  );
}
```

Share `productDetailOptions` with the detail page so keys/`queryFn` match.

### Stage 4 — Harden + verify the loop

- Touch devices: rely on router/`<Link>` viewport prefetch or tap — don't fake hover.
- Cap concurrency if needed (simple in-flight Set).
- Prefetch next **page** of a list after settle (pagination), not every sibling card.

**Verify the loop.** Skim 20 cards quickly: **0** (or ~0) detail GETs. Hover one card 200ms: **1** GET. Click within `staleTime`: detail shows with **no** second GET (or only background if you chose `'always'` on detail). Hover leave before 175ms: no GET.

## Variations

1. **Lodash `debounce`** — same idea; cancel on leave.
2. **IntersectionObserver prefetch** — when a card is ~visible, prefetch detail (throttle).
3. **Router link prefetch** — RR/TanStack link prefetch for the *route module*; still pair with Query prefetch for data.
4. **Idle callback** — `requestIdleCallback` to prefetch the *next* page only.
5. **SSR `Promise.all` prefetch** — server parallel warm ([nextjs bridge](../../server/nextjs-and-rsc-in-practice.md#step-3--the-rsc--tanstack-query-bridge)); different layer.

## Trade-offs and common pitfalls

1. **Undebounced hover** — storm.
2. **`staleTime: 0` on prefetch** — wasted work.
3. **Mismatched keys** vs detail page — cache miss on click.
4. **Prefetching POST/mutable endpoints** — don't.
5. **Ignoring keyboard** — add focus prefetch.
6. **Mobile hover polyfills** — usually wrong.
7. **Prefetch on every mousemove** — worse than enter.
8. **No cancel on leave** — timers fire after skim.
9. **Prefetch storm + no HTTP/2** — connection queue jank.
10. **Measuring only click latency** — miss the skim cost.

### When NOT to prefetch on hover

Sparse UIs with one primary CTA — prefetch on mount/idle of *that* target instead. Huge payloads (video manifests) — prefetch metadata only. Rate-limited APIs — prefer click-to-fetch. Intent must be **cheaper than the miss**.

## See also

- [Prefetch on intent](../../ecosystem/data-fetching-tanstack-query.md#prefetch-on-intent)
- [Request waterfall](./request-waterfall.md)
- [Lazy route flashes blank](../routing/lazy-route-flashes-blank.md) — chunk prefetch sibling
- [Route splitting](../performance/route-splitting-bundle.md)

## References

- TanStack Query — [Prefetching](https://tanstack.com/query/latest/docs/framework/react/guides/prefetching)

## Demo source

- `demos/data-fetching/hover-prefetch-request-storm/` — skim vs dwell Network traces. *(Demo host TBD)*
