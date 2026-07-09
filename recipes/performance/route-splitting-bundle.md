---
recipe_id: route-splitting-bundle
track: performance
primary_concept: ecosystem/performance-profiling
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/performance-profiling
  - concurrent/suspense
  - ecosystem/routing-react-router
  - rendering/error-boundaries
  - recipes/performance/typing-lag-rerender-storm
status:
  drafted: true
  reviewed: false
---

# A 900KB bundle loads before anyone sees the login screen

> **What you'll build:** an app whose initial download contains only what the first screen needs — the rest (each route, the charting library, the rich-text editor, the PDF exporter) split into chunks that load on navigation, interaction, or visibility — turning a 900KB entry bundle and a 6-second mobile TTI into a shell that paints fast and pulls the heavy parts in on demand.

## The scenario

A SaaS dashboard SPA on Vite. The first screen is a simple login, but the initial JavaScript bundle is **900KB gzipped**. On a mid-tier phone on 4G, time-to-interactive is ~6 seconds and LCP is in the poor range; a measurable slice of users bounce before the login form is usable.

The bundle is one chunk because everything is imported eagerly: the router statically imports every route component at the top (`import Dashboard from "./routes/Dashboard"`), and heavy dependencies — a charting library, a rich-text editor used only on a settings page, a PDF exporter, a large date/i18n bundle — are imported at module top level. The bundler puts anything reachable by a static import from the entry into the entry chunk, so the login screen downloads the entire application before it can render.

Why it escaped QA:

- On the **dev machine** — fast CPU, localhost, warm cache — 900KB loads instantly. Nobody feels it.
- **Lighthouse isn't in the QA loop**; functional tests pass no matter the bundle size.
- It **creeps**: each feature adds a dependency to the eager graph, the bundle grows a little per PR, invisibly, until it's 900KB. No single change looks bad.
- Only real users on mobile and slow networks feel it — and the team, on fast machines, doesn't.

## Walkthrough

### Stage 1 — Name it: the entry chunk ships code the first screen never runs

Everything in the entry chunk downloads and parses before first paint, even code the first screen never executes. A module lands in the entry chunk when it's reachable by a **static import** from the entry — so a route component or a heavy library imported at module top is eager, whether or not the first screen uses it. The goal: the initial bundle contains only what the **first meaningful paint** needs; everything else loads on demand — a route change, an interaction, or scrolling into view. And [measure first](../../ecosystem/performance-profiling.md#real-world-patterns) — the bundle analyzer shows exactly what's in the entry chunk and which dependencies dominate it, so you split the things that actually matter instead of guessing.

### Stage 2 — Split by route (the highest-leverage cut)

Turn static route imports into lazy ones, each behind a Suspense boundary — [`lazy` + `<Suspense>` is code splitting](../../concurrent/suspense.md#code-splitting-with-lazy):

```tsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./routes/Dashboard"));
const Settings = lazy(() => import("./routes/Settings"));

// <Route path="/dashboard" element={<Suspense fallback={<RouteSkeleton />}><Dashboard /></Suspense>} />
```

Or let the router own it — [React Router v7's route-level `lazy`](../../ecosystem/routing-react-router.md#step-4--wire-it-up-lazily) code-splits each route without hand-wiring `React.lazy`:

```tsx
{ path: "dashboard", lazy: () => import("./routes/Dashboard") }
```

Now every route is its own chunk, fetched on navigation; the entry chunk shrinks to the shell plus the login screen. Give each lazy route a **skeleton** fallback, not a spinner ([one boundary per lazy region](../../concurrent/suspense.md#stage-1--lazy-panel-one-boundary)). Leave the *first* route eager — splitting the critical path just adds a round-trip before first paint.

### Stage 3 — Split heavy libraries behind interaction and visibility

Even one route can be heavy. A charting library needed only when a tab opens, an editor only when "Edit" is clicked, a PDF exporter only on "Export" — none belongs in that route's initial payload. Load them on demand:

```tsx
// a 200KB PDF library that loads only when the user actually exports
async function handleExport() {
  const { exportPdf } = await import("./lib/pdf");
  exportPdf(data);
}

// a heavy chart rendered only when its tab is active
const RevenueChart = lazy(() => import("./RevenueChart"));
{tab === "revenue" && (
  <Suspense fallback={<ChartSkeleton />}>
    <RevenueChart data={data} />
  </Suspense>
)}
```

For below-the-fold heavy components, trigger the lazy load on visibility (an `IntersectionObserver` that flips a flag). The rule is the same: reachable-only-on-demand code should be dynamically imported, not sitting in a route's first byte.

### Stage 4 — Prefetch to hide the latency, and verify

Splitting trades bundle size for a fetch delay on navigation. Hide it by **prefetching** the next likely chunk before the user commits — on link hover/focus, on idle, or via [the router's intent-based prefetch](../../ecosystem/routing-react-router.md#real-world-patterns):

```tsx
<Link to="/dashboard" onMouseEnter={() => import("./routes/Dashboard")}>Dashboard</Link>
```

Two guardrails. Don't **over-split** — a chunk per component means request overhead and load waterfalls; split at meaningful boundaries (routes, heavy libraries), not everything. And handle the **failed chunk**: after a deploy, a stale page can reference an old chunk hash that now 404s — the [stale-chunk case](../../concurrent/suspense.md#stage-4--the-stale-chunk-case), caught by an [error boundary that offers a reload](../../rendering/error-boundaries.md#the-stale-chunk-fallback) rather than a white screen.

**Verify the loop.** Run the bundle analyzer before and after: the entry chunk drops from ~900KB to ~150KB (the heavy libraries are now separate on-demand chunks), and a throttled [Lighthouse / `web-vitals` run](../../ecosystem/performance-profiling.md#real-world-patterns) shows TTI and LCP move out of the poor range on mid-tier mobile. Navigation to a prefetched route is instant; a cold route shows its skeleton then content; an exported PDF pulls its library only on click. The login screen paints without downloading the dashboard.

## Variations

1. **Router-owned route split** (React Router `lazy`) vs manual `React.lazy` + `<Suspense>` — the router version co-locates the split with the route and can also lazy-load its loader/action.
2. **Heavy lib behind interaction** — `await import()` inside the handler for the exporter/editor/chart that only a fraction of sessions ever trigger.
3. **Visibility-triggered lazy** — an `IntersectionObserver` loads a below-the-fold heavy component only when it scrolls near the viewport.
4. **Prefetch on intent** — hover/focus/idle prefetch so the split is invisible; the router can prefetch on link intent.
5. **Failed-chunk resilience** — an error boundary around lazy regions that catches a 404'd stale chunk and prompts a reload, so a deploy mid-session doesn't blank the app.

## Trade-offs and common pitfalls

1. **Static-importing every route at the top** — one entry chunk with the whole app. Lazy per route.
2. **A heavy library imported at module top level** — in the entry chunk even if one route uses it. Dynamic-import it where it's used.
3. **No Suspense boundary around a lazy route** — the tree suspends too high (or throws). Wrap each lazy region.
4. **Over-splitting** — a chunk per component adds request overhead and waterfalls. Split at routes and heavy libs.
5. **No prefetch** — every navigation has a visible chunk-load delay. Prefetch on intent.
6. **A lazy chunk that 404s after a deploy** — the stale-chunk white screen. Error boundary + reload.
7. **A spinner fallback that flashes** on a fast chunk — use a skeleton, or delay the fallback.
8. **Shared heavy deps duplicated across chunks** — ensure the bundler hoists shared deps into a common chunk instead of copying them.
9. **Measuring on a fast dev machine** — the bloat is invisible. Bundle analyzer + throttled Lighthouse.
10. **Barrel/whole-library imports** (`import { X } from "huge-lib"` that isn't tree-shakeable) — import the specific submodule or a lighter alternative.
11. **Splitting the first/critical route** — adds a round-trip before first paint for no benefit. Keep the critical path eager.
12. **Not measuring after** — assuming the split helped. Confirm the entry chunk actually shrank.

### When NOT to split

If the app is **small** and the bundle is already lean (well under ~150KB) — a landing page, a focused tool — code splitting adds complexity, request overhead, and loading states for no real gain; ship one chunk. And the **first meaningful paint's critical path** should never be split, because splitting it just inserts a network round-trip before the user sees anything. The test: *is this code needed for the first meaningful paint?* If yes, keep it eager. If no, split it — and measure to confirm the entry chunk actually got smaller.

## See also

- [`performance-profiling`](../../ecosystem/performance-profiling.md#real-world-patterns) — measuring the bundle and the LCP/TTI impact before and after, so you split what matters.
- [`suspense`](../../concurrent/suspense.md#code-splitting-with-lazy) — `lazy` + `<Suspense>`, boundary placement, and the stale-chunk case.
- [`routing-react-router`](../../ecosystem/routing-react-router.md#step-4--wire-it-up-lazily) — route-level `lazy` and intent-based prefetch.
- [`error-boundaries`](../../rendering/error-boundaries.md#the-stale-chunk-fallback) — recovering from a chunk that 404s after a deploy.
- [`typing-lag-rerender-storm`](./typing-lag-rerender-storm.md) — the other half of the performance track (render cost, not download cost).

## References

- Vite / Rollup — dynamic `import()` and manual vs automatic chunking.
- MDN — dynamic `import()` and `<link rel="prefetch">`.
- web.dev — reducing JavaScript payloads with code splitting; Core Web Vitals (LCP, INP).

## Demo source

- `demos/performance/route-splitting-bundle/` — the 900KB single-chunk app, then route splitting + on-demand heavy libs + intent prefetch, measured with the bundle analyzer and a throttled Lighthouse run. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*