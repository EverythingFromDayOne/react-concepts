---
recipe_id: spinner-never-resolves-after-unmount
track: data-fetching
primary_concept: effects/effects-and-synchronization
difficulty: intermediate
react_baseline: "19.2"
related:
  - effects/effects-and-synchronization
  - ecosystem/data-fetching-tanstack-query
  - concurrent/suspense
status:
  drafted: true
  reviewed: false
---

# The spinner never resolves after you navigate away and back

> **What you'll build:** you'll take a tabbed dashboard whose Revenue tab spins forever after a fast tab-switch on a slow connection, find the stuck `loading: true` that outlived its request, and fix it by deriving loading from the request itself — first the right way with TanStack Query, then the correct hand-rolled union for when you can't use it. This is the third of the data-fetching trio: [ordering](./search-race-condition.md) and [dedup](./strictmode-double-mount.md) are the other two; this one is **abandonment**.

## The scenario

A tabbed analytics dashboard — Overview, Traffic, Revenue. Each tab fetches its dataset on mount. To keep each tab's data around when you switch away and back, the team hoisted per-tab `{ loading, data }` into a store:

```ts
// stores/dashboard.ts — the setup that will strand a spinner
import { create } from "zustand";
import type { RevenueData } from "../types";

interface DashboardState {
  revenue: { loading: boolean; data: RevenueData | null };
  setRevenue: (patch: Partial<DashboardState["revenue"]>) => void;
}

export const useDashboard = create<DashboardState>((set) => ({
  revenue: { loading: false, data: null },
  setRevenue: (patch) =>
    set((s) => ({ revenue: { ...s.revenue, ...patch } })),
}));
```

```tsx
// RevenueTab.tsx — every line looks correct
import { useEffect } from "react";
import { useDashboard } from "./stores/dashboard";

export function RevenueTab() {
  const { loading, data } = useDashboard((s) => s.revenue);
  const setRevenue = useDashboard((s) => s.setRevenue);

  useEffect(() => {
    if (loading || data) return; // don't refetch if loading or already loaded
    const controller = new AbortController();
    let ignore = false;

    setRevenue({ loading: true });
    fetch("/api/revenue", { signal: controller.signal })
      .then((r) => r.json())
      .then((json) => {
        if (!ignore) setRevenue({ loading: false, data: json });
      })
      .catch(() => {
        if (!ignore) setRevenue({ loading: false }); // skipped on abort
      });

    return () => {
      ignore = true;
      controller.abort();
    };
  }, [loading, data, setRevenue]);

  if (loading) return <Spinner />;
  return <RevenueChart data={data} />;
}
```

On a fast connection it's flawless. On a throttled phone: a user taps **Revenue** (fires a 1.4s fetch, sets `revenue.loading = true`), then taps back to **Overview** before it resolves. `RevenueTab` unmounts; the cleanup aborts the fetch — correct, per [effects-and-synchronization](../../effects/effects-and-synchronization.md#the-abortcontroller-pattern). The abort rejects the promise, the `ignore` guard (correctly) suppresses the stale write — so `setRevenue({ loading: false })` **never runs**. `revenue.loading` is now stuck `true` in the store. Tap **Revenue** again: the tab remounts, the effect's `if (loading || data) return` sees `loading: true`, skips the fetch, and renders `<Spinner />` — forever. Only a full reload clears it. Support tickets: *"Revenue just spins, I have to refresh."* Repros about 1 in 20 mobile sessions.

**Why it escaped QA:** on localhost the fetch resolves in ~4ms — faster than anyone can tap away — so the abort branch never runs in testing. The bug needs a slow request *and* a fast navigation, which only happens on real networks. StrictMode's dev double-mount exercises mount→unmount→mount, but the instant localhost resolve lands the data before the guard can matter.

## Walkthrough

### Stage 1 — Name it

The spinner is a `true` that never became `false`. Loading was **set imperatively** and **hoisted to outlive the component**, but the one code path that resets it (the request resolving) was correctly suppressed when the request was abandoned. Every piece is individually right; the bug is that request-derived state (loading) is being stored as a standalone flag with a reset path that an abandoned request skips. A flag you *set* can get stranded; a value you *derive* cannot.

### Stage 2 — The two naive fixes, both incomplete

**Reset loading in the cleanup:**

```ts
return () => {
  ignore = true;
  controller.abort();
  setRevenue({ loading: false }); // "just reset it on the way out"
};
```

This unsticks the obvious case, but you're now hand-syncing request status across the unmount boundary — and in StrictMode's dev remount it flashes the spinner off then on, and it still permits impossible states (`loading` and `data` both set during a refetch). You've patched a symptom of deriving loading imperatively, not the cause.

**Drop the `if (loading) return` guard** so a remount always refetches: now every tab-switch re-requests, and a slow earlier response can land after a newer one and overwrite it — the exact stale-response race the [search-race-condition recipe](./search-race-condition.md) exists to kill. You've traded a stuck spinner for a data race.

### Stage 3 — The real fix: derive loading from the request

Stop storing loading as a flag. [TanStack Query](../../ecosystem/data-fetching-tanstack-query.md) derives `status` from the query's actual state, so it *cannot* get stuck — there's no boolean to strand — and it dedupes and cancels for you:

```tsx
// useRevenue.ts
import { useQuery } from "@tanstack/react-query";
import type { RevenueData } from "./types";

async function fetchRevenue(signal: AbortSignal): Promise<RevenueData> {
  const res = await fetch("/api/revenue", { signal });
  if (!res.ok) throw new Error("Failed to load revenue");
  return res.json();
}

export function useRevenue() {
  return useQuery({
    queryKey: ["revenue"],
    queryFn: ({ signal }) => fetchRevenue(signal), // Query passes an abort signal
  });
}
```

```tsx
// RevenueTab.tsx — loading is now derived, and the store holds only "which tab"
import { useRevenue } from "./useRevenue";

export function RevenueTab() {
  const { data, isPending, isError } = useRevenue();

  if (isPending) return <Spinner />;
  if (isError) return <p role="alert">Couldn't load revenue.</p>;
  return <RevenueChart data={data} />;
}
```

Abandon the request by navigating away and `isPending` reflects the *live* query state on return; there is no stranded flag. The data survives the tab switch (Query caches it by key), so the "keep the data around" goal that motivated the store is met — without hoisting request status into it. That store now holds only genuine client state (the active tab), exactly as [zustand-goes-stale](../state-management/zustand-goes-stale.md) prescribes: request-derived state never belongs in a client store.

### Stage 4 — If you can't use Query: a status union

When a library isn't an option, keep the status **local** and model it as a discriminated union so the spinner is *derived*, and an abandoned request returns to `idle` instead of stranding `loading`:

```tsx
import { useEffect, useState } from "react";
import type { RevenueData } from "./types";

type RequestState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: RevenueData }
  | { status: "error"; error: string };

export function RevenueTab() {
  const [state, setState] = useState<RequestState>({ status: "idle" });

  useEffect(() => {
    const controller = new AbortController();
    let ignore = false;
    setState({ status: "loading" });

    fetch("/api/revenue", { signal: controller.signal })
      .then((r) => r.json())
      .then((data) => {
        if (!ignore) setState({ status: "success", data });
      })
      .catch((e) => {
        if (!ignore && e.name !== "AbortError") {
          setState({ status: "error", error: String(e) });
        }
      });

    return () => {
      ignore = true;
      controller.abort();
    };
  }, []); // local state unmounts with the component — nothing to strand

  if (state.status === "loading" || state.status === "idle") return <Spinner />;
  if (state.status === "error") return <p role="alert">{state.error}</p>;
  return <RevenueChart data={state.data} />;
}
```

Because the state is local, it's destroyed with the component — on remount it starts at `idle` and re-runs the effect cleanly. No hoisted flag, no survivor guard, no stuck spinner. (You lose Query's cross-switch caching; that's the trade for going library-free.)

**Verify against the incident.** Reproduce it: throttle to slow 3G, tap Revenue, immediately tap Overview, then tap Revenue again. Before: permanent spinner needing a reload. After (either fix): the request re-runs and resolves, because loading is read from the live request, not a flag that outlived it.

## Variations

- **Suspense reads.** A [suspending](../../concurrent/suspense.md) component that unmounts mid-fetch is fine *if* its promise still settles — ensure the abort path rejects the promise rather than leaving it forever pending, or the boundary can hang. Query's `useSuspenseQuery` handles this.
- **A global top-bar loader.** If you track a global in-flight *count* (increment on start, decrement on finish), decrement in a `finally` so an **aborted** request still decrements — otherwise the counter sticks above zero and the global bar spins forever. Same bug, one level up.
- **Route-level loaders.** With [React Router](../../ecosystem/routing-react-router.md) or TanStack Router, the router owns pending state and cancels the previous navigation's loader for you, so this failure mode mostly disappears — the framework resets pending on every navigation.

## Trade-offs and common pitfalls

1. **Hoisting request status into a store or context** gives it a lifetime longer than the request, and an abandoned request leaves it stranded. Keep ephemeral request status local or derived from the request.
2. **Resetting the flag in cleanup** unsticks the obvious case but flashes the spinner during StrictMode's dev remount and still allows impossible states. Derive, don't reset.
3. **A "fetch once" guard in a ref or module scope** survives unmount, so a remount skips the fetch and renders a spinner with no request behind it. Guards belong to the request lifecycle, not to module scope.
4. **The `ignore`-flag `finally`/`catch` that skips the reset** is exactly what strands the flag here. If you insist on an imperative flag, reset it even on abort — but prefer not to have the flag.
5. **No `AbortController` at all** flips the failure: the resolve runs after unmount and writes stale data (or, pre-React-18, warns about setting state on an unmounted component — React 18 removed that warning, so the write is now a silent stale-data bug). See [strictmode-double-mount](./strictmode-double-mount.md).
6. **Deriving `loading = !data && !error`** strands an abandoned request in "no data, no error, not loading" limbo — which many components render as a spinner. Model an explicit union with an `idle` state.
7. **A suspended promise that never settles on abort** hangs the Suspense boundary. Make the abort reject.
8. **A global in-flight counter missing its decrement on abort** sticks above zero. Always decrement in `finally`.
9. **Boolean soup** (`loading`, `data`, `error` as separate booleans) makes `loading && data` representable and stuck states easy. A discriminated union makes them unrepresentable.
10. **Timers you don't clear on unmount** (a retry `setTimeout`, a debounce) can fire after unmount and re-enter a stuck state. Clear every timer in cleanup.
11. **Reading the spinner from the hoisted flag instead of the live request** is the root shape of this whole class. Render loading from whatever actually knows if a request is in flight.

### When NOT to add abandonment handling

If the request is genuinely **fire-and-forget** — an analytics beacon, a best-effort log — you don't track loading at all, so there's no spinner and nothing to strand; adding lifecycle management to it is wasted ceremony (and `navigator.sendBeacon` is the better tool anyway). And if a component is structurally guaranteed **not to unmount mid-request** (a full-screen route with no in-flight navigation away), the elaborate abandonment path is technically unnecessary — but the safe patterns above cost almost nothing, so defaulting to them is still the right habit rather than reasoning case-by-case about who can unmount when.

## See also

- [Effects and synchronization](../../effects/effects-and-synchronization.md#the-abortcontroller-pattern) — the cleanup/abort contract this recipe builds on.
- [Data fetching with TanStack Query](../../ecosystem/data-fetching-tanstack-query.md) — derives status from the request, so it can't strand a spinner.
- [Recipe: search race condition](./search-race-condition.md) — the *ordering* half of the trio; the race the naive guard-removal reintroduces.
- [Recipe: StrictMode double-mount](./strictmode-double-mount.md) — the *dedup* half; the resolve-after-unmount sibling bug.
- [Recipe: server data in a Zustand store goes stale](../state-management/zustand-goes-stale.md) — why request-derived state doesn't belong in a client store.

## References

- React docs — [Synchronizing with Effects: fetching data](https://react.dev/learn/synchronizing-with-effects#fetching-data)
- React docs — [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- TanStack Query — [Query Cancellation](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation)

## Demo source

*Demo pending — see the roadmap's demo-hosting decision (StackBlitz vs. CodeSandbox vs. local `demos/`).*