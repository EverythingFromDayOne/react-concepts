---
article_id: concurrent-rendering
concept_folder: concurrent
wave: 3
related:
  - rendering/how-react-renders
  - effects/escape-hatches-audit
  - state/usereducer-and-state-structure
  - effects/custom-hooks
  - recipes/performance/typing-lag-rerender-storm
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this.** You have a search box over 8,000 rows. Each keystroke re-renders the list, and the render costs ~120ms. You already memoized the rows and let the Compiler do its thing — the render is genuinely that expensive, not accidentally wasteful. So the input freezes: you type "reconcil" and the letters land in three bursts, the caret stutters, and the browser reports INP spikes of 240ms. There is no memoization move left. The problem isn't *how much* work the render does — it's that the work is **blocking the one interaction the user cares about**. Concurrent rendering is the tool for exactly this: it lets React keep the input at 60fps while the expensive list render happens at a lower priority, interruptibly, without you touching the cost of the render itself.

## What concurrent rendering actually is

Concurrent rendering is not a feature you switch on. Since React 18 (and unconditionally in 19.2 with `createRoot`), it is simply *how React renders*: render work is broken into interruptible units, updates carry a priority, and React can pause a render, hand control back to the browser, and either resume or throw the work away.

What you *do* opt into is the ability to **label some updates as lower priority** so React knows which ones it's allowed to interrupt. That's the entire public surface of this article: two hooks (`useTransition`, `useDeferredValue`), one standalone function (`startTransition`), and the mental model that makes them make sense.

The mental-model shift is this. Before concurrent rendering, a render was an atomic, blocking function call: you called `setState`, React rendered the whole tree to completion, committed, and only *then* did the browser get a chance to paint or process input. A 120ms render meant 120ms of frozen page. After concurrent rendering, a render is a **schedulable, pausable, abandonable unit of work**. The 120ms of render can be sliced across several browser tasks, yielding between slices so the input keeps responding, and if you type another key mid-render the half-finished render is discarded and restarted with the new value — nothing torn, nothing committed, no wasted paint.

This builds directly on machinery you already met. [`how-react-renders`](../rendering/how-react-renders.md) owns the fiber tree, double buffering, and **the lane model** — the fact that every update is assigned a priority lane and that React works the highest-priority lane first. This article is where the lane model earns its keep: transitions are just a *lower* lane, and interruption is just React noticing a higher lane appeared and restarting. If the phrase "work-in-progress tree" or "the three bailout checks" isn't fresh, skim that article's under-the-hood section first; everything here assumes it.

One sentence to carry: **concurrent features don't make a slow render fast — they keep the app responsive *during* a slow render.** That distinction is the source of half the misuses in the wild, and we'll return to it in the mistakes section.

## How it works under the hood

### The work loop yields

The reason a render can be interrupted at all is that React's concurrent work loop checks, between units of work, whether it should stop:

```js
// Simplified from react-reconciler's work loop.
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress); // one fiber's beginWork/completeWork
  }
}
```

Contrast the synchronous loop, which has no such check — `while (workInProgress !== null)` — and runs to completion no matter what. `shouldYield()` comes from React's Scheduler package. It returns `true` when the current time slice is spent. The slice is small — the frame budget is ~5ms — and on browsers that expose it, the Scheduler also consults `navigator.scheduling.isInputPending()` so it can bail *early* if a real input event is waiting in the queue. When `shouldYield()` returns true, React stops mid-tree, posts a macrotask (via a `MessageChannel` port — `setTimeout` has a 4ms clamp that would waste budget), lets the browser process events and paint, and picks up where it left off on the next task.

So a "slow" concurrent render doesn't monopolize the main thread. A 120ms render becomes roughly twenty-four 5ms slices with the browser breathing between each. Urgent events queued during those gaps get handled promptly.

### Urgent vs transition lanes

Every update is scheduled into a lane (see [`how-react-renders`](../rendering/how-react-renders.md#lanes-how-react-decides-whats-urgent)). What Wave 2 didn't spell out is *which* lane, and that's the whole game here:

- A default `setState` from an event handler (typing, clicking) lands in a **blocking** lane — high priority. React renders it as close to synchronously as it can; you want the character to appear the instant it's typed.
- A `setState` wrapped in `startTransition` lands in a **transition** lane — low priority, drawn from a pool of ~16 transition lanes that React cycles through so distinct transitions don't clobber each other's identity.

React always works the highest-priority pending lane first. When both a blocking update and a transition are pending, the blocking one renders and commits; the transition waits. This is why the input in the lead example stays live: the character update is blocking, the list update is a transition, and blocking always wins the race.

Batching, which [`how-react-renders`](../rendering/how-react-renders.md) framed as **lane collapse**, extends cleanly: updates in the same lane batch together into one render, but a blocking update and a transition update are *different* lanes and do **not** collapse into each other. They render in separate passes, blocking first.

### Interruption, traced

Here's the sequence that makes concurrent rendering worth its complexity. You're typing fast:

1. Keystroke "a" fires. `setQuery("...a")` schedules a blocking update; the list update it triggers (through `useDeferredValue` or a wrapped `setState`) schedules a transition. React renders the input at blocking priority, commits — caret moves — and begins the transition render of the 8,000-row list, slicing it across tasks.
2. Two slices in, keystroke "b" fires. `setQuery("...ab")` schedules a *new* blocking update. React marks the root as having work in a higher-priority lane than the one it's currently rendering.
3. On the next `shouldYield()` boundary, React sees the higher lane. It calls `prepareFreshStack` — **throws away the half-built work-in-progress list tree** — commits the blocking input update ("b" appears), and restarts the transition render from scratch with the new query.

Step 3 is safe for exactly one reason, and it's the reason double buffering exists ([`how-react-renders`](../rendering/how-react-renders.md#two-trees-double-buffering)): the interrupted render was happening on the **work-in-progress tree**, a separate fiber tree from the **current** (committed, on-screen) tree. Abandoning it discards a scratch buffer. The screen was never showing a half-filtered list, because nothing committed. This is the deep payoff of the pointer-swap design: renders are provisional until commit, so throwing one away costs only the CPU already spent, never visual corruption.

The intermediate renders that got abandoned never painted. If you type ten characters quickly, React may start ten list renders and commit only the last one. That's the mechanism behind "it feels instant even though the render is slow": the slow renders that would have janked you got cancelled before they reached the screen.

And a worry this immediately raises, answered: abandoned renders never run your effects either. Effects fire in the commit phase ([`effects-and-synchronization`](../effects/effects-and-synchronization.md)), and an abandoned render never commits — so a `useEffect` keyed on the deferred value runs only for the values that actually made it to the screen, not once per keystroke. Interruption discards render work, never committed work. This is why concurrent rendering is safe to build on: the observable boundary is commit, and commit is still atomic.

### Tearing — the hazard interruptibility introduces

Interruptibility buys responsiveness and costs a new failure mode. Because a single render can now span multiple browser tasks, **an external mutable value read during render can change partway through**, and two components reading the same source can render with different values of it. That's tearing: the UI is internally inconsistent — literally torn between two states of the world.

Trace it. Suppose a module-level `let livePrice = 100` is mutated by a WebSocket, and two components read it directly during render:

```tsx
// ⚠️ Both read a mutable module value directly during render.
let livePrice = 100;
socket.on("tick", (p) => { livePrice = p; });

function Header() { return <span>{livePrice}</span>; }   // reads livePrice
function Footer() { return <span>{livePrice}</span>; }   // reads livePrice
```

In a *synchronous* render these always agree — the whole tree renders in one uninterrupted shot, so `livePrice` can't change mid-render. Under concurrent rendering, React can render `Header` (reads `100`), yield, let a socket tick fire the mutation (`livePrice = 105`), then resume and render `Footer` (reads `105`). Now the header says 100 and the footer says 105, committed together. They're torn.

React's *own* state cannot tear. `useState`/`useReducer` values are read from the fiber, which is snapshotted for the duration of a render — every read within one render pass sees the same value, no matter how many tasks it spans (this is the snapshot discipline from [`state-and-usestate`](../state/state-and-usestate.md)). Tearing is strictly a hazard of reading a *mutable external source* directly in render.

The fix is not this article's to own — [`escape-hatches-audit`](../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely) owns `useSyncExternalStore`, which subscribes to an external store safely: React compares snapshots and, if it detects the store changed during a concurrent render, synchronously re-renders so it never *commits* a torn tree. What this article owns is *why the hazard exists at all* — interruptible rendering spanning tasks — so that when you reach for a store, you understand what `useSyncExternalStore` is defending against rather than treating it as a magic incantation.

### You can watch all of this now

React 19.2 added **Performance Tracks** to the Chrome DevTools performance profiler — a Scheduler track and a Components track. The Scheduler track labels work by priority: you can literally see "blocking" spans for input and "transition" spans for `startTransition` work, watch React yield to paint, and catch the classic bug where an update you *thought* was a transition is being processed as blocking (a stray synchronous `setState` outside the transition, or a `flushSync` you forgot). This is the concurrent-rendering analog of the **✨ badge** you used to verify Compiler memoization ([`memoization-and-the-compiler`](../rendering/memoization-and-the-compiler.md)): don't assume your transition is a transition — record a profile and confirm the work landed in the transition track.

## Basic usage

Three entry points. All stable, all baseline in 19.2.

### `useTransition` — when you own the setState

```tsx
import { useTransition, useState } from "react";

function TabPanel() {
  const [tab, setTab] = useState<"overview" | "reports">("overview");
  const [isPending, startTransition] = useTransition();

  function selectReports() {
    // The click feedback is urgent; the heavy panel render is not.
    startTransition(() => {
      setTab("reports");
    });
  }

  return (
    <div>
      <button onClick={selectReports} disabled={tab === "reports"}>
        Reports
      </button>
      {isPending && <Spinner />}
      <Panel tab={tab} />
    </div>
  );
}
```

`isPending` is `true` from the moment the transition is scheduled until its render commits — your handle for dimming, spinners, or disabling controls. `startTransition(fn)` marks every `setState` called synchronously inside `fn` as a transition.

### `useDeferredValue` — when you receive a value you don't set

```tsx
import { useDeferredValue, useState } from "react";

function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {/* Renders with the lagging value at transition priority. */}
      <Results query={deferredQuery} />
    </>
  );
}
```

`useDeferredValue(query)` returns the *previous* `query` on the render where `query` changes, then schedules a transition-priority re-render with the new value. The input reads the live `query` (urgent, snappy); `Results` reads `deferredQuery` (lagging, interruptible). React 19 added a second argument, `useDeferredValue(query, initialValue)`, which supplies the value for the *first* render before deferring to the real one — useful when the initial computation is itself expensive.

### `startTransition` — the standalone function

```tsx
import { startTransition } from "react";

// Usable outside components — router code, event utilities, store actions.
function navigate(url: string) {
  startTransition(() => {
    history.pushState(null, "", url);
    setRoute(url);
  });
}
```

Same semantics as the hook's second element, minus `isPending` (there's no component to hang the flag on). Reach for it when the update originates outside a component or you don't need the pending flag.

## Walkthrough: the split-lane search

This is the canonical concurrent-rendering scenario, and it pays off the "search/table split-lane" walkthrough that [`how-react-renders`](../rendering/how-react-renders.md) forward-referenced. We'll build a filterable list of 8,000 items, watch it jank, and fix it *without* reducing the render cost — because the point is that concurrent rendering is orthogonal to how expensive the render is.

### Stage 0 — the naive version janks

```tsx
// SearchableList.tsx
import { useState } from "react";
import { rows } from "./data"; // 8,000 rows

function filterRows(query: string) {
  const q = query.toLowerCase();
  // Deliberately not cheap: substring scan over every row, every keystroke.
  return rows.filter((r) => r.label.toLowerCase().includes(q));
}

export function SearchableList() {
  const [query, setQuery] = useState("");
  const results = filterRows(query);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Filter…"
      />
      <p>{results.length} matches</p>
      <ul>
        {results.map((r) => (
          <li key={r.id}>{r.label}</li>
        ))}
      </ul>
    </div>
  );
}
```

Type "reconciliation" and the caret stutters. Every keystroke runs `filterRows` and re-renders up to 8,000 `<li>`s synchronously — measured at ~120ms per keystroke on a mid-tier laptop, worse on a phone. The input and the list are in the **same blocking lane**, so the input can't update until the list is done. INP goes to 240ms+. Memoizing the rows doesn't help: the list content genuinely changes on every keystroke, so there's nothing to skip. The render *is* the cost.

### Stage 1 — defer the consumer with `useDeferredValue`

The list is what's expensive, and we don't own its setter path directly — we own `query`, and the list merely *reads* it. That's the `useDeferredValue` shape: defer the value the expensive consumer reads.

```tsx
// SearchableList.tsx
import { useDeferredValue, useState } from "react";
import { rows } from "./data";

function filterRows(query: string) {
  const q = query.toLowerCase();
  return rows.filter((r) => r.label.toLowerCase().includes(q));
}

// Extracted so it can be an independent memoization boundary.
function ResultList({ query }: { query: string }) {
  const results = filterRows(query);
  return (
    <>
      <p>{results.length} matches</p>
      <ul>
        {results.map((r) => (
          <li key={r.id}>{r.label}</li>
        ))}
      </ul>
    </>
  );
}

export function SearchableList() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Filter…"
      />
      {/* Dim while the visible list lags a keystroke or two behind. */}
      <div style={{ opacity: isStale ? 0.5 : 1, transition: "opacity 120ms" }}>
        <ResultList query={deferredQuery} />
      </div>
    </div>
  );
}
```

Two details carry the fix, and both depend on the Compiler:

- **The input reads `query` (live), `ResultList` reads `deferredQuery` (lagging).** On each keystroke, React commits the input update at blocking priority immediately — caret stays at 60fps — then renders `ResultList` at transition priority. Type fast and the intermediate list renders get abandoned per the interruption trace above; only the last one paints.
- **`ResultList` must actually skip re-rendering when `deferredQuery` is unchanged**, or the deferral is pointless. With React Compiler on (assumed for all app code), `ResultList` is auto-memoized, so a render where `deferredQuery` didn't change hits **bailout check 3** ([`how-react-renders`](../rendering/how-react-renders.md#bailouts-the-fast-paths)) and does no work. Confirm the ✨ badge is present on `ResultList` in DevTools — don't assume it; a purity violation can silently opt a component out of memoization ([`rules-of-react`](../foundations/rules-of-react.md)). This is the moment the Compiler-first stance and concurrent rendering interlock: the deferred boundary only saves work because the boundary is memoized.

`isStale` is derived, not stored — `query !== deferredQuery` computed during render, per derive-don't-mirror ([`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md)). It's `true` for the brief window where the visible list is a keystroke behind the input, which is exactly when you want the dim.

Measured result: input latency back to sub-16ms, INP under 100ms, and the list catches up within a frame or two on a fast machine — the render still costs 120ms, but it no longer blocks anything the user is touching.

### Stage 2 — the `useTransition` variant, and when to pick which

If the trigger were a *setState you own* rather than a value you receive — say a category toggle that swaps the whole dataset — you'd wrap the setter instead:

```tsx
export function CategoryList() {
  const [category, setCategory] = useState("all");
  const [isPending, startTransition] = useTransition();

  function pick(next: string) {
    startTransition(() => {
      setCategory(next); // heavy list rebuild happens at transition priority
    });
  }

  return (
    <div>
      <CategoryTabs active={category} onPick={pick} isPending={isPending} />
      <ResultList category={category} />
    </div>
  );
}
```

The decision rule, stated once so you never guess again:

| Situation | Use |
| --- | --- |
| You call the `setState` that triggers the expensive render | `useTransition` (wrap the setter; get `isPending` for free) |
| You receive a value (prop, controlled input) whose setter you don't control at the boundary | `useDeferredValue` (defer the value the expensive consumer reads) |
| The update originates outside a component (router, store action) | standalone `startTransition` |

They're the same lane mechanism from two ends: `useTransition` marks the *cause* (the setState) as low priority; `useDeferredValue` marks the *effect* (a value's propagation) as low priority. In the search case you don't own the list's setter path cleanly, so you defer the value; in the category case you own `setCategory`, so you wrap it.

## Real-world patterns

**Tab and route switches.** The most common production use. Wrapping `setTab`/`setRoute` in `startTransition` keeps the click's own feedback (the button's active state, a spinner) instant while the heavy incoming panel renders at low priority — no dead 200ms between click and any response. React 19.2's [`<Activity>`](#how-this-evolved) composes with this: an `<Activity mode="hidden">` subtree pre-renders the *next* tab at transition priority in the background, so the switch is already warm when the user gets there.

**Deferring an expensive subtree by prop.** Pass a deferred value down as a prop to a subtree that's memoized (Compiler-on, or a manual `memo` boundary if it's a library export — commented per convention). The subtree re-renders at transition priority and, because it's memoized, skips entirely on renders where the deferred prop is unchanged. This is the generalized form of the search walkthrough.

**Pairing with the performance ladder.** Concurrent features are the *responsiveness* layer, not the *cost* layer. The [`typing-lag-rerender-storm`](../recipes/performance/typing-lag-rerender-storm.md) recipe's ladder — measure → structure → Compiler-verify → defer → do less → manual memo — puts `useDeferredValue` at the "defer" rung, *after* you've colocated state and let the Compiler memoize, and *before* the heavier structural fixes. If the list is 8,000 rows, the real answer is often virtualization ("do less" — render 30 rows, not 8,000) *and* a transition on top. They're complementary: virtualization cuts the render cost; the transition keeps input responsive during whatever cost remains.

### Async transitions

In React 19, the `startTransition` callback may be `async` — the transition stays pending across `await` boundaries until the awaited work and its resulting state updates settle:

```tsx
startTransition(async () => {
  const data = await save(form);   // transition stays pending across this await
  setResult(data);
});
```

This is the mechanism the entire Actions story is built on. We introduce it here as *transition semantics*; [`actions`](./actions.md) productizes it into `useActionState`/`useFormStatus`/`useOptimistic`, and the [`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) recipe already uses those APIs at the recipe level. When you get there, remember: an Action is an async transition with a form-shaped reducer wrapped around it.

## API and type reference

| API | Signature | Returns | Notes |
| --- | --- | --- | --- |
| `useTransition` | `useTransition(): [boolean, (fn: () => void \| Promise<void>) => void]` | `[isPending, startTransition]` | `isPending` true while the transition renders. Callback may be async (19+). |
| `startTransition` | `startTransition(fn: () => void \| Promise<void>): void` | `void` | Standalone; no `isPending`. Use outside components or when the flag isn't needed. |
| `useDeferredValue` | `useDeferredValue<T>(value: T, initialValue?: T): T` | deferred `T` | Returns previous value on change, schedules transition-priority re-render with new. `initialValue` (19+) seeds the first render. |

Type notes: `useDeferredValue` is generic and preserves `T` exactly — no widening, no `undefined` unless `T` includes it. The transition callback's return type being `void | Promise<void>` is what admits async Actions; a callback that returns a non-promise value is fine, its return is ignored.

## Common mistakes

**1. Wrapping every `setState` in `startTransition`.** Transitions carry scheduling overhead and introduce pending states you now have to design around. Reserve them for updates that trigger a *genuinely expensive* render. A transition around a cheap update is pure cost with a confusing `isPending` flicker on top.

```tsx
// ❌ Cheap update, no expensive render downstream — nothing to defer.
startTransition(() => setIsOpen(true));
// ✅ Just set it.
setIsOpen(true);
```

**2. Using `useDeferredValue` to "debounce" network fetches.** This is the big one, and it's the scheduling-lag-vs-wall-clock-lag distinction promised back in [`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md) and [`custom-hooks`](../effects/custom-hooks.md#real-world-patterns). A `useDebouncedValue(query, 300)` hook adds a **fixed wall-clock delay** and *skips intermediate values* — you fire one fetch after the user stops typing for 300ms, and the delay is 300ms whether the machine is a workstation or a phone. `useDeferredValue` adds **scheduling lag, not wall-clock lag**: it doesn't delay by a fixed amount and it doesn't skip values — every value eventually renders, just at low priority, interruptibly. On a fast machine the deferred render lands next frame; on a slow one it yields as needed. So:

- For an **expensive render** over a value: `useDeferredValue`. It keeps input responsive and abandons intermediate *renders* (which never paint), but it fires the same number of *computations* — it lowers priority, not count.
- For an **expensive side effect** you want to fire *fewer* of (a network request, an analytics event): **debounce or throttle the effect itself**. `useDeferredValue` gives you no *controllable* reduction. A fetch effect keyed on `deferredQuery` fires once per *committed* deferred value — and how many values commit depends on the machine: on a fast machine nearly every keystroke's deferred render completes and commits (≈ one fetch per keystroke, same as `query`), while on a slow machine more deferred renders get abandoned before commit (fewer fetches). You've made your request count a nondeterministic function of CPU speed, which is precisely the wrong property for controlling network traffic. Debounce fires exactly one request per quiet period, deterministically, on every machine.

The one-line test: deferring changes *when and whether React re-renders*; debouncing changes *how often your code deliberately runs*. Network requests are code you want to run a controlled number of times — so control it explicitly, don't leave it to the scheduler.

**3. Expecting concurrent features to make a slow render fast.** They don't reduce work; they reschedule it. A 120ms render is still 120ms of CPU — you've just stopped it from blocking input. If the render itself is the problem (dropped frames even without concurrent input contention, slow initial paint), fix the render: virtualize, split, do less. Concurrent features are a complement to the performance ladder, never a substitute for its lower rungs.

**4. Reading mutable external state directly in render.** Covered under tearing above. A module-level mutable, a ref you mutate and read in render, a store read without a subscription — any of these can tear under interruption. Route external stores through `useSyncExternalStore` ([`escape-hatches-audit`](../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely)).

**5. Deferring the wrong side.** Deferring the *input's own value* makes the input lag — the exact opposite of the goal. The input must always read the **live** value; defer the value the **expensive consumer** reads.

```tsx
// ❌ Input now lags behind the user's typing.
const deferred = useDeferredValue(query);
<input value={deferred} onChange={(e) => setQuery(e.target.value)} />
// ✅ Input live, consumer deferred.
<input value={query} onChange={(e) => setQuery(e.target.value)} />
<Results query={useDeferredValue(query)} />
```

**6. Forgetting to memoize the deferred consumer.** If the expensive child isn't memoized (Compiler badge absent, or a manual `memo` missing at a non-compiled boundary), it re-renders on *every* parent render regardless of whether the deferred value changed — so the deferral saves nothing. The child must be able to hit a bailout check when its deferred prop is stable. Verify the ✨ badge.

**7. Gating the input on `isPending`.** `isPending` describes the *transition*, not the input. Feeding it into the input's `disabled` or `value` couples the responsive thing to the slow thing and reintroduces lag. Use `isPending` for the *deferred region's* affordances (dim, spinner), never the urgent control.

**8. Relying on async-transition pending semantics on the wrong version.** Async `startTransition` callbacks staying pending across `await` is a React 19 behavior. On <19 the transition ended at the first `await` and the pending flag dropped early. Since this project baselines 19.2, use it freely — but flag it if any example claims to run on 18.

**9. Side effects inside a transition render.** A transition render can run multiple times and be abandoned before commit. Any impurity in render — mutation, logging a "this rendered once" side effect, a counter bump — is wrong in general ([`rules-of-react`](../foundations/rules-of-react.md)) and *doubly* visible here, because interruption makes the extra/abandoned renders observable. Renders describe; handlers cause. Keep effects in effects.

**10. Using a transition for something that must be synchronous.** A controlled input's own value, a value the user must see change *this frame*, a measurement that must reflect the committed DOM — none of these tolerate being deprioritized. If the update must be visible immediately, it's blocking by definition; don't wrap it.

## How this evolved

The through-line from [`how-react-renders`](../rendering/how-react-renders.md) — class → hooks → concurrent → 19 Actions → Compiler — runs straight through this article, because concurrent rendering is the hinge the later era swings on.

- **Pre-18: synchronous, blocking renders.** The only tools for "keep the input responsive while something expensive happens" were wall-clock hacks: `debounce`/`throttle` to fire less often, manual `setTimeout` chunking to yield the main thread by hand, `requestIdleCallback` for the brave. None of them made *rendering* interruptible — they made your *code* run less. A slow render was a frozen page, full stop.
- **18: concurrent rendering shipped**, opt-in through `createRoot` plus the concurrent features — `startTransition`, `useTransition`, `useDeferredValue` — and automatic batching across the board. This is where interruptible rendering, the lane model, and `useSyncExternalStore` (as the tearing defense) all arrived together. They were "new features" you adopted deliberately.
- **19: the features became baseline**, and async transitions turned the transition into the substrate for **Actions** — `useActionState`, `useFormStatus`, `useOptimistic` are all async transitions with structure ([`actions`](./actions.md)). `useDeferredValue` gained its `initialValue` argument. `use()` ([`use-and-promises`](./use-and-promises.md)) made promises first-class render inputs, leaning on the same scheduler.
- **19.2 (this baseline):** the tooling and ergonomics matured. **Performance Tracks** in Chrome DevTools finally let you *see* the Scheduler's blocking-vs-transition prioritization directly (use them to verify your transitions land where you think). The **`<Activity>`** component landed: `mode="hidden"` keeps a subtree mounted with its state and DOM preserved while unmounting its effects and deferring its updates until React has nothing else to do — pre-render the tab the user is likely to visit next, at transition priority, without touching foreground performance. It's concurrent rendering applied to *whole subtrees* rather than single values. (A 19.2 bug fix worth knowing if you hit it: an infinite `useDeferredValue` loop on `popstate` — browser back/forward — was patched; if you're on an early 19.2 patch and see it, upgrade.)

The shape of the arc: React spent five years turning rendering from an atomic blocking call into a scheduler, then built its entire modern mutation and data story (Actions, `use`, Suspense) on top of that scheduler. Everything in the rest of Wave 3 assumes the machinery in this article.

## Exercises

**1. Convert and measure.** Take the Stage 0 naive `SearchableList` and apply `useDeferredValue`. Record a DevTools performance profile *before* and *after*, and read the Scheduler track. Confirm the list render moves from the blocking track to the transition track, and that INP drops.
*Hint:* Defer on the consumer (`Results`), not the input. Dim while `query !== deferredQuery`. Verify the ✨ badge on the results component — if the deferral doesn't help, the consumer probably isn't memoizing.

**2. Never-block a tab switch.** Build a three-tab interface where one tab renders an expensive chart. Wrap the tab-setter so clicking a tab never blocks the click's own visual feedback; show a pending affordance on the incoming tab.
*Hint:* `useTransition`, wrap `setActiveTab`. The button's active state is urgent (outside the transition); the panel render is the transition. Use `isPending` for the panel region only.

**3. (Stretch) Reproduce tearing, then fix it.** Create a module-level mutable counter mutated on a fast interval, read it directly in render inside a deferred subtree, and force enough concurrent work to observe two components disagreeing. Then fix it and confirm the disagreement disappears.
*Hint:* The fix is `useSyncExternalStore` — see [`escape-hatches-audit`](../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely). Watch the Scheduler track to confirm the render is actually being sliced (tearing only shows when a render spans tasks).

## Summary

- Concurrent rendering is *how React renders*: interruptible, prioritized units of work. You opt into **labeling** updates as low-priority (transitions), not into the mechanism itself.
- The work loop yields between fibers (`shouldYield()`, ~5ms slices, `MessageChannel` macrotask). Blocking lanes beat transition lanes; a higher-priority update mid-render triggers `prepareFreshStack` — the work-in-progress tree is discarded and restarted, safe because of double buffering.
- **`useTransition`** marks a setState you own as low priority (gives `isPending`); **`useDeferredValue`** marks a value's propagation as low priority (defer the consumer, not the input); **`startTransition`** is the standalone form.
- **Tearing** is the hazard interruptibility introduces — a mutable external value read in render can change mid-render. React state can't tear; external stores need `useSyncExternalStore`.
- Concurrent features keep you **responsive during** a slow render; they don't make it fast. `useDeferredValue` is scheduling lag (lowers priority, not count) — for fewer *side effects*, debounce the effect, don't defer the value.
- Verify with React 19.2's Scheduler track, and let the Compiler memoize the deferred consumer — the deferral only pays off if the boundary can bail out.

## See also

- [`how-react-renders`](../rendering/how-react-renders.md) — the lane model, double buffering, and the three bailout checks this article extends.
- [`escape-hatches-audit`](../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely) — `useSyncExternalStore`, the fix for the tearing hazard traced here.
- [`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md) — transitions-in-one-place and the deliberate-lag exception the debounce contrast pays off.
- [`custom-hooks`](../effects/custom-hooks.md) — the explicit-deferral discussion and why a `useDebouncedValue` hook is a different tool.
- [`memoization-and-the-compiler`](../rendering/memoization-and-the-compiler.md) — the ✨ badge and why the deferred consumer must be memoized.
- [`typing-lag-rerender-storm`](../recipes/performance/typing-lag-rerender-storm.md) — the performance ladder that places `useDeferredValue` at the "defer" rung.
- [`suspense`](./suspense.md) *(planned, next)* — the other half of concurrent rendering: waiting for data, not just deprioritizing renders.
- [`actions`](./actions.md) *(planned)* — async transitions productized into the 19 mutation story.

## References

- React docs — [`useTransition`](https://react.dev/reference/react/useTransition)
- React docs — [`useDeferredValue`](https://react.dev/reference/react/useDeferredValue)
- React docs — [`startTransition`](https://react.dev/reference/react/startTransition)
- React docs — [`<Activity>`](https://react.dev/reference/react/Activity)
- React docs — [React Performance Tracks](https://react.dev/reference/dev-tools/react-performance-tracks)
- React blog — [React 19.2](https://react.dev/blog/2025/10/01/react-19-2)

## Demo source

_Demo source pending — StackBlitz link to be added in the demo pass._