---
article_id: how-react-renders
concept_folder: rendering
wave: 2
related:
  - foundations/thinking-in-react
  - state/state-and-usestate
  - rendering/rendering-lists-and-keys
  - foundations/rules-of-react
  - rendering/memoization-and-the-compiler
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# How React Renders

> **Lead with this:** a re-render is a function call, not a DOM rebuild — [thinking-in-react](../foundations/thinking-in-react.md) established that. This article opens the hood on *how* that function call becomes pixels: the fiber data structure that makes rendering interruptible and skippable, the lane system that decides what's urgent, the two-tree double buffer that lets React work on a scratchpad, and the strictly ordered commit phase that finally touches the DOM. Once you can trace one update through this pipeline, most "why did React do that?" questions stop being mysteries and start being lookups.

## What it is

React's update pipeline has four stages, and only one of them touches the DOM:

```
trigger  →  schedule  →  render phase  →  commit phase  →  (passive effects)
(setState)  (pick lane)   (pure, async,     (sync, DOM       (your useEffects,
                           interruptible)    mutations)        after paint)
```

The **fiber** is the unit that flows through this pipeline. Every component instance, every host element (`<div>`), every text node gets one — a plain JavaScript object that records what the thing is, where it sits in the tree, what props and state it rendered with last time, and what work is pending on it. Elements are the *descriptions* your components return each render ([jsx-and-rendering](../foundations/jsx-and-rendering.md)); fibers are the *persistent instance records* React reconciles those descriptions against. The element is the blueprint delivered fresh every render; the fiber is the building that gets renovated in place.

Two properties of this architecture explain almost everything downstream:

1. **The render phase is pure and disposable.** React can pause it, throw it away, and redo it — which is why your components must be pure ([rules-of-react](../foundations/rules-of-react.md)), and why StrictMode double-invokes them to prove it.
2. **The commit phase is synchronous and atomic.** The user never sees a half-applied tree. Interruption happens *between* renders and commits, never mid-commit.

## How it works under the hood

### From element to fiber

On first mount, React walks the element tree your root component returns and creates a fiber for each node. On every subsequent render, it *reuses* the existing fibers wherever type and key match — that matching algorithm, including `lastPlacedIndex` and its reorder degeneracy, is owned by [rendering-lists-and-keys](./rendering-lists-and-keys.md). What matters here is *where* it runs: child reconciliation happens inside the **parent's** unit of work, during the render phase. By the time a child fiber is processed, its fate (reuse, move, create, delete) was already decided one level up.

### Anatomy of a fiber

Trimmed to the fields that matter for this article. The names are stable across the 19.x line; the exact layout is internal and not a public API:

```ts
interface Fiber {
  // Identity — what kind of thing this is
  tag: WorkTag;              // FunctionComponent, HostComponent ('div'), HostText, ...
  type: any;                 // your component function, or the tag string
  key: string | null;        // your list key, stored verbatim
  stateNode: any;            // the real DOM node for hosts; null for function components

  // Position — the traversal pointers (this is a linked list, not a call stack)
  return: Fiber | null;      // parent in the work tree
  child: Fiber | null;       // first child
  sibling: Fiber | null;     // next sibling
  index: number;             // slot among siblings — the lastPlacedIndex math reads this

  // Data — what it rendered with
  pendingProps: any;         // props for the render in progress
  memoizedProps: any;        // props from the last completed render
  memoizedState: any;        // head of the hook linked list — your useState lives here
  updateQueue: any;          // queued state updates awaiting processing

  // Scheduling — what work is pending
  lanes: Lanes;              // pending updates on THIS fiber (bitmask)
  childLanes: Lanes;         // pending updates somewhere BELOW (bitmask)
  flags: Flags;              // Placement | Update | ChildDeletion | Passive | Ref | ...
  subtreeFlags: Flags;       // children's flags, bubbled up during completeWork

  // The other buffer
  alternate: Fiber | null;   // this fiber's twin in the other tree
}
```

Two fields deserve a pause:

- **`memoizedState` is where your hooks live** — as a linked list, one node per hook call, in call order. When your component renders, React walks this list with a cursor: first `useState` call reads the first node, second reads the second. That cursor is the mechanical reason hooks can't go inside conditions — skip one call and every hook after it reads the wrong node. [rules-of-react](../foundations/rules-of-react.md) owns the full contract; this is the machine it protects.
- **`lanes` vs `childLanes`** is how React skips work without checking every component. If a fiber's own `lanes` are empty *and* its `childLanes` are empty, React doesn't even walk into that subtree — it clones the pointer and moves on. Your `setState` call marks the fiber's lane, then walks *up* the `return` chain marking every ancestor's `childLanes` — leaving a breadcrumb trail from the root straight down to the work.

### Two trees: double buffering

At any moment there are up to two fiber trees:

- The **current** tree — what's on screen. `root.current` points at it.
- The **workInProgress** tree — the scratchpad React builds the next render on.

Each fiber's `alternate` points at its twin in the other tree. During a render, React builds workInProgress fibers by cloning current fibers (or creating fresh ones), does all the diffing and computation there, and — only if the render completes — flips one pointer at commit: `root.current = finishedWork`. The trees swap roles; last render's current tree becomes next render's scratchpad.

This is why interruption is safe. If a higher-priority update arrives mid-render, React abandons the half-built workInProgress tree and starts over. The screen never flickered, because the screen was never involved — the current tree stayed untouched the whole time. It also means every fiber object gets reused in alternation, which is why you must never mutate objects you've handed to React ([state-and-usestate](../state/state-and-usestate.md) owns the handoff rules): both trees may be holding references to them.

### Lanes: how React decides what's urgent

Every update gets a **lane** — one bit in a 31-bit bitmask. The lane is assigned at trigger time based on what caused the update:

| Lane group (illustrative) | Assigned when | Behavior |
| --- | --- | --- |
| `SyncLane` | Discrete events: click, keydown, submit | Rendered without yielding — must feel instantaneous |
| `InputContinuousLane` | Continuous events: scroll, drag, mousemove | High priority, but can be coalesced |
| `DefaultLane` | Timeouts, network callbacks, anything else | Normal priority, time-sliced |
| `TransitionLanes` (multiple) | `startTransition`, `useDeferredValue` | Interruptible background work |
| `RetryLanes` | Suspense retries | Re-render after data arrives |
| `IdleLane` / `OffscreenLane` | Hidden content, idle work | Only when nothing else is pending |

(Exact bit values shift between minors; the groups and their ordering don't.)

Bitmasks make the scheduling questions cheap: "is there sync work pending?" is one `&` operation; "merge these two updates into one render?" is one `|`. Batching — the behavior [state-and-usestate](../state/state-and-usestate.md) traced from the update-queue side — is, from this side, just multiple updates landing in the same lane before a render starts: React processes the whole lane in one pass, one render, one commit. Three `setCount` calls in a click handler are three entries in an `updateQueue`, all tagged `SyncLane`, drained together.

When lanes compete, urgent wins — and here's the part that makes transitions work: a low-priority render in progress gets *abandoned* when urgent work arrives. The urgent lane renders and commits first; the transition lane then **restarts from scratch**. Your component function runs again for the same conceptual update. That's not an edge case — it's the design. Hold that thought for StrictMode.

### The render phase: beginWork down, completeWork up

The render phase is a depth-first traversal using the `child`/`sibling`/`return` pointers — a loop over a linked list, not recursion. That's the whole point of the fiber rewrite: a call stack can't be paused halfway, a loop with a `workInProgress` pointer can.

For each fiber, two operations:

- **`beginWork` (on the way down):** decide whether this fiber needs to render. If yes — call your component function, get the returned elements, reconcile them against the existing child fibers (create/reuse/move/delete decisions, `Placement`/`ChildDeletion` flags set). If no — **bail out** (next section). Then descend to `child`.
- **`completeWork` (on the way back up):** when a fiber has no children left to process, finish it — for host fibers, prepare DOM instances and diff props into an update payload; for everyone, bubble `flags` into the parent's `subtreeFlags`. Then move to `sibling`, or `return` upward.

That `subtreeFlags` bubbling is what makes commit fast: by the time the traversal finishes, the root knows exactly which subtrees contain effects, and commit can skip everything else without looking.

Between units of work, concurrent rendering checks `shouldYield()` — roughly every 5ms of work, React offers the main thread back to the browser. A 40ms render becomes eight 5ms slices with paint opportunities between them, instead of one dropped-frames block. Only *interruptible* lanes (transitions) yield; sync work runs to completion.

### Bailouts: the fast paths

`beginWork`'s first question is "can I skip this?" The bailout check, in order:

1. **Same props reference** — `oldProps === newProps`, reference equality, not shallow comparison. (This is why the Compiler and the stable-element patterns matter: they're all in the business of making this check pass.)
2. **No pending work on this fiber** — its `lanes` don't intersect the lanes being rendered.
3. **No context change** — nothing this fiber reads has a new value.

All three pass → the component function is **not called**. Then the second check: are `childLanes` also clear? If yes, React clones the child pointer and skips the *entire subtree* — thousands of components dismissed in one pointer copy. If no, React walks down without rendering this fiber, following the `childLanes` breadcrumbs to wherever the work actually is.

This is the machinery behind the stable-element bailout that [components-and-props](../foundations/components-and-props.md) owns: `{children}` passed down from an owner that didn't re-render *is* the same reference, check 1 passes, the subtree is skipped. It's also why blast radius is a structural property — [component-composition](../foundations/component-composition.md)'s two performance moves are both techniques for making these checks pass without writing `memo` once.

And it's why mutation is fatal. Push into the array you already handed to state, set the same reference back, and check 1 passes *against you* — React sees identical references, bails out, and your update vanishes. The bailout can't tell "nothing changed" from "you changed it in place."

### Interruption, replay, and why StrictMode double-invokes

Assemble the pieces: renders can be **abandoned** (urgent lane arrived), **restarted** (transition re-begins after the urgent commit), and **repeated** (Suspense retry re-renders after data lands). One user-visible update can legitimately execute your component function two, three, four times — and commit once.

So the render phase's purity requirement isn't etiquette, it's a load-bearing wall. Any observable side effect in render — a network call, an analytics ping, a mutation of shared state — fires an unpredictable number of times depending on scheduling weather you don't control. Code that's only correct when renders and commits are 1:1 is code with a concurrency bug that hasn't been scheduled yet.

**StrictMode is that scheduling weather, on demand.** In development, it invokes your component function twice per render (second pass's logs dimmed in the console), re-runs initializers, and — separately — runs the mount effect cycle as setup → cleanup → setup ([effects-and-synchronization](../effects/effects-and-synchronization.md) owns that audit). It's not simulating a hypothetical; it's rehearsing exactly what the concurrent scheduler does in production under load. Code that breaks under StrictMode double-invocation is code that breaks when a transition gets interrupted on a real user's mid-range phone. The double-invoke is the cheapest concurrency test you'll ever run.

### The commit phase: three synchronous sub-phases

The render phase produced a finished workInProgress tree with flags. Commit applies it — synchronously, uninterruptibly, in three ordered passes over the flagged fibers:

| Sub-phase | When | What runs |
| --- | --- | --- |
| **Before mutation** | DOM still shows the *old* tree | Snapshot reads (`getSnapshotBeforeUpdate`); passive-effect flush gets scheduled |
| **Mutation** | The DOM changes | Insertions, updates, deletions from `flags`; deleted subtrees' cleanup runs; old refs detach |
| **Layout** | DOM is new, **paint hasn't happened** | Refs attach; `useLayoutEffect` setup runs; class lifecycles fire |

Then the browser paints. Then, in a separate task *after* paint, **passive effects** — your `useEffect`s — flush. That gap is the entire difference between `useEffect` and `useLayoutEffect`: layout effects run pre-paint (block it; use only to measure/adjust DOM), passive effects run post-paint (never block pixels). The full decision rules live in [escape-hatches-audit](../effects/escape-hatches-audit.md) *(Wave 2, planned)* — as does `flushSync`, the escape hatch that forces render *and* commit to complete before the next line of your handler.

The pointer swap — `root.current = finishedWork` — happens between mutation and layout, which is why layout effects already see the new tree as "current."

### The owner chain: what DevTools means by "rendered by"

Every fiber records which component's render *created* its element — its **owner**, which is not necessarily its parent. [component-composition](../foundations/component-composition.md) owns the owner-vs-parent distinction and why it decides re-render behavior; the machinery note here is that DevTools reads this chain directly. Select a component, open the "rendered by" list — that's the owner chain, bottom to top. Pair it with Profiler's "Record why each component rendered" and you get, per commit: props changed (and which), state changed (and which hook index), or "parent rendered" — meaning it failed the bailout and the cause is up the owner chain, not local.

## Basic usage: observing the pipeline

You never call this machinery directly — but you can instrument it. Three tools, in escalating precision:

**1. StrictMode** — you already have it; keep it:

```tsx
// main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { App } from "./App";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

**2. The `<Profiler>` component** — programmatic phase and timing data:

```tsx
import { Profiler, type ProfilerOnRenderCallback } from "react";

const logRender: ProfilerOnRenderCallback = (
  id,            // the id prop below
  phase,         // "mount" | "update" | "nested-update"
  actualDuration, // ms spent rendering this subtree this commit
  baseDuration,   // estimated ms for a full no-bailout render — the ceiling
) => {
  console.table({ id, phase, actualDuration, baseDuration });
};

export function App() {
  return (
    <Profiler id="ProductTable" onRender={logRender}>
      <ProductTable />
    </Profiler>
  );
}
```

`actualDuration` far below `baseDuration` means bailouts are doing their job. The two converging means every child rendered — your blast radius is the whole subtree.

**3. React DevTools Profiler** — record an interaction, read the flame graph: gray bars didn't render (bailed out), colored bar width is `actualDuration`, and with "Record why each component rendered" enabled, clicking any bar tells you which check failed. This is rung 1 of the performance ladder — [typing-lag-rerender-storm](../recipes/performance/typing-lag-rerender-storm.md) owns the full measure-first workflow.

## Walkthrough: one keystroke, end to end

Small enough to trace, real enough to matter: a search input filtering a 300-row table.

```tsx
// SearchPage.tsx
import { useState } from "react";
import { ProductTable } from "./ProductTable";
import type { Product } from "./types";

interface SearchPageProps {
  products: Product[];
}

export function SearchPage({ products }: SearchPageProps) {
  const [query, setQuery] = useState("");

  const visible = query
    ? products.filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
    : products;

  return (
    <main>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search products…"
        aria-label="Search products"
      />
      <ProductTable products={visible} />
    </main>
  );
}
```

```tsx
// ProductTable.tsx
import type { Product } from "./types";

interface ProductTableProps {
  products: Product[];
}

export function ProductTable({ products }: ProductTableProps) {
  return (
    <table>
      <tbody>
        {products.map((p) => (
          <tr key={p.id}>
            <td>{p.name}</td>
            <td>{p.price.toLocaleString()}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

The user types "r". Trace it:

1. **Trigger.** The keystroke fires React's root-level listener ([conditional-rendering-and-events](../foundations/conditional-rendering-and-events.md) owns the delegation model). Input events are discrete → the dispatch runs at sync priority.
2. **Lane assignment.** `setQuery("r")` pushes an update onto the input's owning fiber's `updateQueue`, tagged `SyncLane`. The fiber's `lanes` gets the bit; the walk up the `return` chain sets `childLanes` on every ancestor to the root.
3. **Schedule.** The root sees pending sync work and schedules a render pass for `SyncLane` — after your handler finishes, so any further `setState` calls in it join the same queue (one render, not three).
4. **Render phase, downhill.** `beginWork` starts at the root. Ancestors above `SearchPage` have empty `lanes` but marked `childLanes` — they don't re-render, React just walks through them following the breadcrumbs.
5. **The work.** `SearchPage`'s fiber has a pending lane → `beginWork` calls your function. The hook cursor walks `memoizedState`, the update queue drains, `query` is `"r"`. The filter runs during render — derived, never mirrored. New elements come back.
6. **Reconciliation.** `SearchPage`'s children get diffed: `<main>`, `<input>`, `<ProductTable>` all match by type → fibers reused, `pendingProps` updated. `ProductTable` got a new `products` array (the filter created it) → bailout check 1 fails → it renders. Its 300-row map reconciles by `key={p.id}`: matched IDs reuse their `<tr>` fibers; filtered-out rows' fibers get `ChildDeletion` flags on the parent.
7. **Uphill.** `completeWork` runs host-fiber prop diffs — the `<input>` fiber gets an `Update` flag with payload `{ value: "r" }`; text fibers whose content didn't change get nothing. All flags bubble into `subtreeFlags`.
8. **Commit.** ~300 rows of function-component rendering is on the order of 10–25ms — sync lane, so no yielding, one pass. Mutation applies the input's value, removes deleted rows, updates changed text nodes. Pointer swap. Layout: nothing registered. Paint.
9. **Passive.** No effects here — the flush is a no-op. Note what this component *doesn't* have: no effect synchronizing `visible` into state. The filter derived during render kept the whole trace one render long instead of two ([effects-and-synchronization](../effects/effects-and-synchronization.md), branch 1 of the tree).

**Now make it interruptible.** At 300 rows this is fine. At 5,000 rows the filter-plus-render costs ~150ms *per keystroke*, all in `SyncLane`, all blocking — typing feels like wading. Split the urgent from the deferrable:

```tsx
// SearchPage.tsx — the input stays urgent; the table becomes a transition
import { useState, useTransition } from "react";
import { ProductTable } from "./ProductTable";
import type { Product } from "./types";

export function SearchPage({ products }: { products: Product[] }) {
  const [query, setQuery] = useState("");          // urgent: the echo in the box
  const [tableQuery, setTableQuery] = useState(""); // deferred: what the table shows
  const [isPending, startTransition] = useTransition();

  function handleChange(next: string) {
    setQuery(next);                                 // SyncLane
    startTransition(() => setTableQuery(next));     // TransitionLane
  }

  const visible = tableQuery
    ? products.filter((p) =>
        p.name.toLowerCase().includes(tableQuery.toLowerCase()),
      )
    : products;

  return (
    <main>
      <input
        value={query}
        onChange={(e) => handleChange(e.target.value)}
        aria-label="Search products"
      />
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        <ProductTable products={visible} />
      </div>
    </main>
  );
}
```

Re-trace the interesting part: each keystroke now creates **two** updates in **two** lanes. The `SyncLane` render is tiny — `SearchPage` re-renders, but `tableQuery` didn't change yet, so `visible` is the same computation over the same input; the input's value updates, commit, paint. The keystroke echoed in a few milliseconds. Then the `TransitionLane` render starts — the expensive one — sliced into ~5ms units, yielding between slices. If the user types again mid-slice, the new `SyncLane` update **abandons the transition render**, commits the echo, and the transition restarts with the newest value. The half-built workInProgress tree is discarded; the screen, driven by the untouched current tree, never showed a thing. Every render of `SearchPage` in that storm was a pure function call that either committed or evaporated — which is exactly the behavior StrictMode was rehearsing.

The transition/`useDeferredValue` mechanics, tearing, and when *not* to reach for this live in [concurrent-rendering](../concurrent/concurrent-rendering.md) *(Wave 3, planned)*; the full diagnostic ladder that decides whether you need it at all is [typing-lag-rerender-storm](../recipes/performance/typing-lag-rerender-storm.md)'s.

## Real-world patterns

**Debug renders with the owner chain, not console.log archaeology.** "Why did this render?" has exactly four answers — own state, own props, context, or "the bailout failed upstream" — and Profiler's "why did this render" tells you which. When it says *parent rendered*, resist fixing the child; walk the owner chain and fix the origin. Nine times out of ten the real fix is [state colocation or a composition move](../foundations/component-composition.md), not memoization.

**Engineer bailouts structurally.** Every technique in the performance canon is a way to make `oldProps === newProps` true or `lanes` empty: colocate state so fewer ancestors render; pass content as `children` so it rides through as a stable reference; let the Compiler stabilize the inline handlers and objects that used to break check 1 ([memoization-and-the-compiler](./memoization-and-the-compiler.md) *(Wave 2, planned)* owns the verification workflow). Manual `memo` is the last rung for a reason — it's a per-component patch on what's usually a structural problem.

**Use `key` as an identity scalpel.** Changing a fiber's `key` doesn't update it — it *discards the fiber subtree* and mounts a fresh one: new fibers, new hook lists, state gone by design. `<UserProfile key={userId} />` is a one-line "reset everything for the new user" ([state-and-usestate](../state/state-and-usestate.md) ranks it in the props-change escalation ladder). The cost is real — full remount, effects re-run — so it's a scalpel, not a convenience.

**Know which phase your cost lives in.** A slow interaction is either render-phase cost (many/expensive component calls — flame graph is wide) or commit-phase cost (heavy DOM mutation, layout effects forcing sync layout — flame graph is narrow but the frame still janks). They have disjoint fixes: render cost responds to bailouts, slicing, doing less; commit cost responds to touching less DOM and getting measurement code out of the layout phase. Diagnosing one as the other burns days.

**Trust the double buffer in dev tooling.** When a render "ran twice but nothing broke," or a transition's console.logs appear for a state the UI never showed — that's abandoned workInProgress trees, working as designed. Log noise from discarded renders is not a bug report.

## Reference: the pipeline at a glance

| Stage | Interruptible | Your code that runs | DOM |
| --- | --- | --- | --- |
| Trigger | — | Event handlers, `setState` calls | Old |
| Render phase | Yes (transition lanes) | Component functions, initializers | Old |
| Commit: before mutation | No | Snapshot reads | Old |
| Commit: mutation | No | Cleanup of deleted subtrees | Changing |
| Commit: layout | No | `useLayoutEffect` setup, ref attach | New, unpainted |
| Paint | — | — | New, visible |
| Passive flush | No (separate task) | `useEffect` cleanup + setup | New, painted |

| Fiber field | One-line meaning |
| --- | --- |
| `type` / `key` / `stateNode` | What it is; your list key; the real DOM node (hosts) |
| `return` / `child` / `sibling` / `index` | Tree position as a walkable linked list |
| `pendingProps` / `memoizedProps` | Incoming props vs last committed props — bailout check 1 compares these |
| `memoizedState` | Hook linked list; the cursor's track |
| `updateQueue` | Queued `setState` calls awaiting the next render |
| `lanes` / `childLanes` | Pending work here / somewhere below — the breadcrumb trail |
| `flags` / `subtreeFlags` | What commit must do here / below |
| `alternate` | The twin fiber in the other buffer |

## Common mistakes

**1. Reading the DOM during render.** Render describes; it runs against a tree that may never commit, and before any mutation:

```tsx
function Tooltip({ anchorId }: { anchorId: string }) {
  // 🔴 render-phase DOM read: stale, null on first render, impure
  const rect = document.getElementById(anchorId)?.getBoundingClientRect();
  // ✅ measure in useLayoutEffect, store in state, render from that
}
```

**2. Equating "rendered" with "DOM changed."** A component can render 50 times and produce zero DOM operations — `completeWork` diffs host props and emits no `Update` flag when output matches. Profiler counts renders; the Elements panel shows mutations. Optimizing render counts when your actual cost is commit-side (or vice versa) is fixing the wrong phase.

**3. Writing "runs once" render logic.**

```tsx
let impressions = 0;
function Banner({ campaign }: { campaign: string }) {
  impressions++;                       // 🔴 counts renders, not views —
  trackImpression(campaign, impressions); // fires on replays, StrictMode, abandoned trees
  return <img src={`/banners/${campaign}.webp`} alt="" />;
}
// ✅ observable effects belong in useEffect (fires per commit) or the event handler
```

If StrictMode makes a number double, the number was already wrong — production replays just hadn't hit it yet.

**4. Expecting the DOM to be new right after `setState`.**

```tsx
function onExpand() {
  setExpanded(true);
  panelRef.current?.scrollIntoView(); // 🔴 render hasn't happened; panel isn't there
}
// ✅ scroll in an effect keyed to `expanded` — or flushSync when a handler
//    truly must see committed DOM (escape-hatches-audit owns the trade-offs)
```

**5. Mutating into a bailout.**

```tsx
function addTag(tag: string) {
  tags.push(tag);       // 🔴 same array reference
  setTags(tags);        // Object.is says "unchanged" → no render, update swallowed
}
setTags((prev) => [...prev, tag]); // ✅ new reference — the immutable-recipes table has every shape
```

The bailout machinery is fast *because* it trusts references. Mutation lies to it.

**6. Index keys meeting the matcher.** Reconciliation matches by position when keys are the index, so deleting row 2 makes every later row inherit its predecessor's fiber — hook state included. [rendering-lists-and-keys](./rendering-lists-and-keys.md) owns the corruption trace; the machinery view just makes it obvious *why*: the fiber survived, the data moved.

**7. Coordinating siblings through render-time side channels.**

```tsx
export const registry: string[] = [];
function Section({ title }: { title: string }) {
  registry.push(title);  // 🔴 render order + count are scheduler internals;
  return <h2>{title}</h2>; //    replays double-push, abandoned renders push ghosts
}
```

Renders describe; handlers cause. Cross-component coordination goes through state, context, or an external store read via `useSyncExternalStore` ([escape-hatches-audit](../effects/escape-hatches-audit.md) *(planned)*).

**8. Heavy work in the layout phase.** `useLayoutEffect` runs after mutation, *before paint* — the frame waits for it. Data processing or network setup there turns a 16ms frame into a 90ms one. Layout effects are for measure-then-adjust; everything else is passive.

**9. Reflexively memoizing when the parent renders.** "Parent rendered" in Profiler doesn't mandate `memo` — it usually means state lives too high or content could ride through as `children`. The Compiler already stabilized most of what hand-written `useCallback` used to; structural fixes shrink blast radius for whole subtrees at once. Manual memoization is rung 6, not rung 1.

## How this evolved

- **≤ React 15 — the stack reconciler.** Rendering was literal recursion: one component's render called its children's renders down the call stack. Correct, but unpausable — a 200ms tree walk was 200ms of frozen main thread, because you can't yield from the middle of a call stack.
- **React 16 — Fiber.** The rewrite this article describes: recursion became a loop over a linked list, which made pausing, resuming, and discarding work possible. Nothing user-facing changed yet — 16 shipped the engine idling.
- **16.8 — hooks.** Function-component state got its home on `fiber.memoizedState` as the hook list. The rules of hooks are the cursor's requirements, published.
- **18 — concurrency, on.** `createRoot` enabled time slicing, transitions, and automatic batching in all lanes (pre-18, batching only covered React event handlers — the timeout-batching gotchas in old Stack Overflow answers date from this). The purity contract became load-bearing in production.
- **19 — built on it.** Legacy root mode gone; Actions, `use()`, and optimistic updates all assume the interruptible pipeline underneath ([actions](../concurrent/actions.md) *(Wave 3, planned)*).
- **19.x — the Compiler era.** The Compiler doesn't change this pipeline at all — it changes the *inputs*, auto-stabilizing the references that decide bailout check 1. The machinery you just learned is the machinery it optimizes for.

## Exercises

**1. Owner-chain hunt.** Build `App → Layout → Sidebar` where `App` holds a `theme` state and passes it only to `Sidebar`. Toggle the theme with DevTools Profiler recording ("Record why each component rendered" on). Why does `Layout` render, and what single structural change stops it — without `memo`?
*Hint: `Layout`'s "why" says parent rendered. Who is `Sidebar`'s owner? What happens if `Layout` receives `Sidebar` as `children` instead of rendering it?*

**2. Bailout prediction.** Parent holds `count`; it renders `<A />`, `<B items={ITEMS} />` (module-constant array), and `{children}` received from *its* parent. Predict which of the three render when `count` changes, then verify with `console.count` in each. Which bailout check saved each survivor?
*Hint: check 1 is reference equality on props. Which of the three received a reference that was re-created during the parent's render?*

**3. Watch the slices.** Render a 2,000-row list where each row logs `performance.now()` on render. Trigger a re-render sync, then via `startTransition`, and compare the timestamp spreads. Sync: one contiguous block. Transition: clusters with ~gaps between them. Now type into an urgent input mid-transition — find the log cluster from the render that never committed.
*Hint: gaps ≈ yields at ~5ms boundaries. The abandoned render's rows log, but the DOM (and Profiler's commit list) never show that pass.*

## Summary

- One update flows trigger → lane → render phase → commit phase → passive effects. Only commit touches the DOM, and it's atomic.
- Fibers are the persistent per-instance records; elements are disposable descriptions reconciled against them. Hooks live on the fiber as a linked list — the rules of hooks are that list's contract.
- Two trees, one pointer swap: rendering happens on a scratchpad, which is what makes abandoning and restarting work safe and invisible.
- Lanes rank urgency as bitmasks; batching is same-lane collapse; transitions are the interruptible lanes that yield every ~5ms and lose to sync work.
- Bailouts skip work via reference equality plus lane emptiness — every real performance technique is a way of making those checks pass structurally.
- Renders can run any number of times per commit. StrictMode's double-invoke is that reality rehearsed in dev, not an annoyance to silence.
- Commit runs before-mutation → mutation → layout, then paint, then your `useEffect`s. Knowing which phase holds your cost is half of every performance diagnosis.

## See also

- [thinking-in-react](../foundations/thinking-in-react.md) — the mental model this article mechanizes
- [rules-of-react](../foundations/rules-of-react.md) — the contract the pipeline assumes; the enforcement stack
- [state-and-usestate](../state/state-and-usestate.md) — the update queue and batching from the state side
- [rendering-lists-and-keys](./rendering-lists-and-keys.md) — the child-reconciliation algorithm that runs inside `beginWork`
- [component-composition](../foundations/component-composition.md) — owner vs parent; the structural moves that engineer bailouts
- [typing-lag-rerender-storm](../recipes/performance/typing-lag-rerender-storm.md) — the measure-first diagnostic ladder built on this machinery
- [memoization-and-the-compiler](./memoization-and-the-compiler.md) *(Wave 2, planned)* — what the Compiler stabilizes and how to verify it
- [concurrent-rendering](../concurrent/concurrent-rendering.md) *(Wave 3, planned)* — transitions, `useDeferredValue`, and tearing in depth

## References

- [Render and Commit — react.dev](https://react.dev/learn/render-and-commit)
- [<StrictMode> — react.dev](https://react.dev/reference/react/StrictMode)
- [<Profiler> — react.dev](https://react.dev/reference/react/Profiler)
- [useTransition — react.dev](https://react.dev/reference/react/useTransition)
- [Keeping Components Pure — react.dev](https://react.dev/learn/keeping-components-pure)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.