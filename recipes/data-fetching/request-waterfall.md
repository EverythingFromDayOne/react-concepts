---
recipe_id: request-waterfall
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: advanced
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - concurrent/suspense
  - ecosystem/routing-react-router
  - concurrent/use-and-promises
  - data-fetching/search-race-condition
status:
  drafted: true
  reviewed: false
---

> **What you'll build:** a product page that loads in one round-trip's worth of time instead of four stacked back-to-back. You'll learn to *see* a request waterfall (it's invisible on localhost), tell an accidental one from a real data dependency, and flatten it with the right tool — parallel queries, a route loader, or an API change — instead of the wrong one.

## The scenario

A product page is a tree, and every level fetches its own slice:

```tsx
function ProductPage({ id }: { id: string }) {
  const { data: product } = useSuspenseQuery(productOptions(id));
  return (
    <article>
      <h1>{product.name}</h1>
      <Suspense fallback={<SellerSkeleton />}>
        <SellerInfo sellerId={product.sellerId} />
      </Suspense>
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={id} />
      </Suspense>
    </article>
  );
}

function Reviews({ productId }: { productId: string }) {
  const { data: reviews } = useSuspenseQuery(reviewsOptions(productId));
  return (
    <Suspense fallback={<RatingSkeleton />}>
      <RatingSummary productId={productId} />   {/* fetches the aggregate rating */}
      <ReviewList reviews={reviews} />
    </Suspense>
  );
}
```

It works. Each section shows a tidy skeleton, then fills in. Everyone approves the PR — the per-section loading states *look* like a feature.

On a fast connection the whole thing paints in ~1.2s. On a real phone with ~200ms of round-trip latency it takes **2.8s**, and the Network tab shows a staircase:

| Request | Starts at | Duration | Why it waited |
| --- | --- | --- | --- |
| `product` | 0ms | 300ms | — |
| `reviews` | 500ms | 400ms | didn't start until `product` resolved and `<Reviews>` mounted |
| `seller` | 500ms | 250ms | same — `<SellerInfo>` couldn't mount until `product` was done |
| `rating` | 1100ms | 200ms | didn't start until `reviews` resolved and `<RatingSummary>` mounted |

Total critical path: ~1500ms of *serial* fetching plus latency, when the slowest single request is 400ms. The requests are individually fine; the **timeline** is the bug.

**Why it escaped QA:** on localhost and in CI, latency is ~2ms per hop, so four serial requests add ~8ms — invisible. Nobody threw a network throttle. Worse, the Suspense boundaries *masked* the regression: a suspended parent doesn't render its children, so each child's `useSuspenseQuery` can't even *start* until its parent resolves — the boundaries turned independent fetches into a serial chain, and the loading skeletons made it look like intentional progressive loading. It's not progressive; it's a waterfall.

This is the *dedup* recipe's ([`strictmode-double-mount`](./strictmode-double-mount.md)) and *race* recipe's ([`search-race-condition`](./search-race-condition.md)) sibling — same track, third failure mode: not duplicate requests, not out-of-order responses, but requests that should have been concurrent running one-at-a-time.

## Walkthrough

### Stage 0 — See the staircase, name the cause

You cannot fix what you can't see, and you can't see this on localhost. **Throttle the network** (DevTools → Network → Slow 4G) or read the React 19.2 Performance Tracks; a diagonal staircase of request bars is the signature. Two mechanisms produce it, and they stack:

1. **Fetch-on-render.** A component starts its request only when it renders. Push the fetch down the tree and you push its *start time* later.
2. **Suspend-blocks-descendants.** A suspended component doesn't mount its children, so their fetches can't begin until it resolves — the [`suspense`](../../concurrent/suspense.md) mechanic, weaponized against you.

The fix is always the same shape: **start the request before the component that reads it renders.** How you do that depends on whether the dependency is real.

### Stage 1 — Accidental waterfall: parallelize independent data

`seller`, `reviews`, and `rating` don't actually need each other — only `seller` needs `product.sellerId`, and `reviews`/`rating` need just the `id` you already have. They were serialized by *nesting*, not by data. Hoist them and fire them together with `useSuspenseQueries` (the anti-waterfall hook from [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md)):

```tsx
function ProductPage({ id }: { id: string }) {
  const [{ data: product }, { data: reviews }, { data: rating }] = useSuspenseQueries({
    queries: [productOptions(id), reviewsOptions(id), ratingOptions(id)],
  });
  return (
    <article>
      <h1>{product.name}</h1>
      <Suspense fallback={<SellerSkeleton />}>
        <SellerInfo sellerId={product.sellerId} />
      </Suspense>
      <RatingSummary rating={rating} />
      <ReviewList reviews={reviews} />
    </article>
  );
}
```

One suspend, three requests in flight at once. The staircase collapses: `product`/`reviews`/`rating` all start at 0ms; the page's critical path is now the slowest of them (~400ms) plus one latency hop, not their sum. **2.8s → 900ms** on the throttled phone.

### Stage 2 — Real dependency: you can't parallelize a chain

First, a test to tell a *real* dependency from an *accidental* one, because the whole recipe hinges on it:

> Does the child query's **key or input** come from the parent's **response**? If yes, it's a real dependency and it must wait. If the child only needs a URL param or prop you *already have*, the nesting is accidental — flatten it (Stage 1).

`seller` fails the test the right way: `sellerOptions(product.sellerId)` needs a field that only exists *after* `product` resolves. That one hop genuinely must be serial, and — importantly — the scenario already expresses it correctly: `<SellerInfo sellerId={product.sellerId} />` reads `product` in the parent and passes the id down. **Correct nesting for a real dependency is not the bug; the bug is nesting that *isn't* a dependency.** Don't flatten this one.

Outside suspense, the same dependency is expressed with `enabled`, which keeps the child query *idle* (not erroring) until its input exists:

```tsx
const { data: product } = useQuery(productOptions(id));
const { data: seller } = useQuery({
  ...sellerOptions(product?.sellerId),
  enabled: product?.sellerId != null, // stays idle until the id is real
});
```

What you must *not* do is fake independence — firing `sellerOptions(undefined)` to "parallelize" just fetches garbage. So when a chain is real, you have two honest levers:

- **Collapse the round-trips at the API.** When latency dominates, the biggest win isn't client code — it's making `/api/products/:id` return the seller inline, turning two serial hops into one. A UI that needs four sequential fetches is usually a data-modeling smell pointing at a composed endpoint (BFF or GraphQL). One endpoint change can delete a staircase that no amount of `useSuspenseQueries` will.
- **Parallelize *around* the chain.** You can't shorten `product → seller`, but you *can* make sure the independent siblings (`reviews`, `rating`) run alongside it instead of behind it (Stage 1). One unavoidable serial hop is fine; three accidental ones trapped behind it are not.

### Stage 3 — Route-shaped: hoist the fetch to a loader

If this is a route, the cleanest flatten moves fetching *out of the render tree entirely*. A [`routing-react-router`](../../ecosystem/routing-react-router.md) loader fires all the route's data at navigation time, in parallel, before any component renders — so the waterfall never forms, because fetch timing is no longer coupled to the component tree:

```tsx
export const loader =
  (queryClient: QueryClient) =>
  async ({ params }: { params: { id: string } }) => {
    const id = params.id!;
    // Kick all three off at navigation time; the router awaits them together.
    await Promise.all([
      queryClient.prefetchQuery(productOptions(id)),
      queryClient.prefetchQuery(reviewsOptions(id)),
      queryClient.prefetchQuery(ratingOptions(id)),
    ]);
    return null;
  };
```

The components keep reading via `useSuspenseQuery` and hit a warm cache. The request now starts *before* the JS even renders the page.

Two honest knobs here. `prefetchQuery` **swallows errors** (the component's own `useSuspenseQuery` re-throws them into its boundary later) — swap to `ensureQueryData` if you want the loader itself to surface the error or to skip the fetch when the cache is already fresh. And `await Promise.all(...)` blocks the navigation on the *slowest* of the three: parallel, but no progressive reveal. If one section is much slower, don't `await` it — return its promise bare (article 28's deferred pattern) so the fast sections paint while the slow one streams. Blocking-all and streaming are both valid; pick per how uneven the durations are.

### Stage 4 — The general principle: create high, read low

Every fix above is one rule: **start the promise as high (early) as you can, consume it as low (deep) as you need** — the create-high/read-low parallelism from [`use-and-promises`](../../concurrent/use-and-promises.md). When you can't hoist the whole fetch, hoist just its *start* with `usePrefetchQuery` before the Suspense boundary, so the request is in flight while the boundary's fallback shows.

### Verify it — close the loop you opened in Stage 0

Re-run the *same* throttled trace. The tell that you actually fixed it: the request bars now **start together at 0ms** instead of stepping down a staircase, and the page's paint time collapses to the slowest single request plus one latency hop.

| | Before | After (Stage 1) |
| --- | --- | --- |
| `product` / `reviews` / `rating` start | 0 / 500 / 1100ms | 0 / 0 / 0ms |
| Critical path (throttled) | ~2.8s | ~900ms |
| Shape in Network tab | staircase | parallel stack |

If the staircase survives, you flattened the wrong layer — the fetches are still coupled to render order somewhere (a lingering boundary, or a dependency you assumed was accidental).

## Variations

**The code-split waterfall (the nastier one).** Wrapping a route component in `React.lazy` serializes *code and data*: document → JS → lazy chunk downloads → component renders → *then* it fetches. Four serial steps before a byte of data. The fix is the route-level `lazy` from [`routing-react-router`](../../ecosystem/routing-react-router.md), which loads the chunk *in parallel* with the loader — code and data race instead of stacking.

**The UI N+1.** A list where every row fetches its own detail is a horizontal waterfall — dozens of parallel-but-redundant requests, or worse, serial ones. Fix by batching (one request for all ids) or by having the list endpoint include what each row needs.

**Infinite scroll.** Page 2 waiting on page 1 is a *correct* serial chain (you need the cursor). Prefetch the next page as the user nears the end so the wait is hidden, not eliminated.

## Trade-offs and common pitfalls

1. **Adding Suspense boundaries to "improve loading" and serializing fetches instead.** Boundaries shape *reveal*, not *fetch timing*. If the data isn't hoisted, a boundary just delays the child's fetch start. This is the scenario's whole trap.
2. **Parallelizing a real dependency.** Firing a child query with a parent value that isn't there yet sends `undefined`. Confirm the dependency is accidental (nesting) before you flatten it.
3. **Over-hoisting into a god-loader.** Pull *every* fetch to the top and one slow query blocks the entire page — you've traded a waterfall for a single long block with no progressive reveal. Parallelism and reveal granularity are both goals; balance them.
4. **Diagnosing on localhost.** ~2ms hops hide the staircase completely. Throttle to Slow 4G or you're testing a network you'll never ship to. (This is exactly how it reached production.)
5. **Flattening but leaving skeleton jank.** Parallel requests resolve at different times; without stable, correctly-sized skeletons you get layout shift as each lands. Reveal design is still [`suspense`](../../concurrent/suspense.md)'s job.
6. **Unstable per-item keys in `useSuspenseQueries`.** A key that isn't stable per item breaks dedup and can serve the wrong row's data — the same stable-key discipline as everywhere else.
7. **Reaching for client gymnastics when an API change is the real lever.** Collapsing three hops into one endpoint beats any amount of `useSuspenseQueries` when latency dominates. Do the cheap structural fix first.
8. **Speculative prefetch everywhere.** `usePrefetchQuery` for data users rarely reach wastes their bandwidth. Prefetch on *intent* (hover, likely-next route), not on every render.
9. **Mistaking a render storm for a waterfall.** A waterfall is serial *network* (Network tab, staircase); a render storm is serial *CPU* (Profiler, long commits) — see [`typing-lag-rerender-storm`](../performance/typing-lag-rerender-storm.md). Different tab, different fix; don't apply one recipe's cure to the other's disease.
10. **The warm cache hides the cold-start waterfall.** After the first load, TanStack Query serves from cache, so the *second* navigation is instant and the staircase vanishes — devs test the warm path and never see the bug new users hit. Diagnose from a cold cache (hard reload / incognito), not your third click.
11. **Unbounded parallelism.** "Fire everything at once" isn't free: 30 parallel requests hit the browser's per-host connection cap and can trip server rate limits, so they *re-serialize* into batches anyway. Past a handful of concurrent requests, batch them into one endpoint instead of spraying.
12. **Blocking the loader on `Promise.all` of everything.** Awaiting all prefetches makes the navigation wait for the slowest section — parallel, but no progressive reveal, and one rejection (with `ensureQueryData`) sinks the whole navigation. For an uneven mix, defer the slow section (bare promise) rather than blocking the fast ones behind it.
13. **When NOT to "fix" it — because it isn't a waterfall.** Independent slow sections that fetch *in parallel* and reveal *progressively* behind separate Suspense boundaries are the target state, not the bug. A slow "you might also like" widget *should* stream in after the product without blocking it. The disease is *serial* fetching; parallel-and-progressive is health. Don't over-correct by collapsing genuinely-independent data into one blocking mega-request just to have "a single load" — that trades a staircase for a wall.

## See also

- [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md) — owns the prevention: `useSuspenseQueries`, `usePrefetchQuery`, prefetch-into-cache
- [`suspense`](../../concurrent/suspense.md) — why a suspended parent blocks its children's fetches, and reveal design
- [`routing-react-router`](../../ecosystem/routing-react-router.md) — the loader-level flatten and the code-split-waterfall fix
- [`use-and-promises`](../../concurrent/use-and-promises.md) — the create-high/read-low parallelism principle underneath every fix here
- [`search-race-condition`](./search-race-condition.md) — the data-fetching-track hub; the *ordering* failure to this recipe's *serialization* failure

## References

- TanStack Query — [Performance & Request Waterfalls](https://tanstack.com/query/latest/docs/framework/react/guides/request-waterfalls)
- TanStack Query — [Prefetching & Router Integration](https://tanstack.com/query/latest/docs/framework/react/guides/prefetching)
- React — [Suspense](https://react.dev/reference/react/Suspense)
- React Router — [Streaming with Suspense](https://reactrouter.com/how-to/suspense)

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).