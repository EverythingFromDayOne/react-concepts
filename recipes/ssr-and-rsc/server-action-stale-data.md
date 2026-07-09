---
recipe_id: server-action-stale-data
track: ssr-and-rsc
primary_concept: server/nextjs-and-rsc-in-practice
difficulty: intermediate
react_baseline: "19.2"
related:
  - server/nextjs-and-rsc-in-practice
  - ecosystem/data-fetching-tanstack-query
  - concurrent/actions
  - recipes/ssr-and-rsc/hydration-mismatch
  - recipes/forms-and-ux/double-submit-and-optimistic-like
status:
  drafted: true
  reviewed: false
---

# The Server Action succeeded, but the page still shows the old value

> **What you'll build:** a Server Action whose mutation is actually reflected in the UI — by invalidating the *exact* cache the data came from, with the *right* mechanism for read-your-own-writes, and by remembering that a client-side Query cache is a second cache to bust — instead of a write that lands in the database while the page keeps serving a stale render.

## The scenario

An admin edits a product's price. The product page caches the product with `use cache` + `cacheTag("product-" + id)`. The "Save price" Server Action writes the new price to the database and returns success. The form submits, no error — and the price on the page is **unchanged**. The admin edits again, saves again, still the old price. They file a bug: "saving doesn't work." It does; the write landed. The *cache* is serving the old render.

Then it gets subtle. Someone adds `revalidatePath("/products")` — the product *list* route, not `/products/[id]` — so nothing changes. Someone adds `revalidateTag("products")` — but the cache was tagged `product-${id}`, so the tag doesn't match and nothing is busted. Someone gets the tag right with `revalidateTag` — and the admin *still* sees the old price on this request, because in Next 16 that's stale-while-revalidate: it serves stale now and refreshes in the background.

Why it escaped QA:

- **It doesn't reproduce in `next dev`.** Dev doesn't cache the way a production build does, so the action "works" locally — the page always re-renders fresh. The staleness only appears in `next build && next start`, which nobody ran.
- **The action succeeds**, so every functional test passes; only the *displayed* value is stale.
- The **tag string** and the **mechanism choice** are silent: a mismatched tag busts nothing, and the wrong revalidation function fails quietly.

## Walkthrough

### Stage 1 — Name it: a mutation has two halves, and you skipped the second

A Server Action that mutates the source of truth does **not** automatically bust the cache — Next 16 caching is opt-in and explicit, so the mutation must invalidate every cache entry that derived from the data it changed ([the revalidation model](../../server/nextjs-and-rsc-in-practice.md#step-2--a-server-action-that-mutates-and-revalidates)). "The write succeeded" and "the cached render is stale" are both true at once. A mutation is **write the data, then invalidate what it made stale** — skip the second half and you get this bug. And the first thing to fix in your process: **you cannot reproduce this in `next dev`** — test the built app (`next build && next start`), because dev doesn't cache like production.

### Stage 2 — Match the invalidation to the cache

The tag you revalidate must be *exactly* the tag the cache was set with. Derive both from one helper so they can't drift:

```tsx
export const productTag = (id: string) => `product-${id}`;

async function getProduct(id: string) {
  "use cache";
  cacheTag(productTag(id)); // the cache is tagged here...
  return db.product.findById(id);
}
```

```tsx
"use server";
export async function savePrice(id: string, price: number) {
  await db.product.update(id, { price }); // 1) write the source of truth
  updateTag(productTag(id));              // 2) invalidate the SAME tag (Stage 3 explains updateTag vs revalidateTag)
}
```

Reject the near-misses: `revalidatePath("/products")` busts the *list* route, not the `/products/[id]` detail; `revalidateTag("products")` (plural, hand-typed) doesn't match `product-${id}` and silently busts nothing. Tag the thing, invalidate *that* tag, and derive the string once — a typo'd tag is a no-op with no error.

### Stage 3 — Read-your-own-writes: `updateTag` vs `revalidateTag`

Even with the right tag, the *mechanism* decides whether the acting user sees their own change immediately:

- **`updateTag(tag)`** — Server-Action-only, **immediate**. The response the admin gets back reflects the write. This is read-your-own-writes, and it's what you want for the user who just performed the mutation.
- **`revalidateTag(tag, "max")`** — **stale-while-revalidate**: it serves the stale value on this request and refreshes in the background, so the admin sees the *old* price until the next navigation (and thinks it failed). Correct for *other* users' background freshness, wrong for the acting user's immediate feedback.

So the admin's save uses `updateTag` for immediate consistency; if you also want every other visitor's cache to refresh, `revalidateTag(tag, "max")` handles that in the background. Using `revalidateTag` where you needed `updateTag` is the "I saved and nothing happened" report.

### Stage 4 — Don't forget the client cache, and verify

If a [TanStack Query client cache mirrors this data](../../server/nextjs-and-rsc-in-practice.md#step-3--the-rsc--tanstack-query-bridge) — the RSC→Query hydration bridge — then the Server Action revalidated the **server** cache but the **client** cache still holds the old value, so the client shows stale until you invalidate it too. The mutation's second half is now *two* invalidations:

```tsx
// after the action resolves on the client (e.g. in the mutation's onSettled)
queryClient.invalidateQueries({ queryKey: ["product", id] });
```

This is the same lesson as [a client store going stale](../../ecosystem/data-fetching-tanstack-query.md#stage-4--a-mutation-with-the-server-as-source-of-truth): the client cache is a separate thing that must be invalidated on mutation. Two caches, two invalidations.

**Verify the loop.** In `next build && next start` (not dev): the admin edits the price and saves → their page shows the new price immediately (`updateTag`), a second visitor's next load shows it (`revalidateTag` in the background), and if a client Query cache is in play, it's invalidated too. The "saved but stale" report is closed because the mutation now busts exactly what it made stale — on both caches.

## Variations

1. **The invalidation decision.** `updateTag` (read-your-own-writes, Server-Action-only, immediate) vs `revalidateTag(tag, "max")` (stale-while-revalidate, also Route Handlers) vs `revalidatePath` (blunt, path-based) — [the trio, per article 33](../../server/nextjs-and-rsc-in-practice.md#real-world-patterns).
2. **One tag helper.** Derive `cacheTag` and every revalidate call from a single function so the set and the bust can't drift — a typo is otherwise a silent no-op.
3. **Client Query cache too.** The RSC→Query bridge means two caches; invalidate both, or the client stays stale after the server is fresh.
4. **Optimistic UI to bridge the gap.** [`useOptimistic`/`useActionState`](../../concurrent/actions.md#useoptimistic--a-transition-scoped-overlay) shows the new price on the client instantly while the action and revalidation settle — the same pattern as [the optimistic-like recipe](../forms-and-ux/double-submit-and-optimistic-like.md) — so even the stale-while-revalidate window feels instant.
5. **Test caching correctly.** A cache-behavior test runs against the built app (`next build && next start`), asserts the value updates after the action, and would have caught this before prod.

## Trade-offs and common pitfalls

1. **No revalidation after a mutation** — the write lands, the cached render stays. Invalidate the tag.
2. **Tag mismatch** — the `revalidateTag`/`updateTag` string ≠ the `cacheTag` string (typo, different derivation). Silently busts nothing. One helper.
3. **Wrong granularity** — `revalidatePath("/products")` for a `/products/[id]` staleness (or the reverse). Tag and bust the specific thing.
4. **`revalidateTag` where you needed `updateTag`** — stale-while-revalidate means the acting user sees stale and thinks it failed. `updateTag` for immediate read-your-own-writes.
5. **Testing in `next dev`** — dev doesn't cache like prod; the bug is invisible. `next build && next start`.
6. **Forgetting the client Query cache** — you revalidated the server; the client mirror is still stale. `invalidateQueries` too.
7. **`revalidatePath` as a catch-all** — over-busts unrelated caches or misses the specific page. Prefer tags.
8. **Deriving the tag in two places** — drift. Derive it once.
9. **Assuming the action's return value updates the display** — it doesn't bust the cache; you still invalidate.
10. **`updateTag` outside a Server Action** — it's Server-Action-only; reaching for it in a Route Handler is wrong (use `revalidateTag` there).
11. **Ignoring the stale-while-revalidate window** — other users see the old value "for a moment"; fine by design, but know it's happening.
12. **No optimistic UI on a slow mutation** — the user waits and may re-submit (a double-submit). Optimistic UI bridges it.

### When NOT to revalidate

If the data **isn't cached** — a fully dynamic page that reads request-time data with no `use cache` — there's nothing to revalidate; every render already refetches, so a mutation is reflected immediately. Don't bolt `revalidateTag` onto uncached data. And if you're **not on an RSC/caching framework** (a Vite SPA with TanStack Query), the only "server cache" is your Query cache and this is the ordinary [`invalidateQueries`-on-mutation](../../ecosystem/data-fetching-tanstack-query.md#stage-4--a-mutation-with-the-server-as-source-of-truth) flow — no `updateTag`, no tags. The test: *is this data served from a server cache (`use cache` / a tag)?* If yes, match the invalidation to it. If it's always dynamic, or client-cache-only, there's no server revalidation to do.

## See also

- [`nextjs-and-rsc-in-practice`](../../server/nextjs-and-rsc-in-practice.md#step-2--a-server-action-that-mutates-and-revalidates) — the caching model, `updateTag` vs `revalidateTag` vs `revalidatePath`, and the RSC→Query bridge.
- [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md#stage-4--a-mutation-with-the-server-as-source-of-truth) — invalidating the client cache on mutation, the same discipline on the client side.
- [`actions`](../../concurrent/actions.md#useoptimistic--a-transition-scoped-overlay) — optimistic UI to make the revalidation window feel instant.
- [`hydration-mismatch`](./hydration-mismatch.md) — the sibling SSR bug; and [`use-client-sprawl`](./use-client-sprawl.md), the RSC-boundary sibling.
- [`double-submit-and-optimistic-like`](../forms-and-ux/double-submit-and-optimistic-like.md) — the optimistic mutation pattern this leans on.

## References

- Next.js — `use cache`, `cacheTag`, `updateTag`, `revalidateTag`, `revalidatePath`, and where each may be called.
- Next.js — testing cached behavior against a production build (`next build && next start`).
- React — Server Actions and `useOptimistic` / `useActionState`.

## Demo source

- `demos/ssr-and-rsc/server-action-stale-data/` — the price edit that "saved but stayed stale," then the matched tag + `updateTag` + client-cache invalidation, verified against a production build. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*