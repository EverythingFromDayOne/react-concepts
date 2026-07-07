---
article_id: ssr-and-hydration
concept_folder: server
wave: 3
related:
  - effects/escape-hatches-audit
  - rendering/error-boundaries
  - effects/custom-hooks
  - concurrent/suspense
  - server/server-components
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this.** Three symptoms, one root cause. Your app renders fast from the server, then the console throws a hydration mismatch and a chunk of the page flickers and re-renders on the client. Or the server crashes outright with `window is not defined`. Or you get a warning that `useLayoutEffect` does nothing on the server and can't figure out why it matters. All three are the same contract being violated: **the server produced HTML, and the client's first render must agree with it.** Server-side rendering is two renders of the same tree — one on the server that makes HTML, one on the client that adopts that HTML — and everything that goes wrong goes wrong at the seam between them. This article is the framework-agnostic mechanics of that seam: how hydration matches the DOM, why mismatches happen and what React 19 does about them, how to read browser-only values without breaking the agreement, and how errors flow through the streaming pipeline. Next.js applies all of this ([`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md)); here's what it's applying.

## What SSR and hydration are

Client-only React renders into an empty `<div id="root">`: the browser downloads JS, runs it, and only then does content appear. SSR moves the first render to the server:

1. **Server render.** The server runs your component tree and produces HTML — real markup, sent in the initial response. The browser paints it immediately, so the user sees content before any JavaScript has executed. Good for first-contentful-paint, largest-contentful-paint, and crawlers that don't run JS.
2. **Hydration.** The JS bundle loads and React runs the tree *again* on the client — but instead of creating DOM, it **adopts the existing server DOM**: matches its output against what's already there, reuses those nodes, and attaches event handling and state. `hydrateRoot(domNode, <App/>)` instead of `createRoot(domNode).render(<App/>)`.

The mental model that prevents most bugs: **SSR is not one render, it's two, and they must produce the same first frame.** The server render has no browser (no `window`, no DOM, no effects), and the client render has to land on exactly what the server emitted. Hold that and the rest of this article is consequences.

This is the framework-agnostic layer. The `react-dom/server` and `react-dom/client` APIs here are what Next.js, Remix-lineage routers, and hand-rolled SSR setups all sit on top of. Server Components ([`server-components`](./server-components.md)) are a *different* thing that builds on this foundation — SSR renders components to HTML; RSC is about which components run on the server at all. Keep them separate for now.

## How it works under the hood

### The server render

On the server, React renders the tree to HTML through a streaming API — `renderToPipeableStream` (Node streams) or `renderToReadableStream` (Web streams, for edge runtimes). The legacy `renderToString` still exists but is synchronous and can't stream or await Suspense well; the streaming APIs are the baseline.

What the server render *cannot* do is the crux of everything downstream: there is **no DOM, no commit phase, and no effects.** `useEffect` and `useLayoutEffect` never fire on the server (there's nothing to commit to). Refs are never attached to nodes. `window`, `document`, `navigator`, and `localStorage` don't exist. The server render is pure "compute the markup" — anything that needs the browser has to wait for the client.

### The hydration algorithm

`hydrateRoot` is a special render pass. Ordinary client rendering walks the fiber tree and *creates* DOM nodes. Hydration walks the fiber tree **and the existing server DOM in parallel**, and for each host component it expects to find a matching node already sitting in the DOM. Instead of creating a node, it **claims** the existing one — adopts it into the fiber, attaches the component's state and effects to it, and moves on.

Event handling comes from the root, not per-node. As [`portals-and-the-event-system`](../rendering/portals-and-the-event-system.md) and [`conditional-rendering-and-events`](../foundations/conditional-rendering-and-events.md) describe, React delegates events at the root; hydration attaches those root listeners and wires each fiber's handlers through the tree. This produces the SSR **interactive gap**: the HTML is on screen and looks ready the instant it paints, but nothing is clickable until hydration attaches the event system. A button rendered by the server is visible immediately and inert for a beat — a real UX consideration, not a bug.

### The agreement contract, and why mismatches are expensive

Hydration adopts existing DOM rather than creating it. That only works if the DOM already there is *exactly* the DOM the client's first render would have produced. So the contract is strict: **the client's initial render must match the server HTML, node for node.** When it doesn't, React can't safely claim the mismatched nodes — the fiber tree and the DOM disagree about what should be where.

What React 19 does on a mismatch: it **abandons the server HTML for the affected subtree and re-renders it from scratch on the client**, discarding the SSR work for that region. It logs a single consolidated error with a server-vs-client diff (a real improvement over pre-19's node-by-node warnings), and fires `onRecoverableError` on the root — "recoverable" because the app still works, it just fell back to client rendering there. At the root, a mismatch means the whole page client-renders and you've paid for SSR and gotten none of its benefit, plus a flicker as the server content is replaced.

React 19 is more tolerant of *some* divergence it knows is benign — extra attributes injected by browser extensions, for instance, no longer blow up hydration the way they once did. But logic-level divergence in your own render is still a fallback.

For divergence that is *legitimately* unavoidable and correct — a server-rendered timestamp that will differ from the client's clock by definition — there's a targeted escape hatch: the `suppressHydrationWarning` prop on the specific element. It tells React "I know this one node differs; don't warn, don't fall back." It is per-element and for genuine, known divergence only — not a global mismatch silencer (mistake #9).

### Streaming and selective hydration

The streaming APIs don't wait for the whole tree. The **shell** (everything outside Suspense boundaries) flushes first, so the user sees the frame immediately; each Suspense boundary's content streams in later, as its data resolves, injected via a `<template>` and a tiny inline script that swaps the real content in for the fallback. [`suspense`](../concurrent/suspense.md) owns the reveal semantics (and 19.2's reveal batching); this is the server side that produces the stream.

Hydration is **selective** and concurrent ([`concurrent-rendering`](../concurrent/concurrent-rendering.md)): React hydrates boundaries independently as their HTML arrives, and *prioritizes* the boundary the user interacts with first. Click a button whose region hasn't hydrated yet, and React hydrates that region ahead of the others so the interaction lands as soon as possible. Interruptible, prioritized rendering applied to the hydration pass itself.

## Walkthrough: one request, end to end

The two entry files that make SSR work — and one request traced through both. This is the machinery the API table abstracts.

**The app** renders the whole document, with the slow region behind a Suspense boundary so the shell can flush before its data is ready:

```tsx
// App.tsx
import { Suspense } from "react";

export function App({ url }: { url: string }) {
  return (
    <html lang="en">
      <head><title>Shop</title></head>
      <body>
        <div id="root">
          <Header />
          <Suspense fallback={<ProductsSkeleton />}>
            <Products url={url} /> {/* streams in when its data resolves */}
          </Suspense>
        </div>
      </body>
    </html>
  );
}
```

**The server entry** streams HTML and wires the staged error callbacks from the pipeline section:

```tsx
// entry-server.tsx
import { renderToPipeableStream } from "react-dom/server";
import { App } from "./App";

export function handleRequest(req: Request, res: NodeResponse) {
  let didError = false;
  const { pipe } = renderToPipeableStream(<App url={req.url} />, {
    bootstrapScripts: ["/entry-client.js"], // the JS that will hydrate
    onShellReady() {
      // Shell (everything outside Suspense) is ready → start streaming now,
      // before <Products> data has resolved.
      res.statusCode = didError ? 500 : 200;
      res.setHeader("Content-Type", "text/html");
      pipe(res);
    },
    onShellError() {
      // Shell itself failed — nothing to stream. Send a fallback page.
      res.statusCode = 500;
      res.end("<!doctype html><p>Something went wrong.</p>");
    },
    onError(error) {
      didError = true;
      logToTelemetry(error); // every streaming error, recoverable or fatal
    },
  });
}
```

**The client entry** adopts the streamed DOM:

```tsx
// entry-client.tsx
import { hydrateRoot } from "react-dom/client";
import { App } from "./App";

hydrateRoot(document, <App url={window.location.pathname} />, {
  onRecoverableError(error) {
    logToTelemetry(error); // hydration mismatches and recoveries surface here
  },
});
```

Now trace a single request through both:

1. The request arrives. `renderToPipeableStream` renders the **shell** — `<html>`, `<head>`, `<Header/>`, and the `<Suspense>` boundary's *fallback* (`<ProductsSkeleton/>`), because `<Products>`'s data isn't ready. It does **not** wait for `<Products>`.
2. `onShellReady` fires. The server sets the status and `pipe`s the shell HTML to the response. The browser paints the header and the skeleton immediately — first contentful paint before any component JS has run.
3. `<Products>`'s data resolves on the server. React renders it and **streams** its HTML down the still-open response, wrapped in a `<template>` plus a tiny inline script that swaps it in for the skeleton. The user sees products appear — still no application JS required.
4. Meanwhile `bootstrapScripts` has been loading `entry-client.js`. It runs, calls `hydrateRoot(document, …)`, and React walks the fiber tree against the streamed DOM, **claiming** each node instead of creating it and attaching the root event system.
5. **Selective hydration** wires up the shell first and prioritizes whatever the user interacts with — click the not-yet-hydrated header nav and React hydrates that region ahead of the rest.
6. If, and only if, the client's first render disagreed with the streamed HTML anywhere, `onRecoverableError` fires and that subtree is client-rendered. On a clean run it never fires — which is exactly the state you want your telemetry to confirm.

Every later section is a way to keep step 6 from firing (the browser-value patterns) or a way to handle failures gracefully within steps 1–3 (the error pipeline).

## Reading browser-only values without breaking the contract

This is where the `window`/`navigator` guards [`custom-hooks`](../effects/custom-hooks.md) fenced off, the `useLayoutEffect` SSR warning [`escape-hatches-audit`](../effects/escape-hatches-audit.md) flagged, and the `getServerSnapshot` pattern all converge — because they're the same problem: a value that only exists in the browser, needed by a tree that also renders on the server.

### The anti-pattern that guarantees a mismatch

```tsx
// ❌ Server renders the `false` branch (no window); client renders the `true` branch.
// First client render disagrees with server HTML → hydration mismatch.
function Widget() {
  const isWide = typeof window !== "undefined" && window.innerWidth > 1024;
  return <div>{isWide ? <WideLayout /> : <NarrowLayout />}</div>;
}
```

Guarding `window` with `typeof` stops the *server crash*, but branching on it *during render* trades the crash for a mismatch: the server (no `window`) and the client's first pass (has `window`) render different trees. The guard was necessary and insufficient.

### Pattern 1 — render the same on both passes, then update in an effect

The fix is a **two-pass render**: render the server-safe version on both the server and the *first* client pass (so they agree and hydration succeeds), then read the browser value in an effect and update. Effects don't run on the server, so the first client render still matches; the effect runs after commit and triggers a second render with the real value.

```tsx
import { useEffect, useState } from "react";

function Widget() {
  const [isWide, setIsWide] = useState(false); // server-safe default; matches server HTML
  useEffect(() => {
    setIsWide(window.innerWidth > 1024); // runs only on the client, after hydration
  }, []);
  return <div>{isWide ? <WideLayout /> : <NarrowLayout />}</div>;
}
```

The cost is one extra client render and a possible flash from `false`→`true`; the benefit is a clean hydration. For content that genuinely cannot render server-side at all, gate it entirely on a mounted flag (the mounted-gate, below).

### Pattern 2 — `useSyncExternalStore` with `getServerSnapshot`

When the browser value is needed *at render time* and comes from an external source (viewport, media query, `localStorage`, connection status), the right tool is `useSyncExternalStore` — [`escape-hatches-audit`](../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely) owns its `subscribe`/`getSnapshot` invariants and the tearing problem it solves. What this article owns is its **third argument, `getServerSnapshot`**, which exists precisely for SSR:

```tsx
import { useSyncExternalStore } from "react";

function subscribe(callback: () => void) {
  const mq = window.matchMedia("(min-width: 1024px)");
  mq.addEventListener("change", callback);
  return () => mq.removeEventListener("change", callback);
}

// SSR-safe custom hook — the window guards custom-hooks.md deferred here.
export function useIsWide(): boolean {
  return useSyncExternalStore(
    subscribe,
    () => window.matchMedia("(min-width: 1024px)").matches, // client snapshot
    () => false,                                             // server snapshot — the SSR half
  );
}
```

`getServerSnapshot` returns the value React uses during server rendering **and during the first client (hydration) render** — so both passes see `false`, they agree, hydration succeeds, and only afterward does the client subscribe and update to the real value. It resolves the exact tension the anti-pattern hit: a server-safe first frame, then the browser truth. This is the SSR-safe shape for every browser-only custom hook — `useMediaQuery`, `useLocalStorage`, `useOnlineStatus` — and it's why `getServerSnapshot` is mandatory (not optional) for any store read in a tree that renders on the server.

### The `useLayoutEffect` warning, resolved

`useLayoutEffect` runs synchronously after DOM mutation and before paint — it needs a DOM and a commit phase, neither of which exists on the server. So it never runs server-side, and React warns when you use it in SSR: whatever layout measurement it performs won't have happened for the server render, which can cause a flash or a mismatch if the first paint depended on it.

The resolution depends on what the layout effect was for:

- **If the timing isn't truly pre-paint,** move it to `useEffect`. Both are no-ops on the server, but you drop the warning and the intent is honest.
- **If you need a browser value at render time,** use `useSyncExternalStore` + `getServerSnapshot` (Pattern 2) — that's the render-time-safe path.
- **If the content genuinely can't exist until measured in the browser,** gate it on a mounted flag so the server and first client render emit the same placeholder, and the measured version appears post-hydration (the mounted-gate).

This is the payoff of [`escape-hatches-audit`](../effects/escape-hatches-audit.md)'s note that `useLayoutEffect` has an SSR cost: the cost is that it can't participate in the server render, so anything the first frame depends on has to reach render through `getServerSnapshot`, not through a layout effect.

## The SSR error pipeline

[`error-boundaries`](../rendering/error-boundaries.md) owns the boundary itself and the 19 root error options; this article owns how errors flow through the *server render and hydration*, which has more stages than client-only rendering.

**Error boundaries work on the server.** A boundary in the tree catches an error thrown during server rendering and emits its fallback *as HTML*. So SSR resilience is designed the same way as client resilience — place boundaries around risky regions and the server streams a fallback for that region instead of failing the page.

The streaming render exposes staged error callbacks (on `renderToPipeableStream`/`renderToReadableStream`):

- **`onShellError`** — the *shell* (content outside all Suspense boundaries) threw before anything streamed. There's no partial page to salvage, so this is where you send a fallback response: a 500 page, or a client-only entry point. Not handling it means a blank or broken initial response.
- **`onError`** — fires for *every* error during streaming, for logging. React distinguishes **recoverable** errors (it streamed a fallback and the client will retry) from fatal ones. This is your logging hook; most streaming errors that hit it are recoverable.
- **Suspense-boundary errors during streaming.** If a component inside a Suspense boundary errors *after* the shell has already flushed, React can't un-send the HTML — so it streams instructions telling the client to switch that boundary to client rendering and retry there. The error becomes recoverable: the server gave up on that boundary, the client re-renders it, and if it errors again on the client, the client error boundary catches it. This is the streaming-side of the stale-chunk / boundary-retry story [`suspense`](../concurrent/suspense.md) and [`error-boundaries`](../rendering/error-boundaries.md) set up.

On the client, `hydrateRoot` takes the root error options [`error-boundaries`](../rendering/error-boundaries.md#react-19s-root-level-reporting) owns — `onRecoverableError`, `onCaughtError`, `onUncaughtError`. In the SSR context, `onRecoverableError` is the one that fires most: it's called when React recovers from a hydration mismatch (by client-rendering the subtree) or completes a server-initiated boundary retry. Wire it to your telemetry — a spike in recoverable errors is usually a hydration-mismatch bug leaking SSR performance, even though nothing is visibly broken.

## Real-world patterns

**The mounted-gate for client-only UI.** Some widgets can't render server-side meaningfully (a chart reading `window` dimensions, anything depending on `localStorage`). Render a stable placeholder on the server and first client pass, flip after mount:

```tsx
function ClientOnly({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted ? <>{children}</> : <Placeholder />; // server + first client render agree on <Placeholder />
}
```

**`suppressHydrationWarning` for known-good divergence.** A rendered timestamp, a "time since" label, a value that is *supposed* to reflect the client's clock — mark that specific node and only that node. Never blanket the app.

**Streaming shell + Suspense for perceived speed.** Put the slow, data-dependent regions behind Suspense boundaries so the shell flushes instantly and data streams in — the server side of [`suspense`](../concurrent/suspense.md)'s progressive reveal. Pair each risky boundary with an error boundary so a failure streams a fallback instead of toppling the page.

**SSR-safe custom hooks by default.** Any hook that reads the browser should ship with a `getServerSnapshot` from day one, so consumers can render it server-side without a mismatch. A browser-only hook without a server snapshot is a latent SSR bug.

**Partial pre-rendering (emerging).** React 19.2's `prerender` family lets you pre-render a static shell ahead of time and fill dynamic holes at request time — a static-plus-dynamic hybrid. It's mostly consumed through frameworks; [`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md) owns the production shape. Know it exists; reach for it through your framework.

## API and type reference

| API | Module | Purpose |
| --- | --- | --- |
| `renderToPipeableStream(node, options)` | `react-dom/server` | Stream HTML on Node; `onShellReady`, `onShellError`, `onError`, `onAllReady`. |
| `renderToReadableStream(node, options)` | `react-dom/server` | Stream HTML on Web/edge runtimes; `onError`, `signal`, `.allReady`. |
| `hydrateRoot(domNode, node, options?)` | `react-dom/client` | Adopt server HTML; `onRecoverableError`, `onCaughtError`, `onUncaughtError`. |
| `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)` | `react` | Read an external store; third arg supplies the server/first-hydration value. |
| `suppressHydrationWarning` | host element prop | Suppress the mismatch warning on one node with legitimate divergence. |
| `prerender` / `prerenderToNodeStream` | `react-dom/static` | Static/partial pre-render (framework-consumed). |

## Common mistakes

**1. Reading `window`/`document`/`localStorage` during render.** Crashes the server render outright. Read browser APIs in an effect, or through `useSyncExternalStore` + `getServerSnapshot` — never bare in the render body.

**2. Branching on `typeof window` in render.** Fixes the crash, causes a mismatch: server renders one branch, client's first pass renders the other. Use the two-pass pattern (render server-safe, update in an effect) or `getServerSnapshot`.

**3. Non-deterministic render output.** `Date.now()`, `Math.random()`, `new Date().toLocaleString()` (timezone/locale differ server vs client) all produce different HTML on each side → mismatch. Compute deterministically, mark the node with `suppressHydrationWarning` if the divergence is intended, or render a stable value and update after mount.

**4. Invalid HTML nesting.** A `<div>` inside a `<p>`, or a `<p>` inside a `<p>`: the browser auto-corrects the server HTML into valid structure ([`jsx-and-rendering`](../foundations/jsx-and-rendering.md#one-more-dom-reality-the-browser-edits-your-markup)), so the DOM no longer matches what React expects to hydrate → mismatch. Emit valid HTML.

**5. `useLayoutEffect` for server-relevant layout.** It doesn't run on the server, so the first frame lacks the measurement → warning and flash. Move to `useEffect`, use `getServerSnapshot` for render-time values, or mount-gate the measured content.

**6. Expecting effects to fetch SSR data.** Effects don't run on the server, so `useEffect(fetch)` produces an empty first render server-side (loading state → no SSR benefit). Data needed in the server HTML must be available *during* render (loader, RSC, or a passed-in promise via `use()` — [`use-and-promises`](../concurrent/use-and-promises.md)), not fetched in an effect.

**7. No `onShellError` handler.** A shell-level error with no handler yields a blank or broken initial response and no fallback. Always handle it — send an error page or a client-only entry.

**8. Assuming hydration is instant.** The interactive gap is real: HTML paints before the event system attaches. Don't assume a handler works the moment content is visible; for critical interactions, prioritize hydrating that region (selective hydration does this when the user interacts).

**9. Global hydration-warning suppression.** `suppressHydrationWarning` is per-element and for genuine divergence. Reaching for it to silence a mismatch you don't understand hides a real bug that's costing you SSR on that subtree. Diagnose the mismatch; suppress only the node that legitimately differs.

**10. Pre-hydration DOM mutation.** A third-party script that mutates the DOM before React hydrates changes the nodes out from under the hydration pass → mismatch. Defer such scripts until after hydration, or scope them away from React-owned DOM.

## How this evolved

- **`renderToString`.** The original: synchronous, blocking, no streaming, poor Suspense support. You rendered the whole tree to a string, sent it, and hydrated with `ReactDOM.hydrate`. A slow data dependency blocked the entire response.
- **18: streaming SSR + selective hydration.** `renderToPipeableStream`/`renderToReadableStream`, Suspense boundaries that stream independently, selective and prioritized hydration, and `hydrateRoot` replacing `hydrate`. The shell could flush before the data was ready — a step-change in perceived performance.
- **19: a sturdier seam.** Consolidated mismatch errors with a server-vs-client diff, the root error options (`onRecoverableError`/`onCaughtError`/`onUncaughtError`) on `hydrateRoot`, and more tolerant hydration for benign third-party DOM injection (browser extensions). The error pipeline became legible.
- **19.2 (this baseline):** Suspense reveal batching on the server (client/server reveal timing aligned, `<ViewTransition>` for Suspense), the `prerender` family for static and partial pre-rendering, and hydration fixes for re-suspending boundaries. SSR matured from "render a string" into a streaming, recoverable, partially-static pipeline — the foundation Server Components ([`server-components`](./server-components.md)) build on next.

## Exercises

**1. Fix a server crash the right way.** Take a `useWindowWidth` hook that reads `window.innerWidth` during render (crashes SSR) and rewrite it with `useSyncExternalStore` + a `getServerSnapshot` returning a safe default. Confirm no crash and no mismatch.
*Hint:* The server snapshot is the value both the server and the first client render use; they must agree.

**2. Break hydration, fix it twice.** Render `{Date.now()}` in a node and observe the mismatch. Fix it once with `suppressHydrationWarning`, and once by rendering a stable placeholder and setting the real value in an effect. Note the trade-off between the two.
*Hint:* `suppressHydrationWarning` keeps the server value briefly; the effect approach replaces it after mount.

**3. (Stretch) Survive a server-side failure.** Wrap a data-dependent widget in a Suspense boundary and an error boundary. Make it throw during the server render *after* the shell flushes, and confirm React streams a fallback and the client recovers — watch `onError`/`onRecoverableError` fire.
*Hint:* The shell must flush first (put the widget behind Suspense) so the error is recoverable rather than an `onShellError`.

## Summary

- SSR is **two renders of one tree**: the server makes HTML (no DOM, no effects, no `window`), and the client **hydrates** — adopts that HTML, reusing nodes and attaching the root event system. The interactive gap means visible ≠ clickable.
- The contract is strict: the **client's first render must match the server HTML**. React 19 handles a mismatch by discarding the subtree's HTML and client-rendering it, logging a diff and firing `onRecoverableError` — recoverable but a lost SSR benefit and a flicker.
- Browser-only values need a **server-safe first frame**: two-pass (render safe, update in an effect) or `useSyncExternalStore` with **`getServerSnapshot`**. Branching on `typeof window` in render guarantees a mismatch.
- `useLayoutEffect` doesn't run on the server (no DOM/commit) — for render-time browser values use `getServerSnapshot`; otherwise `useEffect` or a mounted-gate.
- The **SSR error pipeline** has stages: error boundaries emit fallback HTML on the server; `onShellError` for a failed shell; `onError` for logging; post-shell Suspense errors stream a fallback and hand off to the client; `hydrateRoot`'s root error options catch recovery client-side.
- Streaming flushes the shell first and streams Suspense boundaries; **selective hydration** prioritizes the region the user touches.

## See also

- [`escape-hatches-audit`](../effects/escape-hatches-audit.md) — `useSyncExternalStore`'s subscribe/getSnapshot invariants and the `useLayoutEffect` cost this article's `getServerSnapshot` and SSR-warning sections pay off.
- [`error-boundaries`](../rendering/error-boundaries.md) — the boundary and the root error options, whose SSR-pipeline behavior lives here.
- [`custom-hooks`](../effects/custom-hooks.md) — the `window`/`navigator` guards; the SSR-safe hook shape this article supplies.
- [`suspense`](../concurrent/suspense.md) — the streaming reveal whose server side produces the stream.
- [`server-components`](./server-components.md) *(planned, next)* — a different server concept built on this SSR foundation.
- [`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md) *(planned)* — production SSR, partial pre-rendering, and the framework wiring.

## References

- React docs — [`renderToPipeableStream`](https://react.dev/reference/react-dom/server/renderToPipeableStream)
- React docs — [`hydrateRoot`](https://react.dev/reference/react-dom/client/hydrateRoot)
- React docs — [`useSyncExternalStore`](https://react.dev/reference/react/useSyncExternalStore)
- React docs — [`renderToReadableStream`](https://react.dev/reference/react-dom/server/renderToReadableStream)

## Demo source

_Demo source pending — StackBlitz link to be added in the demo pass._