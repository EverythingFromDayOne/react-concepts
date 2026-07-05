---
article_id: memoization-and-the-compiler
concept_folder: rendering
wave: 2
related:
  - rendering/how-react-renders
  - foundations/rules-of-react
  - foundations/components-and-props
  - state/context
  - effects/custom-hooks
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Memoization and the Compiler

> **Lead with this:** for a decade, React performance work meant hand-placing `memo`, `useMemo`, and `useCallback` — and hand-maintaining their fragile precondition, referential stability, which one inline object anywhere in the chain silently destroys. React Compiler 1.0 inverts the deal: write plain components that obey the Rules of React, and the build step memoizes for you, at finer granularity than the manual APIs ever offered. The stance this library has assumed since article one — *no cargo-cult memoization, Compiler assumed ON for app code* — gets its full argument here: what the Compiler actually emits, how to **verify** it worked instead of assuming (rung 3 of the performance ladder), and the enumerated boundary cases where manual memoization remains a professional tool rather than a reflex (rung 6, last for a reason).

## What it is

Three manual APIs, presented as *mechanism* — because understanding the machine is prerequisite to knowing when the automation has you covered:

- **`memo(Component, arePropsEqual?)`** wraps a component so that when its parent renders, React compares old and new props **shallowly, per key** (or with your custom comparator) and skips the render on a match. Mechanically it's an upgrade to bailout check 1 from [how-react-renders](./how-react-renders.md): instead of demanding the whole props *object* be reference-identical, it demands each prop *value* be — one level deep, `Object.is` per key.
- **`useMemo(compute, deps)`** caches a computed value on a hook node; on each render, if every dep is `Object.is`-equal to last time, the cached value comes back — same reference — otherwise `compute` runs again. It is a **cache, not a guarantee**: React reserves the right to discard the cache and recompute (offscreen trees, resource pressure), so `compute` must be pure and cheap-to-repeat in principle.
- **`useCallback(fn, deps)`** is `useMemo(() => fn, deps)` — identity stabilization for functions, so they can pass someone's `Object.is` check downstream: a memoized child's props, an effect's deps, a vendor API's contract.

And the structural flaw that made a decade of this miserable: **manual memoization is contagious.** `memo(Row)` does nothing if `Row` receives an inline `onSelect={() => …}` — so the parent needs `useCallback`; whose deps include an object — so that needs `useMemo`; whose inputs come from a custom hook — which must now document identity guarantees. One forgotten link, anywhere, and the entire chain silently degrades to "renders every time," with no error, ever. Teams either under-memoized (and shipped re-render storms) or over-memoized (and shipped deps bugs plus unreadable components).

**React Compiler** is a build-time transform that ends the chain-maintenance job. It analyzes each component and hook *assuming the Rules of React hold* — purity, immutability, hooks discipline ([rules-of-react](../foundations/rules-of-react.md) framed the contract as "what the Compiler enforces"; this is the payoff) — and rewrites the code to cache values, callbacks, and JSX chunks automatically, invalidating each cached slot precisely when its actual inputs change. You don't choose deps; **deps are derived, not chosen** — by static analysis, from what the code reads.

## How it works under the hood

### What compiled output actually looks like

Take a plain component:

```tsx
function ProductList({ products, query, onPick }: ProductListProps) {
  const visible = products.filter((p) => p.name.includes(query));
  return (
    <ul>
      {visible.map((p) => (
        <ProductRow key={p.id} product={p} onPick={() => onPick(p.id)} />
      ))}
    </ul>
  );
}
```

The Compiler emits (simplified from real output, structure faithful):

```js
import { c as _c } from "react/compiler-runtime";

function ProductList(t0) {
  const $ = _c(7);                       // a 7-slot memo cache on this fiber
  const { products, query, onPick } = t0;

  let visible;
  if ($[0] !== products || $[1] !== query) {
    visible = products.filter((p) => p.name.includes(query));
    $[0] = products; $[1] = query; $[2] = visible;
  } else {
    visible = $[2];
  }

  let t1;
  if ($[3] !== visible || $[4] !== onPick) {
    t1 = visible.map((p) => (
      <ProductRow key={p.id} product={p} onPick={() => onPick(p.id)} />
    ));
    $[3] = visible; $[4] = onPick; $[5] = t1;
  } else {
    t1 = $[5];
  }

  let t2;
  if ($[6] !== t1) {
    t2 = <ul>{t1}</ul>;
    $[6] = t1;                           // (real output stores both key and value)
  } else {
    t2 = $[6 /* cached element */];
  }
  return t2;
}
```

Read what happened. `_c(n)` is itself a hook (`useMemoCache`) — a fixed-size array living on the fiber's hook list like any other hook state, slots initialized to a sentinel so first render always computes. Every intermediate value became a **guarded slot**: the filter result caches on `[products, query]`; the mapped rows cache on `[visible, onPick]`; the `<ul>` element caches on its children. Nobody wrote a deps array — the guards *are* the deps, derived from what each expression reads. When the parent re-renders with the same `products`, `query`, and `onPick`, this function runs, hits three cache checks, and returns **the same `<ul>` element object as last time** — which is bailout check 1 gold: `ProductList`'s output being reference-stable means React's reconciler reuses the subtree wholesale. The Compiler doesn't *skip* renders the way `memo` does; it makes renders so cheap and their outputs so stable that the pipeline's own bailouts fire everywhere downstream. It **manufactures the stable-element bailout** that [components-and-props](../foundations/components-and-props.md) taught as a hand technique.

### The granularity difference

`memo` is all-or-nothing at the component boundary. The Compiler works *inside* the function: in a component whose output is 90% static and 10% changing, the static JSX chunks stay cached (stable references → children bail out) while only the changing expression recomputes. It routinely achieves what manual memoization practically can't: per-element stability inside a component that *does* re-render. This is also why compiled code makes `memo` largely redundant in app code — a child receiving all-stable props from a compiled parent already passes check 1 without any wrapper. And it closes the oldest trap in the manual playbook: **children defeated `memo`** ([components-and-props](../foundations/components-and-props.md)) because `<Wrapper>{content}</Wrapper>` minted a fresh `children` element every parent render, failing the shallow compare no matter what you wrapped. Compiled, that children chunk is a cached slot — stable until its real inputs change — so the one prop that reliably broke `memo` now reliably satisfies it.

### What gets stabilized, enumerated

The registry of things that were identity-churn hazards for a decade and are cached slots now — in compiled app code, each of these holds its reference until its actual inputs change:

| Value | Pre-Compiler status | Compiled status |
| --- | --- | --- |
| Inline event handlers (`onClick={() => …}`) | Fresh every render; `memo`-killers | Slot guarded by captures |
| Object/array literals in JSX (`style={{…}}`, `options={[…]}`) | Fresh every render | Slot guarded by contents' inputs |
| JSX elements and `children` chunks | Fresh unless hand-hoisted | Cached; static chunks permanently stable |
| Derived computations (`items.filter(…)`) | Recomputed; `useMemo` candidates | Slot with derived deps |
| Context provider `value` objects (app code) | The classic broadcast bug | Slot — [context](../state/context.md)'s app-code case closed |
| Custom-hook returns (app code) | Manual `useCallback` chores | Slots inside the compiled hook |

**And the second-order payoff: effects fire less wrongly.** Every one of those stabilized values is also a potential *dependency*. Pre-Compiler, an effect depending on an inline-built `options` object re-ran every render — the resubscribe-storm class of bug. Compiled, the dep holds identity until its real inputs change, so the effect re-runs exactly when it should. The Compiler is careful in the other direction too: it preserves reactive semantics, never caching a value *past* a real input change — your deps stay honest, they just stop lying by accident. (Hand-written `useMemo`/`useCallback` it encounters are left in place and compiled around — deleting them is your cleanup, not its.)

### When the Compiler declines

The Compiler only transforms what it can *prove safe under the Rules*. Encounter a violation — mutation of props or state, hooks in conditions, reads it can't reason about — and it **skips that component or hook entirely, emitting your original code**. Skipped is not broken: behavior is unchanged, you've just silently lost the optimization for that function. Three consequences:

1. **The ESLint rules are load-bearing.** `eslint-plugin-react-hooks` (the compiler-aware rules) flags exactly what makes the Compiler bail. A red squiggle here isn't style — it's "this function will not be optimized, and here's the rule it broke."
2. **Verification is a step, not a vibe** — the walkthrough below. Compilation is per-function and skippable; "the Compiler is on" is a build fact, "this component is compiled" is a claim you check.
3. **`"use no memo"`** is the explicit opt-out directive (per function or file) for the rare case the transform miscompiles or you're bisecting a suspected compiler issue. [rules-of-react](../foundations/rules-of-react.md) introduced it; the accounting discipline: every occurrence carries a comment with a reason and a tracking link, and the count trends to zero. It's a tourniquet, not a lifestyle.

### Cost model

Slots trade memory for time: a few dozen array entries per fiber, comparisons that are single `Object.is` checks. For app code this is noise. The honest limits: caches can be discarded (so `useMemo`-for-semantics was always wrong, compiled or not), compilation adds build time (Oxc-era tooling keeps this small; wiring specifics and the Vite 8 `@rolldown/plugin-babel` gotcha are [react-compiler-deep-dive](./react-compiler-deep-dive.md)'s *(Wave 3, planned)* to own), and the pipeline itself is untouched — lanes, phases, commits all work exactly as [how-react-renders](./how-react-renders.md) traced; only the *inputs to the bailout checks* got better.

## Basic usage

This project's template assumes the Compiler is wired (verification of *that* in one line: the DevTools check below; full wiring in article 24). Day-to-day usage is therefore mostly *not writing things*. The manual APIs, as mechanism, where they still appear:

```tsx
import { memo, useCallback, useMemo } from "react";

// memo with a custom comparator — coarse equality the Compiler won't invent:
// re-render only when the row's data version changes, ignore volatile hover props
export const HeavyRow = memo(HeavyRowImpl, (prev, next) => prev.version === next.version);

// useMemo at a boundary — vendor grid demands a stable config object:
// Manual memo: non-compiled boundary — AgGrid compares this by reference internally.
const gridOptions = useMemo(() => buildGridOptions(columns, locale), [columns, locale]);

// useCallback at a boundary — published hook returning a stable action:
// Manual memo: library boundary — consumers may not run the Compiler. Identity documented.
const retry = useCallback(() => dispatch({ type: "retried", at: Date.now() }), [dispatch]);
```

Every manual usage in this codebase carries a comment naming its boundary — that's the convention, and after this article, the reviewer's question for any uncommented `useMemo` is "which boundary?"

**The one-glance verification:** open React DevTools → Components, select a component. Compiled components carry the **"Memo ✨" badge** next to their name. Badge present = the Compiler transformed this function. Badge absent on a component you expected compiled = it declined; go find out why.

## Walkthrough: the verification workflow (rung 3, end to end)

The setup: a pricing dashboard — filter input, product grid, and a `<PriceChart>` that renders ~40ms of SVG. Written plain, per the house style: zero manual memoization.

```tsx
// PricingDashboard.tsx
import { useState } from "react";
import { PriceChart } from "./PriceChart";
import { ProductGrid } from "./ProductGrid";
import type { Product } from "./types";

export function PricingDashboard({ products }: { products: Product[] }) {
  const [query, setQuery] = useState("");

  const visible = products.filter((p) =>
    p.name.toLowerCase().includes(query.toLowerCase()),
  );

  return (
    <main>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        aria-label="Filter products"
      />
      <ProductGrid products={visible} />
      <PriceChart products={products} /> {/* full dataset — independent of the filter */}
    </main>
  );
}
```

```tsx
// PriceChart.tsx — deliberately expensive: ~40ms of SVG path math for 1,000 points
import type { Product } from "./types";

export function PriceChart({ products }: { products: Product[] }) {
  const points = buildPricePaths(products); // heavy: trend lines, percentiles, smoothing
  return (
    <svg viewBox="0 0 800 300" role="img" aria-label="Price distribution">
      {points.map((p) => (
        <path key={p.id} d={p.d} className={p.tone} />
      ))}
    </svg>
  );
}
```

**Step 1 — confirm compilation.** DevTools Components panel: `PricingDashboard`, `ProductGrid`, `PriceChart` all wear the ✨ badge. The claim "this is optimized" is now a fact, not a default.

**Step 2 — profile the interaction (rung 1 never gets skipped).** Record typing five characters. Expected and observed: `PricingDashboard` renders per keystroke (its state changed — it must); `ProductGrid` renders (its `products` prop is a genuinely new filtered array — it must); **`PriceChart` shows gray — zero renders.** Why: the compiled `PricingDashboard` cached the `<PriceChart products={products}/>` element in a slot guarded only by `products`, which didn't change — same element reference, check 1, subtree skipped. ~40ms saved per keystroke, no `memo` anywhere. Keystroke cost: ~6ms. To *quantify* what compilation is buying at any subtree, read the Profiler's two durations ([how-react-renders](./how-react-renders.md) defined them): `actualDuration` ~6ms against `baseDuration` ~46ms — the gap **is** the bailout dividend, and a subtree where the two converge is a subtree where stabilization isn't landing. For a controlled A/B on one component, `"use no memo"` doubles as a bisect tool: opt it out, re-measure, delete the directive.

**Step 3 — someone ships a regression.** A teammate adds a summary row the chart should include:

```tsx
const withSummary = products;
withSummary.push(buildSummaryRow(visible)); // 🔴 mutates the products PROP in render
return (
  /* … */
  <PriceChart products={withSummary} />
);
```

Two alarms fire *before* any user feels it. The **lint** flags the prop mutation (the compiler-aware rule names it). And in DevTools, `PricingDashboard`'s **badge is gone** — the Compiler proved it couldn't prove safety, and declined the whole function. Re-profile: `PriceChart` now renders every keystroke (no cached element without compilation, and its "guard" input mutates besides) — 46ms keystrokes, typing lag territory. Note the failure shape: **zero errors, zero warnings at runtime, pure silent perf regression.** This is why rung 3 is a *verification* rung: the badge and the lint are the only tripwires.

**Step 4 — fix by obeying, not by memoizing.** The reflex fix — wrap `PriceChart` in `memo` — treats the symptom and leaves the mutation bug (the *original* `products` array now has a phantom summary row; every other consumer sees it). The real fix:

```tsx
const chartData = [...products, buildSummaryRow(visible)]; // ✅ derive, immutably
return <PriceChart products={chartData} />;
```

Badge returns; lint is green; profile shows `PriceChart` rendering only when `products` or the summary actually change. (This is the same lesson [typing-lag-rerender-storm](../recipes/performance/typing-lag-rerender-storm.md)'s Compiler-verification stage teaches with its own bug — the recipe owns the full six-rung diagnostic arc; this walkthrough is rungs 3's standard operating procedure in isolation.)

**Step 5 — the one legitimate manual site (rung 6).** The grid migrates to a vendor virtualization component whose contract reads: *"`renderRow` must be referentially stable; the grid memoizes rows internally against it."* The vendor package is outside your compiled graph — its internal comparison can't be manufactured by *your* build step, and the callback you pass crosses that boundary:

```tsx
// Manual memo: non-compiled boundary — VendorGrid memoizes rows against
// renderRow's identity (see their docs §Perf). Compiler covers our side only.
const renderRow = useCallback(
  (product: Product) => <ProductRow product={product} onPick={pick} />,
  [pick],
);

return <VendorGrid items={visible} renderRow={renderRow} />;
```

Last rung, deliberately: it exists because a *foreign contract* demands identity, not because rendering was slow. Measure → structure → **Compiler verify** → defer → do less → **manual memo last**: rungs 3 and 6, both cashed.

## Real-world patterns

**The diagnostics runbook.** Condensed for the wall:

1. Symptom measured first (Profiler, "why did this component render" — [how-react-renders](./how-react-renders.md)' owner-chain reading).
2. ✨ badge check on the components in the hot path. Missing badge → lint output → fix the rules violation → re-verify. This solves the majority of "the Compiler didn't help" reports.
3. Badge present but re-renders persist → the input genuinely changes: is it *supposed* to? New-array-every-render from an uncompiled ancestor, a context value churning identity ([context](../state/context.md)'s provider-value section), a prop derived in a skipped component upstream. Fix at the source.
4. Structure before memo: colocate state, pass content as `children` — the [composition moves](../foundations/component-composition.md) shrink blast radius for whole subtrees.
5. Only then, rung 6, with a boundary comment.

**Manual memoization's remaining estate, enumerated.** The complete list of places hand-written `memo`/`useMemo`/`useCallback` remains correct in this codebase — anything outside it is a review flag:

| Boundary | Why the Compiler can't cover it | Tool |
| --- | --- | --- |
| Published library code (packages, design systems) | Consumers may not compile; your identity guarantees are API | `useCallback`/`useMemo` on everything returned or provided; documented |
| Context provider values at library boundaries | Fan-out cost × uncompiled consumers | `useMemo` on `value` ([context](../state/context.md)) |
| Vendor components with identity contracts | Their internal comparisons live outside your build | `useCallback`/`useMemo` per their documented contract |
| Non-React consumers (imperative embassies, worker messages) | Identity/equality semantics belong to foreign code | Stabilize per the foreign contract ([useref-and-the-dom](../effects/useref-and-the-dom.md)) |
| Custom comparators (`memo(C, areEqual)`) | Coarser-than-`Object.is` equality is a *policy*, not an inference | `memo` with comparator + comment |
| `"use no memo"` zones | Explicitly outside the compiled graph, temporarily | Manual APIs as pre-Compiler, plus a removal ticket |

Note what's *not* on the list: "expensive computation in app code" (the Compiler caches it), "handler passed to a memoized app component" (both sides compiled → both sides stable), "provider value in app code" (cached slot). The estate is boundaries, all the way down.

**Reading compile output when it matters.** The [React Compiler Playground](https://playground.react.dev) takes a component and shows the emitted slots — the fastest way to answer "will this be cached, and invalidated by what?" for a specific piece of code. Reading the guards teaches the invalidation logic better than any prose: find your expensive expression, read its `if ($[i] !== …)` line, and you know its true deps.

**Interaction with `React.memo` you already have.** Legacy codebases arrive with `memo` sprinkled everywhere. Under compilation it's mostly inert weight — props from compiled parents are already stable, so the wrapper's comparison is a redundant tax (tiny) and a misleading signal (large). The pragmatic policy: don't rip it out in a big bang; delete it wherever you touch, keep it where a comparator encodes real policy, and let the boundary comments mark the survivors as intentional.

## Reference

| API | Mechanism | Compiled-app-code verdict |
| --- | --- | --- |
| `memo(C)` | Per-key shallow props compare gates the render | Redundant; delete on touch |
| `memo(C, areEqual)` | Your equality policy gates the render | Keep when the policy is real |
| `useMemo(f, deps)` | Cached value slot, manual deps, discardable | Boundaries only, commented |
| `useCallback(f, deps)` | `useMemo` for identity | Boundaries only, commented |
| Compiler slots (`_c(n)`) | Derived-deps guards over every intermediate value | The default; verify via badge |
| `"use no memo"` | Function/file opt-out from compilation | Tourniquet + ticket |

**Decision flow for "should I write manual memo here?":** Is this code published, or crossing into non-compiled/foreign territory with an identity contract? → yes: memoize, comment the boundary, document the guarantee. → no: is the component compiled (badge)? → no: fix the rules violation instead. → yes: is the re-render's input genuinely changing? → yes: fix the source or restructure. The flow never reaches "sprinkle `useMemo` and hope."

## Common mistakes

**1. Cargo-cult `useCallback`/`useMemo` in compiled code.** Double memoization: the Compiler caches your `useCallback`'s output *and* its input function — extra slots, extra reads, zero benefit, and every future reader must consider whether the wrapper encodes a real contract. The convention exists precisely so absence of a comment means absence of a reason.

**2. `memo` fed inline props.**

```tsx
const Row = memo(RowImpl);
<Row style={{ padding: 8 }} onPick={() => pick(id)} /> // 🔴 two fresh references; memo compares, never matches
```

Mechanism misunderstanding, pre- or post-Compiler: the wrapper can only compare what it's given. (In compiled code the parent's slots stabilize both — which is the point.)

**3. `useMemo` as a semantic guarantee.**

```tsx
useMemo(() => { analytics.pageView(); }, []); // 🔴 a cache is not "once" — and never side effects
```

React may drop the cache and re-run; the Compiler's caches inherit the same license. "Exactly once per X" is an effect's or a handler's job.

**4. Lying to deps.** Hand-written `useCallback(fn, [])` capturing live state is the stale-closure factory; the fix is honest deps — or noticing you're describing an *event* and reaching for `useEffectEvent` ([custom-hooks](../effects/custom-hooks.md)). Deps are derived, not chosen — the Compiler's whole design says so.

**5. Assuming instead of verifying.** "We enabled the Compiler" is a build flag; the badge is per-function truth. The most common real-world state is *90% compiled, 10% silently skipped, hot path in the 10%* — invisible without rung 3.

**6. Perf-critical mutation, silently uncompiled.** The walkthrough's regression generalizes: any rules violation converts to a skip converts to a perf cliff with no runtime symptom. Treat the compiler lint as CI-blocking; it's the only synchronous alarm.

**7. Permanent `"use no memo"`.** Opt-outs without tickets fossilize; six months later nobody remembers whether the miscompile was real, and the file quietly renders like it's 2023. Reason + link + revisit date, every time.

**8. Blanket `memo` on cheap components.** A comparison per render plus a retained props object per fiber, to skip renders that cost less than the comparison. Pre-Compiler this was over-eager; post-Compiler it's pure sediment.

**9. `memo` (or slots) versus context.** No wrapper defeats bailout check 3: a context reader re-renders on value change, period. Stabilize the *provider value* or split the channel — [context](../state/context.md) owns the fix; memoizing the reader is aiming at the receiver again.

**10. Trusting dev-mode timings.** StrictMode double-invokes, dev-build slow paths, and the Profiler's own overhead all inflate dev numbers. Verify *behavior* (which components render) in dev; measure *cost* in a production build. Rung 1's fine print.

## How this evolved

- **`shouldComponentUpdate` / `PureComponent` (class era):** manual render-gating, hand-written comparisons, the original footgun collection.
- **`memo` + hooks (16.8):** the same idea for functions — and the contagion problem industrialized, because now *identity* mattered everywhere and every component minted fresh closures per render.
- **The discourse years (17–18):** "you might not need useMemo" vs. "memoize everything" — both camps right about the other's failure mode, neither able to fix the underlying fragility.
- **React Forget → React Compiler (announced 2021, RC 2024, 1.0 October 2025):** the fragility declared a compiler problem. Rules of React formalized as the analyzable contract; memoization moved from convention to build artifact.
- **19.x era (this baseline):** Compiler-on is the template default for new apps; the ESLint rules ship compiler-aware; the manual APIs settle into their permanent role — mechanism knowledge, plus boundary tools with comments.

## Exercises

**1. Read the slots.** Paste `ProductList` (top of this article) into the Compiler Playground. Identify which slot guards the filter, the row array, and the root element. Predict which slots invalidate when (a) `query` changes, (b) `onPick` changes, (c) the parent re-renders with everything identical. Verify against the output.
*Hint: count the `if ($[n] !== …)` guards and trace each guard's inputs — (c) should invalidate exactly zero.*

**2. Three ways to lose the badge.** In a scratch app with the Compiler wired, break one component three separate ways: mutate a prop array in render, call a hook inside an `if`, and write to a module-level variable during render. For each: does the lint flag it, does the badge disappear, and what does the Profiler show for a stable-props child? Fix all three.
*Hint: one of the three may keep its badge — the Compiler's analysis and the lint don't have perfectly identical coverage. Which caught what is the lesson.*

**3. Justify the estate.** Audit any mid-sized feature (yours or OSS) for every `memo`/`useMemo`/`useCallback`. Sort each into the boundary table or "sediment." Delete the sediment, add boundary comments to the survivors, and prove via Profiler that the deletions changed nothing.
*Hint: expect the split to run roughly 80/20 sediment-to-boundary in post-Compiler app code — and expect at least one deletion to be scary until the flame graph says otherwise.*

## Summary

- `memo`, `useMemo`, `useCallback` are mechanism: shallow-compare render gating and `Object.is`-guarded value caches. Their fatal flaw at scale was contagion — one unstable link anywhere degrades the chain, silently.
- The Compiler rewrites rule-abiding components into guarded cache slots with *derived* deps — finer-grained than `memo`, no arrays to maintain, stable outputs that make the pipeline's own bailouts fire. It changes bailout *inputs*, not the pipeline.
- It only transforms what it can prove: violations mean silent per-function skips. The ✨ badge and the compiler lint are the tripwires; verification (rung 3) is a step you run, not an assumption you hold.
- Fix regressions by restoring the rules (immutability, purity), not by wrapping symptoms in `memo` — the mutation that broke compilation was a correctness bug wearing a perf costume.
- Manual memoization's estate is exactly the boundaries: published code, provider values at those boundaries, vendor identity contracts, foreign consumers, custom comparators, opt-out zones. Every instance commented; rung 6, last.

## See also

- [how-react-renders](./how-react-renders.md) — the bailout checks whose inputs the Compiler manufactures
- [rules-of-react](../foundations/rules-of-react.md) — the contract the Compiler assumes and enforces; `"use no memo"` introduced
- [components-and-props](../foundations/components-and-props.md) — children-defeat-memo and the stable-element bailout, the hand-built versions
- [typing-lag-rerender-storm](../recipes/performance/typing-lag-rerender-storm.md) — the six-rung ladder this article's workflow slots into
- [context](../state/context.md) — provider-value identity, the boundary case with the widest blast radius
- [custom-hooks](../effects/custom-hooks.md) — published-hook stability, the other standing boundary
- [react-compiler-deep-dive](./react-compiler-deep-dive.md) *(Wave 3, planned)* — wiring (the Vite 8 `@rolldown/plugin-babel` gotcha), configuration, and deeper internals

## References

- [React Compiler — react.dev](https://react.dev/learn/react-compiler)
- [React Compiler Playground](https://playground.react.dev)
- [memo — react.dev](https://react.dev/reference/react/memo)
- [useMemo — react.dev](https://react.dev/reference/react/useMemo)
- [useCallback — react.dev](https://react.dev/reference/react/useCallback)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.