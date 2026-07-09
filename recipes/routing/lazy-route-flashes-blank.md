---
recipe_id: lazy-route-flashes-blank
track: routing
primary_concept: ecosystem/routing-react-router
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/routing-react-router
  - concurrent/suspense
  - concurrent/concurrent-rendering
  - recipes/routing/back-button-scroll
  - recipes/performance/route-splitting-bundle
status:
  drafted: true
  reviewed: false
---

# Clicking a lazy route blanks the screen before the page appears

> **What you'll build:** navigation to a code-split route that never flashes blank — the current page stays visible with a subtle pending indicator while the new chunk and its data load, the chunk is warm because you prefetched it on hover, and fast loads show no spinner at all — instead of unmounting the old page and staring at white space until the new one is ready.

## The scenario

A dashboard app with route-level code splitting (`lazy(() => import("./Reports"))` + `<Suspense>`). The user clicks "Reports": the current dashboard **unmounts immediately**, a blank screen (or a bare spinner) shows while the ~80KB chunk downloads and the route's data fetches, and then Reports appears. On the office network that blank is ~250ms and reads as a flash; on a slower connection it's a second or more of nothing. Users report the app "flashes" or "feels broken."

There's a second, opposite flavor of the same bug: on a *fast* or cached navigation, the spinner appears and vanishes in ~80ms — a flicker that's more jarring than showing nothing.

What's happening:

- **The old UI is replaced by the fallback instantly.** Without a transition, when the target route suspends, React swaps the old page for the Suspense fallback right away — so you see the fallback (blank/spinner) instead of the page you were just on.
- **A chunk→data waterfall doubles the gap.** If the route fetches its data in a `useEffect` after the chunk mounts, it's chunk-download *then* fetch — two serial waits, two beats of nothing.
- **A too-eager fallback flickers.** A spinner with no delay flashes even when the chunk is already cached.

Why it escaped QA:

- In **dev**, modules are served instantly (Vite dev is bundleless), so there's no chunk to wait on and no flash.
- On a **fast network** the blank is ~50ms and gets dismissed as fine.
- It's **perceived jank, not a functional bug** — the route does load; every test passes.
- Real users on slower networks and uncached first visits feel the full blank.

## Walkthrough

### Stage 1 — Name it: navigation replaced the old page with a fallback

On navigation to a lazy route, the old page unmounts and Suspense shows its fallback while the chunk and data load — that's the flash. React's default, *without a transition*, is to show the fallback immediately. The fix concept: a navigation should be a **transition** that keeps the old page visible until the new one is ready ([transitions hold the current content instead of flashing a fallback](../../concurrent/concurrent-rendering.md#real-world-patterns)), plus warming the chunk before the click so there's little to wait for. And a fallback that appears for under ~200ms is worse than none — it should be delayed or skipped.

### Stage 2 — Keep the old page during navigation (the core fix)

React Router **data mode does this by default**: a navigation is a transition, the old route stays mounted while the loader and the lazy chunk resolve, and `useNavigation().state` drives a pending indicator instead of a blank. No flash by construction:

```tsx
// routes: the lazy chunk and the loader resolve in parallel; the old page stays until ready
{ path: "reports", lazy: () => import("./Reports"), loader: reportsLoader }
```

```tsx
function Root() {
  const navigation = useNavigation();
  return (
    <>
      {navigation.state === "loading" && <TopProgressBar />} {/* subtle pending hint */}
      <Outlet />
    </>
  );
}
```

If you're on a **manual `React.lazy` + `<Suspense>`** setup instead, wrap the navigation in a transition so Suspense keeps showing the *old* children rather than swapping to the fallback ([`startTransition` is exactly this](../../concurrent/concurrent-rendering.md#starttransition--the-standalone-function)):

```tsx
const [isPending, startTransition] = useTransition();
function go(to: string) {
  startTransition(() => navigate(to)); // old page stays; isPending drives the pending bar
}
```

### Stage 3 — Prefetch the chunk on intent, and parallelize the data

Keeping the old page removes the *blank*; prefetching removes the *wait*. Warm the chunk on hover/focus so it's ready by the click ([the prefetch-on-intent pattern](../performance/route-splitting-bundle.md)):

```tsx
<Link to="/reports" onMouseEnter={() => import("./Reports")}>Reports</Link>
```

And kill the chunk→data waterfall: in a data router the [loader runs in parallel with the lazy chunk download](../../ecosystem/routing-react-router.md#step-1--loader-with-params-deferred-non-critical-data), so data isn't a second serial wait behind the chunk. Fetching route data in a `useEffect` after mount is the waterfall; move it into the loader.

### Stage 4 — Tame the fallback flicker, handle stale chunks, and verify

When a fallback *is* shown — a genuinely slow load, or a cold first paint — avoid the sub-200ms flicker by delaying it, so fast loads show nothing:

```tsx
function DelayedFallback({ delay = 200 }: { delay?: number }) {
  const [show, setShow] = useState(false);
  useEffect(() => {
    const id = setTimeout(() => setShow(true), delay);
    return () => clearTimeout(id);
  }, [delay]);
  return show ? <RouteSkeleton /> : null; // skeleton matches the layout — no CLS jump on swap
}
```

Use a **skeleton that matches the route's layout** (not a centered spinner) so the swap-in isn't a jarring reflow. And handle the **stale chunk**: after a deploy a page can reference an old chunk hash that now 404s — the [stale-chunk case](../../concurrent/suspense.md#stage-4--the-stale-chunk-case), caught by an [error boundary that prompts a reload](../../rendering/error-boundaries.md#the-stale-chunk-fallback).

**Verify the loop.** On a throttled connection, click a lazy route: the previous page stays with a pending bar (no blank), the chunk was prefetched on hover so it arrives fast, and the loader data came in parallel (no double wait). A fast or cached navigation shows no spinner flicker. A slow load shows a layout-matched skeleton after 200ms, not a flash. The "it flashes" complaint is gone.

## Variations

1. **Data-router pending navigation** (`useNavigation().state`) — the old UI stays with a pending bar; the flash-free default.
2. **Manual `React.lazy` + `startTransition`** — wrap the navigate so Suspense holds the old children through the transition.
3. **Prefetch on intent** — hover/focus prefetch so the chunk is warm at click ([route-splitting](../performance/route-splitting-bundle.md)).
4. **Delayed / minimum-display fallback** — no spinner under ~200ms; a skeleton that matches the layout when shown.
5. **Failed-chunk resilience** — an error boundary around the lazy route that catches a 404'd stale chunk and reloads.

## Trade-offs and common pitfalls

1. **No Suspense boundary around the lazy route** — it suspends too high and the whole app shows a fallback. Scope a boundary to the route.
2. **Navigating without a transition** — the old UI is replaced by the fallback immediately (the flash). Data router, or `startTransition`.
3. **A bare spinner fallback** — flickers on fast/cached loads. Delay it, or use a skeleton.
4. **A chunk→data waterfall** — data fetched after the chunk mounts doubles the gap. Parallelize with the loader.
5. **A fallback with a different layout than the content** — CLS jump on swap. Match the skeleton to the layout.
6. **A fallback that flashes and vanishes** (<200ms) — worse than nothing. Delay / minimum-display.
7. **Not prefetching** — every click pays the full chunk load. Prefetch on hover/focus.
8. **A stale chunk 404 after a deploy** — blank or crash. Error boundary + reload.
9. **A too-coarse Suspense boundary** — a slow sub-part blanks the whole route. Scope boundaries to what can load slowly.
10. **Relying on the browser default in a non-data router** — no transition means it always flashes. Use `startTransition`.
11. **Fetching route data in `useEffect` after mount** — the chunk-then-effect waterfall, and no pending state. Use a loader.
12. **Testing only on a fast network** — the flash is invisible on dev/office speeds. Throttle.

### When NOT to worry about the flash

If your routes are **not code-split** (eager), there's no chunk to wait on and no flash — this is specifically a lazy-route problem. And if the case is a **cold first load or a direct deep-link** rather than an in-app navigation, there is *no old UI to keep* — the transition trick doesn't apply, and the right answer is a good layout-matched skeleton (plus route-splitting and, for content pages, [SSR to fix LCP](../performance/lcp-regression-client-only.md)). The test: *is this an in-app navigation (an old page exists to hold) or a cold load (nothing to hold)?* In-app → transition + prefetch, no flash. Cold load → a skeleton, and consider SSR.

## See also

- [`routing-react-router`](../../ecosystem/routing-react-router.md#real-world-patterns) — data-mode navigation as a transition (`useNavigation`), route `lazy`, and loaders that parallelize data with the chunk.
- [`concurrent-rendering`](../../concurrent/concurrent-rendering.md#real-world-patterns) — why a transition holds the current content instead of flashing a fallback.
- [`suspense`](../../concurrent/suspense.md#code-splitting-with-lazy) — `lazy` + `<Suspense>`, boundary placement, and the stale-chunk case.
- [`back-button-scroll`](./back-button-scroll.md) — the sibling routing bug (scroll restoration on Back).
- [`route-splitting-bundle`](../performance/route-splitting-bundle.md) — the code-splitting and prefetch mechanics this recipe smooths over.

## References

- React Router — `useNavigation`, route `lazy`, and pending navigation UI.
- React — `startTransition` / `useTransition` and Suspense fallback behavior during transitions.
- MDN — dynamic `import()` and `<link rel="prefetch">`.

## Demo source

- `demos/routing/lazy-route-flashes-blank/` — a route-split app flashing blank on navigation, then a pending-bar transition + hover prefetch + a 200ms-delayed layout-matched skeleton. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*