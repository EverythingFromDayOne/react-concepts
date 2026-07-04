---
recipe_id: typing-lag-rerender-storm
track: performance
primary_concept: component-composition
difficulty: foundational
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Typing Lag: The Re-render Storm

> **What you'll build:** a fix for the most-reported React performance symptom — *typing in one input lags the whole page* — done in the professional order: **measure → structure → verify the Compiler → defer the residue.** A 300-row ops dashboard goes from 182ms per keystroke to under 25ms, and the striking part is the diff: state moves, elements change owners, one purity bug gets fixed — and not a single `memo`, `useMemo`, or `useCallback` is written by hand. This recipe is the performance track's hub: it establishes the escalation ladder every later performance recipe slots into.

## The scenario

An internal ops dashboard: a header with a **"Filter deployments"** input, a 300-row deployments table (status pill, environment, a small inline sparkline per row), and a sidebar with four summary charts. A support ticket arrives: *"The search box freezes while typing. Letters appear in bursts."*

Reproduction on a dev machine with 6× CPU throttle (the honest stand-in for the ops team's fleet laptops):

| Measurement | Value |
| --- | --- |
| Scripting per keystroke (Performance panel) | **~182ms** |
| Profiler commit duration per keystroke | ~178ms — `DashboardPage` at the root of every flame |
| Typing "production" (10 keys) | ~1.8s of jank; input echoes in visible bursts |
| INP, field p75 | **420ms** (Core Web Vitals "good" threshold: ≤200ms) |

The cause is structural, and it's the most common one in React apps: the filter `query` lives in `DashboardPage`, so **every keystroke re-renders the entire page** — 300 rows recomputing sparkline paths (~0.5ms each ≈ 150ms) plus four charts (~30ms) for a state change that only the input and the filtered list care about.

Three wrong fixes get tried first, in this order, everywhere: **debounce the input** (now the letters *themselves* lag — the echo is a controlled render, you've delayed the symptom's cure into the symptom; note this is the opposite situation from [`../data-fetching/search-race-condition.md`](../data-fetching/search-race-condition.md), where debounce was correct because it throttled the *network*, never the input's own state); **sprinkle `React.memo` on everything** (a week of wrapping, mostly defeated by fresh inline handlers and objects — the `Object.is` rows from [`../../foundations/components-and-props.md`](../../foundations/components-and-props.md) — and a permanent maintenance tax); and **blame React**. The actual fix order follows.

## Walkthrough

### Stage 0 — the patient

```tsx
// ❌ DashboardPage v0 — everything correct except one thing: placement
export function DashboardPage({ deployments }: { deployments: Deployment[] }) {
  const [query, setQuery] = useState('');

  const visible = deployments.filter((d) =>
    d.service.toLowerCase().includes(query.trim().toLowerCase()),
  );

  return (
    <div className="dashboard">
      <header>
        <input
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Filter deployments…"
        />
      </header>
      <main>
        <DeploymentsTable rows={visible} />
      </main>
      <aside>
        <SummaryCharts deployments={deployments} />
      </aside>
    </div>
  );
}
```

Give the code its due before surgery: the filtering is **derived during render** — correct, keep it ([`../../foundations/thinking-in-react.md`](../../foundations/thinking-in-react.md)); the input is controlled — correct for a filter that drives UI ([`../../forms/forms-controlled-and-uncontrolled.md`](../../forms/forms-controlled-and-uncontrolled.md)); rows are keyed by `d.id` — correct. The one flaw: `query` — state consumed by the input and the table — is placed where its re-renders also tax four charts and the page shell. Placement, not purity, is the disease.

### Stage 1 — measure before touching anything

Two tools, five minutes, and the difference between surgery and flailing:

**React DevTools Profiler** — in the Profiler tab, enable **"Record why each component rendered while profiling"** in settings first; record, type three characters, stop. Read the flamegraph (widths = self time per commit; gray bars = did not render): `DashboardPage` renders per keystroke; hovering `SummaryCharts` and any `DeploymentRow` shows *"why did this render? → the parent component rendered."* Now apply the diagnostic that separates real work from waste: **did the re-render produce different output?** The charts render `deployments` — unchanged — so theirs is pure waste. Rows are mixed: the visible *set* legitimately changes per keystroke, but each surviving row's own props didn't. Baseline recorded: **~178ms/commit**, charts ~30ms of it, rows ~145ms.

**Chrome Performance panel** — confirm the cost is *scripting* (render phase), not style/layout (commit phase). That distinction picks the toolbox: this recipe's tools fix scripting storms; a commit-side storm (per-row layout reads, CSS-in-JS recalculation) is a different disease (pitfall 10). One calibration note: dev-mode numbers include StrictMode double renders and dev checks — treat them as *relative* (before/after on the same build), and reach for a production profiling build when absolute numbers matter.

### Stage 2 — structure: the two moves

Both moves from [`../../foundations/component-composition.md`](../../foundations/component-composition.md), applied verbatim.

**Move 1 — push the state down.** `query`'s consumers are the input and the filtered table; its lowest common parent is *not* the page. Give the pair their own component:

```tsx
// ✅ FilterableDeployments — the blast radius, fenced
function FilterableDeployments({ deployments }: { deployments: Deployment[] }) {
  const [query, setQuery] = useState('');

  const visible = deployments.filter((d) =>
    d.service.toLowerCase().includes(query.trim().toLowerCase()),
  );

  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Filter deployments…"
      />
      <DeploymentsTable rows={visible} />
    </>
  );
}

export function DashboardPage({ deployments }: { deployments: Deployment[] }) {
  return (
    <div className="dashboard">
      <main><FilterableDeployments deployments={deployments} /></main>
      <aside><SummaryCharts deployments={deployments} /></aside>
    </div>
  );
}
```

`DashboardPage` is now stateless — keystrokes never reach it, so `SummaryCharts` and the shell are out of the storm *structurally*, not via a comparison function that can be defeated.

**Move 2 — lift content up.** Design then insists the charts live *visually inside* the filterable card. The reflex is to move `<SummaryCharts />` back inside `FilterableDeployments` — which re-adopts it into the blast radius. Instead, change its **owner**, not its position:

```tsx
<FilterableDeployments deployments={deployments}>
  <SummaryCharts deployments={deployments} />     {/* owner: DashboardPage */}
</FilterableDeployments>

// inside FilterableDeployments:
//   …input + table…
//   <aside>{children}</aside>
```

Per the owner/parent mechanics, `children` is the same element reference across every keystroke render of `FilterableDeployments` — React bails out of the charts subtree automatically, zero `memo`. Re-profile: **~96ms/keystroke.** Charts gone from the flame; the table remains.

### Stage 3 — Compiler verification (and the bug it finds)

The remaining 90ms is 300 `DeploymentRow`s re-running per keystroke. Their props are stable per row (`deployment` objects come from the same source array; only membership in `visible` changes), so under this project's Compiler-first stance they should be *bailing out automatically* — memoized rows with unchanged props skip. Here's the row:

```tsx
// DeploymentRow.tsx — pure at a glance; one line says otherwise
export function DeploymentRow({ deployment }: { deployment: Deployment }) {
  const tagLine = deployment.tags.sort().join(', ');        // ← (spoiler)
  return (
    <tr>
      <td><StatusPill status={deployment.status} /></td>
      <td>{deployment.service}</td>
      <td>{deployment.env}</td>
      <td>{tagLine}</td>
      <td><Sparkline points={deployment.latencyHistory} /></td>
    </tr>
  );
}
```

Are the rows bailing? Verify, don't assume:

1. **DevTools:** compiler-optimized components carry the optimization badge. `DeploymentsTable` has it. `DeploymentRow`… doesn't.
2. **Lint:** the compiler-powered rules ([`../../foundations/rules-of-react.md`](../../foundations/rules-of-react.md)'s enforcement stack) point at the reason:

```tsx
// ❌ the bailout cause
const tagLine = deployment.tags.sort().join(', ');
```

`sort()` mutates the prop's array in place — a purity violation ([`../../foundations/rules-of-react.md`](../../foundations/rules-of-react.md)'s clinic patient B, in the wild), and the Compiler responds exactly as designed: it **can't prove the component pure, so it compiles nothing**, and the row silently forfeits memoization. The fix is one method:

```tsx
const tagLine = deployment.tags.toSorted().join(', ');
```

Badge appears; re-profile: unchanged rows now show *"did not render"* in the Profiler, only rows entering/leaving the filtered set do work. **~24ms/keystroke, INP p75 → 140ms.** Total hand-written memoization in the entire fix: zero. The Compiler was always willing to do this job; a mutated prop was disqualifying one component, and *this is the Compiler-era shape of performance debugging* — bailout diagnostics first, memo APIs almost never ([`../../rendering/memoization-and-the-compiler.md`](../../rendering/memoization-and-the-compiler.md), Wave 2).

### Stage 4 — the residue: when the remaining work is real

Scale the same dashboard to 5,000 rows and stage 3's honest floor reappears: filtering 5k strings plus rendering the changed visible set is *legitimate* work (~140ms), and no memoization removes work whose inputs genuinely changed. The next rung is concurrency — split the update into an urgent part (the input echo) and a deferrable part (the list):

```tsx
const [query, setQuery] = useState('');
const deferredQuery = useDeferredValue(query);          // lags behind during heavy renders

const visible = useMemo(                                 // see note — this one is the Compiler's
  () => filterDeployments(deployments, deferredQuery),
  [deployments, deferredQuery],
);
// …input renders from `query` (instant); table renders from `visible` (catches up)
const stale = query !== deferredQuery;                   // free "updating…" signal
```

```tsx
<div className={stale ? 'results results--stale' : 'results'} aria-busy={stale}>
  <DeploymentsTable rows={visible} />
</div>
```

Typing stays under a frame (the urgent render touches only the input); the expensive list render happens at deferred priority — *interruptible*, so the next keystroke abandons it rather than queueing behind it. (In compiled code you write the plain derivation and the Compiler provides the caching; the mechanics of priorities and interruption are [`../../concurrent/concurrent-rendering.md`](../../concurrent/concurrent-rendering.md), Wave 3 — this is its trailer.) Past ~10k rows even the deferred render is too much per pass, and the answer changes *kind*: virtualization renders only the viewport (queued recipe, `performance/` track).

The audit trail — every stage owed a number, and paid:

| Stage | Change | ms/keystroke | INP p75 |
| --- | --- | --- | --- |
| 0 | baseline (300 rows, state at page root) | 182 | 420ms |
| 2 | push state down + lift charts up | 96 | — |
| 3 | one `toSorted()` → rows compile → unchanged rows bail | **24** | **140ms** |
| 4 | (5k-row variant) `useDeferredValue` | input <16; list catches up, interruptible | — |

Hand-written `memo`/`useMemo`/`useCallback` across all stages: **zero.**

The full ladder, which every performance recipe in this library slots into:

| Rung | Tool | When |
| --- | --- | --- |
| 1 | **Measure** (Profiler + Performance panel) | Always first; pick scripting vs commit |
| 2 | **Structure** — push state down, lift content up | Wasted renders with an identifiable blast radius |
| 3 | **Compiler verification** — badges, lint, fix bailouts | Pure components not bailing when props are stable |
| 4 | **Concurrency** — `useDeferredValue` / transitions | The remaining work is real but deferrable |
| 5 | **Do less** — virtualization, workers, pagination | The work itself exceeds a frame no matter its priority |
| 6 | **Manual memo** — `memo`/`useMemo`/`useCallback` | Non-compiled boundaries only (variation below) |

Manual memoization is rung *six*. Most codebases run the ladder upside down.

## Variations

### The non-compiled boundary — the one legitimate hand-written `memo`

The rows come from a third-party grid library (not compiled by your build). Its `Row` re-renders with the parent regardless of purity — the Compiler only covers code it compiles. This is the boundary case the conventions reserve manual APIs for: wrap your row adapter in `memo`, and *hold up your end of the shallow compare* — stable handlers and no fresh object props at the call site, since nobody is stabilizing them for you here. One component, commented with why, is the correct total amount of hand-rolled memoization in this app.

### When the expensive part is the compute, not the components

Fuzzy-matching 5k rows at 60ms per evaluation makes the *derivation* the cost. The Compiler caches it between unrelated re-renders (typing elsewhere doesn't recompute), but per-keystroke recomputation is irreducible in-render work. `useDeferredValue` keeps the input live while it happens; if the compute grows past what any render pass should hold (~50ms+), it stops being a rendering problem — move it off the main thread (web-worker recipe, queued) or to the server.

### The commit-side storm

Same symptom, different flame: scripting is modest but *style/layout* dominates the Performance trace. Usual suspects: a per-row effect reading `getBoundingClientRect` (300 forced layouts — batch into one measurer or `ResizeObserver`), or runtime CSS-in-JS generating styles per row per render (static extraction). Rung-2/3 tools don't touch this; diagnosing which side of render/commit you're on is why stage 1 exists.

## Trade-offs and common pitfalls

### When NOT to use this

- **Don't restructure without a profile.** The two moves are cheap but not free (review churn, ownership changes); spending them on a 6ms re-render is ritual, not engineering. Baseline first — the fix should have a number it's chasing.
- **Don't debounce controlled-input echo.** Worth its own line because it's the #1 reflex: debounce belongs on derived downstream cost (network — the search recipe; or the deferred list here), never on the state feeding `value=`.
- **Don't virtualize 300 rows.** Windowing at this scale buys complexity, breaks ⌘F and screen-reader continuity, and saves milliseconds structure already saved. It's rung 5 for rung-5 problems.

### Pitfalls

1. **`memo` defeated at the call site.** Wrapping rows while passing `onSelect={() => …}` and `style={{…}}` fresh each render — paid for the compare, lost it to `Object.is`, every time. In compiled code the Compiler stabilizes these; at non-compiled boundaries *you* must, or the `memo` is decoration.
2. **`memo` on a component that takes `children`.** The children element is fresh per owner render, so the compare always fails ([`../../foundations/components-and-props.md`](../../foundations/components-and-props.md)). Restructure ownership instead — that's Move 2.
3. **Index keys under filtering.** Filtering re-indexes survivors; index keys then *remount* rows per keystroke — destroying row state **and** bypassing all memoization, compiled or manual, since a remount is never a props-compare ([`../../rendering/rendering-lists-and-keys.md`](../../rendering/rendering-lists-and-keys.md)). Stable ids are a performance prerequisite, not just a correctness one.
4. **Colocating past the sharing boundary.** If `query` must also drive the URL or a sibling export button, pushing it into `FilterableDeployments` breaks the feature — the right scope is the *lowest common* parent of all consumers, which sometimes is the page. Then the wins come from Move 2 and stage 3 instead; the ladder has multiple rungs precisely because rung 2 isn't always available.
5. **Reading dev-mode absolute numbers.** StrictMode double-renders and dev assertions inflate everything ~2×+. Compare like with like; confirm final numbers on a production profiling build.
6. **Misreading "parent rendered."** The Profiler's "why" tells you the *trigger*, not the *verdict*. Parent-caused re-renders that produce changed output are the system working; the waste test is unchanged output, and only that earns a fix.
7. **The global-store exit.** Moving `query` to Zustand "so the page doesn't re-render" without selector discipline re-renders every subscriber anyway — and now placement problems are invisible in the tree. Stores solve sharing problems, not placement problems (state-management track).
8. **`useDeferredValue` on the wrong side.** Deferring the value the *input* renders from makes typing itself lag — the exact bug, reintroduced deliberately. The input reads the urgent value; only the expensive consumer reads the deferred one.
9. **Assuming the Compiler compiled it.** Bailed-out components are byte-identical in your source — the forfeit is silent. Stage 3's verification loop (badge → lint → fix the rule) is a standing audit habit, not a one-time step.
10. **Fixing render when commit is burning.** Memoizing components while 300 layout reads thrash per keystroke moves nothing. The Performance panel's scripting/rendering split is the fork in the road; take it before choosing tools.
11. **Hand-memoizing the derivation with curated deps.** `useMemo(() => filter(...), [query])` — omitting `deployments` because "it rarely changes" — is a stale list waiting for the day it does. In compiled code, write the plain derivation; where you do memoize manually, deps are derived, not chosen ([`../../effects/effects-and-synchronization.md`](../../effects/effects-and-synchronization.md) — same contract, same reason).
12. **Optimizing the demo dataset.** 300 rows on an M-series laptop hides everything. Profile at the fleet's percentile: real row counts, 6× throttle, production build — the scenario's numbers came from there, and so did the ticket.

## See also

- [`../../foundations/component-composition.md`](../../foundations/component-composition.md) — the two moves and the owner/parent mechanics this entire fix runs on
- [`../../foundations/components-and-props.md`](../../foundations/components-and-props.md) — `Object.is`, children-defeat-memo: why sprinkled `memo` loses
- [`../../foundations/rules-of-react.md`](../../foundations/rules-of-react.md) — the enforcement stack that stage 3's verification is an application of
- [`../../rendering/memoization-and-the-compiler.md`](../../rendering/memoization-and-the-compiler.md) *(Wave 2)* — bailout diagnostics and the manual-memo boundary, in full
- [`../../concurrent/concurrent-rendering.md`](../../concurrent/concurrent-rendering.md) *(Wave 3)* — priorities and interruption behind `useDeferredValue`
- [`../data-fetching/search-race-condition.md`](../data-fetching/search-race-condition.md) — the sibling recipe: where debounce *does* belong, and the union/tagging discipline shared by both
- [`../../ecosystem/performance-profiling.md`](../../ecosystem/performance-profiling.md) *(Wave 4)* — the measurement workflow stage 1 compresses

## References

- [React Developer Tools — the Profiler (react.dev)](https://react.dev/learn/react-developer-tools)
- [`useDeferredValue` (react.dev)](https://react.dev/reference/react/useDeferredValue)
- [`memo` (react.dev)](https://react.dev/reference/react/memo)
- [React Compiler (react.dev)](https://react.dev/learn/react-compiler)
- [Render and Commit (react.dev)](https://react.dev/learn/render-and-commit)
- [Interaction to Next Paint (web.dev)](https://web.dev/articles/inp)
- [Before You memo() — Dan Abramov](https://overreacted.io/before-you-memo/)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../../progress.md) TODOs.