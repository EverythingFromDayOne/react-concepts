---
article_id: error-boundaries
concept_folder: rendering
wave: 2
related:
  - rendering/how-react-renders
  - foundations/rules-of-react
  - state/usereducer-and-state-structure
  - concurrent/suspense
  - ecosystem/routing-react-router
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Error Boundaries

> **Lead with this:** a component that throws during render doesn't break a component — with no boundary in the tree, React unmounts **everything**, and the user gets a blank page because one sparkline divided by zero. Error boundaries are React's `catch` for the render phase: a component that intercepts throws from its subtree and renders a fallback instead of letting the unwind reach the root. They are also the last place classes exist in a modern codebase — the API has no hook equivalent — which is why the house rule is: understand the class once, then never write it again; `react-error-boundary` ships the class plus everything you'd bolt on (resets, retry keys, and the bridge that routes async errors into the same fallback). The deeper skill this article locks is the taxonomy: which errors are *state* you render, and which are *throws* a boundary catches.

## What it is

An error boundary is a class component implementing either or both of:

- **`static getDerivedStateFromError(error)`** — called during the render phase when a descendant throws; returns state that makes the *next* render show a fallback.
- **`componentDidCatch(error, errorInfo)`** — called during commit with the error and a `componentStack`; the reporting hook.

What it catches: errors thrown while React is doing its work in the subtree below — **rendering** (any component function throwing, including a rejected promise read with `use()`), **effects** (setup and cleanup — they run inside React's commit machinery), and class lifecycles/constructors. What it does **not** catch: **event handlers** (your `onClick` runs from a DOM dispatch, not the render pipeline — plain `try/catch` territory), **async callbacks** (`setTimeout`, promise chains detached from render), errors thrown **by the boundary itself** (they propagate to the *next* ancestor boundary), and server-side rendering errors (a different pipeline — [ssr-and-hydration](../server/ssr-and-hydration.md) *(Wave 3, planned)*).

The scoping rule is structural: a boundary protects its **children**, not itself, and the nearest boundary above the throw wins. Nesting boundaries is therefore how you buy granularity — which is most of the placement craft below.

## How it works under the hood

### The unwind

A throw during the render phase interrupts the beginWork/completeWork traversal ([how-react-renders](./how-react-renders.md)) mid-tree. React catches it and walks **up the `return` chain** from the throwing fiber, looking for a class fiber that declares the boundary methods. Found: React records the error on that boundary, schedules it to re-render, and rendering resumes *from the boundary* — `getDerivedStateFromError` runs, the boundary's render returns the fallback, and the finished tree that eventually commits contains the fallback where the broken subtree would have been. The broken subtree's fibers are discarded; its cleanups run. Not found: the "boundary" is the root, and React's contract since 16 is deliberate — **unmount the entire tree** rather than leave half-committed UI lying about the app's state. A corrupted checkout form is worse than a blank page.

### The concurrent retry

In an interruptible render, a throw might be an artifact of timing (a store mutated mid-slice, a tear) rather than a real bug. So before committing a fallback, React **retries the render synchronously** — no yielding, no interleaving. Only if the error reproduces does the boundary engage. Practical consequence: breakpoints and logs in a throwing component can fire twice for one committed fallback; that's the retry, not a double bug.

### Commit-phase throws

Errors from `useLayoutEffect`/`useEffect` setup or cleanup can't unwind a render that's already done — React catches them during the (layout or passive) flush and routes them to the nearest boundary the same way, scheduling its fallback in a follow-up render.

### React 19's root-level reporting

Before 19, centralized error telemetry meant a boundary at the root plus `window.onerror` glue. `createRoot` now takes the hooks directly:

```tsx
// main.tsx
createRoot(document.getElementById("root")!, {
  onUncaughtError: (error, info) => {
    telemetry.fatal(error, info.componentStack); // no boundary caught it — page is gone
  },
  onCaughtError: (error, info) => {
    telemetry.handled(error, info.componentStack); // a boundary caught it — fallback shown
  },
  onRecoverableError: (error, info) => {
    telemetry.recovered(error, info.componentStack); // React auto-recovered (e.g. hydration retry)
  },
}).render(<App />);
```

Every render-pipeline error in the app now reaches exactly one reporting seam, boundary or not — and 19 also cleaned up the dev experience (one console entry with real context, not the old duplicate logs).

### Why a class

The boundary contract is "React calls *these methods on this fiber* during unwind" — and no hook exposes that seam; as of 19.2 none is planned into stability. Hence the standing convention from the [roadmap](../../roadmap.md): classes appear in this library exactly once, here, immediately wrapped:

```tsx
// The class, written once for understanding — you will import, not write, this.
import { Component, type ErrorInfo, type ReactNode } from "react";

interface Props {
  fallback: (error: unknown, reset: () => void) => ReactNode;
  onError?: (error: unknown, componentStack: string | undefined) => void;
  children: ReactNode;
}
interface State {
  error: unknown | null;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null };

  static getDerivedStateFromError(error: unknown): State {
    return { error };
  }

  componentDidCatch(error: unknown, info: ErrorInfo) {
    this.props.onError?.(error, info.componentStack ?? undefined);
  }

  reset = () => this.setState({ error: null });

  render() {
    if (this.state.error !== null) {
      return this.props.fallback(this.state.error, this.reset);
    }
    return this.props.children;
  }
}
```

Thirty lines, no mystery — and three production concerns it *doesn't* handle (reset-on-data-change, async bridging, fallback-component ergonomics), which is exactly what `react-error-boundary` adds.

## Basic usage

The library, in its three shapes:

```tsx
import { ErrorBoundary, useErrorBoundary } from "react-error-boundary";

// 1. Static fallback — for regions with nothing to retry
<ErrorBoundary fallback={<p role="alert">The activity feed hit an error.</p>}>
  <ActivityFeed />
</ErrorBoundary>

// 2. Fallback with recovery — the workhorse
function WidgetFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert" className="widget widget--errored">
      <p>This panel failed to render.</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

<ErrorBoundary
  FallbackComponent={WidgetFallback}
  onError={(error, info) => telemetry.handled(error, info.componentStack)}
  resetKeys={[selectedRegion]}   // new region ⇒ old error no longer relevant ⇒ auto-reset
  onReset={() => refetchRegion(selectedRegion)}
>
  <RevenueWidget region={selectedRegion} />
</ErrorBoundary>

// 3. The async bridge — routing handler/async errors into the SAME surface
function SaveButton({ draft }: { draft: Draft }) {
  const { showBoundary } = useErrorBoundary();

  async function handleSave() {
    try {
      await api.saveDraft(draft);
    } catch (error) {
      showBoundary(error); // handlers aren't caught by boundaries — this opts in
    }
  }

  return <button onClick={handleSave}>Save</button>;
}
```

`resetKeys` deserves a highlight: it ties the error's *lifetime* to data identity — when the keys change, the failure that belonged to the old data auto-clears and the children re-render fresh. It's the boundary-world sibling of the `key`-reset idiom from [state-and-usestate](../state/state-and-usestate.md).

## Walkthrough: containing a dashboard

The scenario: a 14-widget ops dashboard. At 02:00, a malformed metrics payload makes one sparkline's render divide by zero — and pre-boundaries, on-call gets paged for a **blank page**, because one widget's throw unwound to the root and unmounted navigation, filters, and the thirteen healthy widgets with it.

**Step 1 — one boundary per independently useful region.**

```tsx
// Dashboard.tsx
import { ErrorBoundary } from "react-error-boundary";
import { WidgetFallback } from "./WidgetFallback";
import { widgets } from "./widget-registry";

export function Dashboard({ filters }: { filters: DashboardFilters }) {
  return (
    <main className="dashboard">
      {widgets.map((w) => (
        <ErrorBoundary
          key={w.id}
          FallbackComponent={WidgetFallback}
          onError={(error, info) =>
            telemetry.handled(error, info.componentStack, { widget: w.id })
          }
          resetKeys={[filters]}   // new filters ⇒ new data ⇒ every errored widget retries
        >
          <w.Component filters={filters} />
        </ErrorBoundary>
      ))}
    </main>
  );
}
```

Re-run the incident: the sparkline throws, its boundary renders a small "failed to render / try again" card, telemetry records exactly which widget with its component stack, and the other thirteen widgets — plus filters and nav — never notice. Blast radius: one grid cell. Changing any filter auto-resets the errored cell via `resetKeys`, because a new query invalidates the old failure.

**Step 2 — the taxonomy line, drawn in code.** The same widget fetches data. A 500 from the API is *not* a throw-to-boundary case — it's an **expected, renderable outcome**, which means it's *state*, in the status-union discipline from [usereducer-and-state-structure](../state/usereducer-and-state-structure.md):

```tsx
// Inside RevenueWidget — expected failure = state you render
const [state, dispatch] = useReducer(widgetReducer, { status: "loading" });
/* fetch dispatches { type: "failed", message } — the union has an "error" arm the JSX renders */

// The boundary catches what render CANNOT survive — the unexpected:
// malformed payload → buildSparkline() throws → boundary fallback.
```

The line: **if the UI has a sensible thing to show, model it in the union and render it; throw only for states the component cannot render its way out of.** Boundaries are the outermost `default:` of your UI, not its `else`.

**Step 3 — bridge the async gap.** The dashboard's "export report" handler hits a generation service; a failure there should present in the same errored-card style, not an `alert()`. `useErrorBoundary().showBoundary(error)` from the basic-usage section routes it into the widget's boundary — one visual and telemetry surface for both worlds.

**Step 4 — root telemetry.** The 19 root options from under-the-hood go in `main.tsx`, so anything that slips every boundary (or gets auto-recovered) still lands in monitoring with a component stack. Boundary `onError` gives per-region tags; root hooks guarantee nothing is silent.

**Step 5 — test the fallback path.**

```tsx
// Dashboard.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { expect, it, vi } from "vitest";

function Bomb({ defused }: { defused: boolean }) {
  if (!defused) throw new Error("payload malformed");
  return <p>sparkline</p>;
}

it("contains the throw and recovers on retry", async () => {
  const consoleSpy = vi.spyOn(console, "error").mockImplementation(() => {}); // expected noise
  const user = userEvent.setup();
  let defused = false;

  const { rerender } = render(
    <ErrorBoundary FallbackComponent={WidgetFallback}>
      <Bomb defused={defused} />
    </ErrorBoundary>,
  );

  expect(screen.getByRole("alert")).toHaveTextContent(/failed to render/i);

  defused = true; // fix the underlying condition FIRST…
  rerender(
    <ErrorBoundary FallbackComponent={WidgetFallback}>
      <Bomb defused={defused} />
    </ErrorBoundary>,
  ); // …(the boundary still shows the fallback — an error isn't cleared by new props)…

  await user.click(screen.getByRole("button", { name: /try again/i })); // …THEN retry

  expect(screen.getByText("sparkline")).toBeInTheDocument();
  consoleSpy.mockRestore();
});
```

Two habits worth copying: the boundary test asserts the **role="alert"** fallback (the accessible contract, not the styling), and the console spy acknowledges that a caught error still logs in dev — silence it *per expected-error test*, never globally. And one ordering lesson the test encodes: fix the condition, *then* reset — retrying first just re-catches, which is exactly what your users will discover if the retry button doesn't actually change anything.

## Real-world patterns

### Placement: the granularity ladder

Root boundary (full-page "something went wrong" — the last resort that should almost never render) → **route-level** (each page fails alone; in React Router data mode this is the router's own `errorElement`/`ErrorBoundary` per route, which also catches loader errors — [routing-react-router](../ecosystem/routing-react-router.md) *(Wave 4, planned)* owns that integration) → **feature/widget level** (this walkthrough — regions that are independently useful fail independently) → and no lower: a boundary per button is ceremony catching nothing render-shaped. The placement question is always "what's the largest region whose death this fallback adequately replaces?"

### Reset design

A fallback without an exit is a dead end wearing an apology. Three exits, composable: a **retry button** (`resetErrorBoundary`) for transient faults; **`resetKeys`** bound to the data identity the region renders (route params, filters, selected id) so navigation heals errors automatically; and **`onReset`** to clear whatever cached poison caused the throw — with a server cache in play, that's the `QueryErrorResetBoundary` pairing so the retry actually refetches instead of re-throwing from cache ([data-fetching-tanstack-query](../ecosystem/data-fetching-tanstack-query.md) *(Wave 4, planned)*).

### The `try/catch` redirect, paid

[rules-of-react](../foundations/rules-of-react.md) banned `try/catch` around hooks and promised the real strategy here. The division: **`try/catch` is for operations** — handlers and effects doing async work, where you catch, classify, and usually dispatch an error *state*. **Boundaries are for rendering** — the declarative phase can't be wrapped statement-by-statement (hooks break, and half-rendered output is exactly what React refuses to commit), so React gives you catch-as-a-component instead. If you feel the urge to `try/catch` inside render, you're either handling an expected case (make it state) or an unexpected one (let it throw to the boundary).

### The stale-chunk fallback

The most common production boundary firing in SPAs isn't your code — it's `import()` rejecting after a deploy, when a lazily loaded route's chunk hash no longer exists. A route-level boundary that detects the chunk-load error shape and renders "A new version is available — **Reload**" converts a mystery white screen into a one-click recovery. (The `lazy`/Suspense mechanics belong to [suspense](../concurrent/suspense.md) *(Wave 3, planned)*; the boundary is the half you can ship today.)

### Fallbacks stay dumb

The fallback renders *instead of* the region that just proved the data can be poison — so it must not touch that data. Static text, the error message at most, a retry control. A fallback that formats `error.data.summary` is a second throw waiting, and *that* one propagates to the next boundary up.

## Reference

| Where the error happens | Caught by a boundary? | Your tool |
| --- | --- | --- |
| Component render (incl. `use()` rejection) | ✅ nearest ancestor | Boundary fallback |
| `useEffect` / `useLayoutEffect` (setup or cleanup) | ✅ | Boundary fallback |
| Class lifecycle / constructor | ✅ | Boundary fallback |
| Event handler | ❌ | `try/catch` → state, or `showBoundary` |
| `setTimeout` / detached promise | ❌ | `try/catch` → state, or `showBoundary` |
| The boundary's own render | ❌ (next ancestor) | Keep fallbacks dumb |
| SSR render | ❌ (client boundaries) | Server pipeline (Wave 3) |

| `react-error-boundary` API | Job |
| --- | --- |
| `fallback` / `FallbackComponent` / `fallbackRender` | Static node / component with `{error, resetErrorBoundary}` / inline render fn |
| `onError(error, info)` | Reporting seam (componentStack included) |
| `resetKeys` | Auto-reset when listed values change |
| `onReset` | Clear caches / refetch alongside a reset |
| `useErrorBoundary()` | `showBoundary(error)` — the async/handler bridge; also `resetBoundary()` |

## Common mistakes

**1. Zero boundaries, or one at the root.** Zero means any render throw blanks the app. Root-only means a widget bug takes the whole *page* — technically caught, practically still an outage. Boundaries are a granularity tool; use the ladder.

**2. Expecting handler errors to be caught.**

```tsx
<ErrorBoundary fallback={<Oops />}>
  <button onClick={() => { throw new Error("nope"); }}>…</button> {/* 🔴 sails past the boundary */}
</ErrorBoundary>
```

Handlers never enter the render pipeline. `try/catch` and decide: state (expected) or `showBoundary` (unrenderable).

**3. Boundaries as control flow for expected failures.** Throwing on a 404 or a validation miss to "let the boundary handle it" turns your most normal outcomes into telemetry noise and your UI into a fallback slideshow. Expected outcomes are union arms you *render* — the taxonomy line from the walkthrough.

**4. `try/catch` in render.** Around hooks it's a rules violation (call-order corruption); around JSX it's a category error — the throw you'd catch is the mechanism boundaries are built on. Redirect per the pattern above.

**5. A boundary trying to catch itself.**

```tsx
function Widget() {
  return (
    <ErrorBoundary fallback={<Oops />}>{/* 🔴 protects children — Widget's own render throw escapes */}
      {computeRows(data).map(/* … */)}
    </ErrorBoundary>
  );
}
```

If `computeRows` throws, it throws in `Widget`'s render, *above* this boundary. The boundary belongs in the parent — protection is always from outside.

**6. Smart fallbacks.** Reading the failed region's data in the fallback re-throws with the same poison and escalates the blast radius one boundary up. Dumb text, retry button, done.

**7. Swallowing silently.** A boundary with no `onError` and no root hooks converts crashes into invisible degradation — users see "try again" cards; dashboards see nothing. Wire both seams: per-boundary tags, root-level catch-all.

**8. No way back.** A fallback without retry/resetKeys makes a transient 2am payload glitch permanent until refresh. Every fallback ships an exit; `resetKeys` on data identity makes most exits automatic.

**9. Misreading the dev double-invoke.** The synchronous retry (and StrictMode's rehearsals) can make a throwing component log twice per incident in dev. Two logs ≠ two errors — count committed fallbacks, not console lines.

## How this evolved

- **Pre-16:** a render throw left React holding corrupted internal state — subsequent renders on the broken tree misbehaved unpredictably. There was no story.
- **React 16:** the story arrives as a pair: error boundaries, and the hard default that an *uncaught* render error unmounts everything — corruption traded for a blank page, with boundaries as the tool for anything better. `componentDidCatch` first; **16.6** splits the render-phase half into `getDerivedStateFromError`.
- **Hooks era (16.8+):** every lifecycle gets a hook equivalent except this one; `react-error-boundary` becomes the community-standard wrapper and the reason nobody's written the class since.
- **React 18:** concurrent rendering adds the synchronous-retry-before-fallback semantics.
- **React 19 / 19.2:** root `onUncaughtError`/`onCaughtError`/`onRecoverableError` give a first-class telemetry seam; error logs deduplicated with better context; `use()` makes rejected promises a *render-phase* error, pulling data failures into boundary jurisdiction (with Suspense as the pending-side sibling — Wave 3).

## Exercises

**1. Blast-radius audit.** Take any multi-region page and rig one region to throw on a flag. Run it three times: no boundaries, root boundary only, per-region boundaries — screenshot what the user keeps each time, and check what `onUncaughtError` vs `onCaughtError` received. Write the one-sentence placement policy your app should adopt.
*Hint: the root-only run is the persuasive one — "caught" and "acceptable" are different claims.*

**2. One surface for two worlds.** A widget both renders server data (expected failures) and exports a file from a handler (async failures). Implement it so an API 500 renders as a status-union arm, a malformed-payload render throw hits the boundary, and an export failure routes via `showBoundary` — all three visually distinct or deliberately identical, your call, but *decided*.
*Hint: the union arm and the fallback are different components with different powers — one may retry via refetch, the other via `resetErrorBoundary`. Which failure deserves which?*

**3. The stale-chunk drill.** With a lazy route, mock the dynamic import to reject once with a chunk-load-shaped error (then succeed). Build the route-level fallback that recognizes the shape and offers Reload vs the generic fallback for everything else. Test both branches.
*Hint: `vi.mock` the module with a rejecting factory on first call; the error-shape check is `error.name`/message sniffing — keep it in one `isStaleChunkError(error)` helper you can unit-test separately.*

## Summary

- A render throw unwinds up the fiber tree to the nearest boundary — or unmounts the app by design. Boundaries are catch-as-a-component: fallback via `getDerivedStateFromError`, reporting via `componentDidCatch`, protection always from outside, children only.
- The pipeline details that matter: concurrent renders retry synchronously before committing a fallback; effect errors route to boundaries too; handlers and detached async never do — bridge those with `showBoundary` or model them as state.
- The taxonomy carries the design: expected outcomes are **status-union arms you render**; only unrenderable states throw. `try/catch` owns operations; boundaries own rendering — that's the whole redirect from the rules article.
- Place boundaries per independently useful region, on the granularity ladder; give every fallback an exit (retry, `resetKeys` on data identity, `onReset` cache clearing); keep fallbacks too dumb to re-throw.
- Wire both telemetry seams — per-boundary `onError` for tags, 19's root options for the catch-all — so nothing fails silently, including the errors React recovered from on its own.

## See also

- [how-react-renders](./how-react-renders.md) — the render-phase unwind and commit machinery boundaries hook into
- [rules-of-react](../foundations/rules-of-react.md) — the `try/catch`-around-hooks ban this article's redirect completes
- [usereducer-and-state-structure](../state/usereducer-and-state-structure.md) — status unions: the errors-as-state half of the taxonomy
- [state-and-usestate](../state/state-and-usestate.md) — the `key`-reset idiom `resetKeys` generalizes
- [suspense](../concurrent/suspense.md) *(Wave 3, planned)* — the pending-side sibling; boundaries pair with it for `use()` and `lazy`
- [routing-react-router](../ecosystem/routing-react-router.md) *(Wave 4, planned)* — route-level `errorElement` and loader-error integration
- [data-fetching-tanstack-query](../ecosystem/data-fetching-tanstack-query.md) *(Wave 4, planned)* — `QueryErrorResetBoundary` and retry-that-refetches

## References

- [Catching rendering errors with an error boundary — react.dev](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [getDerivedStateFromError — react.dev](https://react.dev/reference/react/Component#static-getderivedstatefromerror)
- [componentDidCatch — react.dev](https://react.dev/reference/react/Component#componentdidcatch)
- [createRoot options — react.dev](https://react.dev/reference/react-dom/client/createRoot)
- [react-error-boundary — GitHub](https://github.com/bvaughn/react-error-boundary)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.