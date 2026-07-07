---
recipe_id: strictmode-double-mount
track: data-fetching
primary_concept: effects/effects-and-synchronization
difficulty: intermediate
react_baseline: "19.2"
related:
  - effects/effects-and-synchronization
  - recipes/data-fetching/search-race-condition
  - ecosystem/data-fetching-tanstack-query
  - rendering/how-react-renders
  - recipes/forms-and-ux/double-submit-and-optimistic-like
status:
  drafted: true
  reviewed: false
---

> **What you'll build:** a product page that fires **one** request per view instead of two — and the diagnostic habit to tell a *dev-only StrictMode fire drill* apart from a *real duplicate that ships to production*. You'll fix the read path with the locked abort-and-ignore pattern, classify the write path separately, and hand the whole problem off to a query cache that dedupes by key.

## The scenario

A product page fetches on mount:

```tsx
function ProductDetail({ id }: { id: string }) {
  const [product, setProduct] = useState<Product | null>(null);

  useEffect(() => {
    fetch(`/api/products/${id}`)
      .then((r) => r.json())
      .then(setProduct);
  }, [id]);

  // ...also records a view
  useEffect(() => {
    fetch(`/api/products/${id}/views`, { method: "POST" });
  }, [id]);

  if (!product) return <Spinner />;
  return <ProductCard product={product} />;
}
```

In development, the Network tab shows **two** `GET /api/products/42` and **two** `POST /api/products/42/views` for a single page load. A teammate "fixes" it by deleting `<StrictMode>` from `main.tsx`. The doubles vanish locally, the PR merges.

Three weeks later, analytics flags the views endpoint: product view counts are running **~1.9× actual traffic** on pages where users bounce back and forth through search results. The read endpoint is also over-served, inflating a rate-limited third-party pricing call downstream (**$0.002/req × the overage**, which is now a line item someone noticed).

**Why it escaped QA:** StrictMode's double-invoke is **development-only**. `npm run build` strips it, and QA tested the built preview — so QA saw *one* request and signed off. The dev double was never the production bug. It was a **fire drill** for a bug that was there the whole time: the effects have no cleanup, so any real remount (fast back/forward, a route transition landing mid-flight, an `<Activity>` reveal) re-runs them. Deleting StrictMode didn't fix that — it unplugged the smoke detector and shipped the fire.

This recipe is the *dedup* half of the data-fetching story. Its sibling, [`search-race-condition`](./search-race-condition.md), owns the *ordering* half (a stale response overwriting a fresh one). Same tools, different failure: here two identical requests fire; there two *different* requests resolve out of order.

## Walkthrough

### Stage 0 — Read the symptom correctly

Before touching anything, name what StrictMode is doing. On mount in development it runs each component **mount → unmount → remount** — setup, cleanup, setup — deliberately, to surface effects that forgot to clean up. The mechanics live in [`how-react-renders`](../../rendering/how-react-renders.md#interruption-replay-and-why-strictmode-double-invokes) and the audit checklist lives in [`effects-and-synchronization`](../../effects/effects-and-synchronization.md#strictmode-the-symmetry-auditor); don't re-derive them here. The one thing to internalize:

> The second request in dev is not the bug. It's StrictMode telling you the **first** effect had no cleanup — which is a bug that fires for real reasons in production.

So the fix is never "make the double go away." It's "make each effect survive being torn down and set up again," which is the definition of a correct effect.

### Stage 1 — The wrong fix, made explicit

Deleting `<StrictMode>`:

```tsx
// DON'T: this "fixes" the dev symptom by disabling the diagnostic
createRoot(document.getElementById("root")!).render(<App />);
```

This changes *nothing* about production behavior. The two effects still have no cleanup. The `POST .../views` still double-counts whenever the component genuinely remounts. You've spent your one warning and gotten nothing for it. Put StrictMode back.

### Stage 2 — Fix the read path: abort + ignore

The read (`GET`) is idempotent and safe to cancel, so it takes the locked pattern from [`effects-and-synchronization`](../../effects/effects-and-synchronization.md#fetching-in-an-effect-the-full-bill-and-the-locked-pattern) verbatim — an `AbortController` to cancel the in-flight fetch on cleanup, plus an ignore flag to guard the state write against a resolve-after-teardown race:

```tsx
function ProductDetail({ id }: { id: string }) {
  const [status, setStatus] = useState<
    | { kind: "loading" }
    | { kind: "success"; product: Product }
    | { kind: "error"; error: Error }
  >({ kind: "loading" });

  useEffect(() => {
    const controller = new AbortController();
    let ignore = false;

    setStatus({ kind: "loading" });
    fetch(`/api/products/${id}`, { signal: controller.signal })
      .then((r) => r.json() as Promise<Product>)
      .then((product) => {
        if (!ignore) setStatus({ kind: "success", product });
      })
      .catch((error: unknown) => {
        if (controller.signal.aborted) return; // aborts are not failures
        if (!ignore) setStatus({ kind: "error", error: error as Error });
      });

    return () => {
      ignore = true;
      controller.abort();
    };
  }, [id]);

  if (status.kind === "loading") return <Spinner />;
  if (status.kind === "error") return <RetryPanel error={status.error} />;
  return <ProductCard product={status.product} />;
}
```

Now in dev: two requests still *start*, but StrictMode's cleanup aborts the first before it resolves — the Network tab shows one `(canceled)` and one real response, and `setStatus` runs exactly once. In production under a rapid remount, the same cleanup cancels the stale in-flight request instead of letting it land after teardown. The status union (from [`thinking-in-react`](../../foundations/thinking-in-react.md#model-states-not-booleans)) means there's no `loading && !error && data` soup to get wrong while this races.

Note what did **not** change: the double *starts* in dev. Abort makes the double *harmless*, not absent. Making it truly absent is Stage 4's job.

### Stage 3 — The write path is a different bug

Aborting the `GET` was clean because a cancelled read has no consequence. The `POST .../views` is not clean, and this is the trap:

> A cancelled request may have already reached the server. `abort()` stops your client from *waiting*; it does not un-send the bytes.

So you cannot abort your way out of a double-count. Writes need one of two things instead:

- **Idempotency.** Give the request a client-generated dedup key the server honors, so a replay is a no-op. This is the same idea as the [`double-submit`](../forms-and-ux/double-submit-and-optimistic-like.md) recipe's server-idempotency half — a mount effect that double-fires and a button that double-clicks are the *same* class of write bug.
- **Relocation.** More often, a fire-and-forget side effect does not belong in a mount effect at all. A view beacon belongs at a stable boundary — a router loader, a `sendBeacon` on a navigation event, or server-side logging — none of which participate in StrictMode's mount/unmount churn.

For this scenario the beacon moves *out* of the component entirely, so there's no mount effect to double-fire in the first place:

```tsx
// A route loader runs once per navigation, not once per mount/remount.
// (Loaders are routing-react-router's territory — shown here only to place the write.)
export async function productLoader({ params }: LoaderFunctionArgs) {
  const product = await fetchProduct(params.id!);
  navigator.sendBeacon(`/api/products/${params.id}/views`); // fire-and-forget, once
  return product;
}
```

The double-count disappears at the source, not the symptom — and the read comes back from the loader already resolved, so `ProductDetail` renders without its own fetch at all.

### Stage 4 — Make the double *absent*: dedup by key

The read fix from Stage 2 is correct but still fires two requests in dev and pays two round-trips on any genuine remount. The production answer for **server state** is a query cache keyed by the request, which is exactly why the state-placement rule sends server state to TanStack Query and never to a client store:

```tsx
function ProductDetail({ id }: { id: string }) {
  const { data, error, isPending } = useQuery({
    queryKey: ["product", id],
    queryFn: ({ signal }) =>
      fetch(`/api/products/${id}`, { signal }).then((r) => r.json()),
  });

  if (isPending) return <Spinner />;
  if (error) return <RetryPanel error={error} />;
  return <ProductCard product={data} />;
}
```

Two mounts of the same `queryKey` while a request is in flight **share the single in-flight request** — the library dedupes by key, so StrictMode's double-mount produces exactly *one* network call, and the remount reads a warm cache instead of re-fetching. Abort is wired in through the `signal` the query passes to `queryFn`. The full API surface — `staleTime`, invalidation, `useSuspenseQuery`, the retry boundary — is [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md)'s territory (planned); here it's enough to see *why* the dedup lands where the effect couldn't.

## Variations

**One-shot init that must run once (analytics SDK, feature-flag bootstrap).** Genuinely once-per-app, remount-proof, and *not* server state. The honest tool is a module-scope guard outside React's lifecycle, or an init call in `main.tsx` before `render`, not an effect at all. Reach for a `useRef` "did I run" guard only if the init must be component-scoped — and know it's a last resort that masks missing cleanup for anything that isn't truly one-shot.

**Subscriptions, not fetches (WebSocket, SSE).** StrictMode double-connects the same way. The fix is the same shape (cleanup that tears the connection down) but the write-path warning bites harder: a dropped-then-reopened socket can miss messages. That's the `real-time/` track's `double-socket-connections` recipe (planned) — the dedup logic here doesn't transfer, because sockets aren't request/response.

**Prefetch on hover.** If you prefetch a product on link hover and then the user clicks, a naive effect fires a second request on mount. With a keyed cache this is free — the hover prefetch and the mount read collapse to one entry. Without one, you're back to hand-rolling an in-flight map (see pitfall 7).

## Trade-offs and common pitfalls

1. **Deleting `<StrictMode>` to silence the double.** The headline anti-fix. It disables a production-relevant diagnostic and ships the missing-cleanup bug it was warning about. If StrictMode noise is genuinely blocking you, fix the effect; the noise is the point.
2. **The `useRef` "hasFetched" guard for reads.** `if (ranRef.current) return;` makes the dev double vanish and feels clever. It also skips the fetch on a *legitimate* remount (new `id`, cache eviction, `<Activity>` reveal), stranding stale data on screen. It papers over exactly the cleanup bug you need to fix.
3. **Assuming the dev double ships to prod — or that prod is clean because dev is.** The double-invoke is dev-only; it will not appear in a build. Conversely, a *clean* dev run after you disabled StrictMode tells you nothing about prod remounts. Read the built preview's Network tab under real navigation, not the dev server's.
4. **Aborting a `POST`/mutation and assuming it didn't happen.** `abort()` cancels the client wait, not the server work. A cancelled write may have committed. Never treat aborted writes as rolled back.
5. **Abort without the ignore flag.** Abort stops most in-flight work, but a request can resolve in the microtask window before cleanup runs; without the `ignore` guard you still write state into an unmounted tree. Ship both, always.
6. **Forgetting to `return` the cleanup.** No returned function means no teardown means no protection — you've written the pattern and disabled it. The whole fix lives in the return.
7. **Hand-rolling a module-scope in-flight `Map` to dedup.** This reinvents a query cache badly: it leaks across tests (no reset between renders), never expires, and has no invalidation. If you're building a dedup table, you want the library.
8. **Keying the query without the variable.** `queryKey: ["product"]` (no `id`) dedupes across *different* products and serves whatever landed first. The key must contain everything the request depends on — same discipline as effect deps ("[deps are derived not chosen](../../rendering/memoization-and-the-compiler.md#how-it-works-under-the-hood)").
9. **Blaming StrictMode for every production double.** Once shipped, doubles come from remounts you caused — an unstable `key` recycling a subtree, a parent that unmounts on every render, deps that churn identity. Find the remount source; StrictMode is not in the building.
10. **"It works in prod without cleanup, so I'll drop it."** True until the first route transition remounts the component mid-request, or `<Activity>` reveals a hidden tree. Cleanup is not optional insurance; it's the contract.
11. **When NOT to reach for abort/dedup at all.** If the request is a genuine fire-and-forget side effect that must happen exactly once per user action — a purchase, a "record view" beacon, an audit log — do **not** contort a mount effect into behaving idempotently with abort gymnastics. The side effect is in the wrong home. Move it to a stable boundary (router loader/action, an event-fired mutation, `navigator.sendBeacon`, or the server). Mount effects are for *synchronizing with external systems*, not for *causing* one-time events; forcing the latter is how you got here.
12. **Expecting query dedup to cover writes.** The keyed cache dedupes *reads*. Mutations are not deduped by key the same way — two `useMutation` calls fire twice. Write idempotency is still your job, and it's the [`double-submit`](../forms-and-ux/double-submit-and-optimistic-like.md) recipe's whole subject.

## See also

- [`effects-and-synchronization`](../../effects/effects-and-synchronization.md) — owner of the StrictMode audit, the effect lifecycle, and the abort-and-ignore pattern this recipe applies
- [`search-race-condition`](./search-race-condition.md) — the *ordering* sibling: stale responses overwriting fresh ones (this recipe is the *dedup* sibling)
- [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md) — *(planned)* the keyed-cache resolution; owns dedup, `staleTime`, invalidation, and `useSuspenseQuery`
- [`double-submit-and-optimistic-like`](../forms-and-ux/double-submit-and-optimistic-like.md) — the write-idempotency half; a double-firing mount effect and a double-clicked button are the same bug
- [`how-react-renders`](../../rendering/how-react-renders.md#interruption-replay-and-why-strictmode-double-invokes) — why StrictMode double-invokes, mechanically

## References

- React — [`StrictMode`](https://react.dev/reference/react/StrictMode) (the "fixing bugs found by re-running effects" section)
- React — [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects) and [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- MDN — [`AbortController`](https://developer.mozilla.org/docs/Web/API/AbortController)
- TanStack Query — [Query docs](https://tanstack.com/query/latest) (request deduplication by query key)

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).