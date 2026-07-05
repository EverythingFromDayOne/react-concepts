---
article_id: useref-and-the-dom
concept_folder: effects
wave: 2
related:
  - effects/effects-and-synchronization
  - rendering/how-react-renders
  - foundations/rules-of-react
  - foundations/component-composition
  - effects/custom-hooks
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# useRef and the DOM

> **Lead with this:** state is for values that drive what renders; refs are for values that must *survive* renders without driving them — timer IDs, library instances, abort controllers, and handles to real DOM nodes. A ref is a box React hands your component instance once and never looks inside again: writing to it schedules nothing, reading it snapshots nothing. React 19 flattened years of ceremony around the DOM half of this story — `ref` is now an ordinary prop, and ref callbacks return cleanup functions — so the modern patterns are shorter than the workarounds they replaced. This article covers both jobs of the ref, the commit-phase timing that governs when `ref.current` actually exists, and the embassy pattern for keeping imperative code contained.

## What it is

`useRef(initialValue)` returns the same `{ current }` object on every render of a component instance. That's the entire API — and each half of the sentence carries a rule:

- **The same object** — its identity never changes, so it's exempt from dependency arrays and safe to close over anywhere.
- **On every render** — it's per-instance, like state. Two `<Stopwatch />`es get two boxes.
- **`{ current }` is mutable and non-reactive** — assign freely from handlers and effects; nothing re-renders, nothing is queued, nothing snapshots.

That gives refs exactly two legitimate jobs:

1. **Instance variables** — values the render output doesn't depend on: interval IDs, WebSocket instances, `AbortController`s, "have we already animated?" flags, imperative library objects.
2. **DOM handles** — the `ref` prop on a host element, which React fills with the real node at commit so effects and handlers can focus, scroll, measure, observe.

The dividing line against state ([state-and-usestate](../state/state-and-usestate.md)): *if the screen should change when the value changes, it's state.* Everything about state — snapshots, update queues, batching, re-renders — exists to keep renders pure and predictable. Refs opt out of all of it, which is precisely why the rules of render purity draw a border around them.

## How it works under the hood

### Where the box lives

`useRef` is a hook, so it occupies one node on the fiber's `memoizedState` linked list ([how-react-renders](../rendering/how-react-renders.md)). At mount, React creates `{ current: initialValue }` and stores it on that node. On every later render, the hook cursor reaches the node and returns *the stored object* — no comparison, no cloning. Both fiber alternates in the double buffer share the same hook state, so the box survives tree swaps, abandoned renders, and replays untouched. Stability isn't an optimization; it's the data structure.

One consequence hides in the argument: `useRef(expensiveThing())` evaluates the argument on *every* render and discards it after the first — the classic wasted-work bug. The sanctioned fix is the **lazy-init exception**, the one render-phase ref write [rules-of-react](../foundations/rules-of-react.md) licenses, because it's idempotent and observable only by the instance itself:

```tsx
const playerRef = useRef<AudioEngine | null>(null);
if (playerRef.current === null) {
  playerRef.current = new AudioEngine(); // legal: write-once, own-instance, deterministic
}
```

### How DOM refs attach: commit-phase choreography

A `ref` on a host element doesn't do anything during render — render produces descriptions, and `ref.current` is still yesterday's value (or `null`). The attachment is commit work, and its ordering is worth knowing precisely ([how-react-renders](../rendering/how-react-renders.md) owns the phase pipeline):

1. **Mutation phase:** the *entire* DOM for this commit is built and inserted. Refs on deleted nodes detach here (cleanup runs / `null` assigned).
2. **Layout phase:** refs attach — **children before parents**, interleaved with `useLayoutEffect` in the same child-first pass.
3. Paint, then passive effects.

Two guarantees fall out. Inside a ref callback or layout effect, *your* node exists and the surrounding DOM tree is complete — `node.parentElement`, `node.closest()`, and measurements all work, because mutation finished before layout began. But *other components' ref objects* may not be populated yet: a child's ref callback runs before its parent's `ref.current` is assigned. Traverse the DOM from your node; don't read sibling refs during attach.

### Ref callbacks and the 19 cleanup contract

Passing a function as `ref` gets you called at attach time with the node. Since React 19, that function can **return a cleanup**, which runs at detach — the same symmetry contract effects live by ([effects-and-synchronization](./effects-and-synchronization.md)):

```tsx
<div
  ref={(node) => {
    const observer = new ResizeObserver(handleResize);
    observer.observe(node);
    return () => observer.disconnect(); // 19: runs on detach
  }}
/>
```

```tsx
// legacy: written for React <19 — modernized in the upgrade pass
// Pre-cleanup convention: React called the ref again with null on detach,
// and you branched on it.
<div
  ref={(node) => {
    if (node) {
      observerRef.current = new ResizeObserver(handleResize);
      observerRef.current.observe(node);
    } else {
      observerRef.current?.disconnect();
    }
  }}
/>
```

Identity matters here exactly as it does for bailouts: if the callback you pass is a *different function* than last render's, React runs the old one's cleanup and calls the new one — a detach/attach cycle per render. The Compiler stabilizes inline ref callbacks like any other function value when their captures allow, so in compiled app code an inline callback typically attaches once; the discipline that makes you safe either way is the same as for effects — **setup must be idempotent and cleanup must be symmetric**, because StrictMode will rehearse the cycle on mount regardless.

### `ref` is a prop now

Since 19, function components receive `ref` in props like anything else — no wrapper, no second argument:

```tsx
interface FancyInputProps extends ComponentPropsWithoutRef<"input"> {
  ref?: Ref<HTMLInputElement>;
}

export function FancyInput({ ref, ...rest }: FancyInputProps) {
  return <input ref={ref} className="fancy" {...rest} />;
}
```

```tsx
// legacy: written for React <19 — modernized in the upgrade pass
export const FancyInput = forwardRef<HTMLInputElement, FancyInputProps>(
  function FancyInput(props, ref) {
    return <input ref={ref} className="fancy" {...props} />;
  },
);
```

`forwardRef` still works in 19.2 (deprecated, not removed), so you'll read it in dependencies for years — recognize it, don't write it. The related element-level changes (`element.ref` access removed, ref no longer extracted specially during element creation) are part of the 19 element anatomy [jsx-and-rendering](../foundations/jsx-and-rendering.md) owns. The `ComponentPropsWithoutRef` + destructure-then-spread technique for prop pass-through is owned by [components-and-props](../foundations/components-and-props.md) — the pattern above is that technique with `ref` added back explicitly, on your terms.

### `useImperativeHandle`: publishing a designed surface

Forwarding the raw node hands the parent *everything* — any DOM method, any style write. `useImperativeHandle` replaces the node with an object you design:

```tsx
useImperativeHandle(ref, () => ({
  focus: () => inputRef.current?.focus(),
  shake: () => inputRef.current?.animate(shakeKeyframes, 300),
}), []);
```

The parent's `ref.current` becomes that object — a two-method treaty instead of open borders. Where this sits in API design: it's past the top of the [escalation ladder](../foundations/component-composition.md) (children → slots → compound → render props), reserved for genuinely imperative verbs — focus, scroll, play, select — that have no declarative shape. If the handle you're designing has *nouns* in it (`getValue`, `setItems`), that's state trying to escape upward; lift it instead.

## Basic usage

The two jobs, minimally:

```tsx
// Job 1: DOM handle — focus on demand
import { useRef } from "react";

export function SearchBar() {
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <div role="search">
      <input ref={inputRef} aria-label="Search" />
      <button onClick={() => inputRef.current?.focus()}>Focus search</button>
    </div>
  );
}
```

```tsx
// Job 2: instance variable — a timer ID the UI never displays
import { useEffect, useRef, useState } from "react";

export function Stopwatch() {
  const [elapsedMs, setElapsedMs] = useState(0);
  const intervalRef = useRef<number | null>(null);

  function start() {
    if (intervalRef.current !== null) return; // already running — idempotent
    const startedAt = Date.now() - elapsedMs;
    intervalRef.current = window.setInterval(() => {
      setElapsedMs(Date.now() - startedAt);
    }, 50);
  }

  function stop() {
    if (intervalRef.current !== null) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  }

  useEffect(() => {
    return () => {
      if (intervalRef.current !== null) clearInterval(intervalRef.current);
    };
  }, []); // symmetric shutdown if unmounted mid-run

  return (
    <div>
      <output>{(elapsedMs / 1000).toFixed(2)}s</output>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

Note the split: `elapsedMs` is state because the screen shows it; the interval ID is a ref because nothing rendered depends on it. Put the ID in state and every tick's bookkeeping would be a pointless render; put the elapsed time in a ref and the display would freeze.

## Walkthrough: a chat log that behaves

The brief: a chat panel that auto-scrolls to new messages *only* when the user is already at the bottom, shows a "Jump to latest" button when they've scrolled up, and lets a Reply button on any message pre-fill and focus the composer. Three ref techniques, each earning its place.

**Step 1 — the composer, with a designed handle.**

```tsx
// MessageInput.tsx
import { useImperativeHandle, useRef, useState, type Ref } from "react";

export interface MessageInputHandle {
  focus(): void;
  prefill(text: string): void;
}

interface MessageInputProps {
  onSend: (text: string) => void;
  ref?: Ref<MessageInputHandle>;
}

export function MessageInput({ onSend, ref }: MessageInputProps) {
  const inputRef = useRef<HTMLInputElement>(null);
  const [draft, setDraft] = useState("");

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    prefill: (text) => {
      setDraft(text);
      inputRef.current?.focus();
    },
  }), []);

  function handleSubmit() {
    const text = draft.trim();
    if (!text) return;
    onSend(text);
    setDraft("");
    inputRef.current?.focus(); // keep the user typing
  }

  return (
    <div className="composer">
      <input
        ref={inputRef}
        value={draft}
        onChange={(e) => setDraft(e.target.value)}
        onKeyDown={(e) => e.key === "Enter" && handleSubmit()}
        aria-label="Message"
      />
      <button onClick={handleSubmit}>Send</button>
    </div>
  );
}
```

The handle exposes two verbs and zero nouns. The draft stays owned by the composer; the parent can *cause* things, not *read* things.

**Step 2 — the log, with a sentinel watching the bottom.**

```tsx
// ChatLog.tsx
import { useLayoutEffect, useRef, useState } from "react";
import { MessageInput, type MessageInputHandle } from "./MessageInput";
import type { Message } from "./types";

interface ChatLogProps {
  messages: Message[];
  onSend: (text: string) => void;
}

export function ChatLog({ messages, onSend }: ChatLogProps) {
  const scrollerRef = useRef<HTMLDivElement>(null);
  const composerRef = useRef<MessageInputHandle>(null);
  const [atBottom, setAtBottom] = useState(true);

  // 19 ref-callback cleanup: the observer's lifetime IS the sentinel's lifetime
  function observeBottom(node: HTMLDivElement) {
    const observer = new IntersectionObserver(
      ([entry]) => setAtBottom(entry.isIntersecting),
      { root: node.parentElement, threshold: 0.99 },
    );
    observer.observe(node);
    return () => observer.disconnect();
  }

  // Pin to bottom when messages arrive — pre-paint, so no visible jump.
  // (Scroll adjustment is the canonical useLayoutEffect case; the full
  // layout-vs-passive decision lives in escape-hatches-audit, Wave 2 planned.)
  useLayoutEffect(() => {
    const el = scrollerRef.current;
    if (!el) return;
    const nearBottom = el.scrollHeight - el.scrollTop - el.clientHeight < 60;
    if (nearBottom) el.scrollTop = el.scrollHeight;
  }, [messages]);

  return (
    <section className="chat">
      <div ref={scrollerRef} className="chat-scroll">
        {messages.map((m) => (
          <article key={m.id}>
            <strong>{m.author}</strong> {m.text}
            <button onClick={() => composerRef.current?.prefill(`@${m.author} `)}>
              Reply
            </button>
          </article>
        ))}
        <div ref={observeBottom} aria-hidden className="sentinel" />
      </div>

      {!atBottom && (
        <button
          className="jump"
          onClick={() => {
            const el = scrollerRef.current;
            if (el) el.scrollTop = el.scrollHeight;
          }}
        >
          Jump to latest ↓
        </button>
      )}

      <MessageInput ref={composerRef} onSend={onSend} />
    </section>
  );
}
```

Three deliberate choices to read closely:

- **`root: node.parentElement`, not `root: scrollerRef.current`.** The sentinel's ref callback runs during the layout phase, child-first — the *DOM* parent exists (mutation finished), but `scrollerRef` may not be assigned yet. Traversing from the node you were handed sidesteps the ordering entirely.
- **The scroll decision reads the DOM, not `atBottom` state.** The layout effect measures live scroll position at the moment it matters. Depending on `atBottom` would both re-run the effect on every observer flip and race the observer's own commit timing — a synchronization contract with two masters. Measure once, from the source.
- **`atBottom` *is* state**, because the Jump button's visibility depends on it — screen changes, so state, per the dividing line.

**Step 3 — StrictMode audit.** Mount under StrictMode: the sentinel's ref callback runs attach → cleanup → attach; because setup constructs a fresh observer and cleanup disconnects it, the second cycle is invisible. The composer's handle is recreated by the double-invoked `useImperativeHandle` — also fine, it closes over stable refs. Symmetry bought correctness before any bug existed; that's the contract paying rent.

## Real-world patterns

**The containment embassy, built out.** When you must host an imperative library — a map, a chart, a video engine — the pattern is a single component that acts as sovereign territory with a React-shaped border. Everything imperative lives inside; props cross the border inward as synchronization, events cross outward as callbacks, and *no imperative handle leaks out*:

```tsx
interface MapPanelProps {
  center: LatLng;
  zoom: number;
  onSelectPin: (id: string) => void;
}

export function MapPanel({ center, zoom, onSelectPin }: MapPanelProps) {
  const mapRef = useRef<LibMap | null>(null);

  // Ref-mirror: the map subscribes ONCE, but always calls the freshest callback.
  const onSelectPinRef = useRef(onSelectPin);
  useEffect(() => {
    onSelectPinRef.current = onSelectPin;
  }, [onSelectPin]);

  // Lifetime = node lifetime (19 cleanup); construction is idempotent
  function initMap(node: HTMLDivElement) {
    const map = createLibMap(node, { center, zoom });
    map.on("select", (id: string) => onSelectPinRef.current(id));
    mapRef.current = map;
    return () => {
      map.destroy();
      mapRef.current = null;
    };
  }

  // Props → imperative calls: each effect synchronizes one concern
  useEffect(() => {
    mapRef.current?.setView(center, zoom);
  }, [center, zoom]);

  return <div ref={initMap} className="map-panel" role="application" />;
}
```

Every ref technique in this article shows up with a job: instance-in-ref for the library object, ref-callback cleanup binding the instance's lifetime to the node's, sync effects translating declarative props into imperative calls, and the **ref-mirror** (latest-ref) idiom letting a once-registered listener read current values without re-subscribing — the effect that writes the mirror re-runs cheaply; the subscription never churns. React 19.2 ships `useEffectEvent` as the codified form of exactly this mirror — same semantics, less plumbing; [custom-hooks](./custom-hooks.md) *(Wave 2, planned)* covers packaging it. The payoff of the embassy: from outside, `MapPanel` is indistinguishable from a normal declarative component. Blast radius of the imperative world: one file.

**Controller-in-ref for handler-driven aborts.** When a fetch starts in an *event handler* rather than an effect, there's no cleanup slot to abort from — the controller has to live somewhere across renders. A ref is that somewhere:

```tsx
const controllerRef = useRef<AbortController | null>(null);

async function handleSearch(query: string) {
  controllerRef.current?.abort();               // cancel the in-flight one
  const controller = new AbortController();
  controllerRef.current = controller;
  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal: controller.signal,
    });
    setResults(await res.json());
  } catch (err) {
    if (controller.signal.aborted) return;      // swallowed, per the contract
    throw err;
  }
}
```

This is the handler-abort variation of [search-race-condition](../recipes/data-fetching/search-race-condition.md); the recipe owns the full race-policy table and the reasons state is the *wrong* home for a controller (renders you don't need, staleness you do).

**Focus management.** Route changes, modal opens, and destructive-action confirmations should move keyboard focus deliberately — `ref.current?.focus()` in the effect that reacts to the transition. Doing this well (focus traps, restoration on close, `inert` on the background) is owned by [accessibility-in-react](../ecosystem/accessibility-in-react.md) *(Wave 4, planned)*; the mechanism is always a ref.

**Handles at design-system leaves only.** In production component libraries, `useImperativeHandle` earns its keep at the leaves — inputs, players, canvases — where imperative verbs are the honest API. If you find handles on *containers* ("give me the form's values"), the escalation ladder was skipped: that's lifted state wearing a trench coat.

## Reference

| | `useState` | `useRef` | Plain variable in component |
| --- | --- | --- | --- |
| Survives re-renders | ✅ | ✅ | ❌ recreated every call |
| Triggers re-render on change | ✅ | ❌ | ❌ |
| Read during render | ✅ (that's its job) | 🚫 (lazy-init only) | ✅ |
| Safe to write in handlers/effects | via setter | ✅ direct | pointless |
| Identity across renders | value may change | same box, always | new every render |

**When `ref.current` is populated (DOM refs):**

| Moment | `ref.current` is |
| --- | --- |
| During render (any render) | Last commit's node, or `null` — never this render's |
| Commit: mutation phase | Detached (`null` / cleanup ran) for replaced or deleted nodes |
| Commit: layout phase | Attached — children's refs before parents' |
| `useLayoutEffect` / `useEffect` / event handlers | ✅ current committed node |
| After conditional unmount of the target | `null` again — guard with `?.` |

**API surface:**

| API | Use | Status in 19.2 |
| --- | --- | --- |
| `useRef(init)` | The box: instance variables + DOM handles | Baseline |
| `ref` prop on function components | Passing refs through your components | Baseline (19+) |
| Ref callback returning cleanup | Node-lifetime setup/teardown | Baseline (19+) |
| `useImperativeHandle(ref, create, deps)` | Publish a designed handle instead of the node | Baseline |
| `forwardRef(render)` | Legacy ref passing | Deprecated — read, don't write |

## Common mistakes

**1. Reading or writing `ref.current` during render.** Renders replay, get abandoned, and run twice under StrictMode ([how-react-renders](../rendering/how-react-renders.md)) — a render-phase ref read is nondeterministic, a write is a side effect. The single sanctioned exception is lazy init: `if (ref.current === null) ref.current = create()`. Everything else belongs in handlers and effects — [rules-of-react](../foundations/rules-of-react.md) owns the full defense.

**2. Expecting a ref write to update the screen.**

```tsx
const likesRef = useRef(0);
<button onClick={() => { likesRef.current++; }}>
  {likesRef.current} likes {/* 🔴 renders only when something ELSE renders */}
</button>
```

If it's displayed, it's state. The tell in the wild: UI that "updates when I click twice" — the second click's render is displaying the first click's mutation.

**3. Grabbing the node before commit.**

```tsx
const rect = boxRef.current?.getBoundingClientRect(); // 🔴 in render: null or stale
```

`ref.current` is a commit-phase artifact. Measure in a layout effect, act in handlers, and always `?.` — a conditional render can legitimately return the target to `null`.

**4. Losing `ref` in prop pass-through.** Spreading `{...rest}` while your props type came from `ComponentPropsWithoutRef` means `ref` was never in the bag — callers' refs silently go nowhere. Accept `ref` explicitly in the props interface and place it deliberately, per the destructure-then-spread discipline in [components-and-props](../foundations/components-and-props.md).

**5. Observers without cleanup.**

```tsx
<div ref={(node) => { new IntersectionObserver(cb).observe(node); }} /> // 🔴 leaks per attach
```

Every attach without a detach is a leak the StrictMode rehearsal will double instantly — which is the feature. Return the disconnect.

**6. Non-idempotent setup that assumes "runs once."** A ref callback's attach can legitimately run more than once per node lifetime (identity change, StrictMode). Setup that appends, increments, or registers must either be paired with symmetric cleanup or guard itself (`if (mapRef.current) return`). Same contract as effects; same reasons.

**7. Handles that puppet state.**

```tsx
useImperativeHandle(ref, () => ({
  setItems: (items: Item[]) => setItems(items), // 🔴 state escaping through a handle
  getDraft: () => draft,                        // 🔴 parent reading child state
}));
```

Nouns in a handle mean ownership is wrong. Lift the state to the component that needs it; keep handles to verbs the DOM understands.

**8. The controller in state.**

```tsx
const [controller, setController] = useState<AbortController | null>(null); // 🔴
```

Every keystroke now renders twice (results + controller), and the handler that reads `controller` sees its render's snapshot — the stale one. Controllers are instance plumbing: ref, per the pattern above and [search-race-condition](../recipes/data-fetching/search-race-condition.md)'s variation.

**9. The broken mirror.**

```tsx
const latestQuery = useRef(query);
latestQuery.current = query; // 🔴 render-phase write — mutates during replays/abandons
```

The mirror idiom is legal only when the write happens *after commit* — in the small effect shown in the embassy pattern — or via `useEffectEvent`, which does the same thing inside React's own timing. Mid-render mirror writes leak state from renders that may never commit.

## How this evolved

- **String refs (pre-15 era):** `ref="input"`, resolved magically on `this.refs` — implicit, uncomposable, long gone.
- **`createRef` + callback refs (16):** explicit boxes for classes; callback refs as the flexible form — called with the node, then with `null`, the branching convention that survived a decade.
- **`useRef` (16.8):** the box becomes a hook; instance variables for function components arrive.
- **The `forwardRef` era (16.3–18):** function components couldn't receive `ref` as a prop, so a wrapper API existed solely to smuggle it through — a workaround that shaped thousands of component libraries.
- **React 19:** the workaround retired — `ref` is a prop; ref callbacks gain cleanup returns; `element.ref` extraction removed from element anatomy. `forwardRef` deprecated but present.
- **19.2 / Compiler era:** `useEffectEvent` codifies the latest-ref mirror; the Compiler stabilizes inline ref callbacks alongside every other function value, making the "detach/attach every render" concern mostly historical for compiled app code.

## Exercises

**1. Modernize the wrapper.** Take a `forwardRef`-based `TextField` from any pre-19 codebase (or write one) and convert it to ref-as-prop with `ComponentPropsWithoutRef<"input">`, preserving exact TypeScript behavior for callers. Then add a `MessageInputHandle`-style two-verb handle.
*Hint: the props interface gains an explicit `ref?: Ref<...>` — decide whether the type parameter is the DOM node or your handle, because that choice is your public API.*

**2. Prove the cleanup symmetry.** Build a `<LazyImage>` whose ref callback creates an IntersectionObserver that swaps in the real `src` on first visibility, with cleanup. Mount it under StrictMode and log attach/cleanup. Then break it deliberately (remove the cleanup) and describe what the logs — and the observer count in DevTools' memory tab — show after mounting 50 of them.
*Hint: `observer.disconnect()` in cleanup, and disconnect-after-first-hit inside the callback are two different lifetimes; you need both.*

**3. Fix the frozen handler.** An effect opens a WebSocket once (`[]` deps) and its `onmessage` calls `props.onMessage` — which is stale forever. Fix it three ways: deps (and observe the reconnect churn), ref-mirror, and `useEffectEvent`. Compare connection counts in the Network tab across ten prop changes.
*Hint: the deps version reconnects ten times; the other two connect once. What's the mirror's write timing relative to the socket's next message?*

## Summary

- One box, two jobs: instance variables the render doesn't display, and handles to committed DOM. If the screen depends on it, it's state — no exceptions.
- The box's identity is the data structure: same object every render, shared across the double buffer, exempt from deps.
- `ref.current` for DOM refs is a commit artifact — attached in the layout phase, children before parents, `null` during any render. Traverse from your node inside ref callbacks; don't read sibling refs.
- React 19: `ref` is a prop (retire `forwardRef`), and ref callbacks return cleanups — bringing node-lifetime setup under the same symmetry contract as effects. StrictMode rehearses both.
- `useImperativeHandle` publishes verbs, never nouns. Nouns in a handle are misplaced state.
- The embassy pattern contains imperative libraries: instance in a ref, lifetime bound to a ref-callback cleanup, props synced inward by effects, callbacks read through a post-commit ref-mirror (or `useEffectEvent`).

## See also

- [effects-and-synchronization](./effects-and-synchronization.md) — the symmetry contract ref-callback cleanups now share; the AbortController pattern in effect position
- [how-react-renders](../rendering/how-react-renders.md) — the commit sub-phases that dictate every timing rule in this article
- [rules-of-react](../foundations/rules-of-react.md) — refs-not-in-render, the lazy-init exception, and why render-phase writes leak
- [component-composition](../foundations/component-composition.md) — the escalation ladder that `useImperativeHandle` sits above
- [search-race-condition](../recipes/data-fetching/search-race-condition.md) — controller-in-ref as the handler-abort variation
- [custom-hooks](./custom-hooks.md) *(Wave 2, planned)* — packaging observers, mirrors, and `useEffectEvent` into reusable hooks
- [escape-hatches-audit](./escape-hatches-audit.md) *(Wave 2, planned)* — the full `useLayoutEffect` decision rules the chat walkthrough leaned on

## References

- [useRef — react.dev](https://react.dev/reference/react/useRef)
- [Manipulating the DOM with Refs — react.dev](https://react.dev/learn/manipulating-the-dom-with-refs)
- [useImperativeHandle — react.dev](https://react.dev/reference/react/useImperativeHandle)
- [Referencing Values with Refs — react.dev](https://react.dev/learn/referencing-values-with-refs)
- [forwardRef (legacy) — react.dev](https://react.dev/reference/react/forwardRef)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.