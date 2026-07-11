---
recipe_id: infinite-scroll-rams-the-tab
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: advanced
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - recipes/performance/virtualization-long-list
  - recipes/data-fetching/request-waterfall
status:
  drafted: true
  reviewed: false
---

# Infinite scroll loads page 80 and the tab runs out of memory

> **What you'll build:** a feed that pages forever without keeping forever — `useInfiniteQuery` + `maxPages`, an intersection sentinel for `fetchNextPage`, and virtualization so the DOM stays a window — optionally bidirectional with `fetchPreviousPage`.

## The scenario

A social feed uses `useInfiniteQuery` without `maxPages`. Power users scroll for 20 minutes → 90 pages × 20 items in `data.pages`, and the UI maps **every** item into the DOM. Chrome tab climbs past 1GB; scroll janks; low-end Androids OOM-kill the tab. "Load more" works; the product dies from success.

A partial fix virtualizes the list but still retains 90 pages in the Query cache — better DOM, still fat RAM + slow serialize to DevTools/persist.

**Why it escaped QA:** fixtures stop at 2–3 pages; QA scrolls casually; memory isn't in the checklist.

## Walkthrough

### Stage 1 — Name it: pages accumulate until you cap them

[Infinite queries](../../ecosystem/data-fetching-tanstack-query.md#infinite-queries) store `{ pages, pageParams }`. Each `fetchNextPage` **appends**. Without `maxPages`, growth is unbounded. Virtualization ([sibling recipe](../performance/virtualization-long-list.md)) caps **DOM**; `maxPages` caps **cache**.

### Stage 2 — Reject fake fixes

- **Only virtualize** — DOM fixed, cache still grows (and persist/hydrate suffers).
- **Window `onscroll` + `useState` concat** — lose dedup/cancel/GC; race city.
- **`gcTime: 0`** — nukes cache on unmount; doesn't help a long-lived feed mount.

### Stage 3 — Infinite query + sentinel + maxPages (+ virtualize)

```tsx
import { useInfiniteQuery } from "@tanstack/react-query";
import { useEffect } from "react";
import { useInView } from "react-intersection-observer";

async function fetchFeedPage({ pageParam, signal }: { pageParam: number; signal: AbortSignal }) {
  const res = await fetch(`/api/feed?page=${pageParam}&limit=20`, { signal });
  if (!res.ok) throw new Error("feed failed");
  return res.json() as Promise<FeedItem[]>;
}

export function useFeed() {
  return useInfiniteQuery({
    queryKey: ["feed", "home"],
    queryFn: ({ pageParam, signal }) => fetchFeedPage({ pageParam, signal }),
    initialPageParam: 1,
    getNextPageParam: (last, all) => (last.length === 0 ? undefined : all.length + 1),
    maxPages: 3, // keep ~60 items warm
  });
}

export function Feed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, isPending } = useFeed();
  const { ref, inView } = useInView({ rootMargin: "200px" });

  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) void fetchNextPage();
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isPending) return <FeedSkeleton />;
  const items = data.pages.flat();

  return (
    <div>
      {/* Prefer useVirtualizer here when items are heavy — see virtualization recipe */}
      {items.map((item) => (
        <FeedCard key={item.id} item={item} />
      ))}
      <div ref={ref}>{isFetchingNextPage ? "Loading…" : hasNextPage ? "…" : "End"}</div>
    </div>
  );
}
```

Compose with TanStack Virtual when row cost is high — same as variation 3 in the virtualization recipe, now with a real owner section.

### Stage 4 — Bidirectional + verify

When `maxPages` drops early pages, scrolling **up** needs `getPreviousPageParam` + `fetchPreviousPage` + a top sentinel — otherwise users hit empty space. Virtualizers often restore scroll offset when prepending; test that UX.

**Verify the loop.** Scroll until page 6+: DevTools shows ≤ `maxPages` entries in `pages`. Memory stable vs uncapped baseline. Near-end sentinel triggers exactly one in-flight `fetchNextPage`. Optional: scroll up after cap → previous page reloads.

## Variations

1. **Button "Load more"** — no IntersectionObserver; same query.
2. **Cursor/nextToken pages** — `getNextPageParam: (last) => last.nextCursor`.
3. **Virtualized window** — required for heavy rows.
4. **`useSuspenseInfiniteQuery`** — initial suspend; see [suspense-infinite-refresh](./suspense-infinite-refresh-blanks-feed.md).
5. **Persist feed** — usually **don't**; or persist only page 1 (filter in shouldDehydrate).

## Trade-offs and common pitfalls

1. **No `maxPages`** — RAM death.
2. **Flatten + mount all** — DOM death even with maxPages if max is large.
3. **Wrong `getNextPageParam`** — infinite refetch loop (always returns next).
4. **Missing `initialPageParam` (v5)** — required.
5. **Observer fires every pixel without guard** — check `isFetchingNextPage`.
6. **Keys on page index** — remount flicker; key on item id.
7. **Ignoring upward scroll after trim** — blank hole.
8. **Prefetching 50 pages on mount** — not infinite scroll, a DOS.
9. **Abort-less `queryFn`** — page changes pile HTTP.
10. **Testing only 2 pages** — never see growth.

### When NOT to infinite-scroll

Search results, admin tables, and shareable page URLs usually want **discrete pagination**. Infinite scroll is for feeds where position-in-stream matters more than addressable pages — and you must budget RAM.

## See also

- [Infinite queries](../../ecosystem/data-fetching-tanstack-query.md#infinite-queries)
- [Virtualization long list](../performance/virtualization-long-list.md)
- [Suspense infinite refresh blanks feed](./suspense-infinite-refresh-blanks-feed.md)

## References

- TanStack Query — [Infinite Queries](https://tanstack.com/query/latest/docs/framework/react/guides/infinite-queries)

## Demo source

- `demos/data-fetching/infinite-scroll-rams-the-tab/` — capped vs uncapped memory. *(Demo host TBD)*
