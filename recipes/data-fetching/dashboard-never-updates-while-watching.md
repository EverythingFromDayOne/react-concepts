---
recipe_id: dashboard-never-updates-while-watching
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - recipes/data-fetching/refetch-on-mount-always-spams
  - recipes/state-management/zustand-goes-stale
status:
  drafted: true
  reviewed: false
---

# The ops board sits stale while the operator stares at it

> **What you'll build:** an on-call dashboard that updates while someone is actively watching — without turning every idle tab into a polling botnet — by using `refetchInterval` only for live surfaces, and keeping the default "stale ≠ refetch" model everywhere else.

## The scenario

A status board shows open incidents. `staleTime: 60_000`. At T+61s the data is **stale**, but the operator never leaves the tab, never blurs the window, never remounts the route. Nothing refetches. A Sev-1 opened 40 seconds ago never appears until they click away and back. Ticket: *"the board doesn't update unless I refresh."*

Someone "fixes" it with `staleTime: 0` + default focus refetch — still **idle-tab silent**. Someone else sets `refetchInterval: 5_000` on the **root** client. The board works. So do 200 other queries in backgrounded tabs, each polling every 5s → API cost 10× overnight.

**Why it escaped QA:** demos always remount or switch tabs; focus refetch masks the idle path; load tests don't leave a single tab open for minutes.

## Walkthrough

### Stage 1 — Name it: stale is a flag, not a timer fire

[The staleness lifecycle](../../ecosystem/data-fetching-tanstack-query.md#how-it-works-under-the-hood) marks data stale after `staleTime`. Network work waits for a [trigger](../../ecosystem/data-fetching-tanstack-query.md#refetch-triggers-and-dials): focus, remount, reconnect, or **`refetchInterval`**. An operator who never triggers those events watches forever-stale UI. That is intentional resource policy — not a broken `staleTime`.

### Stage 2 — Reject the wrong dials

- **Shorter `staleTime` alone** — still no fetch until a trigger.
- **Global `refetchInterval`** — every query polls; battery and bill explode.
- **Zustand + `setInterval`** — hand-rolled racing cache ([zustand-goes-stale](../state-management/zustand-goes-stale.md)); you re-own abort/dedup/visibility.

### Stage 3 — Poll only the live query

```tsx
export function useIncidentBoard() {
  return useQuery({
    queryKey: ["incidents", "board"],
    queryFn: ({ signal }) => fetchIncidents(signal),
    staleTime: 10_000,
    refetchInterval: 15_000, // while this observer is mounted
    refetchIntervalInBackground: false, // pause when tab hidden
    refetchOnWindowFocus: true, // catch up immediately on return
  });
}
```

Leave catalogs on defaults (no interval). Prefer push (`WebSocket` → `queryClient.setQueryData`) when the backend already streams; interval is the HTTP fallback.

### Stage 4 — Harden + verify the loop

- Cap concurrency: one board query, not one interval per widget row.
- Disable interval when `document.visibilityState === 'hidden'` via `refetchIntervalInBackground: false` (default).
- On mutation (ack incident), `invalidateQueries` so you don't wait for the next tick.

**Verify the loop.** Open the board, leave focus on it, inject a server incident: within ~15s the row appears with no tab switch. Background the tab for a minute: Network shows **no** board polls. Refocus: one fetch. Product list in another route: still no interval traffic.

## Variations

1. **Conditional interval** — `refetchInterval: (q) => (q.state.data?.length ? 30_000 : 5_000)`.
2. **SSE/WebSocket primary** — interval only as heartbeat / reconnect fallback.
3. **`refetchIntervalInBackground: true`** — rare (alarm consoles); justify the cost.
4. **Per-widget intervals** — usually wrong; one board query, select in UI.
5. **Pause while `isMutating`** — avoid stomping optimistic rows mid-edit.

## Trade-offs and common pitfalls

1. **Expecting `staleTime` to poll** — it only expires freshness.
2. **Global interval** — cost disaster.
3. **Background polling defaulted on** — ghost tabs hammer prod.
4. **Interval without `staleTime` thought** — still fine; interval forces fetch regardless, but focus behavior still uses stale rules for *other* triggers.
5. **N widgets × interval** — N× load; consolidate.
6. **Ignoring invalidate after write** — UI lags up to one interval.
7. **Polling auth-sensitive PII on shared screens** — lock the machine; don't poll forever on a wall display without auth posture.
8. **Replacing Query with setInterval+useState** — lose dedup/cancel.
9. **Testing only with tab switches** — never exercises the idle path.
10. **5s interval "just to be safe"** — measure; 15–60s often enough for ops boards.
11. **Push available but still polling hard** — prefer push; poll as backup.

### When NOT to poll

If users leave and return (email-style apps), default focus/remount refetch is enough — polling wastes money. If the screen is a report opened once, invalidate on navigation instead. Poll when **someone is watching and silence is a bug**.

## See also

- [Refetch triggers and dials](../../ecosystem/data-fetching-tanstack-query.md#refetch-triggers-and-dials)
- [Refetch on mount always spams](./refetch-on-mount-always-spams.md) — commitment freshness without global storms
- [Zustand goes stale](../state-management/zustand-goes-stale.md) — why not invent a client-store poller

## References

- TanStack Query — [Disabling/Pausing Queries](https://tanstack.com/query/latest/docs/framework/react/guides/disabling-queries)
- TanStack Query — [Important Defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults)

## Demo source

- `demos/data-fetching/dashboard-never-updates-while-watching/` — idle tab vs interval vs background pause. *(Demo host TBD)*
