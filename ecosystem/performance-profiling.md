---
article_id: performance-profiling
concept_folder: ecosystem
wave: 4
related:
  - rendering/memoization-and-the-compiler
  - rendering/react-compiler-deep-dive
  - concurrent/concurrent-rendering
  - rendering/how-react-renders
  - effects/escape-hatches-audit
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Performance profiling

> You do not make React fast by sprinkling `useMemo`. You make it fast by measuring first, finding the one component that is actually dragging the tree, and confirming the number moved after you touched it. Every tool in this article exists to answer one of three questions: *what rendered*, *how much did it cost*, and *did the user feel it*. Guessing at any of those is how teams spend a sprint memoizing components that were never the problem.

## What it is

Profiling is measurement, not optimization. The optimization is a separate, later step that measurement earns you the right to take. In React there are two layers to measure, and conflating them is the most common way profiling goes wrong.

The **React layer** answers *what rendered and how much did it cost*: which components re-rendered on an interaction, why, and how many milliseconds each render took. The React DevTools Profiler, the `<Profiler>` component, React Scan, and — new in 19.2 — React's custom Chrome Performance Tracks all live here.

The **browser layer** answers *what did the user actually feel*: Core Web Vitals — Largest Contentful Paint, Interaction to Next Paint, Cumulative Layout Shift — measured with the Chrome Performance panel in the lab and the `web-vitals` library in the field.

The two layers do not always agree, and that disagreement is the point. A component can re-render a hundred times and cost nothing the user feels, because the React Compiler kept each render cheap. A single component can re-render *once* and blow your INP past 500ms because that one render ran a 400ms sort on the blocking path. Render **count** is a diagnostic breadcrumb; render **cost on the interaction path** is the thing that ships as a slow app. Keep the layers separate and you profile the right problem. Collapse them and you optimize render count that never mattered.

The discipline this article teaches is a loop: **baseline → change exactly one thing → re-measure → keep it only if the number moved.** Three tools feed the loop, and knowing which one to reach for is most of the skill.

## How it works under the hood

### Where the numbers come from

React ships three builds. The default **development** build is instrumented for warnings and is slow — its absolute timings are meaningless for optimization decisions, though its *relative* comparisons and its "why did this render" reasons are trustworthy. The default **production** build has profiling instrumentation stripped out entirely. Between them sits a dedicated **profiling** build: production semantics plus the timing hooks, imported by swapping `react-dom/client` for `react-dom/profiling` (framework build flags like `next build --profile` do this for you). Rule of thumb: reach for the profiling build any time you quote an absolute millisecond number as evidence.

The instrumentation works by hooking React's **commit phase** (the render/commit split is owned by [how React renders](../rendering/how-react-renders.md#render-and-commit)). Every time a subtree commits, React records how long the render took and hands it to two consumers: the DevTools extension, and any `<Profiler>` you have mounted. That single commit hook is why the DevTools flamegraph and the `<Profiler>` `onRender` callback report the same `actualDuration` — they are two faces of one measurement.

`actualDuration` is the time actually spent rendering the subtree this commit, *with* whatever memoization was in effect. `baseDuration` is React's estimate of what the same subtree would have cost with **no** memoization at all — the worst case, computed by summing each component's most recent standalone render time. The relationship between them is the whole game: if `actualDuration` on an update is far below `baseDuration`, memoization (manual or Compiler-inserted) is doing real work; if `actualDuration` sits *at* `baseDuration` every commit, nothing is being skipped and your memoization — if you added any — is inert.

The "why did this render?" reason comes from React tracking, per fiber, which inputs changed: a parent that re-rendered and dragged the child along, a specific hook slot (`Hook 3 changed` — the number is the hook's position top-to-bottom in the component), or a named prop whose reference changed (`Props changed: (onClick)`). That last one is the fingerprint of an unstable inline prop.

### The 19.2 Performance Tracks

Before 19.2, the Chrome Performance panel showed you a "Long Task" with no idea which component caused it. React 19.2 adds custom tracks that annotate the browser timeline with React's own view of the work:

- **Scheduler track** — four subtracks, one per priority *lane*: **Blocking** (synchronous work, typically user input), **Transition** (work started inside `startTransition`), **Suspense** (fallback/reveal work), and **Idle**. This track is how you *see* React's concurrency: whether a piece of work ran as blocking or as a transition is now visible, not inferred.
- **Components track** — a flamegraph of component render durations for the recorded window, using the same color scheme as the scheduler phases. Effects show too, in a distinct color, but only if they ran ≥ 0.05ms or scheduled an update (to keep the track readable).

Each render pass shows its phases on the timeline — **Update** (what scheduled the render) → **Render** → **Commit** → **Remaining Effects** (passive effects, usually after paint; the exception is discrete events like clicks, where passive effects can run *before* paint). One high-value extra: if an update was scheduled *during* render, React flags a **Cascading update** entry with the stack trace of the component that scheduled it — the fastest way to catch a `useEffect`-sets-state feedback loop.

In development the tracks appear automatically. In a profiling build only the Scheduler track is on by default, and the Components track lists only subtrees wrapped in `<Profiler>` — *unless* the React DevTools extension is installed, which restores full component coverage. Server Components and Server Request tracks are development-only.

## Basic usage

**Open the DevTools Profiler.** With the React DevTools extension installed, open DevTools → **Profiler** tab → click record (⏺) → perform one interaction (a keystroke, a filter change, a route navigation) → stop. You get one bar per **commit**. Scrub commits with the picker at the top.

Two views matter. The **flamegraph** shows the component tree for the selected commit, each node sized by its render time — yellow is expensive, blue is cheap, grey did not render this commit. The **ranked chart** throws away the tree and sorts every component that rendered by descending duration; it is the fastest way to find the single worst offender in a busy commit.

The one setting that changes everything: the gear icon → **"Record why each component rendered while profiling."** Turn it on, record again, and every component tells you *why* it re-rendered.

**Measure programmatically with `<Profiler>`.** For automated checks or production RUM, wrap a subtree and read the commit data directly:

```tsx
import { Profiler, type ProfilerOnRenderCallback } from "react";

const onRender: ProfilerOnRenderCallback = (
  id,             // the Profiler's id prop
  phase,          // "mount" | "update" | "nested-update"
  actualDuration, // ms rendering this subtree this commit (with memoization)
  baseDuration,   // ms it would cost with no memoization (worst case)
  startTime,      // when React began this render
  commitTime,     // when React committed it (shared across profilers in the commit)
) => {
  // 16ms is one frame at 60fps — anything above it risks a dropped frame.
  if (actualDuration > 16) {
    console.warn(`[${id}] ${phase} ${actualDuration.toFixed(1)}ms (base ${baseDuration.toFixed(1)}ms)`);
  }
};

export function App() {
  return (
    <Profiler id="Dashboard" onRender={onRender}>
      <Dashboard />
    </Profiler>
  );
}
```

`<Profiler>` is lightweight but not free — each one adds a little CPU and memory overhead, so instrument specific subtrees, not every component.

**Log a Core Web Vital.** The `web-vitals` library is the field-side measurement:

```ts
// reportWebVitals.ts
import { onINP, onLCP, onCLS } from "web-vitals";

onINP(console.log); // { name: "INP", value: 342, rating: "poor", ... }
```

## Walkthrough — the measurement loop, end to end

The lesson here is not a fix. It is the *loop*: how three tools hand off to each other, and how the profiler tells you which fix is even the right kind.

The app is a product dashboard: a search box over a 500-row table. Typing feels laggy on a mid-range phone. We will not guess — we will measure our way to the cause, pick the fix the measurement points at, and confirm the number moved.

### Starting files

```ts
// types.ts
export interface Product {
  id: string;
  name: string;
  price: number;
}
```

```tsx
// ProductDashboard.tsx
import { useState } from "react";
import { ProductTable } from "./ProductTable";
import type { Product } from "./types";

interface ProductDashboardProps {
  products: Product[];
}

export function ProductDashboard({ products }: ProductDashboardProps) {
  const [query, setQuery] = useState("");

  // Derived during render — never mirrored into state.
  const visible = products.filter((p) =>
    p.name.toLowerCase().includes(query.toLowerCase()),
  );

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Filter products…"
        aria-label="Filter products"
      />
      <p>{visible.length} of {products.length}</p>
      <ProductTable products={visible} />
    </div>
  );
}
```

```tsx
// ProductTable.tsx
import { ProductRow } from "./ProductRow";
import type { Product } from "./types";

interface ProductTableProps {
  products: Product[];
}

export function ProductTable({ products }: ProductTableProps) {
  return (
    <table>
      <tbody>
        {products.map((p) => (
          <ProductRow key={p.id} product={p} />
        ))}
      </tbody>
    </table>
  );
}
```

```tsx
// ProductRow.tsx — deliberately does a little work per row
import type { Product } from "./types";

interface ProductRowProps {
  product: Product;
}

export function ProductRow({ product }: ProductRowProps) {
  const priceLabel = new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  }).format(product.price);

  return (
    <tr>
      <td>{product.name}</td>
      <td>{priceLabel}</td>
    </tr>
  );
}
```

### Step 1 — Triage with React Scan (seconds, not minutes)

Before opening any panel, get a visual read. Drop React Scan in for the dev session — one import, before your app mounts:

```ts
// main.tsx (top of file, dev only)
import { scan } from "react-scan";

if (import.meta.env.DEV) {
  scan({ enabled: true });
}
```

Type into the filter. The entire table lights up on every keystroke, each row stamped with a rising render count. React Scan's whole job is this fast necessary-vs-unnecessary read: it highlights renders that produced no change to the component's DOM subtree. Here the glow tells us the table re-renders per keystroke — expected, since `visible` is recomputed — so this is not "unnecessary renders" to eliminate. It is *cost per keystroke* to investigate. React Scan pointed us at the table; now we need the number.

> React Scan is triage, not diagnosis. Run it for a few minutes, find the components that glow hot, then switch to the Profiler for the "how much" and the "why." It's dev-only; the `dangerouslyForceRunInProduction` flag exists purely so its name can warn you never to.

### Step 2 — Locate and cost it with the DevTools Profiler

Profiler tab → gear → enable **"Record why each component rendered."** Record one keystroke, stop, open the **ranked chart**.

`ProductTable` sits at the top at ~110ms `actualDuration`, with 500 `ProductRow` renders stacked beneath it. Click `ProductDashboard`: **"Hook 1 changed"** — that's `query`. The state update at the top re-runs the filter, hands a fresh `visible` array down, and the whole table re-renders. The commit is ~120ms.

This is the fork in the road, and the profiler just told us which branch we're on. `actualDuration` (~110ms) is close to `baseDuration` here, so nothing is being wrongly skipped — this is a genuine *render-cost* problem, not a wasted-render problem. There are two honest kinds of fix for a real render-cost problem:

1. **Make the render cheaper** — colocate the input state so the table isn't in its update path at all, or lean on composition so unchanged rows never re-render. This is the arc the [typing-lag rerender-storm recipe](../recipes/performance/typing-lag-rerender-storm.md) owns end to end; that recipe is the right home for it.
2. **Move the cost off the blocking path** — let the keystroke commit instantly and let the expensive table update run at a lower priority, so it never blocks input even though it still costs ~110ms.

We'll take route 2 here, because it's the one that lets us *measure a concurrent feature* — and verifying that a transition actually helped is exactly the kind of claim you must never make without the profiler.

### Step 3 — Confirm the user impact before touching code

Locating the cost isn't the same as proving the user feels it. Throttle CPU to 4× in the Chrome Performance panel (never profile responsiveness on your unthrottled dev machine), record a few keystrokes, and read the **19.2 Scheduler track**: every keystroke produces a long bar on the **Blocking** subtrack — React is doing the whole table render synchronously, in the same frame as the input. Meanwhile `web-vitals` reads the field-equivalent metric:

```ts
import { onINP } from "web-vitals/attribution";

onINP((m) => {
  console.log(m.value, m.rating, m.attribution.interactionTarget);
  // 342 "poor" "input[aria-label='Filter products']"
});
```

INP ≈ 342ms — **poor** (the good/poor line is 200ms/500ms). The attribution build even names the culprit element. Now we have a baseline that is about the user, not about a flamegraph.

### Step 4 — Apply exactly one change

Split the update into two priorities: the input value updates synchronously (stays on the Blocking lane, so typing feels instant), and the filtered result is a *deferred* value that React is free to compute at Transition priority. The mechanism — `useDeferredValue`, transitions, tearing — is owned by [concurrent rendering](../concurrent/concurrent-rendering.md#usedeferredvalue); here we only wire it and measure it.

```tsx
// ProductDashboard.tsx — one change
import { useState, useDeferredValue } from "react";
import { ProductTable } from "./ProductTable";
import type { Product } from "./types";

interface ProductDashboardProps {
  products: Product[];
}

export function ProductDashboard({ products }: ProductDashboardProps) {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query); // table follows this, lazily

  const visible = products.filter((p) =>
    p.name.toLowerCase().includes(deferredQuery.toLowerCase()),
  );

  return (
    <div>
      <input
        value={query}                                   // driven by the instant value
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Filter products…"
        aria-label="Filter products"
      />
      <p>{visible.length} of {products.length}</p>
      <ProductTable products={visible} />
    </div>
  );
}
```

### Step 5 — Verify the number moved (this is not optional)

Re-record. Two things must be true, and you must *see* both:

- **Scheduler track:** the keystroke now produces a short Blocking bar (just the input), and the ~110ms table render has moved to the **Transition** subtrack. The expensive work still happens — it's just no longer between the user's key press and the next paint.
- **INP:** `web-vitals` now reads ≈ 180ms — **good**. The render cost didn't shrink; it stopped blocking.

That distinction is the entire payoff of measuring instead of guessing. A `useMemo` sprinkle would have done nothing — the render wasn't wasted, it was mispriorized, and only the Scheduler track makes the difference legible.

### Step 6 — And confirm the Compiler is doing its half

Open the DevTools **Components** tab and select `ProductRow`. With the React Compiler on (assumed for app code — see [the compiler deep dive](../rendering/react-compiler-deep-dive.md)), it carries a **✨ badge**, meaning the Compiler memoized it. Prove it pays off: during a filter that only removes rows, scrub to that commit — surviving `ProductRow`s show `actualDuration` ≈ 0 (skipped), and only the table's own reconciliation cost remains. That's the Compiler doing what a wall of hand-written `memo`/`useMemo` used to do — and the profiler is how you confirm it, rather than assuming.

**Verify-the-loop table:**

| Metric | Before | After |
| --- | --- | --- |
| Keystroke lane (Scheduler track) | Blocking, ~120ms | Blocking ~4ms + Transition ~110ms |
| `ProductTable` `actualDuration` | ~110ms | ~110ms (unchanged — that's the point) |
| INP (4× throttle) | 342ms · poor | 180ms · good |

## Real-world patterns

**Did the transition actually help?** The only honest answer comes from the Scheduler track. A transition that "feels" faster but still shows its work on the **Blocking** subtrack didn't take — usually because the state update wasn't inside `startTransition`/`useDeferredValue`, or because a synchronous read forced it back to blocking priority. The classic miss: wrapping the *table* update in a transition but leaving the *input* value update blocking is right; wrapping the *input* value in the transition makes typing itself lag. The track tells you which you did.

**Did the Compiler actually help?** Two checks. In the **Components** panel, compiled components carry the ✨ badge — a component you expected to be optimized that lacks the badge means the Compiler bailed (a rules-of-React violation is the usual cause; the [compiler deep dive](../rendering/react-compiler-deep-dive.md#bailouts) owns the why). In the **Profiler**, compare `actualDuration` to `baseDuration` on updates: a healthy compiled tree shows `actualDuration` dropping well below `baseDuration` as unchanged subtrees skip. If they track each other commit-for-commit, nothing is being memoized and you have a bailout to chase — *not* a reason to hand-add `memo`.

**Profiling in production.** The DevTools Profiler is dev-only, but real-user numbers come from a profiling build feeding `<Profiler>` `onRender` to your observability stack — sample it (log only `actualDuration > 16`) so the overhead and the data volume stay sane.

**Field vs lab, and the RUM contract.** Lab tools (Chrome Performance panel, Lighthouse) give you reproducibility; field tools (`web-vitals` + CrUX) give you truth about real devices. You need both: field data tells you *there is* a slow interaction and on which route/device/country; lab data lets you *reproduce and fix* it. Wire all three vitals to an endpoint, and slice — an aggregate INP hides a page that's fine on desktop in one region and terrible on mobile in another:

```ts
import { onINP, onLCP, onCLS, type Metric } from "web-vitals";

function report(metric: Metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating, // "good" | "needs-improvement" | "poor"
    id: metric.id,
    route: window.location.pathname,
    device: /Mobi|Android/i.test(navigator.userAgent) ? "mobile" : "desktop",
  });
  navigator.sendBeacon?.("/api/vitals", body) ??
    fetch("/api/vitals", { body, method: "POST", keepalive: true });
}

onINP(report);
onLCP(report);
onCLS(report);
```

**The SPA vitals trap.** Core Web Vitals are anchored to a *hard* navigation. In a client-routed React app, a route change does not reset the LCP or CLS window and INP keeps accumulating across the whole session. A number that looks fine on your dev machine can be a route-transition artifact in the field. If your router doesn't emit soft-navigation vitals, treat SPA CWV numbers with suspicion and lean harder on per-interaction INP attribution.

**Attributing a slow interaction to a script.** When INP attribution points at an element but not a cause, a Long Animation Frame observer names the exact script:

```ts
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      // entry.scripts[] names the handler/source that overran the frame
      console.log(entry.duration, entry.scripts.map((s) => s.sourceURL));
    }
  }
}).observe({ type: "long-animation-frame", buffered: true });
```

Pair this with `onINP` and you can walk from "the worst interaction in production" to "this specific handler" without reproducing anything locally.

## API / tool reference

**`<Profiler onRender>` callback** — `(id, phase, actualDuration, baseDuration, startTime, commitTime)`

| Arg | Meaning |
| --- | --- |
| `id` | The Profiler's `id` prop — identifies which tree committed. |
| `phase` | `"mount"` \| `"update"` \| `"nested-update"` (the third = a state update triggered *during* another update). |
| `actualDuration` | ms rendering this subtree this commit, *with* memoization. Should fall well below `baseDuration` on updates. |
| `baseDuration` | ms to render the subtree with *no* memoization — the worst case. |
| `startTime` / `commitTime` | When React began / committed this render (`commitTime` is shared across profilers in one commit). |

> The legacy 7th `interactions` argument was removed with interaction tracing in React 18 — code or tutorials still passing it are stale.

**`web-vitals`** — `onLCP` / `onINP` / `onCLS` (also `onFCP`, `onTTFB`); each gives a `Metric { name, value, rating, delta, id, navigationType }`. Import from `web-vitals/attribution` for `metric.attribution`.

| Metric | Good | Poor (> ) | Note |
| --- | --- | --- | --- |
| LCP | ≤ 2.5s | 4.0s | Largest content paint. |
| INP | ≤ 200ms | 500ms | Replaced FID (Mar 2024). Input delay + processing + presentation; worst interaction (~98th pct over 50). |
| CLS | ≤ 0.1 | 0.25 | Unitless layout-shift score. |

Thresholds are evaluated at the 75th percentile of field sessions. Use `{ reportAllChanges: true }` for low-traffic pages or lab runs.

**DevTools Profiler settings** — "Record why each component rendered while profiling" (the essential one); "Hide commits below _ ms" to cut noise; flamegraph vs ranked toggle.

**React Scan** — `scan(options)` (imperative), `useScan(options)` (hook), or `npx react-scan@latest <url>` (CLI), or the `unpkg` script tag. Key options: `enabled`, `log`, `showToolbar`, `animationSpeed`, `onRender(fiber, renders)`, and `dangerouslyForceRunInProduction` (never).

## Common mistakes

**1. Optimizing before measuring.** Adding `memo`/`useMemo`/`useCallback` on a hunch is the cardinal sin — it adds comparison cost and code, and with the Compiler on it's usually inert. Profile, find the actual bar, then act.

```tsx
// ❌ Cargo-cult: no measurement, and the Compiler already handles this.
const Row = memo(function Row({ item }: { item: Item }) { /* ... */ });

// ✅ Ship the plain component; open the Profiler; only reach for manual
//    memo at a non-compiled boundary the ✨ badge shows the Compiler skipped.
export function Row({ item }: { item: Item }) { /* ... */ }
```

**2. Trusting dev-build absolute numbers.** The development build is instrumented and slow; a "300ms render" there may be 40ms in production. Use dev for *relative* comparisons and the "why" reasons; use a **profiling build** whenever you quote a millisecond number as evidence.

**3. Chasing render count instead of render cost.** "This renders 60 times a second" is not a bug if each render is 0.2ms and the Compiler keeps it cheap. React Scan's glow shows *count*; the Profiler shows *cost*. Optimize cost on the interaction path, ignore cheap renders.

**4. Reading `baseDuration` as the real cost.** `baseDuration` is a *hypothetical* no-memoization worst case, not what happened. The number that happened is `actualDuration`. Use `baseDuration` only as the yardstick you compare `actualDuration` against.

**5. Fixing the widest flamegraph bar without expanding it.** A bar's width includes its children. The wide bar is often a cheap parent over one expensive child. Expand before concluding, or use the ranked chart, which attributes self-time correctly.

**6. Profiling on an unthrottled machine.** Your laptop is not your user's phone. Throttle CPU 4×–6× in the Performance panel; an interaction that's fine at 1× can be a 400ms INP at 4×. Improvements for slow hardware help fast hardware too.

**7. Concluding from the React Profiler when the cost isn't React's.** The Profiler only sees React renders. A slow `fetch`, a layout thrash from reading `offsetHeight` in a loop, or a 200ms third-party script won't show as a wide component bar. When the Profiler looks innocent but the app feels slow, switch to the Chrome Performance panel and the LoAF observer.

**8. Confusing FID with INP.** FID was retired in March 2024. It only measured the delay before the *first* interaction's handler ran; INP measures *every* interaction's full latency including processing and paint. Targets, tooling, and fixes differ — chase INP.

**9. Believing a transition helped without checking the lane.** "It feels smoother" is not evidence. If the work still shows on the **Blocking** subtrack, the transition didn't take. The Scheduler track is the proof.

**10. Leaving `<Profiler>` in a hot production path unsampled.** It adds per-commit overhead. Wrap targeted subtrees, gate reporting behind `actualDuration > 16`, and don't wrap the whole app.

## How this evolved

Manual instrumentation came first: `console.time` around suspected components, then community tools like *why-did-you-render* that patched React to log which prop reference changed. React DevTools v4 (2019) shipped the Profiler tab with the "why did this render" reasons, folding most of that into the browser. React 16–17 also had *interaction tracing* — the now-removed 7th `onRender` argument — which was cut in React 18 as concurrent rendering made single-interaction attribution unreliable.

The two changes that reshaped the workflow are recent. The **React Compiler** (1.0, Oct 2025) auto-memoizes most components at build time, which flips the profiling question from "where should I add `memo`?" to "did the Compiler memoize what I expected, and where did it bail?" — a verification task, not an authoring one. And **INP replacing FID** (Mar 2024) moved the field metric from "first interaction delay" to "worst full interaction," which made the responsiveness of filtering, validation, and dynamic updates — exactly the React-heavy interactions — the thing that ships as a passing or failing grade. React 19.2's Performance Tracks close the loop by making React's own scheduling visible inside the browser timeline, so "did the transition run as a transition?" is finally an observation instead of a guess.

## Exercises

**1. Count vs cost.** Take a list where every row re-renders on a parent state change. Before optimizing, use the Profiler to decide whether it's a *count* problem (many cheap renders) or a *cost* problem (few expensive renders). *Hint:* open the ranked chart and read `actualDuration` per component; if the total across all rows is a few milliseconds, you're chasing a non-problem — verify with `baseDuration` that memoization has nothing to skip.

**2. Prove the Compiler memoized a component.** Pick a pure leaf component in a compiled app. Confirm the ✨ badge in the Components panel, then profile an update that shouldn't affect it and confirm its `actualDuration` is ~0. *Hint:* if the badge is missing, the Compiler bailed — check the ESLint plugin output for a rules-of-React violation before assuming you need manual `memo`.

**3. Reproduce a poor INP and watch it recover.** Add an artificial 250ms synchronous loop to a click handler, measure INP with `web-vitals` under 4× throttle, then move the work off the blocking path (a transition, or `scheduler.yield()` to chunk it) and re-measure. *Hint:* use the attribution build so you can confirm the `interactionTarget` is the element you expect, and watch the Scheduler track move the work off the Blocking subtrack.

## Summary

Profiling is the discipline that earns you the right to optimize. Measure at two layers — React (what rendered, how much, and why: DevTools Profiler, `<Profiler>`, React Scan, the 19.2 tracks) and browser (what the user felt: Core Web Vitals via the Performance panel and `web-vitals`). Reach for React Scan to triage, the Profiler to locate and cost, and the Scheduler track plus INP to prove user impact. Read `actualDuration` against `baseDuration`, not in isolation. Let the measurement pick the *kind* of fix — cheaper render, or off the blocking path — and never claim a transition or the Compiler helped without seeing the lane change or the ✨ badge. Then close the loop: change one thing, re-measure, keep it only if the number moved.

## See also

- [Memoization and the Compiler](../rendering/memoization-and-the-compiler.md) — the mechanism this article verifies; where manual `memo` still legitimately lives.
- [React Compiler deep dive](../rendering/react-compiler-deep-dive.md#bailouts) — why the ✨ badge is missing, and how to read a bailout.
- [Concurrent rendering](../concurrent/concurrent-rendering.md#usedeferredvalue) — transitions and `useDeferredValue`, the mechanism behind the walkthrough's fix.
- [How React renders](../rendering/how-react-renders.md#render-and-commit) — the render/commit split the Profiler instruments.
- [Recipe: typing lag / rerender storm](../recipes/performance/typing-lag-rerender-storm.md) — the colocation-and-composition fix arc for render-cost problems; the hub of the performance recipe track.
- [Recipe: 900KB initial bundle](../recipes/performance/route-splitting-bundle.md) *(planned)* — profiling load, not interaction.
- [Recipe: 6s freeze on tab switch](../recipes/performance/virtualization-long-list.md) *(planned)* — when the fix is virtualization, not memoization.

## References

- React docs — [`<Profiler>`](https://react.dev/reference/react/Profiler)
- React docs — [React Performance Tracks](https://react.dev/reference/dev-tools/react-performance-tracks)
- React 19.2 release notes — [react.dev/blog](https://react.dev/blog/2025/10/01/react-19-2)
- web.dev — [Interaction to Next Paint (INP)](https://web.dev/articles/inp)
- web.dev — [Core Web Vitals thresholds](https://web.dev/articles/vitals)
- `web-vitals` — [github.com/GoogleChrome/web-vitals](https://github.com/GoogleChrome/web-vitals)
- React Scan — [github.com/aidenybai/react-scan](https://github.com/aidenybai/react-scan)

## Demo source

*Demo pending — see the roadmap's demo-hosting decision (StackBlitz vs. CodeSandbox vs. local `demos/`).*