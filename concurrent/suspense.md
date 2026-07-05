---
article_id: suspense
concept_folder: concurrent
wave: 3
related:
  - rendering/error-boundaries
  - concurrent/concurrent-rendering
  - concurrent/use-and-promises
  - ecosystem/data-fetching-tanstack-query
  - server/ssr-and-hydration
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this.** You have a dashboard where three widgets each fetch their own data. The naive version threads an `isLoading` boolean through every one, and the render is a thicket of `if (loading) return <Spinner/>; if (error) return <Error/>; return <Data/>` repeated per widget, plus a parent that has to decide whether to gate the whole page on all three. Suspense inverts this: a component *suspends* — it tells React "I'm not ready" — and the nearest `<Suspense>` boundary shows a fallback until it is. Loading stops being a value you plumb through props and becomes a **boundary in the tree**, the same way an error boundary made failure a boundary instead of a return value. The catch worth stating up front: Suspense handles *pending*. It does **not** handle *rejected* — that's still an error boundary's job, and the two are designed to be stacked.

## What Suspense is

`<Suspense>` is a boundary that renders a fallback while any component in its subtree is suspended, and swaps to the real content once everything under it is ready.

```tsx
<Suspense fallback={<Skeleton />}>
  <Profile />   {/* if Profile suspends, we see <Skeleton /> */}
</Suspense>
```

"Suspended" is a specific state, not a metaphor. A component suspends when it needs something that isn't ready yet — code that hasn't downloaded, or data that hasn't resolved — and it signals that to React by throwing a thenable (a promise-like) during render. React catches it at the nearest Suspense boundary, shows the fallback, subscribes to the promise, and **retries** the subtree when it resolves.

Two distinct things suspend, and Suspense treats them identically:

- **Code.** `lazy(() => import("./Chart"))` returns a component whose module downloads on first render. Until the chunk arrives, it suspends. This is code-splitting, and it's the original (React 16.6) use of Suspense.
- **Data.** A component reads a pending promise — via `use(promise)` ([`use-and-promises`](./use-and-promises.md) owns the mechanics), a Suspense-enabled query, or an RSC — and suspends until the promise resolves. This is the 19-era use of Suspense.

The mental unlock: **loading is a boundary, not a prop.** You stop asking "is this loading?" at every leaf and instead declare "here is the region that has a loading state, and here is what it looks like while waiting." Suspense composes with error boundaries the way it does precisely because they're the same shape — a boundary that catches a signal thrown from below and renders something else. [`error-boundaries`](../rendering/error-boundaries.md) owns the errors-as-state-vs-errors-as-thrown taxonomy; Suspense is the *pending* counterpart of errors-as-thrown.

## How it works under the hood

### Suspending is a throw, catching is a boundary

When a component suspends, React doesn't get a return value — it gets a throw, structurally like the throw an error boundary catches (see [`error-boundaries`](../rendering/error-boundaries.md#unwind-mechanics) for the unwind machinery this reuses). But instead of a plain error, what's thrown is a **thenable**. React unwinds the fiber stack to the nearest `<Suspense>` boundary fiber, and that boundary switches which children it commits:

- Its **primary** children (the real subtree) are set aside — kept as pending work, not committed to the DOM.
- Its **fallback** is committed instead.

React then calls `.then()` on the thrown promise. When it resolves, React schedules a **retry**: it re-renders the boundary's primary children. If they now succeed, the fallback is swapped out for the real content. If they suspend *again* (a second, nested async dependency), the fallback stays and the cycle repeats — this is how a waterfall becomes visible as a fallback that lingers.

Which boundary catches is the same "nearest ancestor" rule error boundaries use: the *innermost* enclosing `<Suspense>` catches a suspension, and an outer boundary only sees it if there's no inner one between the suspending component and the root. That's what makes granular placement work — an inner boundary around a slow widget absorbs its suspension locally, so an outer page-level boundary never fires for it.

The retry is a *render*, and renders that suspend are discarded and retried — which means, exactly as with transitions ([`concurrent-rendering`](./concurrent-rendering.md)), **a suspended render never commits and never runs effects.** Effects fire only after a component successfully renders and commits ([`effects-and-synchronization`](../effects/effects-and-synchronization.md)). Don't put logic in render expecting it to run once "when loading starts"; it may run several times across retries, none of which commit until the last.

### The transition interplay — holding content instead of flashing a spinner

This is the single most important Suspense behavior and the reason it's in Wave 3 next to concurrent rendering rather than in Wave 1.

When a component suspends during the **initial** render of a boundary — there's no committed content yet — you get the fallback. Correct: there's nothing else to show.

But when an **update** to already-visible content causes a suspension — say a refresh, or navigating a tab whose new content fetches — the default is jarring: React hides the content you're already looking at and shows the fallback again. You had a rendered profile; you clicked refresh; it vanished and became a skeleton. That's worse than the loading state you were trying to improve.

Wrapping the update in a transition fixes it. When the suspending update is inside `startTransition` ([`concurrent-rendering`](./concurrent-rendering.md#basic-usage)), React **keeps the current committed UI visible** and renders the new (suspending) tree in the background at transition priority, surfacing `isPending` instead of the fallback. It only commits — swapping content — once the new tree is ready. The Suspense fallback appears only when there's no prior content to hold onto.

```tsx
function ProfilePage() {
  const [userId, setUserId] = useState("1");
  const [isPending, startTransition] = useTransition();

  function open(id: string) {
    // Without the transition, the profile below flashes to <Skeleton/> on every switch.
    // With it, the current profile stays put (dimmed) until the next one is ready.
    startTransition(() => setUserId(id));
  }

  return (
    <div style={{ opacity: isPending ? 0.6 : 1 }}>
      <UserList onOpen={open} />
      <Suspense fallback={<Skeleton />}>
        <Profile userId={userId} />
      </Suspense>
    </div>
  );
}
```

The rule that falls out: **the Suspense fallback is for "we have nothing yet"; the transition `isPending` is for "we have something, and something better is coming."** Initial load → fallback. Subsequent updates → transition.

### Reveal throttling

React doesn't reveal content the instant a promise resolves. It throttles reveals to avoid two ugly outcomes: a fallback that flashes for 30ms then vanishes (worse than no fallback), and a cascade of boundaries popping in one-by-one like popcorn. React 19.2 tightened this on the server side — streamed SSR Suspense boundaries now **batch their reveals** over a short window so related content appears together and aligns with client-rendered timing ([`ssr-and-hydration`](../server/ssr-and-hydration.md) owns the streaming side). The practical consequence: don't build logic that assumes the fallback is visible for exactly as long as the promise took. React decides reveal timing; you don't.

What React does *not* give you as a stable API is **coordinated ordering** — forcing sibling boundaries to reveal top-to-bottom regardless of which resolves first (the job `SuspenseList` was prototyped for). `SuspenseList` remains experimental and is not part of this 19.2 baseline, so don't reach for it in app code. If you need a strict reveal order across independent regions, coordinate it yourself — a single boundary over the group, or gating later regions on earlier data — rather than assuming a coordination primitive exists.

## Basic usage

### Code splitting with `lazy`

```tsx
import { lazy, Suspense } from "react";

// The import() runs on first render of <Chart/>, not at module load.
const Chart = lazy(() => import("./Chart"));

export function Analytics() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <Chart />
    </Suspense>
  );
}
```

`lazy` takes a function returning a `Promise<{ default: Component }>` — the shape a dynamic `import()` produces for a default export. The component downloads once, then it's cached; subsequent renders don't re-suspend.

### Data with `use`

```tsx
import { Suspense, use } from "react";

// The promise must be created OUTSIDE render and stable across renders —
// use-and-promises.md owns why. In production this comes from TanStack Query
// or an RSC, not a hand-rolled cache.
function Profile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // suspends until resolved; throws to error boundary if rejected
  return <h1>{user.name}</h1>;
}

export function ProfileSection({ userPromise }: { userPromise: Promise<User> }) {
  return (
    <Suspense fallback={<Skeleton />}>
      <Profile userPromise={userPromise} />
    </Suspense>
  );
}
```

`use()` reads a promise and suspends while it's pending. The critical constraint — the promise must be stable, not created fresh in render — belongs to [`use-and-promises`](./use-and-promises.md); we pass it in as a prop here so the examples are correct without re-teaching that article.

## Walkthrough: the loading / error / success triad

We'll build a user panel that lazy-loads, fetches data, handles both loading and failure declaratively, and refreshes without flashing. The point is composition: **Suspense for pending, ErrorBoundary for rejected, real content for resolved** — three states, three boundaries, zero `isLoading` booleans.

### Stage 1 — lazy panel, one boundary

```tsx
import { lazy, Suspense } from "react";

const ActivityPanel = lazy(() => import("./ActivityPanel"));

export function Dashboard() {
  return (
    <Suspense fallback={<PanelSkeleton />}>
      <ActivityPanel />
    </Suspense>
  );
}
```

First render downloads the chunk; the skeleton shows meanwhile. Note the fallback is a *skeleton matching the panel's layout*, not a centered spinner — a spinner that's replaced by a differently-sized panel shifts the page (CLS). Match the shape.

### Stage 2 — add data suspense with a granular boundary

The panel itself reads data. We give it its own inner boundary so the panel's chrome (header, tabs) can render as soon as the *code* arrives, and only the data region waits:

```tsx
// ActivityPanel.tsx
import { Suspense, use } from "react";

function ActivityFeed({ feedPromise }: { feedPromise: Promise<Activity[]> }) {
  const items = use(feedPromise);
  return (
    <ul>
      {items.map((a) => (
        <li key={a.id}>{a.summary}</li> // stable key from identity
      ))}
    </ul>
  );
}

export default function ActivityPanel({ feedPromise }: { feedPromise: Promise<Activity[]> }) {
  return (
    <section>
      <header>Recent activity</header>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed feedPromise={feedPromise} />
      </Suspense>
    </section>
  );
}
```

Now there are **nested boundaries**: the outer one (Stage 1) catches the code suspension; the inner one catches the data suspension. The header paints the moment the chunk loads; the list fills in when the feed resolves. Boundary placement *is* the reveal design — this is the same placement-granularity ladder [`error-boundaries`](../rendering/error-boundaries.md#placement-granularity-ladder) describes for errors, applied to loading. One boundary at the top means one big spinner; granular boundaries mean progressive reveal.

### Stage 3 — the error half (paying off the debt)

A pending promise shows the fallback. A **rejected** promise throws during render — and Suspense does not catch that. `use(feedPromise)` on a rejected promise re-throws the rejection, which propagates to the nearest **error boundary**. So the complete triad wraps the Suspense boundary in an error boundary:

```tsx
import { ErrorBoundary } from "react-error-boundary"; // error-boundaries.md owns this API

export function Dashboard({ feedPromise }: { feedPromise: Promise<Activity[]> }) {
  return (
    <ErrorBoundary FallbackComponent={PanelError} resetKeys={[feedPromise]}>
      <Suspense fallback={<PanelSkeleton />}>
        <ActivityPanel feedPromise={feedPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

The ordering is deliberate and it's the canonical Suspense pattern: **ErrorBoundary outside, Suspense inside.** Pending → the Suspense fallback (inner). Rejected → the error fallback (outer). Resolved → the real content. Three outcomes, cleanly separated, each declared once. This is the pairing [`error-boundaries`](../rendering/error-boundaries.md) promised and deferred to here: it owns the boundary itself, `resetKeys`, and the errors-as-thrown taxonomy; this article owns the fact that a *rejected `use()`* is one of the things that gets thrown, and that Suspense sits *inside* the error boundary so pending and rejected land in different fallbacks.

### Stage 4 — the stale-chunk case

Here's the Suspense-side half of the stale-chunk boundary [`error-boundaries`](../rendering/error-boundaries.md#reset-design) set up. A `lazy` import can *fail*, not just be slow: a user has a tab open, you deploy, the old chunk hash 404s, and their next lazy render rejects. That rejection throws to the error boundary exactly like a data rejection — `lazy` failures and `use()` failures are both errors-as-thrown. The fix is a reset path that retries the import (or, for a hard chunk-hash mismatch, reloads):

```tsx
function ChunkError({ resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert">
      <p>This section failed to load — you may be on an outdated version.</p>
      {/* resetErrorBoundary re-renders the subtree, which re-attempts the import().
          If the chunk is genuinely gone, prompt a reload instead. */}
      <button onClick={() => location.reload()}>Reload</button>
      <button onClick={resetErrorBoundary}>Retry</button>
    </div>
  );
}
```

The Suspense side of the story: because `lazy` re-suspends on a retry render, resetting the error boundary sends the subtree back through the Suspense fallback while the re-import is in flight — the loading and error states hand off to each other cleanly. `error-boundaries` owns the reset design; Suspense owns the re-suspend that the reset triggers.

## Real-world patterns

**Route-level code splitting.** The highest-leverage use: `lazy` each route, wrap the router outlet in a Suspense boundary, and pair it with the router's error handling (`errorElement` in React Router data mode — [`routing-react-router`](../ecosystem/routing-react-router.md) owns that integration). Each route's chunk downloads on navigation; the boundary shows a page-level skeleton.

**Preloading on intent.** A `lazy` component only starts downloading when it *renders*. To hide the load entirely, kick off the import on hover or focus — before the click — so the chunk is already cached when the user commits:

```tsx
const Editor = lazy(() => import("./Editor"));
function openEditor() { import("./Editor"); /* warm the chunk early */ }
// <button onMouseEnter={openEditor} onClick={showEditor}>Edit</button>
```

**Granular boundaries per independent region.** Independent data → independent boundaries → independent reveal. A slow "recommendations" widget shouldn't hold up a fast "profile header." One boundary per region that can fail or load on its own.

**Skeletons over spinners.** Layout-stable fallbacks (skeletons shaped like the content) keep CLS at zero and read as "this content is arriving here" rather than "something is happening somewhere." Reserve spinners for genuinely unknown-shape or full-page waits.

**Streaming SSR reveal.** On the server, Suspense boundaries stream: the shell flushes immediately with fallbacks, and each boundary's real content streams in as its data resolves, with React 19.2 batching the reveals. This is [`ssr-and-hydration`](../server/ssr-and-hydration.md)'s territory; the client-side mental model (boundary = a point where the page can flush early) carries straight over.

## API and type reference

| API | Signature | Notes |
| --- | --- | --- |
| `Suspense` | `<Suspense fallback={ReactNode}>{children}</Suspense>` | Catches suspensions from anywhere in `children`. `fallback` shows while any descendant is suspended. Nesting is legal and common. |
| `lazy` | `lazy(load: () => Promise<{ default: ComponentType<P> }>): ComponentType<P>` | Downloads the module on first render; caches after. The promise must resolve to a `{ default }` export. |
| `use` | `use<T>(promise: Promise<T>): T` | Suspends on pending, throws on rejected. Mechanics + the stable-promise rule: [`use-and-promises`](./use-and-promises.md). |

Type note: there is no "Suspense-aware" type distinction on a component — suspending is a runtime behavior (a throw), not a type-level contract. TypeScript can't tell you a component suspends; the Suspense boundary is a runtime safety net you place by knowing your data flow.

## Common mistakes

**1. Creating the promise inside render.** `use(fetch(...))` or `use(new Promise(...))` in a component body creates a *new* promise every render. Each render suspends on a promise that never resolves before the next render replaces it → infinite fallback or a retry loop. The promise must be stable (cached, or from a Suspense-enabled library). This is the central `use()` footgun — [`use-and-promises`](./use-and-promises.md) owns the full treatment; just know the Suspense symptom is "fallback that never resolves."

**2. Expecting Suspense to catch errors.** Suspense catches *pending*. A rejected promise or a thrown error goes to the nearest **error boundary**, not the Suspense boundary. If a failing fetch shows a blank screen instead of an error UI, you're missing the ErrorBoundary wrapper.

**3. Fallback that shifts layout.** A centered spinner replaced by a full data table jumps the page (CLS). Shape the fallback like the content it replaces.

**4. Boundary placed too high.** One `<Suspense>` at the app root means the entire page flashes to a fallback whenever *any* descendant suspends — one slow widget blanks everything. Place boundaries at the granularity you want to reveal at (the placement ladder from [`error-boundaries`](../rendering/error-boundaries.md#placement-granularity-ladder)).

**5. Not using a transition for updates.** The default on a suspending *update* is to hide existing content and show the fallback — jarring on refresh/navigation. Wrap the update in `startTransition` to hold the current content and show `isPending` instead. Initial load → fallback; update → transition.

**6. Request waterfalls.** Nested components each fetching their own data, each suspending, serialize the loads: the child can't start fetching until the parent resolves and renders it. Hoist the fetches so they start in parallel (kick off all the promises high, pass them down), or use a library that dedups and parallelizes. Suspense makes waterfalls *visible* (nested fallbacks lingering) but doesn't prevent them — [`data-fetching-tanstack-query`](../ecosystem/data-fetching-tanstack-query.md) owns the prevention.

**7. Reaching for Suspense when a boolean is clearer.** Suspense shines when loading is a *boundary* — a region with a coherent loading state, especially with code-splitting or Suspense-native data. For a single button's inline pending state, `useActionState`/`useFormStatus` ([`actions`](./actions.md)) or a plain flag from your query library is more direct. **When NOT:** don't wrap every leaf in a boundary to avoid writing one conditional; that's boundary sprawl, and it fragments your reveal into popcorn.

**8. Forgetting the reset path for lazy failures.** A failed chunk import stays failed — the error boundary catches it, but without a reset (`resetKeys` or `resetErrorBoundary`) the user is stuck on the error UI. Provide retry/reload, especially for deploy-induced stale chunks (Stage 4).

**9. Assuming exact fallback timing.** React throttles/batches reveals (19.2). A promise that resolves in 40ms may still show its fallback slightly longer to avoid a flash. Don't gate logic on "the fallback was visible for exactly N ms."

**10. Side effects in a suspending component's render.** A suspended render is discarded and retried; effects run only after a committed render. Logic that must run "once, on load" goes in a `useEffect` in the resolved component, not in render — same discipline as transition renders and the same rule from [`rules-of-react`](../foundations/rules-of-react.md).

## How this evolved

- **Pre-16.6:** no Suspense. Code-splitting via `react-loadable` or hand-rolled dynamic imports; loading state was a forest of `isLoading` booleans plumbed through props, one per async leaf.
- **16.6:** `React.lazy` + `Suspense` shipped — but for **code only**. Data suspense was unsupported in app code; the boundary caught lazy imports and nothing else.
- **18: concurrent Suspense.** Streaming SSR (`renderToPipeableStream`), selective hydration, and — the big one — transitions that **hold committed content instead of falling back**. Data suspense became real inside frameworks (RSC, Relay), though not yet in hand-written app code.
- **19:** `use()` made data suspense first-class in app code (with cached promises), and the loading/error/success triad became fully declarative — the ErrorBoundary-outside-Suspense-inside pattern is the idiom of this era.
- **19.2 (this baseline):** SSR Suspense **reveal batching** — server-streamed boundaries reveal together over a short window, aligning with client timing and enabling `<ViewTransition>` for Suspense; plus a hydration fix so re-suspending boundaries hide/unhide correctly. The trajectory: Suspense went from a code-splitting convenience to the unifying primitive for *every* "not ready yet" state in React.

## Exercises

**1. Boolean to boundary.** Take a widget that manages `isLoading`/`error`/`data` with `useState` and effects, and rewrite it as a cached resource read through `use()`, wrapped in Suspense (loading) inside an ErrorBoundary (error). Count the lines removed.
*Hint:* The promise comes from outside render — a module cache or a prop. If the fallback never resolves, you're creating the promise in render (mistake #1).

**2. Survive a stale deploy.** Wrap a `lazy` route in an ErrorBoundary. Simulate a chunk-load failure (throw from the `import`), confirm the error fallback shows, and give it a retry that re-suspends through the Suspense fallback on recovery.
*Hint:* `resetKeys` / `resetErrorBoundary` from [`error-boundaries`](../rendering/error-boundaries.md#reset-design); a genuinely-gone chunk needs a reload, not just a retry.

**3. (Stretch) Refresh without flashing.** Build a detail view with a refresh button. First without a transition — watch it blank to the skeleton on every refresh. Then wrap the refresh in `startTransition` and confirm the current content stays (dimmed) until the new data arrives.
*Hint:* The Suspense fallback is for "nothing yet"; `isPending` is for "something better coming" ([`concurrent-rendering`](./concurrent-rendering.md#the-transition-interplay--holding-content-instead-of-flashing-a-spinner)).

## Summary

- Suspense makes **loading a boundary, not a prop**: a component *suspends* (throws a thenable) and the nearest `<Suspense>` shows its fallback until the subtree is ready, then retries.
- Two things suspend, treated identically: **code** (`lazy`) and **data** (`use()` / Suspense-enabled sources). Suspended renders never commit and never run effects.
- Suspense handles **pending only.** Rejections (failed `use()`, failed `lazy` chunk) throw to an **error boundary** — stack ErrorBoundary *outside*, Suspense *inside* for the loading/error/success triad.
- On **updates**, wrap in `startTransition` so React holds existing content and shows `isPending` instead of blanking to the fallback. Fallback = "nothing yet"; transition = "something better coming."
- Placement is reveal design: granular boundaries reveal progressively; a too-high boundary blanks the whole page. React throttles reveal timing (batched on SSR in 19.2) — don't assume exact fallback duration.
- Suspense makes waterfalls visible but doesn't prevent them; parallelize fetches or use a library that dedups.

## See also

- [`concurrent-rendering`](./concurrent-rendering.md) — transitions, the mechanism that lets Suspense hold content instead of flashing a fallback.
- [`error-boundaries`](../rendering/error-boundaries.md) — the *rejected* half of the triad: the boundary, `resetKeys`, placement ladder, errors-as-thrown.
- [`use-and-promises`](./use-and-promises.md) *(planned, next)* — `use()` mechanics and the stable-promise rule the data examples depend on.
- [`data-fetching-tanstack-query`](../ecosystem/data-fetching-tanstack-query.md) *(planned)* — Suspense-enabled queries, dedup, and waterfall prevention.
- [`ssr-and-hydration`](../server/ssr-and-hydration.md) *(planned)* — streaming Suspense on the server and reveal batching.

## References

- React docs — [`<Suspense>`](https://react.dev/reference/react/Suspense)
- React docs — [`lazy`](https://react.dev/reference/react/lazy)
- React docs — [`use`](https://react.dev/reference/react/use)
- React blog — [React 19.2](https://react.dev/blog/2025/10/01/react-19-2)

## Demo source

_Demo source pending — StackBlitz link to be added in the demo pass._