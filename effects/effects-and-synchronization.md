---
article_id: effects-and-synchronization
concept_folder: effects
wave: 1
related:
  - state/state-and-usestate
  - foundations/thinking-in-react
  - foundations/conditional-rendering-and-events
  - effects/useref-and-the-dom
  - effects/escape-hatches-audit
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Effects and Synchronization

> **Lead with this:** `useEffect` is the most misused API in React, and the misuse has one root: reading it as a *lifecycle hook* — "after render, do X" — when it is a *synchronization contract* — "while these values hold, keep this external system matched to them." Under the lifecycle reading, effects become the junk drawer: derived state, event logic, state chains, one-time init, all stuffed into `useEffect` because it's "where code runs." Under the synchronization reading, most of those effects visibly don't belong (there's no external system in sight), and the ones that remain get a precise shape: setup that connects, cleanup that disconnects, dependencies that are *derived from the code, not chosen*. This article builds that model down to the timing and snapshot mechanics, gives you the decision tree that deletes half the effects in a typical codebase, and locks the project's connection and fetch conventions — including the `AbortController` pattern everything async builds on.

## What it is

An effect synchronizes your component with a system **React doesn't manage**: a WebSocket, a browser API (`window` listeners, `matchMedia`, `document.title`), a timer, a non-React widget (a map library, a chart), a network request. Renders are pure descriptions ([`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md)); handlers respond to specific interactions ([`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md)); effects cover the remainder — things that must *stay true* about the outside world as long as the component shows what it shows.

```tsx
useEffect(() => {
  const connection = createConnection(roomId);   // setup: start synchronizing
  connection.connect();
  return () => connection.disconnect();          // cleanup: stop synchronizing
}, [roomId]);                                    // the values this synchronization reads
```

Read it as a sentence: *"While this component is on screen showing `roomId`, keep a connection to that room open."* Not "on mount, connect" — the mount/unmount framing is the class-era residue this article exists to remove. Three commitments follow from the sentence:

- **Setup and cleanup are symmetric.** Whatever setup starts, cleanup stops — completely. An effect whose cleanup fully undoes its setup can run any number of times and leave the world identical; that idempotence is what React's re-runs (and StrictMode) are built to exploit and verify.
- **Dependencies are derived, not chosen.** They are the reactive values the effect's code reads — all of them. The array isn't a knob for *when* to run; it's a declaration of *what this synchronization depends on*, and the linter computes it from your code better than you will.
- **One effect per synchronization.** Split effects by external system, not by lifecycle moment. "Everything that happens on mount" is not a concern; "keep the ticker connected" and "keep the tab title updated" are two.

And the meta-commitment, the article's thesis: **before writing any effect, prove you need one.** Two questions eliminate most candidates — *can this be computed during render?* and *is this caused by a specific user interaction?* — and the decision tree below runs the full elimination.

## How it works under the hood

### Timing: after paint, by design

The commit phase updates the DOM, the browser paints, **then** effects run. That ordering is deliberate: effects are for work that shouldn't block pixels — connecting, subscribing, logging. (The rare synchronization that must happen *before* paint — measuring layout to position a tooltip — is `useLayoutEffect`'s narrow job; the audit of it, `useInsertionEffect`, and `flushSync` lives in [`escape-hatches-audit.md`](escape-hatches-audit.md).) One structural note: on mount, children's effects run before their parent's — the tree completes bottom-up.

### The lifecycle, traced with snapshots

Every render that has an effect produces *its own* setup and cleanup closures, capturing that render's snapshot ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)) — this single fact explains almost everything effects do:

```
mount, roomId = 'general'
  → setup('general')                        // connects to general

re-render, roomId = 'travel'
  → cleanup from the 'general' render runs  // disconnects general — OLD snapshot
  → setup('travel')                         // connects travel — NEW snapshot

unmount
  → cleanup from the 'travel' render runs   // disconnects travel
```

Cleanup always runs **with the values of the render that created it** — which is exactly what you want (you must disconnect from the room you connected to, not the new one), and occasionally what surprises you (cleanup "sees stale state" is not a bug; it's the contract). Dependency comparison between renders is `Object.is` per element — the same identity semantics as everywhere else ([`../foundations/components-and-props.md`](../foundations/components-and-props.md)), with the same consequence: object and function literals in deps are new every render.

### StrictMode: the symmetry auditor

In development, StrictMode mounts every component, runs its effects, then immediately runs cleanup and setup once more: **setup → cleanup → setup**. This is not noise to suppress; it is a *proof obligation*. If your effect is a true synchronization — setup starts, cleanup fully stops — the extra cycle is invisible: connect, disconnect, connect, and the world holds one connection, same as production. If the double-cycle breaks something (two WebSockets, duplicate listeners, a doubled fetch actually *mattering*), the effect was leaking, and StrictMode found it on day one instead of after a weekend of users navigating back and forth. The reflex "my effect runs twice, how do I stop it" has exactly one right answer: you don't — you make cleanup real. (Why React does this: it reserves the right to unmount and remount trees while preserving state — back/forward cache-style — so effects must survive stop/start cycles anyway.)

### Stale closures: what lying to the linter buys you

Omit a dependency and the effect keeps running the **old render's closure** forever:

```tsx
// ❌ The interval is created once, closing over count = 0 — forever
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);   // count is THIS render's 0
  return () => clearInterval(id);
}, []);   // linter says [count]; we "knew better"
// Result: 0 → 1 → 1 → 1 → …  (every tick computes 0 + 1)
```

The effect isn't "seeing stale state" — it's executing exactly the snapshot it captured, as designed. The fixes aren't linter appeasement; they're better designs: make the effect *not read* the value (`setCount(c => c + 1)` — the updater form, again earning its keep), or include the dep and accept the re-sync (fine when setup is cheap), or — when an effect genuinely needs *fresh* values without re-synchronizing — that's the gap the still-experimental `useEffectEvent` is designed to close; today's stable idioms are the updater form, cheap re-syncs, or a ref mirror ([`useref-and-the-dom.md`](useref-and-the-dom.md) and [`escape-hatches-audit.md`](escape-hatches-audit.md) cover the trade-offs). What is *never* the fix: suppressing the lint rule and shipping the frozen closure.

## The decision tree: do you need an effect at all?

Run every effect candidate through this, in order. Most die on the first two branches — and each branch's fix is an article you've already read.

**1. Computing something from existing props/state?** → Render-time derivation. No effect, no extra state. The mirror-state-plus-sync-effect version renders twice per change and can desynchronize; the derived version can't ([`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md), [`../state/state-and-usestate.md`](../state/state-and-usestate.md)).

**2. Resetting all state when a prop changes?** → `key`. Identity, not synchronization ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)).

**3. Adjusting part of the state when a prop changes?** → The escalation ladder: derive → `key` → guarded set-during-render → effect *last* ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)). The effect version paints a stale frame and then corrects it; the ladder's earlier rungs never do.

**4. Responding to a user interaction?** → The handler. This is the event-first rule from [`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md), and the laundering version deserves its autopsy:

```tsx
// ❌ An event, laundered through state into an effect
const [shouldSave, setShouldSave] = useState(false);
useEffect(() => {
  if (!shouldSave) return;
  saveDraft(draft);
  setShouldSave(false);
}, [shouldSave, draft]);
<button onClick={() => setShouldSave(true)}>Save</button>

// ✅ The event, handled
<button onClick={() => saveDraft(draft)}>Save</button>
```

The laundered version: fires twice under StrictMode's mount cycle if the flag is ever true at mount, races itself when clicks arrive faster than renders, saves a *different* `draft` than the one the user saw (the effect runs with a later snapshot), and scatters one action across three code sites. One function call replaces all of it. The tell in review: an effect whose dependency is a boolean named like a verb.

**5. Notifying a parent about a state change?** → Call the callback in the same handler that sets the state. `onChange`-via-effect delivers the notification a render late and double-fires in dev; the handler version is synchronous and single-sourced.

**6. A chain of effects, each setting state the next one watches?** → One event handler computing all consequences. Effect cascades (`setCard` → effect sets `goldCount` → effect sets `round`) take N renders for one logical transition, hide the causality, and each link can misfire independently — this is [`../state/usereducer-and-state-structure.md`](../state/usereducer-and-state-structure.md)'s home turf: transitions belong in one place.

**7. Something that must run once per app, not per component?** → Module scope, or a module-level `didInit` guard. Component mount is the wrong lifetime — components mount many times (StrictMode, remounts, multiple instances).

**8. Subscribing to an external *store* (something with getSnapshot semantics)?** → `useSyncExternalStore`, which handles the tearing and subscription bookkeeping effects can't ([`escape-hatches-audit.md`](escape-hatches-audit.md)).

**9. Fetching data?** → It *is* a synchronization, so an effect is legal — and the full bill for doing it by hand is itemized in the patterns section, with `TanStack Query` as where the bill gets paid ([`../ecosystem/data-fetching-tanstack-query.md`](../ecosystem/data-fetching-tanstack-query.md)).

What survives the tree: connections, browser-API subscriptions, third-party widget lifecycles, timers, and hand-rolled fetching you've chosen with open eyes. Notice the shape they share — *an external thing that must be started and stopped.*

## Basic usage

The canonical survivors, each a complete synchronization:

```tsx
// Browser API: keep `matches` synchronized with a media query
function useReducedMotion(): boolean {
  const [reduced, setReduced] = useState(
    () => window.matchMedia('(prefers-reduced-motion: reduce)').matches,
  );

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    const onChange = (e: MediaQueryListEvent) => setReduced(e.matches);
    mq.addEventListener('change', onChange);
    return () => mq.removeEventListener('change', onChange);
  }, []);   // reads no reactive values — genuinely dependency-free

  return reduced;
}
```

```tsx
// Third-party widget: initialize + destroy, options as deps
useEffect(() => {
  const chart = createChart(containerRef.current!, { symbol, theme });
  return () => chart.destroy();
}, [symbol, theme]);   // option change = full re-sync; cheap widgets make this fine
```

```tsx
// Document state: sync the title; "cleanup" restores the invariant
useEffect(() => {
  document.title = unread > 0 ? `(${unread}) Inbox` : 'Inbox';
  return () => { document.title = 'Inbox'; };
}, [unread]);
```

And the shape rule for async: effects can't *be* async (an async function returns a promise, and React expects a cleanup function), so the convention is an inner function:

```tsx
useEffect(() => {
  const controller = new AbortController();
  async function load() { /* … */ }
  load();
  return () => controller.abort();
}, [url]);
```

## Walkthrough — a live price tile, synchronized honestly

The scenario: a dashboard tile shows a live price for a symbol. The symbol is a prop; the feed is an external push API; there's a pause button; and StrictMode is on, because it's on in this project. Four steps, three of which are lessons.

### Step 1 — the external system, made concrete

```tsx
// ticker.ts — the thing React doesn't manage (stand-in for your WS/SSE client)
export interface TickerHandle {
  close: () => void;
}

export function subscribeTicker(
  symbol: string,
  onPrice: (price: number) => void,
): TickerHandle {
  console.log(`[ticker] OPEN ${symbol}`);
  const id = setInterval(() => onPrice(100 + Math.random() * 10), 1000);
  return {
    close: () => {
      console.log(`[ticker] CLOSE ${symbol}`);
      clearInterval(id);
    },
  };
}
```

The logs are the instrumentation — the whole walkthrough is about watching them.

### Step 2 — the leak, observed

The version everyone writes first:

```tsx
// ❌ PriceTile v1 — no cleanup
export function PriceTile({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);

  useEffect(() => {
    subscribeTicker(symbol, setPrice);
  }, [symbol]);

  return <div className="tile">{symbol}: {price?.toFixed(2) ?? '—'}</div>;
}
```

Console on mount, StrictMode dev:

```
[ticker] OPEN AAPL
[ticker] OPEN AAPL      ← the audit cycle's second setup. Nothing closed the first.
```

Two live subscriptions, one tile — and every symbol switch adds another, because the effect from the old render never stopped its ticker. This *is* the production bug (navigate around an SPA for ten minutes and count the zombie intervals); StrictMode merely surfaced it at mount instead of at scale. The fix is not `useRef` guards or "run once" hacks — it's the missing half of the synchronization:

```tsx
useEffect(() => {
  const handle = subscribeTicker(symbol, setPrice);
  return () => handle.close();
}, [symbol]);
```

```
[ticker] OPEN AAPL
[ticker] CLOSE AAPL     ← audit cycle: cleanup proves symmetric
[ticker] OPEN AAPL      ← world state: one subscription. Same as prod.
```

And switching `symbol` to `MSFT` now reads exactly like the lifecycle trace section said it would: `CLOSE AAPL`, `OPEN MSFT` — cleanup with the old snapshot, setup with the new.

### Step 3 — state joins the synchronization

Pause arrives. Paused means *disconnected* — not "connected but ignoring," which would keep a socket open to throw its data away:

```tsx
export function PriceTile({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);
  const [paused, setPaused] = useState(false);

  useEffect(() => {
    if (paused) return;                                  // no setup → nothing to clean up
    const handle = subscribeTicker(symbol, setPrice);
    return () => handle.close();
  }, [symbol, paused]);

  return (
    <div className="tile">
      {symbol}: {price?.toFixed(2) ?? '—'} {paused && <em>(paused)</em>}
      <button onClick={() => setPaused((p) => !p)}>{paused ? 'Resume' : 'Pause'}</button>
    </div>
  );
}
```

Toggling pause re-runs the effect — `CLOSE`, then either nothing (paused) or `OPEN` (resumed) — and that's *fine*: re-synchronization is the mechanism, not a cost to engineer around. The early-return-without-setup shape is the idiom for conditional synchronizations: when there's nothing to start, return no cleanup, and the contract still holds. Note also what the button does — it flips *state*, and the synchronization follows declaratively. It does **not** reach for the handle imperatively; the effect owns the connection's lifetime, the handler owns the user's intent, and the boundary stays clean.

### Step 4 — the second system gets its own effect

"Show the live price in the tab title." Different external system → different effect, even though "it also happens when price changes" makes it *feel* like the first effect's business:

```tsx
useEffect(() => {
  if (price === null) return;
  document.title = `${symbol} ${price.toFixed(2)}`;
  return () => { document.title = 'Dashboard'; };
}, [symbol, price]);
```

Split by system, the two effects have different dep lists, different cleanup, different reasons to change — merged, every price tick would tear down and reopen the ticker. "One effect per synchronization" is not style; it's what keeps re-sync cheap.

## Real-world patterns

### Fetching in an effect: the full bill, and the locked pattern

A fetch is a legitimate synchronization ("keep `results` matched to `query`"), and hand-rolling it correctly costs more than anyone budgets. The naive version and its four invoices:

```tsx
// ❌ Every failure mode in eleven lines
useEffect(() => {
  fetch(`/api/search?q=${encodeURIComponent(query)}`)
    .then((r) => r.json())
    .then(setResults);
}, [query]);
```

1. **Race:** type `rea`, then `react`; the slow `rea` response lands last and overwrites the right answer. Out-of-order responses are the default on real networks.
2. **Zombie updates:** unmount mid-flight and the response still lands — wasted work, and state writes into a component that's gone (React 18 removed the warning, not the waste).
3. **StrictMode double-fetch:** the audit cycle fires two requests; without cancellation, both land.
4. **No modeled failure:** non-2xx and network errors fall through; the UI has no error or loading truth.

The locked convention solves 1–3 with **one mechanism** — cancel the previous synchronization in cleanup — and 4 with the state-machine discipline from [`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md):

```tsx
type SearchState =
  | { status: 'idle' }
  | { status: 'loading'; query: string }
  | { status: 'success'; query: string; results: Result[] }
  | { status: 'error'; query: string; message: string };

const [search, setSearch] = useState<SearchState>({ status: 'idle' });

useEffect(() => {
  if (query === '') { setSearch({ status: 'idle' }); return; }

  const controller = new AbortController();
  setSearch({ status: 'loading', query });

  (async () => {
    try {
      const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
        signal: controller.signal,
      });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const results: Result[] = await res.json();
      setSearch({ status: 'success', query, results });
    } catch (err) {
      if (controller.signal.aborted) return;   // superseded, not failed — say nothing
      setSearch({ status: 'error', query, message: err instanceof Error ? err.message : 'Request failed' });
    }
  })();

  return () => controller.abort();
}, [query]);
```

The load-bearing lines: `controller.abort()` in cleanup means a new query (or unmount, or StrictMode's audit) *cancels the previous synchronization at the network level* — stale responses can't land because they never complete; and the `signal.aborted` check in `catch` keeps cancellation from masquerading as an error. This is the project's **locked fetch-in-effect pattern**: every hand-rolled fetch in an effect carries an `AbortController`, aborts in cleanup, and ignores aborts in `catch` — no exceptions.

And the honest conclusion: that's ~30 lines per endpoint, and it *still* lacks caching, request dedup across components, retries, and revalidation. Which is not an argument to skip the abort discipline — it's the argument for why [`../ecosystem/data-fetching-tanstack-query.md`](../ecosystem/data-fetching-tanstack-query.md) is this project's baseline, and this exact code is stage one of the `search-race-condition` recipe, where the arc continues.

### The non-passive listener escape hatch — paid in full

[`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md) left a debt: React's root `wheel`/`touch` listeners are passive, so `onWheel` can't `preventDefault`. The escape hatch is a native listener, and it's an effect because it's a synchronization with a DOM node:

```tsx
const canvasRef = useRef<HTMLCanvasElement>(null);   // refs in full: useref-and-the-dom.md

useEffect(() => {
  const el = canvasRef.current;
  if (!el) return;

  const onWheel = (e: WheelEvent) => {
    e.preventDefault();                               // works: we own this listener's options
    applyZoom(e.deltaY);
  };

  el.addEventListener('wheel', onWheel, { passive: false });
  return () => el.removeEventListener('wheel', onWheel);
}, []);
```

The same shape covers every "React's synthetic system can't express this" case: `window`/`document` targets, capture-with-options, custom DOM events from web components. Note the dep honesty question it raises: if `applyZoom` reads props/state, the linter wants it in deps — include it (listeners are cheap to re-attach), or route through the updater/ref idioms from the stale-closure section. Never the suppress comment.

### The registration pattern — the composition debt, paid

[`../foundations/component-composition.md`](../foundations/component-composition.md)'s Tabs used DOM-sibling focus movement and promised the real version. Registration is compound parts announcing themselves to the family — and it's an effect, because "the root's list of parts" is external state the part synchronizes with:

```tsx
// In the family context: register/unregister provided by the root
function Tab({ value, children }: TabProps) {
  const { register, unregister /* … */ } = useTabsContext('Tab');

  useEffect(() => {
    register(value);
    return () => unregister(value);
  }, [value, register, unregister]);
  /* … */
}

// In the root: an ordered id list in state; roving/typeahead/Home/End all read it
const register = useCallback((id: string) => {
  setIds((prev) => (prev.includes(id) ? prev : [...prev, id]));
}, []);
```

Mount registers, unmount unregisters, `value` change re-registers — the part's presence in the family is now exactly as true as its presence in the tree, including for tabs behind conditionals. (The `useCallback` on `register` is the rare pre-Compiler necessity in *library* code: it's in consumers' dep arrays, so its identity is part of the contract — the boundary case [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md) enumerates.)

### The symmetry table

Cleanup is never creative — it's the inverse lookup of whatever setup did:

| Setup | Cleanup |
| --- | --- |
| `addEventListener(t, fn, opts)` | `removeEventListener(t, fn)` — same function reference, mind |
| `setInterval(fn, ms)` / `setTimeout` | `clearInterval(id)` / `clearTimeout(id)` |
| `subscribe(...)` → returns unsubscribe | call the unsubscribe |
| `new WebSocket(url)` / `new EventSource(url)` | `.close()` |
| `observer.observe(el)` (Intersection/Resize/Mutation) | `observer.disconnect()` |
| `fetch(url, { signal })` | `controller.abort()` |
| widget `create(...)` | widget `.destroy()` |
| write `document.title` / body class | restore the previous value |

If a setup line has no row-mate, the effect is a leak in waiting — that's the review heuristic.

## API quick reference

| Form | Meaning | Field notes |
| --- | --- | --- |
| `useEffect(setup)` | Re-synchronize after **every** render | Almost never right; usually a missing deps array |
| `useEffect(setup, [a, b])` | Re-sync when `Object.is` says `a` or `b` changed | Deps = every reactive value the code reads; the linter derives them |
| `useEffect(setup, [])` | Sync once with the **mount snapshot** | Legal only when the code truly reads no reactive values — not "I want mount-only" |
| `return () => {…}` from setup | Cleanup | Runs before the next setup and at unmount, with **its own render's snapshot** |
| async setup | Not allowed directly | Inner `async` function + abort in cleanup — the locked shape |
| `useLayoutEffect` | Same contract, runs **before paint** | DOM measurement only; blocks pixels ([`escape-hatches-audit.md`](escape-hatches-audit.md)) |
| `useSyncExternalStore` | Subscribing to stores with `getSnapshot` | The correct tool where effect-based subscriptions tear ([`escape-hatches-audit.md`](escape-hatches-audit.md)) |

## Common mistakes

**Deriving state in an effect.** The flagship, branch 1 of the tree — mirror state + sync effect where a plain expression belonged. Extra render per change, desync windows, and the gateway to effect chains. Delete the state; compute it.

**Missing cleanup.** The walkthrough's leak: subscriptions, listeners, and intervals that outlive their render. StrictMode's doubled setup is the alarm; the fix is always the symmetric inverse, never a "did I already run" guard.

**Lying to the deps linter.** Omitting a dep freezes the closure at an old snapshot — the `0 → 1 → 1 → 1` interval. Fix the *design* (updater form, restructure, include-and-cheapen), not the lint output.

**Unstable object/function deps.** `options` built fresh each render in the deps array = re-sync every render = at best waste, at worst an effect↔state loop. Depend on the primitives inside it, build the object *in* the effect, or hoist it to module scope; the Compiler stabilizes app-code cases, but deps you *export* to consumers are contract ([`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md)).

**Fetching without abort.** Races, zombies, and StrictMode doubles — all three are the same missing `AbortController`. The locked pattern or the library; nothing in between.

**`useEffect(async () => …)`.** Returns a promise where React expects a cleanup function; the cleanup silently never exists. Inner function, always.

**Event logic laundered through flags.** Branch 4's autopsy: `setShouldX(true)` + watching effect. Double-fires, races itself, acts on the wrong snapshot. If a user did it, a handler handles it.

**The self-feeding loop.** An effect that unconditionally sets state listed in its own deps: render → effect → set → render → effect. Usually a derivation in disguise (branch 1) or a missing condition; occasionally a sign the whole thing is a reducer transition.

**`[]` as `componentDidMount` cosplay.** Hardcoding "run once" onto code that reads `roomId` means the second room never connects. Empty deps are a *conclusion* the linter reaches when the code reads nothing reactive — never a starting intention.

**Fighting cleanup's snapshot.** Trying to make cleanup see the *new* value ("disconnect from the room we're *going to*") misreads the contract: cleanup's job is to undo *its own* setup, and its own setup used the old value. If cleanup seems to need fresh data, the synchronization is drawn around the wrong boundary.

## How this evolved

| Era | Change | What it means now |
| --- | --- | --- |
| Class era | `componentDidMount` / `DidUpdate` / `WillUnmount` | One concern smeared across three methods, three concerns tangled in each method — the split-by-*time* model `useEffect` replaced with split-by-*concern* |
| React 16.8 (2019) | `useEffect` ships | Setup+cleanup+deps as one unit; the synchronization model becomes expressible |
| React 18 (2022) | StrictMode adds the mount audit cycle; unmounted-setState warning removed | Cleanup symmetry becomes machine-checked in dev; the warning that taught a generation `isMounted` hacks retires |
| React 19 (2024) | Ref callbacks gain cleanup functions | The setup/cleanup contract extends to refs — same symmetry, per-node ([`useref-and-the-dom.md`](useref-and-the-dom.md)) |
| Compiler era (2025+) | Deps-adjacent identity problems shrink in app code; `useEffectEvent` still experimental | Fewer accidental re-syncs; the fresh-values-without-resync gap remains officially open — design around it, don't suppress around it |

```tsx
// legacy: class-era split-by-time — one subscription across three methods,
// modernized in the upgrade pass
componentDidMount() { this.conn = connect(this.props.roomId); }
componentDidUpdate(prev) {
  if (prev.roomId !== this.props.roomId) {
    this.conn.close();
    this.conn = connect(this.props.roomId);
  }
}
componentWillUnmount() { this.conn.close(); }
```

## Exercises

### 1. Predict the audit

StrictMode is on. Write the exact console output for: mount with `symbol="AAPL"`, then a re-render changing it to `"MSFT"`, then unmount — for walkthrough v1 (no cleanup) *and* the fixed version. State how many live intervals exist at each stage of each.

*Hint: v1 — `OPEN AAPL`, `OPEN AAPL`, `OPEN MSFT`, then silence at unmount: three zombies. Fixed — `OPEN AAPL`, `CLOSE AAPL`, `OPEN AAPL`, `CLOSE AAPL`, `OPEN MSFT`, `CLOSE MSFT`: zero. The exercise is done when the fixed sequence stops surprising you.*

### 2. Three cures for one stale interval

The `0 → 1 → 1 → 1` counter from the stale-closure section. Fix it three ways — updater form with `[]`, honest `[count]` deps, and a version that ticks a `delay` from props — and rank them: which re-creates the interval per tick, which never re-creates it, and which is the only one that can respect a changing `delay`?

*Hint: the updater form never re-creates (the effect reads nothing reactive — `[]` is now *true*); `[count]` re-creates every second (correct, wasteful, and it drifts — each tick restarts the 1000ms clock); `delay` must be a dep in any version, and re-creating on delay change is the *point*. The ranking teaches the real skill: deps aren't good or bad, they're an accounting of what the sync reads.*

### 3. Un-launder the autosave

Inherited: a `dirty` flag set in every field's `onChange`, and an effect watching `[dirty]` that saves and clears it. Product also wants "saving…"/"saved" status. Refactor: the save moves to an explicit trigger (a debounced call in the change handler is acceptable — write it with `setTimeout` + cleanup-style cancellation in the handler's closure, or as a tiny `useDebouncedCallback` you sketch), status becomes a discriminated union, and the flag dies. Then list the three bugs the flag version had that yours structurally can't.

*Hint: the three — StrictMode double-save when mounting dirty, saving a later `draft` than the one that set the flag, and the flag-clear race when edits land mid-save. Your version can't have them because the save call sites are the edits themselves, each closing over exactly the draft it saw. If your debounce lives in an effect watching `draft`… you've re-laundered it; the timer belongs to the event.*

## Summary

You learned effects as they were designed: synchronization contracts, not lifecycle hooks. The mechanics — after-paint timing, per-render setup/cleanup closures with their own snapshots, `Object.is` dep comparison, StrictMode's setup→cleanup→setup symmetry audit, and stale closures as the honest price of lying to the linter. The method: run every candidate through the decision tree (derive it, `key` it, ladder it, handle it, notify synchronously, collapse the chain, hoist the init, `useSyncExternalStore` it) and let only real external systems through — then write them as the walkthrough did: one effect per system, symmetric cleanup proven by the audit cycle, conditional syncs via early return, state changing the sync declaratively. The locked conventions: inner-async with `AbortController`, abort in cleanup, `signal.aborted` swallowed in catch — the one mechanism that kills races, zombies, and StrictMode doubles together — and the symmetry table as the review heuristic. Effects are the escape hatch that earns its name: used for what only they can do, they're small, boring, and correct; used as the junk drawer, they're where React apps go to become haunted.

## See also

- [`../state/state-and-usestate.md`](../state/state-and-usestate.md) — snapshots, the escalation ladder, `key` resets: branches 1–3 of the tree
- [`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md) — the event-first rule and the passive-listener debt this article paid
- [`../foundations/component-composition.md`](../foundations/component-composition.md) — the registration pattern's consumer side
- [`useref-and-the-dom.md`](useref-and-the-dom.md) — refs in effects, ref-callback cleanup, the mirror idiom
- [`escape-hatches-audit.md`](escape-hatches-audit.md) — `useLayoutEffect`, `useSyncExternalStore`, `flushSync`: the specialized siblings
- [`../ecosystem/data-fetching-tanstack-query.md`](../ecosystem/data-fetching-tanstack-query.md) — where the fetch bill gets paid properly
- [`custom-hooks.md`](custom-hooks.md) — packaging the survivors (`useReducedMotion` above is already one)

## References

- [Synchronizing with Effects (react.dev)](https://react.dev/learn/synchronizing-with-effects)
- [You Might Not Need an Effect (react.dev)](https://react.dev/learn/you-might-not-need-an-effect)
- [Lifecycle of Reactive Effects (react.dev)](https://react.dev/learn/lifecycle-of-reactive-effects)
- [Separating Events from Effects (react.dev)](https://react.dev/learn/separating-events-from-effects)
- [Removing Effect Dependencies (react.dev)](https://react.dev/learn/removing-effect-dependencies)
- [`useEffect` API (react.dev)](https://react.dev/reference/react/useEffect)
- [`AbortController` (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.