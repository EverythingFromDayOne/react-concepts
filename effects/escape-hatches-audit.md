---
article_id: escape-hatches-audit
concept_folder: effects
wave: 2
related:
  - effects/effects-and-synchronization
  - rendering/how-react-renders
  - effects/useref-and-the-dom
  - effects/custom-hooks
  - state/state-and-usestate
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Escape Hatches Audit

> **Lead with this:** three hooks sit outside the mainstream: `useLayoutEffect`, `useSyncExternalStore`, and `flushSync`. They exist because the ordinary tools — `useEffect`, `useState`, normal batching — have edge cases the design couldn't close without them. Each one is precise about *why it can't wait* and exacts a cost for that impatience: `useLayoutEffect` blocks paint and must be fast; `useSyncExternalStore` reads external state synchronously and must never tear; `flushSync` collapses the scheduler and must be used sparingly. The wrong one applied to the wrong problem is worse than nothing — `useLayoutEffect` on every effect produces jank you'd never profile; `useSyncExternalStore` replacing `useState` is over-engineering for data that was never external. This article draws the exact lines, audits the one known correct site for each, and closes every forward reference the Wave 2 articles opened.

## What each one is, and what the ordinary tool missed

### `useLayoutEffect` — synchronous after mutation, before paint

`useLayoutEffect(setup, deps)` has an identical signature to `useEffect`. It differs in exactly one property: it fires in the **layout sub-phase** of commit — after React has mutated the DOM, *before the browser paints the frame*. This means:

- **The DOM is current and measurable.** `ref.current.getBoundingClientRect()` returns the final layout for this frame.
- **State dispatched here re-renders synchronously**, before paint — so a measure → set-state → re-render cycle is invisible to the user.
- **It blocks paint.** Every millisecond of layout-effect work is a millisecond the user waits. `useEffect` is "after paint, non-blocking"; `useLayoutEffect` is "before paint, blocking." That's the whole trade.

`useEffect` missed this because it was designed not to block: fire *after* the browser had a chance to draw, let the frame be fast, let the DOM stay interactive. That's right 95% of the time. The 5%: when you must **read the DOM and react to it in the same frame** — a tooltip that repositions to stay in-viewport, a scroll-lock that restores position, an autosized textarea, an animation that anchors to a measured element. React's rule of thumb: if the user would see a flash or a jump with `useEffect`, the use case belongs in `useLayoutEffect`.

The commit-phase ordering, in full ([how-react-renders](./how-react-renders.md) owns the broader pipeline):

```
commit: before-mutation → mutation → layout (refs attach, useLayoutEffect runs) → paint → passive (useEffect runs)
```

`useLayoutEffect`'s cleanup also runs in the layout phase — synchronously, before the next layout effect's setup — which is the same symmetry contract as effects ([effects-and-synchronization](../effects/effects-and-synchronization.md)).

**The server-rendering caveat.** `useLayoutEffect` can't run on the server (no DOM, no layout), so a server render of a component using it produces a warning and skips the effect. When you control what runs on the server you generally don't hit this; when you're in a meta-framework context ([ssr-and-hydration](../server/ssr-and-hydration.md) *(Wave 3, planned)*), gate with `typeof window !== "undefined"` or use `useEffect` with an immediate-sync-needed guard.

### `useSyncExternalStore` — subscribing to external state safely

```tsx
const snapshot = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?);
```

The problem it solves is subtle. In concurrent rendering, a component's read of an external mutable store — a Zustand slice, a Redux `store.getState()`, a module-level variable — and React's decision to commit that render can be separated by yielded frames. Another update can mutate the store mid-render. Two sibling components that *should* agree on the store can now have read at different moments — one sees the old value, one the new — and React commits both: **tearing**, a visually inconsistent frame.

`useState` doesn't tear because React *owns* the update: the snapshot is frozen at the start of the render and can't be mutated mid-render. An external store has no such guarantee.

`useSyncExternalStore` closes the gap:

- **`subscribe(listener)`**: register for store changes; return unsubscribe. React subscribes at mount and whenever `subscribe` changes (keep it stable — `useCallback` or module-level).
- **`getSnapshot()`**: read the current value *synchronously*. Must be pure and return a referentially stable value when the store hasn't changed — `Object.is` equality, same reference. React calls it on render and after every store notification; unstable snapshots (new object every call) produce infinite render loops.
- **React's guarantee**: if the snapshot changes between when React starts rendering and when it commits, the commit is torn — React discards it and re-renders synchronously. No user sees the inconsistent state.

`useEffect` missed this because its subscription-and-read pattern has the window: subscribe in the effect (after render), update state on notification (another render) — the gap between initial render's read and effect's subscription is a tear window. [tree branch 8 from effects-and-synchronization](./effects-and-synchronization.md) warned "this store pattern belongs in `useSyncExternalStore`"; this is the mechanism and the tool.

When the snapshot covers server rendering, `getServerSnapshot` supplies a deterministic value for the server/hydration pass (same contract: stable, pure).

### `flushSync` — collapsing the scheduler, right now

```tsx
import { flushSync } from "react-dom";

flushSync(() => {
  setState(value);          // or: dispatch, any state updater
});
// After this line: render and commit have already happened.
// DOM is updated. ref.current is fresh. Layout effects ran.
```

React 18 made batching the default in all contexts — `setTimeout`, promise callbacks, native event handlers — and that's almost always right: fewer renders, smoother animation, less work. `flushSync` is the deliberate opt-out: "I need the committed DOM *right here*, synchronously, because the next line of my code reads it."

`useEffect` and normal batching missed this because they defer by design. The one canonical use: a handler that **must see the committed DOM before it continues** — scroll-into-view after a state-driven insert, focus management that depends on a newly mounted node, third-party integrations that synchronously read the DOM after an imperative call.

`flushSync` is not a way to escape async React or to make state updates "immediate" in component logic — that's the stale-closure confusion [state-and-usestate](../state/state-and-usestate.md) resolves. It's for the specific case where JavaScript that runs *after* `flushSync(…)` must operate on a DOM that reflects the state change inside it.

**The cost**: synchronous render + commit in the middle of a handler, blocking the thread. Use once per interaction at most; nested `flushSync` calls are an error.

## How they work under the hood

### `useLayoutEffect` — a flag bit and a synchronous flush

Structurally identical to `useEffect` on the hook list; the difference is a flag on the fiber: `HookLayout` vs `HookPassive`. During commit, React processes `HookLayout` effects **inline**, synchronously, in the same call stack as mutation — before returning from the commit function that the browser's frame callback invoked. `HookPassive` effects get *scheduled* into a separate task (a `MessageChannel` post or `setTimeout`) — the browser paints between them. Two flag bits; one paint boundary.

Cleanup ordering inside a commit: React runs cleanup for every `HookLayout` effect in the subtree **before** running any new setup — child cleanups before parent cleanups, then parent setups before child setups. The same depth-first order as ref attaches; the same reason a layout effect can safely measure a freshly attached ref.

### `useSyncExternalStore` — a `subscribe` hook with a tear-detection read

Internally, `useSyncExternalStore` stores the snapshot in hook state and registers a store listener that calls `scheduleUpdateOnFiber` on notification. On every render, React calls `getSnapshot()` and compares the result with `Object.is` to the stored snapshot. If different and the render is a *concurrent* (transition) render, React abandons it and re-renders synchronously — tearing averted at the cost of one extra render. If different and the render is *sync*, the re-render happens immediately. The `subscribe`→`getSnapshot` pairing is the external analogue of React's own state queue: subscribe for "something changed," read deterministically on demand.

### `flushSync` — raising the work loop's priority ceiling

`flushSync` temporarily sets the scheduler's lane to `SyncLane` and calls `performSyncWorkOnRoot` — the same path that runs sync-priority renders in normal event handling — immediately, inline. The outer scheduler is paused for the duration (no interleaving), the render and commit complete, and control returns. Any state updates that arrive during the `flushSync` callback are folded into one synchronous pass; any updates *outside* the callback continue batching normally. After it returns, `workLoop` resumes wherever it left off.

## Basic usage

Each hook, at its canonical site:

```tsx
// useLayoutEffect: measure and reposition in the same frame
import { useLayoutEffect, useRef, useState } from "react";

export function Tooltip({ label, anchor }: { label: string; anchor: DOMRect }) {
  const [style, setStyle] = useState<React.CSSProperties>({ visibility: "hidden" });
  const tooltipRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    const el = tooltipRef.current;
    if (!el) return;
    const rect = el.getBoundingClientRect();
    const left = anchor.left + anchor.width / 2 - rect.width / 2;
    const fitsBelow = anchor.bottom + rect.height + 8 < window.innerHeight;
    setStyle({
      visibility: "visible",
      top: fitsBelow ? anchor.bottom + 8 : anchor.top - rect.height - 8,
      left: Math.max(8, Math.min(left, window.innerWidth - rect.width - 8)),
    });
  }, [anchor]); // re-measure if anchor moves

  return (
    <div ref={tooltipRef} role="tooltip" className="tooltip" style={{ position: "fixed", ...style }}>
      {label}
    </div>
  );
}
```

The `visibility: hidden` → measure → `visibility: visible` dance guarantees the user never sees the tooltip at the wrong position. With `useEffect`, the tooltip flashes at (0, 0) for one frame before repositioning — the entire class of layout-effect use cases in one example.

```tsx
// useSyncExternalStore: subscribing to a hand-rolled store
import { useSyncExternalStore } from "react";

// Module-level store (hand-rolled for illustration; use Zustand/Redux in practice)
let networkStatus: "online" | "offline" = navigator.onLine ? "online" : "offline";
const listeners = new Set<() => void>();

function subscribeNetwork(listener: () => void) {
  listeners.add(listener);
  return () => listeners.delete(listener);
}

function getNetworkSnapshot() {
  return networkStatus; // string primitive → always stable under Object.is
}

window.addEventListener("online", () => {
  networkStatus = "online";
  listeners.forEach((l) => l());
});
window.addEventListener("offline", () => {
  networkStatus = "offline";
  listeners.forEach((l) => l());
});

export function useNetworkStatus() {
  return useSyncExternalStore(subscribeNetwork, getNetworkSnapshot);
}
```

`subscribeNetwork` is module-level — stable identity across renders, zero churn. `getNetworkSnapshot` returns a primitive — `Object.is`-stable when unchanged. Two invariants to internalize and enforce on every `useSyncExternalStore` at a call site.

```tsx
// flushSync: scroll-to-bottom only after the message is in the DOM
import { flushSync } from "react-dom";
import { useRef, useState } from "react";

export function ChatLog() {
  const [messages, setMessages] = useState<string[]>([]);
  const endRef = useRef<HTMLDivElement>(null);

  function sendMessage(text: string) {
    flushSync(() => {
      setMessages((prev) => [...prev, text]);
    });
    // DOM is committed — scroll-into-view sees the new message node
    endRef.current?.scrollIntoView({ behavior: "smooth" });
  }

  return (
    <div className="log">
      {messages.map((m, i) => <p key={i}>{m}</p>)}
      <div ref={endRef} />
    </div>
  );
}
```

Without `flushSync`, `scrollIntoView` runs before the render commits — `endRef` sits at the old last message, scroll lands one short. With `flushSync`, the DOM reflects the new message, and scroll is precise. Note this is the *only* legitimate reason to reach for `flushSync` in a handler — the scroll needs committed DOM, and there's no other hook to reach for.

## Walkthrough: a virtualised list that needs all three

A virtualised log viewer — 100k entries, only the visible window rendered, with an "auto-scroll" mode that pins to the bottom as new entries stream in. One interaction, three escape hatches, each at its only honest site.

```tsx
// LogViewer.tsx
import { flushSync } from "react-dom";
import { useLayoutEffect, useRef, useState, useSyncExternalStore } from "react";
import type { LogEntry } from "./log-store";
import { logStore } from "./log-store"; // external mutable store

// 1. useSyncExternalStore: log entries from an external streaming source
function useLogEntries(): LogEntry[] {
  return useSyncExternalStore(logStore.subscribe, logStore.getSnapshot);
}

export function LogViewer() {
  const entries = useLogEntries();
  const scrollerRef = useRef<HTMLDivElement>(null);
  const [autoScroll, setAutoScroll] = useState(true);
  const [range, setRange] = useState({ start: 0, end: 50 });

  // 2. useLayoutEffect: update the visible window BEFORE paint so
  //    the user never sees the old slice replaced by the new slice.
  //    Also the auto-scroll pin: measure, decide, act — one frame.
  useLayoutEffect(() => {
    const el = scrollerRef.current;
    if (!el) return;
    const nearBottom = el.scrollHeight - el.scrollTop - el.clientHeight < 80;
    if (autoScroll && nearBottom) {
      el.scrollTop = el.scrollHeight; // pin — before paint means no visible jump
    }
  }, [entries, autoScroll]);

  // 3. flushSync for user-initiated "Jump to bottom" —
  //    must scroll AFTER the full range is committed to DOM.
  function jumpToBottom() {
    flushSync(() => setRange({ start: Math.max(0, entries.length - 50), end: entries.length }));
    const el = scrollerRef.current;
    if (el) el.scrollTop = el.scrollHeight;
  }

  const visible = entries.slice(range.start, range.end);

  return (
    <section>
      <div
        ref={scrollerRef}
        className="log-scroller"
        onScroll={(e) => {
          const el = e.currentTarget;
          const near = el.scrollHeight - el.scrollTop - el.clientHeight < 80;
          setAutoScroll(near);
        }}
      >
        {visible.map((entry) => (
          <LogRow key={entry.id} entry={entry} />
        ))}
      </div>
      {!autoScroll && (
        <button className="jump" onClick={jumpToBottom}>Jump to bottom ↓</button>
      )}
    </section>
  );
}
```

Each hook at its justified site:

- **`useSyncExternalStore`**: the log store is external and mutable; a concurrent render tearing across an entry-count change would commit a partially-updated list. The hook's tear-detection re-render guarantee is load-bearing, not precautionary.
- **`useLayoutEffect`**: auto-scroll needs the DOM's actual `scrollHeight` for *this commit*; `useEffect` would let the browser paint the old position first. The measure-then-pin must complete in the layout phase.
- **`flushSync`**: "Jump to bottom" is user-initiated; after the handler returns, the next line must scroll the *newly committed* rows. Without `flushSync`, `setRange` is batched, scroll runs on the old DOM, and we land at what *was* the bottom a render ago.

The absence of each hook at the other sites is also deliberate: the `onScroll` handler reads DOM and sets state, but the state change can safely batch and paint normally (`setAutoScroll` — no synchrony needed). The `autoScroll`→`scrollTop` path uses `useLayoutEffect` (paint-blocking is correct) rather than `flushSync` (which would be misapplied here — `flushSync` is for handlers that *immediately* need committed DOM, not for post-commit reactions).

## Real-world patterns

**`useLayoutEffect` sites, enumerated.** The canonical list is short; anything not on it probably belongs in `useEffect`:

- Tooltip/popover positioning (measure after mount, reposition if out-of-viewport)
- Scroll restoration (`scrollTop = savedPos` must happen before paint or the user sees the top)
- DOM-synchronized animations (starting an animation from a measured element's current rect)
- Reading layout before a child's paint (a parent that resizes based on child content)
- Third-party library initialization that reads layout synchronously (Popper, floating-ui)

One mechanical note: calling `setState` in a layout effect causes an *additional synchronous render* before paint — two renders per commit, both invisible. This is the design; it's also why layout effects must be fast. A `useLayoutEffect` that does 50ms of work produces a 50ms frame regardless of CPU throttling.

**`useSyncExternalStore` in practice.** Most app code never writes this directly — Zustand, Redux Toolkit, and Jotai all expose properly implemented `useSyncExternalStore` selectors via their hooks. The cases where you write it yourself are: a module-level singleton (network status, a global event bus, the browser geolocation), a browser API that notifies via event (media query matches — the upgrade from the `useMediaQuery` in [custom-hooks](../effects/custom-hooks.md)), or a third-party non-React store you're wrapping. Each time, the two invariants from basic usage are the checklist: module-stable `subscribe`, `Object.is`-stable `getSnapshot`.

**`flushSync` outside React contexts.** Non-React code that calls a React state setter and immediately inspects the DOM is the one case where `flushSync` isn't inside a React event handler at all — a WebSocket message handler that updates state and then needs to scroll, a D3 drag callback that reads a freshly mounted label. Wrap only the setter; the DOM read goes after. One per frame; never nested.

**The performance escape hatch that usually isn't.** `useLayoutEffect` with a guard (`if (condition) return;`) is not `useEffect` — it blocks paint even when it no-ops, because the layout flush itself has overhead. If most renders should not do the work, put the guard at the top so the effect returns before any measurement, and consider whether `useEffect` with a deferred correction (accept the flash, it won't be visible) is actually cheaper.

## Reference

| | `useEffect` | `useLayoutEffect` | `useSyncExternalStore` | `flushSync` |
| --- | --- | --- | --- | --- |
| When it runs | After paint (passive task) | After mutation, before paint | On render + on store notification | Immediately, synchronously |
| Blocks paint | ❌ | ✅ | ❌ | ✅ (for the duration) |
| Primary job | Synchronize with external systems | Measure DOM and react before paint | Subscribe to external mutable store | Force immediate render + commit |
| Wrong use | Anything whose timing doesn't matter | Anything that doesn't need pre-paint measurement | Replacing `useState` for internal state | Replacing `useEffect`/`useState` |
| SSR safe | ✅ (skips; no-op on server) | ⚠️ (warning; skips) | ✅ (with `getServerSnapshot`) | N/A (client only) |
| Cost | None (deferred) | Every ms blocks the frame | One extra render on tear | One synchronous render mid-handler |

**Decision tree: which one do I need?**

```
Does the setup touch or depend on the committed DOM?
  └─ Yes → useLayoutEffect
  └─ No → Does it read an external mutable store that must not tear?
       └─ Yes → useSyncExternalStore
       └─ No → useEffect (almost certainly) or:
            Does a handler need committed DOM RIGHT AFTER a state update?
              └─ Yes → flushSync inside that handler
```

## Common mistakes

**1. `useLayoutEffect` for every effect.** "It's more predictable" — no, it's just blocking. An effect that sets up a WebSocket, logs an event, or syncs with localStorage has no business running before paint. The frame janks for zero benefit. `useEffect` is the default; `useLayoutEffect` earns its site.

**2. Slow work in layout effects.** Long computations, network calls, heavy DOM traversals — all of these before paint are all of these before paint. The commit phase doesn't yield; 80ms of layout-effect work is an 80ms frozen frame. Measure fast; compute slowly in passive effects or during render.

**3. `useState` for external mutable stores.** The effect-subscribe pattern:

```tsx
const [price, setPrice] = useState(() => priceStore.get(ticker));
useEffect(() => priceStore.subscribe(ticker, setPrice), [ticker]);
```

Has a tear window between the initial read and the subscription. Under concurrent rendering, the component can render with the stale initial value, yield, the store updates, the effect hasn't subscribed yet. `useSyncExternalStore` is the correct primitive; the effect version is the pattern it replaced.

**4. Unstable `getSnapshot`.**

```tsx
// 🔴 new object every call — infinite re-renders
useSyncExternalStore(subscribe, () => ({ items: store.items, total: store.total }));

// ✅ memoized at store level or: return a primitive / stable reference
useSyncExternalStore(subscribe, () => store.getStableSnapshot()); // store caches the object
```

React calls `getSnapshot()` on every render and after every notification, comparing with `Object.is`. An inline object literal fails that check always — React keeps seeing "changed," keeps re-rendering. The snapshot must be a primitive, or a reference the store holds stable until the data actually changes.

**5. `flushSync` inside a React event handler that already batches.** React event handlers batch automatically since 18; `flushSync` inside one forces a mid-handler render that you almost never want — it breaks the handler's own batching for subsequent `setState` calls. The site is handlers that run *outside* React's event system: `setTimeout`, `Promise.then`, native `addEventListener` — places where you'd historically have needed `ReactDOM.unstable_batchedUpdates` for the opposite reason.

**6. Nested `flushSync`.** It's an error React throws on. If you feel the need to nest, the design is wrong — there's one too many layers of imperative DOM dependency, and the right fix is restructuring so the sequence doesn't need two synchronous rendering passes mid-handler.

**7. `useSyncExternalStore` for internal state.** If the store lives inside the component (or is React state), `useSyncExternalStore` adds tear-detection complexity for a case that can't tear — React already owns the update. `useState`/`useReducer` are cheaper and semantically correct.

**8. No cleanup in `useLayoutEffect`.** Same contract as `useEffect` — every setup with observable side effects returns a cleanup. A layout effect that attaches a DOM event listener, sets a class, or modifies a ref without cleaning up will leak on every dependency change and on unmount, but faster, because it runs synchronously.

## How this evolved

- **Pre-hooks class era:** `componentDidMount`/`componentDidUpdate` ran synchronously after commit — effectively layout-phase behavior, but with no passive alternative. Everything was "layout," and expensive effects just blocked paint as a matter of course.
- **Hooks (16.8):** the split landed — `useEffect` deferred to after paint, `useLayoutEffect` preserved the synchronous slot, and the *vast majority* of effects correctly moved to passive. The distinction made the performance contract explicit for the first time.
- **Concurrent rendering (18):** tearing became a real runtime concern, not a theoretical one; `useSyncExternalStore` was extracted from its internal implementation and published as the stable subscription primitive. Zustand, Redux, and Jotai rewrote their hooks around it immediately.
- **Automatic batching (18):** `flushSync` became the necessary escape hatch — before 18, most non-React-event-handler contexts didn't batch, so the problem it solves existed in fewer places.
- **React 19.2:** the three escape hatches are stable and unchanged; the ecosystem of stores that implement `useSyncExternalStore` correctly is now mature, so the cases where you write the hook directly are genuinely narrow.

## Exercises

**1. Prove the flash.** Build a `Tooltip` component twice — once with `useEffect`, once with `useLayoutEffect` — both measuring and repositioning from the left edge to centered. Open DevTools Performance panel, record a slow-CPU (6× throttle) hover over the trigger. Find the frame boundary between the two renders in the waterfall. The `useEffect` version has two painted frames in the profile; `useLayoutEffect` has one.
*Hint: the tell is a layout-paint event followed by a second layout-paint event in the passive case; in the layout case, two renders appear in the scripting block before a single paint.*

**2. Break and fix the tear.** Build a store with two components that read it: a header reading the count, and a body reading whether count is even. Update the store directly (outside React) every 16ms. Under concurrent rendering with a 2-second artificial delay in the body component's render, both using `useEffect`-based subscription: observe a frame where the header reads odd but the body reads "even." Then migrate both to `useSyncExternalStore` and observe the tearing disappear.
*Hint: `useDeferredValue` wrapped around the body's read can artificially surface the tear window in development; the store must have two readers whose snapshots can differ.*

**3. The scroll precision audit.** The chat log from basic usage: comment out `flushSync`, send a message, and record in the Performance tab where the scroll lands relative to where the new message is. Restore `flushSync` and repeat. Measure the pixel delta between "scrolled to" and "new message top" in both cases — the delta shrinks to ≤ 1px only with `flushSync`.
*Hint: `el.scrollTop` after a batched update reads the pre-render scroll; after `flushSync` it reads the post-render one. The exact delta depends on message height; the direction of the error is always "one message short."*

## Summary

- `useLayoutEffect` runs synchronously after mutation, before paint — giving you committed DOM measurements in the same frame. The cost is blocking paint; the use cases fit on a short list: tooltips, scroll restoration, DOM-synchronized animations, layout-reading libraries. Everything else stays in `useEffect`.
- `useSyncExternalStore` closes the subscribe-then-read tear window for external mutable stores. Its two invariants — module-stable `subscribe`, `Object.is`-stable `getSnapshot` — are the whole correctness model. Most app code uses it via the library hooks that wrap it; you write it directly only for browser singletons and store integrations.
- `flushSync` collapses batching for one synchronous render-and-commit, for handlers that must immediately read the committed DOM. One per interaction; never nested; almost never inside React event handlers (which batch already).
- The decision flows cleanly: DOM dependency before paint → layout effect; external mutable store → `useSyncExternalStore`; handler needs committed DOM now → `flushSync`; everything else → `useEffect` / `useState` / normal rendering.
- Each carries a real cost that makes it wrong outside its site. The audit closes Wave 2.

## See also

- [effects-and-synchronization](./effects-and-synchronization.md) — the decision tree that routes here; the symmetry contract layout effects share
- [how-react-renders](../rendering/how-react-renders.md) — the commit sub-phases `useLayoutEffect` and passive effects occupy
- [useref-and-the-dom](./useref-and-the-dom.md) — the attach timing `useLayoutEffect` reads and the measurement patterns it executes
- [custom-hooks](./custom-hooks.md) — packaging `useMediaQuery` via `useSyncExternalStore`; the `useSyncExternalStore` upgrade
- [state-and-usestate](../state/state-and-usestate.md) — batching and the `flushSync` escape context
- [portals-and-the-event-system](../rendering/portals-and-the-event-system.md) — `useLayoutEffect` for anchored portal positioning
- [ssr-and-hydration](../server/ssr-and-hydration.md) *(Wave 3, planned)* — the `useLayoutEffect` SSR warning and the `getServerSnapshot` pattern

## References

- [useLayoutEffect — react.dev](https://react.dev/reference/react/useLayoutEffect)
- [useSyncExternalStore — react.dev](https://react.dev/reference/react/useSyncExternalStore)
- [flushSync — react.dev](https://react.dev/reference/react-dom/flushSync)
- [Subscribing to an external store — react.dev](https://react.dev/learn/you-might-not-need-an-effect#subscribing-to-an-external-store)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.