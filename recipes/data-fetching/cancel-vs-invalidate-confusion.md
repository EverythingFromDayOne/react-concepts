---
recipe_id: cancel-vs-invalidate-confusion
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: foundational
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - recipes/forms-and-ux/double-submit-and-optimistic-like
  - recipes/data-fetching/search-race-condition
status:
  drafted: true
  reviewed: false
---

# After "cancel" the list never refreshes — or after invalidate a ghost write returns

> **What you'll build:** muscle memory for two Query verbs — `cancelQueries` stops in-flight work; `invalidateQueries` marks stale and refetches — used together in the optimistic mutation contract so you neither freeze on cancelled data nor lose an optimistic write to a late refetch.

## The scenario

Two bugs from the same team in one week:

**A.** A "Refresh" button calls `queryClient.cancelQueries({ queryKey: ['orders'] })`. The spinner stops. The list **never** updates — support still sees yesterday's orders until a full reload. Dev thought cancel meant "drop cache and reload."

**B.** An optimistic "Mark shipped" uses `setQueryData` in `onMutate` but skips `cancelQueries`. A background refetch that started before the click resolves **after** the optimistic write and paints the old row. Flicker: shipped → unshipped → shipped after a manual refresh. They "fixed" it by removing optimism.

**Why it escaped QA:** cancel "feels" like it did something (in-flight abort); invalidate is tested only on the happy mutation path; the race in B needs an overlapping refetch (focus/remount) during the click — rare in click-scripted tests.

## Walkthrough

### Stage 1 — Name the two verbs

From [Cancel vs invalidate](../../ecosystem/data-fetching-tanstack-query.md#cancel-vs-invalidate):

| | `cancelQueries` | `invalidateQueries` |
| --- | --- | --- |
| Intent | Stop / ignore in-flight | Mark stale + refetch active |
| New network? | No | Yes (if observers active) |
| Typical moment | `onMutate`, Abort button | `onSettled` after writes |

Cancel without invalidate → no refresh. Invalidate without cancel during optimism → late GET can win the race ([mistake 6](../../ecosystem/data-fetching-tanstack-query.md#common-mistakes)).

### Stage 2 — Reject the confused APIs

- **Refresh button = only cancel** — aborts; does not refetch. Use `invalidateQueries` or `refetchQueries`.
- **Optimism = only `setQueryData`** — missing cancel invites the flicker in B.
- **`removeQueries` as refresh** — drops cache → full pending spinner; heavier than invalidate.

### Stage 3 — The real fix (both call sites)

**Refresh control:**

```tsx
function RefreshOrders() {
  const qc = useQueryClient();
  return (
    <button type="button" onClick={() => qc.invalidateQueries({ queryKey: ["orders"] })}>
      Refresh
    </button>
  );
}
```

**Optimistic ship (contract from the article walkthrough):**

```tsx
useMutation({
  mutationFn: shipOrder,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ["orders"] });
    const previous = queryClient.getQueryData(["orders"]);
    queryClient.setQueryData(["orders"], (old: Order[] | undefined) =>
      old?.map((o) => (o.id === id ? { ...o, status: "shipped" } : o)),
    );
    return { previous };
  },
  onError: (_e, _id, ctx) => {
    if (ctx?.previous) queryClient.setQueryData(["orders"], ctx.previous);
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ["orders"] }),
});
```

### Stage 4 — Harden + verify the loop

- Prefer `onSettled` over `onSuccess` for invalidate ([mistake 7](../../ecosystem/data-fetching-tanstack-query.md#common-mistakes)).
- Manual cancel buttons for **downloads** still use `cancelQueries` — then optionally invalidate if you want a clean idle state.

**Verify the loop.** A: click Refresh → Network GET `/orders`, UI updates. B: start a slow refetch (throttle), click Ship mid-flight → row stays shipped (no flicker to old); on error, rolls back then resettles from server.

## Variations

1. **`refetchQueries` on Refresh** — force fetch even if fresh; invalidate only marks stale first.
2. **Partial key cancel** — `cancelQueries({ queryKey: ['orders', id] })` during row edit.
3. **React `useOptimistic`** for local-only toggles — [double-submit recipe](../forms-and-ux/double-submit-and-optimistic-like.md); cache optimism still needs cancel.
4. **Logout teardown** — `cancelQueries` then `clear()` ([logout-leaves-stale-cache](../auth/logout-leaves-stale-cache.md)), not invalidate (which would refetch).
5. **Search key change** — Query auto-cancels prior key; still wire `signal` ([search-race](./search-race-condition.md)).

## Trade-offs and common pitfalls

1. **Cancel ≟ refresh** — the core confusion.
2. **Invalidate on logout** — refetches into a dying session; use `clear`.
3. **Optimism without cancel** — flicker.
4. **Invalidate only on success** — failed mutation skips resync.
5. **Broad cancel** — cancels unrelated in-flight UI; scope keys.
6. **Ignoring abort in `queryFn`** — cancel marks Query cancelled but HTTP continues ([search-race](./search-race-condition.md)).
7. **`setQueryData` after invalidate without await** — races; settle order matters.
8. **Teaching cancel as "clear cache"** — it doesn't remove data.
9. **Double invalidate storms** — debounce rapid mutation bursts if needed.
10. **No verify with throttling** — race B never appears on localhost.

### When NOT to cancel

Don't cancel a **payment** mutation mid-flight from a route change — writes aren't auto-aborted for a reason ([upload-cant-be-cancelled](./upload-cant-be-cancelled.md)). Cancel reads freely; cancel writes only with an explicit user "Abort" and server idempotency.

## See also

- [Cancel vs invalidate](../../ecosystem/data-fetching-tanstack-query.md#cancel-vs-invalidate)
- [Stage 5 optimistic walkthrough](../../ecosystem/data-fetching-tanstack-query.md#stage-5--make-it-optimistic-with-rollback)
- [Double-submit and optimistic like](../forms-and-ux/double-submit-and-optimistic-like.md)

## References

- TanStack Query — [Query Cancellation](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation)
- TanStack Query — [Invalidations from Mutations](https://tanstack.com/query/latest/docs/framework/react/guides/invalidations-from-mutations)

## Demo source

- `demos/data-fetching/cancel-vs-invalidate-confusion/` — refresh button + optimistic race with/without cancel. *(Demo host TBD)*
