---
recipe_id: back-button-scroll
track: routing
primary_concept: ecosystem/routing-react-router
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/routing-react-router
  - ecosystem/data-fetching-tanstack-query
  - recipes/routing/params-out-of-sync
  - recipes/performance/virtualization-long-list
status:
  drafted: true
  reviewed: false
---

# Hitting Back dumps the user at the top of the list instead of where they were

> **What you'll build:** scroll restoration that returns the user to exactly where they were on Back — item 47 in a long feed, not the top — while still starting new pages at the top, by letting the router save and restore scroll per history entry, loading list data before render so the target exists when restoration fires, and handling the cases the built-in can't (custom scroll containers, virtualized lists).

## The scenario

A product listing. The user scrolls down to item 47, clicks into a product's detail page, reads it, and hits **Back** — and lands at the **top** of the list, having to scroll all the way back down to find their place. On a long feed this is one of the most-reported UX complaints, and it quietly tanks engagement.

Two things conspire:

- **The browser's native scroll restoration can't work in an SPA.** On Back, the browser tries to restore scroll immediately — but the new route's content is rendered *asynchronously* (after the JS runs, after data fetches), so when the browser attempts to scroll to the saved offset, the list content doesn't exist yet. There's nothing at that position to scroll to.
- **Something scrolls to top on every navigation.** A common `useEffect(() => window.scrollTo(0, 0), [pathname])` — added to make *forward* navigation start at the top — also fires on Back, clobbering any restoration.

And the subtlety underneath: forward navigation (a **PUSH** — a genuinely new page) *should* start at the top, while Back/Forward (a **POP**) should *restore*. Treating both the same is the root of the bug in either direction.

Why it escaped QA:

- Test data is **short** — the list fits on screen, so there's no scroll to lose. Nobody tests Back on a long, scrolled list.
- **Dev is fast** — data loads instantly, so the restoration-timing race never surfaces.
- It's a **UX regression, not a functional bug** — every test passes; the page works, it just forgets where you were.

## Walkthrough

### Stage 1 — Name it: SPA content is async, and PUSH ≠ POP

Native browser restoration fails because the content isn't rendered when it fires, and a scroll-to-top-on-every-nav clobbers Back. Correct restoration needs three things: **save** the scroll position per history entry before leaving, **distinguish** PUSH (new page → top) from POP (Back/Forward → restore), and **restore** on POP *after* the target content has rendered. Miss any one and you either lose the position or apply it at the wrong time.

### Stage 2 — Let the router do it (the right default)

React Router's data mode ships `<ScrollRestoration />`, which does exactly the three things above — saves scroll keyed by location, restores on POP, tops on PUSH, and waits for the route's loader data before restoring. Add it once at the root:

```tsx
import { Outlet, ScrollRestoration } from "react-router";

export function Root() {
  return (
    <>
      <Outlet />
      <ScrollRestoration />
    </>
  );
}
```

That's the answer for most apps — don't hand-roll it. And **delete the `useEffect(() => window.scrollTo(0, 0))`**: it fights the restoration and is now redundant (the router tops PUSH navigations for you). The router also sets `history.scrollRestoration = "manual"` so the browser's native attempt doesn't race it.

### Stage 3 — Make the target exist before restoration fires: load data in the loader

`<ScrollRestoration />` restores *after* the route's [loader data is ready](../../ecosystem/routing-react-router.md#step-1--loader-with-params-deferred-non-critical-data). So if the list data is fetched in the route loader, the rows are rendered by the time restoration runs and the saved offset lands correctly:

```tsx
// list route — data ready before render AND before restoration
export async function loader() {
  return { products: await getProducts() };
}
```

The common failure is fetching the list in a **client** `useQuery` *after* render: at restoration time the list is still empty, so the saved offset overshoots to the bottom of a short page, then jumps when the data arrives. Moving the fetch into the loader (or pre-populating the [Query cache in the loader](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns)) closes the race.

### Stage 4 — Custom containers, virtualized lists, and verify

`<ScrollRestoration />` restores `window` scroll. For a scroll container that isn't the window, or a virtualized list where the target row isn't mounted, you need manual control — save keyed by `location.key`, restore on POP in a layout effect:

```tsx
import { useLocation, useNavigationType } from "react-router";
import { useLayoutEffect, useEffect, type RefObject } from "react";

const positions = new Map<string, number>();

export function useScrollRestoration(ref: RefObject<HTMLElement>) {
  const location = useLocation();
  const navType = useNavigationType(); // "PUSH" | "POP" | "REPLACE"

  useEffect(() => {
    const el = ref.current;
    return () => { if (el) positions.set(location.key, el.scrollTop); }; // save on leave
  }, [location.key, ref]);

  useLayoutEffect(() => {
    const el = ref.current;
    if (!el) return;
    el.scrollTop = navType === "POP" ? positions.get(location.key) ?? 0 : 0; // restore on POP, top on PUSH
  }, [location.key, navType, ref]);
}
```

For a [virtualized list](../performance/virtualization-long-list.md), restore the virtualizer's offset (`scrollToOffset(saved)`) rather than the container's `scrollTop`, since the target row isn't in the DOM until the virtualizer scrolls near it. **Verify the loop.** Scroll to item 47, open a detail, hit Back → you land back at item 47, not the top. Forward-navigate to a fresh list → it starts at the top. A loader-fetched list restores cleanly with no flash-to-top-then-jump. A custom scroll panel restores its own `scrollTop`. The "it forgot where I was" complaint is closed.

## Variations

1. **Router `<ScrollRestoration />`** — the default: save/restore per key, PUSH-top / POP-restore, data-aware. Prefer it.
2. **Load list data in the loader** — so the rows exist when restoration fires; the fix for the client-fetch race.
3. **Custom scroll container** — restore the element's `scrollTop`, not `window` scroll, with the manual hook.
4. **Virtualized list** — restore the virtualizer's `scrollToOffset`; the row re-measures as it mounts.
5. **Infinite scroll** — the genuinely hard case: the items at the saved position aren't loaded on Back. Either persist the loaded pages (keep them in the cache so Back re-renders them) or accept top-of-list. Be honest about which.

## Trade-offs and common pitfalls

1. **`useEffect(() => window.scrollTo(0, 0), [pathname])`** — clobbers Back restoration by topping on POP too. Let the router top PUSH navigations.
2. **Relying on native `history.scrollRestoration = "auto"` in an SPA** — the content isn't rendered when it fires. The router owns restoration (`manual`).
3. **Not distinguishing PUSH from POP** — either always-top (loses Back) or always-restore (a new page starts mid-scroll).
4. **Restoring before the content is rendered** — overshoot/undershoot. Restore after the loader data is ready.
5. **Fetching the list in a client `useQuery` after render** — it's empty at restoration time. Load in the loader.
6. **Restoring `window` scroll for a custom container** — nothing happens. Restore the container's `scrollTop`.
7. **Virtualized list restoring `window` scroll** — the rows aren't mounted. Restore the virtualizer offset.
8. **Infinite scroll expecting restoration to an unloaded page** — nothing to restore to. Persist pages or accept top.
9. **Both `<ScrollRestoration />` and a manual hook** — they fight. Pick one per scroll region.
10. **Saving scroll too late** (after the reset) — the position is already gone. Save on leave / before unmount.
11. **Keying by `pathname` instead of `location.key`** — the same path revisited restores the wrong entry's position. Key by history entry.
12. **Testing only on short lists** — the whole bug is invisible without a long, scrolled list.

### When NOT to restore

If pages are **short** — everything fits on screen with no meaningful scroll — restoration buys nothing; there's nothing to save. And a genuinely **new page** reached by forward navigation should start at the **top**, not restore — restoring on PUSH is the opposite bug. If you're on React Router data mode, `<ScrollRestoration />` already handles the common case, so **don't hand-roll** — reach for the manual hook only for what it can't cover (non-window containers, virtualized lists, per-route policy). The test: *do users scroll meaningfully and return via Back to a position they'd want preserved?* If yes, restore (prefer the built-in). Short pages, or always-top-on-return, need nothing.

## See also

- [`routing-react-router`](../../ecosystem/routing-react-router.md#real-world-patterns) — data-mode navigation, `<ScrollRestoration />`, and loaders that make content ready before restoration.
- [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns) — pre-populating the Query cache in a loader so the list renders before restoration fires.
- [`params-out-of-sync`](./params-out-of-sync.md) — the sibling routing bug: URL as the source of truth, and why Back restores the URL but not mirrored state.
- [`virtualization-long-list`](../performance/virtualization-long-list.md) — restoring scroll in a virtualized list (offset, not `scrollTop`).
- `routing/lazy-route-flashes-blank` — the sibling lazy-route bug *(planned)*.

## References

- React Router — `<ScrollRestoration />`, `useNavigationType`, `getKey`.
- MDN — `history.scrollRestoration` and the browser's scroll-restoration behavior.
- web.dev — scroll restoration and back/forward cache interactions.

## Demo source

- `demos/routing/back-button-scroll/` — a long product list that loses its scroll on Back, then `<ScrollRestoration />` + loader-loaded data, plus a manual hook for a custom scroll panel and a virtualized variant. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*