---
recipe_id: suspense-infinite-refresh-blanks-feed
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: advanced
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - concurrent/suspense
  - concurrent/concurrent-rendering
  - recipes/data-fetching/infinite-scroll-rams-the-tab
  - rendering/error-boundaries
status:
  drafted: true
  reviewed: false
---

# Pull-to-refresh on a Suspense infinite feed flashes the whole skeleton

> **What you'll build:** a `useSuspenseInfiniteQuery` feed whose **initial** load uses Suspense/ErrorBoundary cleanly, while refresh/invalidation keeps the current list on screen via `startTransition` — plus `QueryErrorResetBoundary` so Retry actually retries.

## The scenario

A news feed uses `useSuspenseInfiniteQuery` behind `<Suspense fallback={<FeedSkeleton />}>`. First paint: skeleton → feed. Good. A "Refresh" button calls `queryClient.invalidateQueries({ queryKey: ['feed'] })` (or `refetchQueries`). Because the update suspends again without a transition, React **hides the existing feed and shows the skeleton** for the whole refetch — including when 40 items were already on screen. Users call it "the app blinks."

Worse: after a network error, ErrorBoundary shows Retry; Retry only remounts the boundary. Query still has the error cached → **immediate rethrow** → Retry appears broken until a full reload. Missing [`QueryErrorResetBoundary`](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns).

**Why it escaped QA:** refresh tested on empty/slow first load (skeleton expected); error retry clicked once after DevTools "clear cache"; transitions never mentioned in the Suspense+Query copy-paste.

## Walkthrough

### Stage 1 — Name it: Suspense updates need transitions

[The transition interplay](../../concurrent/suspense.md#the-transition-interplay--holding-content-instead-of-flashing-a-spinner): suspending **updates** default to showing the fallback. Wrap the refresh in `startTransition` so React holds the previous UI and surfaces pending instead. Initial mount still uses the skeleton — correct.

### Stage 2 — Reject "just use non-Suspense infinite"

You can — `useInfiniteQuery` + local `isPending` — but then you lose the declarative boundary loading model. The bug is the refresh path, not Suspense itself.

### Stage 3 — Suspense infinite + transition refresh + error reset

```tsx
import { Suspense, useTransition } from "react";
import { useSuspenseInfiniteQuery, useQueryClient, QueryErrorResetBoundary } from "@tanstack/react-query";
import { ErrorBoundary } from "react-error-boundary";

function FeedList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useSuspenseInfiniteQuery({
    queryKey: ["feed", "home"],
    queryFn: fetchFeedPage,
    initialPageParam: 1,
    getNextPageParam: (last, all) => (last.length === 0 ? undefined : all.length + 1),
    maxPages: 3,
  });

  return (
    <>
      {data.pages.flat().map((item) => (
        <FeedCard key={item.id} item={item} />
      ))}
      {hasNextPage && (
        <button type="button" disabled={isFetchingNextPage} onClick={() => void fetchNextPage()}>
          {isFetchingNextPage ? "Loading…" : "More"}
        </button>
      )}
    </>
  );
}

function RefreshButton() {
  const qc = useQueryClient();
  const [isPending, startTransition] = useTransition();

  return (
    <button
      type="button"
      disabled={isPending}
      onClick={() => {
        startTransition(() => {
          void qc.refetchQueries({ queryKey: ["feed", "home"] });
        });
      }}
    >
      {isPending ? "Refreshing…" : "Refresh"}
    </button>
  );
}

export function FeedPage() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ resetErrorBoundary, error }) => (
            <div role="alert">
              <p>{error.message}</p>
              <button type="button" onClick={resetErrorBoundary}>
                Retry
              </button>
            </div>
          )}
        >
          <RefreshButton />
          <Suspense fallback={<FeedSkeleton />}>
            <FeedList />
          </Suspense>
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

Note: wire `onReset={reset}` from `QueryErrorResetBoundary` so Retry clears Query's error state ([article Suspense block](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns)).

### Stage 4 — Harden + verify the loop

- `fetchNextPage` should **not** blank the list — `isFetchingNextPage` is local; don't put next-page fetches in a transition that remounts the boundary incorrectly.
- Prefer `refetchQueries` in a transition over `resetQueries` if you want to keep showing data.

**Verify the loop.** Load feed → Refresh: skeleton does **not** replace the list; pending on the button. Throttle offline → error UI → Retry: one new GET, feed returns. Without reset wiring: Retry loops the error (reproduce once to confirm the bug).

## Variations

1. **Pull-to-refresh library** — call the same `startTransition` + refetch.
2. **`invalidateQueries` in transition** — same hold behavior.
3. **Granular Suspense per section** — refresh one column without blanking the chrome.
4. **Non-Suspense infinite** — if the team won't adopt transitions yet.
5. **Router loader + dehydrate** — first paint without client suspend; refresh still needs the transition discipline.

## Trade-offs and common pitfalls

1. **Invalidate without transition** — full skeleton flash.
2. **Retry without Query error reset** — infinite error loop.
3. **Suspense around the whole app shell** — refresh blanks nav too; tighten the boundary.
4. **Using `isPending` from Suspense query** — not returned; use transition pending / `isFetching`.
5. **Resetting queries on every focus** — fighting the cache.
6. **Forgetting `maxPages`** — [RAM recipe](./infinite-scroll-rams-the-tab.md) still applies.
7. **ErrorBoundary inside Suspense** — order is Error outside, Suspense inside ([article](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns)).
8. **Testing refresh only on cold cache** — miss the flash.
9. **`key={Date.now()}` on feed to "force refresh"** — nuclear remount; loses scroll.
10. **Blocking next-page on transition** — overkill; next page isn't a full resource remount.

### When NOT to use Suspense infinite

If refresh/filter is constant and the team won't use transitions, plain `useInfiniteQuery` with `isPending`/`isFetching` is simpler. Suspense shines when **boundaries already orchestrate** loading/error for the route.

## See also

- [Infinite queries](../../ecosystem/data-fetching-tanstack-query.md#infinite-queries)
- [Suspense transition interplay](../../concurrent/suspense.md#the-transition-interplay--holding-content-instead-of-flashing-a-spinner)
- [`startTransition`](../../concurrent/concurrent-rendering.md#starttransition--the-standalone-function)
- [Infinite scroll rams the tab](./infinite-scroll-rams-the-tab.md)

## References

- TanStack Query — [Suspense](https://tanstack.com/query/latest/docs/framework/react/guides/suspense)
- TanStack Query — [QueryErrorResetBoundary](https://tanstack.com/query/latest/docs/framework/react/reference/QueryErrorResetBoundary)

## Demo source

- `demos/data-fetching/suspense-infinite-refresh-blanks-feed/` — refresh with/without transition + Retry reset. *(Demo host TBD)*
