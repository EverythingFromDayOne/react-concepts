---
article_id: react-compiler-deep-dive
concept_folder: rendering
wave: 3
related:
  - rendering/memoization-and-the-compiler
  - foundations/rules-of-react
  - rendering/how-react-renders
  - ecosystem/performance-profiling
  - recipes/performance/typing-lag-rerender-storm
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this.** You turned on React Compiler. Some components show the ✨ badge in DevTools; some don't. Nobody told you why the un-badged ones opted out, or whether that's a bug you caused or a decision the compiler made. Worse, on a fresh Vite 8 project you followed a tutorial, the badge never appeared at all, and there was no error — the compiler just silently wasn't running, because the tutorial showed the Babel wiring that Vite 8 deleted. This article is the deep dive [`memoization-and-the-compiler`](./memoization-and-the-compiler.md) pointed forward to: what the compiler actually does to your code, *why* it sometimes refuses to compile a component (and why refusing is the safe outcome), the exact Vite 8 / `@rolldown/plugin-babel` wiring that most guides still get wrong, and how the ESLint plugin turns "it silently bailed" into "here's the line to fix." [`memoization-and-the-compiler`](./memoization-and-the-compiler.md) owns *when and why* to memoize and the manual APIs; this owns the compiler as a program that rewrites your components.

## What the compiler is

React Compiler is a **build-time optimizing compiler** that inserts memoization into your components and hooks automatically — the equivalent of hand-written `useMemo`/`useCallback`/`memo`, but derived by static analysis and applied comprehensively, at a granularity no human maintains by hand. It's a Babel plugin (`babel-plugin-react-compiler`) that runs during your build; the runtime cost is a small per-component memo cache (`_c`, from `react/compiler-runtime`).

The editorial stance for this project (roadmap §5): the compiler is **assumed ON for all app code**, and manual memoization is demoted to a commented escape hatch at non-compiled boundaries. This article is what justifies that stance — once you understand what the compiler infers and how it protects itself from unsafe code, "just write pure components and let it memoize" stops being a leap of faith.

The division of labor with [`memoization-and-the-compiler`](./memoization-and-the-compiler.md): that article owns the three manual APIs as mechanism, the traced `_c(n)` output, contagion, the ✨ badge, and the manual-memo estate table. This article owns the *compiler itself* — the pipeline, the analyses, the soundness contract that lets it skip work safely, the toolchain wiring, and the full opt-in/opt-out accounting.

## How it works under the hood

### The pipeline

The compiler runs as a Babel pass over each module. Conceptually it moves through four stages:

1. **Parse** your source to an AST (Babel's job).
2. **Lower to an intermediate representation.** The compiler builds a control-flow graph of each component/hook in a High-level IR (HIR) in SSA-like form. This is what lets it reason about *order of operations* and *what depends on what* — things an AST alone doesn't make explicit.
3. **Analyze.** Several passes run over the HIR (below): inferring which values are reactive, grouping code into memoizable scopes, computing each scope's exact dependencies, and tracking mutation so it never memoizes something unsafe.
4. **Codegen.** The compiler re-emits JavaScript with a memo cache: it allocates cache slots (`_c(n)` — the traced output [`memoization-and-the-compiler`](./memoization-and-the-compiler.md#what-compiled-output-actually-looks-like) already dissected) and wraps each scope in a "did these inputs change? if not, reuse the cached value" guard.

The output is behaviorally identical to your source — same renders, same values — with recomputation skipped wherever the inputs are unchanged.

### Reactive scopes and per-expression granularity

The central concept the compiler introduces is the **reactive scope**: a span of instructions whose outputs depend on the same set of reactive inputs, memoized together as a unit. A value is "reactive" if it can change between renders — props, state, context, and anything derived from them. The compiler groups your component's work into scopes and gates each scope's re-execution on its dependency set.

The consequence is a granularity no hand-written memoization achieves: the compiler memoizes **per expression**, not per component. Where you might wrap one big `useMemo` around a computed list (and recompute the whole thing when any input changes), the compiler can split the work into several scopes, recomputing only the sub-expressions whose specific inputs changed. It's finer *and* it's derived mechanically, so it doesn't rot when you edit the code — the classic failure mode of hand-maintained dependency arrays.

### Dependency inference — "deps are derived, not chosen," mechanized

The project's recurring principle — **deps are derived, not chosen** — is exactly what the dependency-inference pass does. For each reactive scope, the compiler computes the *minimal, exact* set of inputs that determine the scope's output. No guessing, no `// eslint-disable exhaustive-deps`, no stale closure from a dependency you forgot. The compiler reads the data flow and produces the dependency list that a perfectly disciplined human *would* write and routinely doesn't. This is why the compiler's memoization is more correct than most hand-written memoization, not just more convenient: humans choose deps and get them wrong; the compiler derives them.

### Mutability and aliasing analysis — why local mutation stays legal

Memoization is only safe if the compiler knows a value won't be mutated after it's cached. So the compiler tracks **mutation and aliasing**: where each value is written, and which references alias the same underlying object. Two payoffs:

- **Local mutation of freshly-created values stays legal** — the "local-mutation-legal" carve-out from [`rules-of-react`](../foundations/rules-of-react.md#how-it-works-under-the-hood). Building an array with a `for` loop and `push` inside render is fine; the compiler sees the array is created and mutated locally, then never mutated after, and memoizes it correctly. The rule was never "no mutation ever" — it's "no mutation of values that escaped your scope," and the compiler models exactly that boundary.
- **Scopes split where mutation demands.** If a later instruction mutates a value an earlier scope produced, the compiler adjusts scope boundaries so the memoization stays correct rather than caching a value that's about to change.

### The soundness contract: bail, don't miscompile

Here's the part that makes the whole thing trustworthy. The compiler's correctness rests on your code following the **Rules of React** ([`rules-of-react`](../foundations/rules-of-react.md) owns them): components and hooks are pure, props/state/hook-returns aren't mutated, hooks follow their rules. If you break those, memoization *could* change behavior — serve a stale value, skip an update that should have happened.

The compiler's defense is a design principle worth internalizing: **when it cannot statically prove a component is safe to compile, it bails on that component — skips memoizing it — rather than compiling it and risking wrong behavior.** The bailout is *per component/hook*, not per app: one un-compilable component opts itself out while everything around it stays compiled. The result is conservative and safe: a rule violation almost always means "that component just didn't get optimized," not "your app broke."

This is what a missing ✨ badge actually means. The badge ([`memoization-and-the-compiler`](./memoization-and-the-compiler.md#walkthrough-the-verification-workflow-rung-3-end-to-end)) marks a compiled component. No badge means one of three things: it's not a component/hook (nothing to compile), it's excluded by config, or **the compiler detected something it couldn't safely compile and bailed**. The third case is the interesting one — and diagnosing it is exactly what the ESLint plugin is for. The compiler bails *silently* (that's the point — it degrades gracefully); lint is how you make the silence visible.

## Walkthrough: what the compiler does to a component

Description is cheaper than tracing, so let's trace. Take a component that, in the manual era, you'd have sprinkled with three memoization calls:

```tsx
function ProductPanel({ product, currency, onAddToCart }: ProductPanelProps) {
  const priceLabel = formatPrice(product.price, currency); // derived from product.price + currency
  const related = expensiveRelatedLookup(product.category); // expensive, derived from product.category
  const handleAdd = () => onAddToCart(product.id);          // a new function every render, manually

  return (
    <section>
      <Header title={product.name} price={priceLabel} />
      <RelatedList items={related} />
      <AddButton onClick={handleAdd} />
    </section>
  );
}
```

Manually, you'd reach for `useMemo` on `related`, maybe `useMemo` on `priceLabel`, and `useCallback` on `handleAdd` — and you'd have to hand-write each dependency array. The compiler does the whole thing by analysis. Here's what it infers, without you writing a single dep array:

- **It builds reactive scopes.** The compiler identifies distinct spans of work and the reactive inputs each depends on. `priceLabel` is a scope depending on `product.price` and `currency`. `related` is a *separate* scope depending only on `product.category`. `handleAdd` is a scope depending on `onAddToCart` and `product.id`. The rendered JSX is itself a scope depending on the outputs of the others. Crucially, these are *separate* scopes — the compiler didn't lump them into one "recompute everything if any prop changed" block the way a single hand-written `useMemo` around the whole body would.

- **It derives each scope's exact dependencies.** Not "everything the function closes over" — the *minimal* set. `related`'s scope depends on `product.category` and nothing else, so a change to `product.price` does **not** recompute the expensive related-lookup. A human writing `useMemo(() => expensiveRelatedLookup(product.category), [product])` would over-depend on the whole `product` object and recompute needlessly; the compiler derives `[product.category]` precisely. This is "deps are derived, not chosen" paying off in fewer wasted computations than careful manual code typically achieves.

- **It guards each scope with a slot in the memo cache.** Codegen wraps each scope in a "did these specific deps change since last render? if not, reuse the cached value" check, backed by cache slots (`_c(n)` from `react/compiler-runtime` — [`memoization-and-the-compiler`](./memoization-and-the-compiler.md#what-compiled-output-actually-looks-like) shows the exact emitted syntax; the point here is *which* scopes and *why*, which that article doesn't trace).

Now watch a concrete re-render. The parent re-renders and passes a `product` whose `price` changed but whose `category` and `id` did not, with a stable `currency` and `onAddToCart`:

- `priceLabel` scope: `product.price` changed → **recomputes**. (Correct — the price label is stale otherwise.)
- `related` scope: `product.category` unchanged → **skips** the expensive lookup, reuses the cached list. (This is the win — the costly work is gated on the one input that actually feeds it.)
- `handleAdd` scope: `onAddToCart` and `product.id` unchanged → **reuses the same function reference**. So `AddButton`, if it's compiled (or memoized), sees an identical `onClick` prop and can bail on its own re-render (bailout check 3, [`how-react-renders`](./how-react-renders.md#bailouts-the-fast-paths)) — the stable-reference problem `useCallback` existed to solve, handled without a `useCallback`.

The manual version needed three memo calls and three correct dependency arrays to achieve this, and would have quietly regressed the moment someone edited the body and forgot to update a dep array. The compiled version derives it all and stays correct across edits. To confirm it's actually happening rather than trusting the description, the ✨ badge marks `ProductPanel` as compiled, and the preset's `logger` will print a compile event for it — the verify-don't-assume discipline this whole article turns on.

## The Vite 8 wiring (and the gotcha)

This is the setup step that most guides get wrong, because the ground shifted under them. **Vite 8 ships Rolldown** (a single Rust bundler) and **`@vitejs/plugin-react` v6 switched to Oxc** for the JSX and Fast Refresh transforms — which means **v6 dropped Babel as a dependency entirely.** The old, everywhere-online form is now dead:

```ts
// ❌ DEAD on Vite 8 + @vitejs/plugin-react v6.
// Babel was removed from the plugin, so this `babel` option is silently ignored —
// no error, no badge, the compiler simply never runs. This is THE gotcha.
react({ babel: { plugins: ["babel-plugin-react-compiler"] } });
```

Because the compiler *is* a Babel plugin, and the React plugin no longer runs Babel, you bring Babel back explicitly with `@rolldown/plugin-babel` and feed it the compiler via the `reactCompilerPreset` helper (exported from `@vitejs/plugin-react`, which bundles `babel-plugin-react-compiler` behind a preconfigured file filter):

```bash
npm i -D @rolldown/plugin-babel @babel/core babel-plugin-react-compiler
# TypeScript projects also need the Babel core types:
npm i -D @types/babel__core
```

```ts
// vite.config.ts — the correct Vite 8 wiring.
import { defineConfig } from "vite";
import react, { reactCompilerPreset } from "@vitejs/plugin-react";
import babel from "@rolldown/plugin-babel";

export default defineConfig({
  plugins: [
    react(),
    babel({ presets: [reactCompilerPreset()] }),
  ],
});
```

A few things worth knowing so this doesn't bite you:

- **The preset carries a file filter.** `reactCompilerPreset()` only runs the compiler on relevant source files, so you're not paying Babel over your whole `node_modules`. To exclude specific directories (a legacy folder you're not ready to compile), grab the preset and adjust its filter: `const preset = reactCompilerPreset(); preset.rolldown.filter.id.exclude = ["src/legacy/**"];`.
- **Compiler-before-other-Babel-plugins.** If you add *other* Babel plugins alongside the compiler, the compiler must run first in the Babel plugin order — it needs the original source shape for its analysis. This is about ordering *within* the Babel preset, not about the Vite `plugins` array.
- **The v5 escape hatch.** If you haven't moved to plugin-react v6 yet, v5 still works on Vite 8 and still accepts the old `react({ babel: {...} })` form — so "the old way" isn't wrong everywhere, just wrong on v6. Match your wiring to your plugin-react major version.
- **Verify, don't assume.** After wiring, open React DevTools and confirm the ✨ badge on your components ([`memoization-and-the-compiler`](./memoization-and-the-compiler.md#walkthrough-the-verification-workflow-rung-3-end-to-end)), or pass a `logger` to the preset to print compile events per file: `reactCompilerPreset({ logger: { logEvent(filename, event) { console.log(filename, event); } } })`. A silent no-op is the failure mode this whole section exists to prevent.

## The ESLint plugin

The compiler's static analysis is available to you as **lint**, whether or not you've turned the compiler on. As of v6, `eslint-plugin-react-hooks` **bundles the React Compiler's rules** under the same `react-hooks/` namespace as the classic `rules-of-hooks` and `exhaustive-deps`. When the compiler would detect a pattern it can't safely compile, the corresponding lint rule flags it in your editor — turning the compiler's silent bailout into a visible, fixable diagnostic.

```ts
// eslint.config.ts (flat config)
import reactHooks from "eslint-plugin-react-hooks";
import { defineConfig } from "eslint/config";

export default defineConfig([
  reactHooks.configs.flat.recommended,          // core + recommended compiler rules
  // reactHooks.configs.flat["recommended-latest"], // + newest experimental compiler rules
]);
```

The compiler-derived rules, all under `react-hooks/`, encode the soundness contract as individual diagnostics — among them:

- `purity` — flags known-impure work in render (the contract the compiler assumes).
- `immutability` — flags mutation of props/state/hook returns.
- `refs` — flags reading or writing a ref during render.
- `set-state-in-render` and `set-state-in-effect` — flag the setState-in-render loop and setState-synchronously-in-effect antipatterns.
- `static-components` — flags components defined inside other components (a fresh identity every render).
- `preserve-manual-memoization` — validates that your *existing* manual `useMemo`/`useCallback` is preserved (and correct) rather than silently discarded — important reassurance that the compiler doesn't fight your intentional boundaries.
- `use-memo` — validates `"use memo"` / `"use no memo"` directive usage.

Two facts make this the load-bearing part of adoption:

1. **It works without the compiler.** You can enable these rules and fix violations *before* flipping the compiler on — so adoption is "clean up the code, then compile," not "compile and hope."
2. **A lint warning explains a missing badge.** When a component isn't compiling and you don't know why, the lint rule points at the exact violation the compiler bailed on. Lint is the diagnostic layer over the compiler's graceful silence.

Note the preset surface is evolving quickly (the plugin recently slimmed to `recommended` and `recommended-latest`); if `flat.recommended` doesn't resolve for your installed version, check the plugin's README for the current preset path. The rule *names* under `react-hooks/` are stable; the config export path is the churny part.

## Opting in and out — the accounting

[`memoization-and-the-compiler`](./memoization-and-the-compiler.md#when-the-compiler-declines) introduced `"use no memo"`. Here's the full accounting of the levers, from whole-project down to a single function.

### Compilation mode (project-wide)

`reactCompilerPreset` takes a `compilationMode`:

- **`'infer'` (default):** the compiler compiles everything it identifies as a component or hook. This is the greenfield choice, and this project's baseline — compiler on, everywhere, from day one.
- **`'annotation'`:** the compiler compiles *only* functions explicitly marked with the `"use memo"` directive. This is the incremental-adoption choice for a large existing codebase — you opt components in one at a time and verify each, rather than compiling the whole app at once.

```ts
babel({ presets: [reactCompilerPreset({ compilationMode: "annotation" })] });
```

### Directives (per-function)

```tsx
function LegacyWidget() {
  "use no memo"; // opt this function OUT of compilation (infer mode)
  // …intentionally hand-tuned or knowingly rule-bending code…
}

function OptedIn() {
  "use memo"; // opt this function IN (required under annotation mode)
  // …
}
```

`"use no memo"` is an **escape hatch, not a fix.** If a component misbehaves under compilation, the cause is almost always a Rules-of-React violation the compiler *should* have bailed on but a subtle case slipped through — and the honest response is to find and fix the violation (lint will usually name it), not to silence the compiler on that function and ship the impurity. Reserve `"use no memo"` for genuinely exceptional cases: a hot path you've hand-optimized in a way the compiler can't match, or a temporary unblock while you fix the real cause. Every `"use no memo"` should carry a comment explaining why, same discipline as a manual `memo` at a boundary ([`memoization-and-the-compiler`](./memoization-and-the-compiler.md#reference)).

### File and directory exclusion (config)

For whole areas you're not ready to compile (a vendored legacy tree), exclude by path via the preset filter rather than sprinkling directives:

```ts
const preset = reactCompilerPreset();
preset.rolldown.filter.id.exclude = ["src/legacy/**", "src/vendor/**"];
babel({ presets: [preset] });
```

### Gating (runtime rollout)

For a cautious production rollout, the compiler supports **gating** — emitting both compiled and uncompiled paths and choosing between them at runtime via a feature flag, so you can enable compiled output for a percentage of traffic and watch your metrics before committing. It's an experimentation lever, not something a greenfield project needs; reach for it when retrofitting a large app where you want a measured rollout.

## Real-world patterns

**Greenfield adoption (this project).** `compilationMode: 'infer'`, the ESLint recommended preset on, and a simple rule of thumb: **a missing ✨ badge on a real component is a bug to diagnose, not to ignore.** Run lint, read the flagged rule, fix the violation, watch the badge return. You almost never write manual memoization; when you do, it's a commented boundary ([`memoization-and-the-compiler`](./memoization-and-the-compiler.md)).

**Incremental adoption (existing codebase).** Lint first — turn on the `react-hooks/` compiler rules and fix violations with the compiler still off, so you're fixing code, not chasing compile output. Then either switch to `annotation` mode and opt components in file by file, or go to `infer` with the worst legacy directories excluded, and shrink the exclude list over time. The two strategies converge; pick based on how much of the codebase you trust to be rule-clean.

**Manual memo coexistence.** The compiler *preserves* correct manual memoization (`preserve-manual-memoization` guards this), so adopting it doesn't require ripping out every `useMemo` first. But most manual memo becomes redundant noise once the compiler is on — clean it up opportunistically, keeping only the commented boundary cases from [`memoization-and-the-compiler`](./memoization-and-the-compiler.md#reference).

**Where the wins actually are.** The compiler's biggest gains are expensive children that re-render because a frequently-rendering parent re-renders — the exact shape the [`typing-lag-rerender-storm`](../recipes/performance/typing-lag-rerender-storm.md) recipe fixes, where the Compiler-verification rung catches a purity bug. Leaf components and already-hand-optimized code barely move. Don't expect the compiler to make the whole app faster; expect it to erase a specific, common class of wasted re-renders.

**Testing and frameworks.** Vitest/Jest don't apply the Babel transform by default, so components generally aren't compiled *in tests* — usually fine (you test behavior, which is unchanged), but a fixture that intentionally mutates to prove a point will trip the ESLint rules. And in Next.js, the framework wires the compiler itself — don't also add the Babel plugin by hand (double compilation). That's [`nextjs-and-rsc-in-practice`](../server/nextjs-and-rsc-in-practice.md)'s territory.

## Configuration reference

| Lever | Where | Effect |
| --- | --- | --- |
| `compilationMode: 'infer'` | `reactCompilerPreset()` | Compile all inferred components/hooks (default). |
| `compilationMode: 'annotation'` | `reactCompilerPreset()` | Compile only `"use memo"`-marked functions. |
| `target: '17' \| '18' \| '19'` | `reactCompilerPreset()` | Runtime target; older targets use `react-compiler-runtime`. |
| `logger` | `reactCompilerPreset()` | Callback for per-file compile events (debugging). |
| `rolldown.filter.id.exclude` | preset object | Exclude paths from compilation. |
| `"use memo"` | function body directive | Opt a function in (annotation mode). |
| `"use no memo"` | function body directive | Opt a function out (escape hatch). |
| `reactHooks.configs.flat.recommended` | ESLint flat config | Core + recommended compiler lint rules. |
| `reactHooks.configs.flat['recommended-latest']` | ESLint flat config | Adds newest experimental compiler rules. |

## Common mistakes

**1. The dead Vite wiring.** Using `react({ babel: { plugins: ["babel-plugin-react-compiler"] } })` on Vite 8 + plugin-react v6. Babel was removed from the plugin, so the option is ignored — no error, no badge, no compilation. Use `@rolldown/plugin-babel` + `reactCompilerPreset()`. This is the single most common "why isn't the compiler working" cause in 2026.

**2. Expecting the compiler to fix impure code.** It doesn't fix impurity — it *bails* on it, silently, leaving that component unoptimized. You still write pure components; the compiler rewards purity with memoization, it doesn't manufacture it.

**3. Misreading a missing badge.** No badge isn't "the compiler is broken." It's "not a component," "excluded," or "bailed on a detected violation." Diagnose with the ESLint rules or the `logger`, don't guess.

**4. Leaving manual memoization everywhere.** After adopting, most `useMemo`/`useCallback`/`memo` in app code is redundant. It's not harmful (the compiler preserves it), but it's noise that obscures the boundary cases that genuinely still matter. Prune it.

**5. Using `"use no memo"` to "fix" a bug.** If compiling a component breaks it, the component is violating the Rules of React. Opting it out hides the symptom and ships the violation. Find the rule (lint), fix the cause. `"use no memo"` is for hand-tuned hot paths, not for silencing correctness problems.

**6. Expecting uniform speedups.** The compiler shines on expensive children under frequently-rendering parents. Leaves and already-optimized code barely change. Measure ([`performance-profiling`](../ecosystem/performance-profiling.md)); don't assume every component got faster.

**7. Skipping lint because "the compiler handles it."** The compiler handles it by *silently declining to optimize*. Without the ESLint rules on, you never see the violations — you just quietly lose optimization on those components. Lint is the visibility layer; running the compiler without it forfeits the diagnostics.

**8. Mutating props or state.** Beyond being a Rules-of-React violation, mutation is exactly what the compiler's memoization assumes never happens. In the rare case a mutation slips past the bailout, memoization can serve a stale value. The `immutability` lint rule exists to catch this.

**9. Relying on compiled behavior in tests.** Tests don't compile by default. Don't write tests that depend on the compiler's memoization; test observable behavior, which is identical compiled or not.

**10. Over-broad exclusion.** A sloppy `exclude` glob (`src/**` "just to be safe") turns the compiler off for trees you actually wanted optimized. Exclude the minimum, and shrink the list over time.

## How this evolved

- **The manual era.** `useMemo`, `useCallback`, and `memo` by hand; dependency arrays maintained by eyeball; the contagion problem where one unstable reference defeats a whole memoized subtree ([`memoization-and-the-compiler`](./memoization-and-the-compiler.md)). Memoization was a permanent tax on writing React.
- **The compiler, experimental → stable.** React Compiler ran in Meta production through 2023–2024, went through a public beta and RC, and reached **1.0 stable in October 2025**. The pitch inverted the tax: write pure components, let the build memoize.
- **The toolchain shift (late 2025 / early 2026).** Vite 8 unified on Rolldown and `@vitejs/plugin-react` v6 moved to Oxc, dropping Babel — which is why the compiler's wiring changed to the `@rolldown/plugin-babel` bridge and why every pre-2026 tutorial's setup is now wrong. In parallel, `eslint-plugin-react-hooks` v6 absorbed the compiler's rules, so a single plugin now covers hooks rules *and* compiler diagnostics.
- **19.2 (this baseline).** The compiler is the default. Manual memoization is demoted to a documented, commented escape hatch at non-compiled boundaries, and "did it compile?" is a first-class thing you verify (the badge, the logger) rather than assume.

## Exercises

**1. Wire it, break it, fix it.** Set up the compiler in a Vite 8 project with the correct `@rolldown/plugin-babel` form and confirm the ✨ badge. Then swap in the dead `react({ babel: {...} })` form and watch the badge vanish with no error — the gotcha, reproduced.
*Hint:* Keep DevTools open on a known component; the badge is your signal that Babel actually ran the compiler.

**2. Cause a bailout, then diagnose it.** Write a component that reads or mutates something impurely so the compiler bails, confirm the badge is missing, then turn on the `react-hooks/` compiler rules and let lint name the violation. Fix it and watch the badge return.
*Hint:* A `set-state-in-render` or a ref read during render will do it; the lint rule points at the exact line.

**3. (Stretch) Incremental adoption.** Start in `annotation` mode, mark two components with `"use memo"`, and confirm only those two carry the badge. Then switch to `infer` with one directory excluded via the preset filter, and confirm the excluded tree stays un-badged while everything else compiles.
*Hint:* `compilationMode` toggles the default; directives and the exclude filter are the per-unit overrides.

## Summary

- React Compiler is a build-time Babel plugin that auto-memoizes via static analysis: parse → HIR → analyze → codegen with a `_c` memo cache. Granularity is **per-expression**, finer than hand-written memoization and derived, not hand-maintained.
- It infers **reactive scopes** and their **exact dependencies** (deps derived, not chosen), and tracks **mutation/aliasing** so local mutation stays legal and unsafe caching is avoided.
- Its correctness rests on the **Rules of React**; when it can't prove safety it **bails on that component** (skips memoization) rather than miscompiling. A missing ✨ badge means not-a-component, excluded, or a silent bailout.
- **Vite 8 gotcha:** plugin-react v6 dropped Babel for Oxc, so `react({ babel: {...} })` is dead — wire the compiler via `@rolldown/plugin-babel` + `reactCompilerPreset()`. Verify with the badge or a `logger`.
- The compiler's rules ship in **`eslint-plugin-react-hooks` v6+** (the `react-hooks/` namespace), work **without** the compiler, and turn a silent bailout into a nameable diagnostic — lint is how you find why a component didn't compile.
- Levers, from broad to narrow: `compilationMode` `infer`/`annotation`, `"use memo"`/`"use no memo"` directives, path exclusion, and gating for rollout. `"use no memo"` is an escape hatch, never a fix for a rule violation.

## See also

- [`memoization-and-the-compiler`](./memoization-and-the-compiler.md) — when/why to memoize, the manual APIs, the traced `_c(n)` output, the ✨ badge, the estate table.
- [`rules-of-react`](../foundations/rules-of-react.md) — the contract the compiler assumes and bails when you break; local-mutation-legal.
- [`how-react-renders`](./how-react-renders.md) — the render/commit and bailout machinery the memo cache plugs into.
- [`performance-profiling`](../ecosystem/performance-profiling.md) *(planned)* — measuring whether compilation actually helped, with the Profiler.
- [`typing-lag-rerender-storm`](../recipes/performance/typing-lag-rerender-storm.md) — the re-render storm the compiler erases, with Compiler-verification catching a purity bug.

## References

- React docs — [React Compiler](https://react.dev/learn/react-compiler)
- React docs — [React Compiler: Installation](https://react.dev/learn/react-compiler/installation)
- React docs — [`eslint-plugin-react-hooks`](https://react.dev/reference/eslint-plugin-react-hooks)
- Vite — [Vite 8.0 announcement](https://vite.dev/blog/announcing-vite8)
- npm — [`@vitejs/plugin-react`](https://www.npmjs.com/package/@vitejs/plugin-react)

## Demo source

_Demo source pending — StackBlitz link to be added in the demo pass._