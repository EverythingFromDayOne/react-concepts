---
article_id: thinking-in-react
concept_folder: foundations
wave: 1
related:
  - foundations/jsx-and-rendering
  - state/state-and-usestate
  - rendering/how-react-renders
  - foundations/rules-of-react
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Thinking in React

> **Lead with this:** React's core bet is that your UI is a function of your state: `UI = f(state)`. You never edit the screen — you edit data, and React re-projects the screen from it. A component is not a thing on the page; it's a function that *describes* what the page should look like for the current data, and React's job is to make the real DOM match that description as cheaply as possible. Every React API — hooks, Suspense, Actions, the Compiler — falls out of this one idea, and the majority of React bugs come from fighting it: mutating data in place, touching the DOM behind React's back, or storing things that should be computed. This article builds the model precisely, traces one click all the way through the machinery, and then applies the model to design a small feature the way you'd design a real one.

## What it is

React is a **declarative** UI library. You describe each possible state of the interface once, and React figures out the transitions between them. The alternative — the imperative model — is what UI code looks like without it: find a node, toggle a class, update a counter label, remember to also update the badge in the header, remember to undo all of it when the user clears the filter. With N pieces of state and M places they show up, imperative code hand-writes every combination and every transition. Declarative code writes N descriptions and lets the library handle the M×N bookkeeping. That trade is the entire product.

Three ideas make the model work:

**Components are functions that return descriptions.** A component takes props, reads state, and returns React *elements* — lightweight objects describing what should exist ([`jsx-and-rendering.md`](jsx-and-rendering.md) dissects them). It does not create DOM nodes and does not return them. Calling the same component twice with the same inputs must produce the same description; that purity requirement is not a style preference, it's what the entire machine is built on ([`rules-of-react.md`](rules-of-react.md) states the contract precisely).

**State is the source of truth; the DOM is a projection.** If something on screen can change, there is a piece of state it derives from. The rendered output is disposable — React can throw it away and recompute it at any time, because your state holds everything needed to rebuild it.

**Data flows one way.** Props flow down the tree; changes flow up through callbacks. A child never reaches over and edits its parent's data — it asks the parent to change it, the parent updates state, and the new data flows back down. This is what makes a large React tree debuggable: to find why something rendered wrong, you walk *up* to the state that produced it, never sideways.

## How it works under the hood

### The render loop

Everything React does is one loop with three steps:

**1. Trigger.** There are exactly two triggers: the initial mount, and a state update. Calling a state setter does not re-run anything on the spot — it *schedules* a render and enqueues the update. (This is why reading state right after setting it gives you the old value; state behaves like a snapshot per render — full mechanics in [`../state/state-and-usestate.md`](../state/state-and-usestate.md).)

**2. Render.** React calls your component functions, top-down from wherever the update originated, and collects the element trees they return. "Render" in React vocabulary means *calling your functions* — nothing more. No DOM work happens in this phase, which is exactly why this phase is allowed to be interrupted, thrown away, or run twice: it's supposed to have no observable side effects.

**3. Commit.** React diffs the new element tree against the previous one (reconciliation — comparison by element `type` and `key`, the subject of [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md)) and computes the minimal set of real DOM mutations. Then it applies them in one synchronous pass, and the browser paints.

The performance implication is the part most people get backwards: **renders are cheap, commits are expensive — and React already minimizes commits for you.** A component re-rendering 30 times that produces the same description each time results in zero DOM work. When React apps feel slow, the cause is almost never "too many renders" in the abstract; it's specific expensive work happening *during* renders, or commits thrashing layout. Which is why the fix is rarely "prevent renders" and usually "make renders pure and cheap" — the stance the React Compiler now automates ([`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md)).

### A click, traced end-to-end

Here's the whole loop in one component. The `console.log` lines are the instrumentation:

```tsx
import { useState } from 'react';

export function LikeButton() {
  const [likes, setLikes] = useState(0);
  console.log('render: likes =', likes);

  return (
    <button
      onClick={() => {
        setLikes(likes + 1);
        console.log('handler: likes is still', likes);
      }}
    >
      {likes} ♥
    </button>
  );
}
```

Click the button once, with `likes` at `0`, and this happens in order:

1. **The browser dispatches the click.** React catches it at the root (React uses event delegation — one real listener at the container, not one per button) and routes it to your handler.
2. **The handler runs.** `setLikes(1)` does *not* change `likes`. It records the update in a queue on this component's internal node and schedules a render. The log prints `handler: likes is still 0` — `likes` is a `const` captured by this closure, a snapshot from the render that created the handler. It will never change; the *next render* gets a new one.
3. **The handler finishes; React flushes.** All state updates from the same event are batched into a single render pass (automatic batching — one render even if the handler called three setters).
4. **Render phase.** React calls `LikeButton()` again. `useState` now returns `1`. The log prints `render: likes = 1`. A brand-new element tree comes back.
5. **Reconciliation.** Same element type (`button`) in the same position → keep the existing DOM node, diff the props. The text child changed: `0 ♥` → `1 ♥`.
6. **Commit.** React updates one text node. Nothing else in the document is touched.
7. **Paint.** The browser renders the frame.
8. **Effects run** (if the component had any) — after paint for `useEffect` ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) owns that story).

Every interaction in every React app is this sequence. When something confuses you — stale values in handlers, "why did this render twice", "why didn't this render at all" — locate which step your assumption broke in, and the confusion usually resolves.

### What React keeps between renders: fibers

Your function's local variables die when it returns. So where do state, the previous props, and the previous tree live? On a **fiber** — one internal node per component instance (and per host element) that persists across renders. Simplified shape:

```ts
// Heavily simplified — the real structure has ~25 fields
{
  type: LikeButton,        // the function itself (or 'button' for host fibers)
  key: null,
  stateNode: …,            // host fibers point at the real DOM node
  memoizedProps: {…},      // props from the last completed render
  memoizedState: {…},      // hooks live here — a linked list, one node per hook call
  child, sibling, return,  // pointers React walks to traverse the tree
  flags: 0b…,              // what commit must do here (update / place / delete)
  lanes: 0b…,              // priority of pending updates
  alternate: {…},          // the work-in-progress twin
}
```

Two details here explain rules you'd otherwise have to memorize:

- **Hooks are a linked list on the fiber, matched by call order.** First `useState` call → first node, second → second. That's the entire reason hooks can't go inside conditions or loops: a skipped call would shift every subsequent hook onto the wrong node, silently handing state to the wrong variable. The rule isn't ceremony; it's pointer arithmetic ([`rules-of-react.md`](rules-of-react.md)).
- **`alternate` is double buffering.** React renders into a work-in-progress copy of the fiber tree while the current one keeps driving the screen; commit atomically swaps them. Discarding a render is therefore structurally free — throw away the WIP tree, the current one was never touched. This is what makes interruptible rendering ([`../concurrent/concurrent-rendering.md`](../concurrent/concurrent-rendering.md)) possible at all.

The full machinery — render phases, priority lanes, how the traversal actually walks — is [`../rendering/how-react-renders.md`](../rendering/how-react-renders.md). At this article's level you need exactly one takeaway: *there is a persistent tree, your function is called against it, and everything React "remembers" lives there, not in your closures.*

### What purity buys

Because the render phase is a pure computation over state, React is free to:

- **Run it twice in development.** `<StrictMode>` deliberately double-invokes component bodies, `useState`/`useReducer` initializers, and updater functions, and runs one extra effect setup → cleanup → setup cycle on mount. If a component misbehaves under StrictMode, the component is wrong, not StrictMode — it just told you on day one instead of in a production incident.
- **Interrupt and discard it.** Concurrent rendering can pause a render mid-tree to handle something urgent, then restart. A render that mutated something outside itself can't safely be discarded — pure ones can, for free (see `alternate` above).
- **Skip it entirely.** The React Compiler memoizes components and values automatically, on the assumption that same inputs → same output. The purity rules are the contract that makes that legal.

This is the deepest version of the mental model: your components describe; React decides *when*, *whether*, and *how often* to ask for the description.

## Basic usage

A minimal but complete example of the model — one piece of state, two derived projections, one event flowing back up:

```tsx
import { useState } from 'react';

interface Product {
  id: string;
  name: string;
  inStock: boolean;
}

interface ProductListProps {
  products: Product[];
}

export function ProductList({ products }: ProductListProps) {
  const [query, setQuery] = useState('');

  // Derived during render — not stored, never out of sync
  const visible = products.filter((p) =>
    p.name.toLowerCase().includes(query.toLowerCase()),
  );

  return (
    <section>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Filter products"
      />
      <p>
        Showing {visible.length} of {products.length}
      </p>
      <ul>
        {visible.map((p) => (
          <li key={p.id}>
            {p.name} {p.inStock ? '' : '(out of stock)'}
          </li>
        ))}
      </ul>
    </section>
  );
}
```

Read it through the model: the filtered list and the count are both *projections* of `query`, computed fresh each render — they cannot drift apart, because neither is stored. The handler doesn't update the UI; it updates state, and the input's value, the list, and the count all follow on the next render. And nothing reads or writes the DOM — no `querySelector`, no class toggling, no "remember to also update the counter."

## Walkthrough — designing a feature with the five-step workflow

The example above was handed to you finished. Real work starts from a spec, and the failure mode is designing state *first* and components around it — you end up with state nobody renders, stored copies of derived data, and props tunneling through layers that don't care. This workflow inverts it. The spec: **a product table with a search box, an "only show in stock" checkbox, out-of-stock rows styled differently, and a live "Showing X of Y" count.**

### Step 1 — break the UI into a component hierarchy

One component per thing with a single responsibility or that repeats:

```
<FilterableProductTable>        owns the feature
├── <SearchBar>                 the input + checkbox
└── <ProductTable>              count line + table
    └── <ProductRow>            one product (repeats)
```

### Step 2 — build the static version first: props only, no state

Render the whole tree from data down, with zero interactivity. This step feels skippable and isn't: if you can't build the static version cleanly, the hierarchy is wrong, and fixing it now costs minutes instead of a refactor later.

```tsx
// product.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}
```

```tsx
// ProductRow.tsx
import type { Product } from './product';

export function ProductRow({ product }: { product: Product }) {
  return (
    <tr className={product.inStock ? undefined : 'row--out-of-stock'}>
      <td>{product.name}</td>
      <td>${product.price.toFixed(2)}</td>
      <td>{product.inStock ? 'In stock' : 'Out of stock'}</td>
    </tr>
  );
}
```

```tsx
// ProductTable.tsx
import type { Product } from './product';
import { ProductRow } from './ProductRow';

interface ProductTableProps {
  products: Product[]; // already filtered — this component just projects
  total: number;
}

export function ProductTable({ products, total }: ProductTableProps) {
  return (
    <>
      <p aria-live="polite">
        Showing {products.length} of {total}
      </p>
      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Price</th>
            <th>Availability</th>
          </tr>
        </thead>
        <tbody>
          {products.map((p) => (
            <ProductRow key={p.id} product={p} />
          ))}
        </tbody>
      </table>
    </>
  );
}
```

Note what the static step already settled: `ProductTable` and `ProductRow` are pure projections with no idea filtering exists. They'll never need to change again.

### Step 3 — find the minimal state

Inventory every piece of data in the feature and interrogate it. State is only what survives all three questions:

| Data | Changes over time? | Passed in as props? | Computable from something else? | Verdict |
| --- | --- | --- | --- | --- |
| `products` | (server-owned; a prop here) | ✓ | — | not state |
| search `query` | ✓ | ✗ | ✗ | **state** |
| `inStockOnly` | ✓ | ✗ | ✗ | **state** |
| visible rows | ✓ | — | ✓ (`products` + `query` + `inStockOnly`) | derive |
| visible count | ✓ | — | ✓ (`visible.length`) | derive |

Five candidates, two survive. Every entry you wrongly promote to state becomes a synchronization job you now own — this table is cheap insurance against that.

### Step 4 — decide where the state lives

Find every component that renders from each piece of state, then put it in their lowest common parent. `SearchBar` displays `query`/`inStockOnly`; `ProductTable` (via the parent's filtering) depends on them. Lowest common parent: `FilterableProductTable`. Not `App`, not a store, not context — *lowest*, because every level higher widens the re-render scope and couples components for no benefit.

### Step 5 — add inverse data flow

`SearchBar` renders state it doesn't own, so it receives handlers to request changes:

```tsx
// SearchBar.tsx
interface SearchBarProps {
  query: string;
  inStockOnly: boolean;
  onQueryChange: (query: string) => void;
  onInStockOnlyChange: (inStockOnly: boolean) => void;
}

export function SearchBar({
  query,
  inStockOnly,
  onQueryChange,
  onInStockOnlyChange,
}: SearchBarProps) {
  return (
    <form role="search">
      <input
        type="search"
        value={query}
        onChange={(e) => onQueryChange(e.target.value)}
        placeholder="Search products…"
      />
      <label>
        <input
          type="checkbox"
          checked={inStockOnly}
          onChange={(e) => onInStockOnlyChange(e.target.checked)}
        />
        Only show products in stock
      </label>
    </form>
  );
}
```

```tsx
// FilterableProductTable.tsx
import { useState } from 'react';
import type { Product } from './product';
import { SearchBar } from './SearchBar';
import { ProductTable } from './ProductTable';

export function FilterableProductTable({ products }: { products: Product[] }) {
  const [query, setQuery] = useState('');
  const [inStockOnly, setInStockOnly] = useState(false);

  const visible = products.filter((p) => {
    if (inStockOnly && !p.inStock) return false;
    return p.name.toLowerCase().includes(query.trim().toLowerCase());
  });

  return (
    <section>
      <SearchBar
        query={query}
        inStockOnly={inStockOnly}
        onQueryChange={setQuery}
        onInStockOnlyChange={setInStockOnly}
      />
      <ProductTable products={visible} total={products.length} />
    </section>
  );
}
```

Audit the result against the model: two `useState` calls for exactly the two facts the user controls; filtering derived in render, stored nowhere; children that are pure projections; changes flowing up through two named handlers. No effects, no refs, no store — and nothing to keep in sync, because there's only one copy of everything. When `products` graduates from a prop to a server fetch, this shape doesn't change; the fetch concern lands in [`../ecosystem/data-fetching-tanstack-query.md`](../ecosystem/data-fetching-tanstack-query.md) and hands this component the same array.

## Real-world patterns

### Derive, don't store

If a value can be computed from existing state or props, compute it during render. Storing it creates a second source of truth that you now have to keep synchronized — usually with an effect, which is the single most common self-inflicted React bug (the full anti-pattern catalog lives in [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)):

```tsx
// ❌ Two sources of truth, synced by hand — and an extra render per change
const [products, setProducts] = useState<Product[]>([]);
const [visibleCount, setVisibleCount] = useState(0);
useEffect(() => {
  setVisibleCount(products.filter((p) => p.inStock).length);
}, [products]);

// ✅ One source of truth, one projection
const [products, setProducts] = useState<Product[]>([]);
const visibleCount = products.filter((p) => p.inStock).length;
```

"But the computation is expensive" is the usual objection, and the Compiler-era answer is: write it plainly first; the Compiler caches it when inputs haven't changed, and the profiler will tell you if a genuine hotspot remains ([`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md)).

### Model states, not booleans

Enumerate the states your UI can actually be in, and store *which one you're in* — not a pile of flags that can combine into nonsense:

```tsx
// ❌ 2⁴ = 16 combinations, ~4 valid, 0 compiler help
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isEmpty, setIsEmpty] = useState(false);
const [hasResults, setHasResults] = useState(false);

// ✅ Exactly the states that exist, and TypeScript enforces it
type SearchState =
  | { status: 'idle' }
  | { status: 'loading'; query: string }
  | { status: 'success'; query: string; results: Product[] }
  | { status: 'error'; query: string; message: string };

const [search, setSearch] = useState<SearchState>({ status: 'idle' });
```

With the discriminated union, `search.results` doesn't even *exist* unless `status === 'success'` — the "loading spinner and error toast showing at the same time" bug becomes uncompilable. Every transition is one `setSearch` with a whole consistent object, instead of four setter calls that must never be half-applied.

### Colocate state; lift only when shared

State belongs in the lowest component that needs it. Every level you lift it "for later" widens the subtree that re-renders per change and couples components that didn't need coupling. Lift when two components genuinely render from the same data — and no further than their lowest common parent. This single habit prevents most of the re-render problems people later reach for memoization to solve, and it's the backbone of the performance recipe track.

### Handlers carry intent, not mechanics

Name callback props for what the user *did*, not for the state change you happen to make today: `onAddToCart`, not `onSetCartOpen`. The child expresses domain intent; the parent owns the mechanics and can change them (open a drawer today, fire a toast tomorrow, batch an analytics event next quarter) without touching the child's contract. Mechanical names leak the parent's implementation into every consumer and turn refactors into cross-file surgery.

## Common mistakes

**Mutating state in place.** React decides whether anything changed with a reference comparison (`Object.is`). Mutating an array or object keeps the reference identical, so React sees "no change" and skips the render entirely:

```tsx
// ❌ Same array reference — bailout, nothing renders
items.push(newItem);
setItems(items);

// ✅ New reference
setItems([...items, newItem]);
```

**Expecting a state setter to update the variable immediately.** `query` is a snapshot for the current render; `setQuery` schedules the *next* one. Code that sets state and reads the variable on the next line is reading the old snapshot — as the click trace showed step by step. (Full mechanics, including updater functions: [`../state/state-and-usestate.md`](../state/state-and-usestate.md).)

**Storing derived data and syncing it with an effect.** The tell is `useEffect` + `setState` where the effect's only inputs are other state/props. Delete the state, compute during render — every time.

**Touching the DOM behind React's back.** `document.querySelector(...).classList.add(...)` works until React's next commit overwrites it — or worse, doesn't, and the screen now shows something no state can explain. If the change matters, it belongs in state; if you need direct DOM access (focus, measurement, third-party widgets), that's what refs are for ([`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md)).

**Side effects during render.** Fetching, subscribing, writing to storage, or mutating module-level variables in the component body breaks the contract that renders are discardable. StrictMode's double-invoke exposes it immediately — twice the fetches, twice the log lines — which is a gift, not a bug report.

**Copying props into state.** `useState(props.initialValue)` reads the prop *once*, on mount; later prop changes are silently ignored. Sometimes that's exactly right (a true "initial" value — name the prop `initialX` so the contract is visible). If the component should follow the prop, render from the prop directly, no state at all.

**Treating re-renders as the enemy.** Wrapping everything in manual memoization before measuring optimizes the cheap phase while ignoring the model: pure, fast renders are the design, and the Compiler handles memoization mechanically. Profiler first, `memo` later, if ever.

## How this evolved

The mental model — UI as a function of state — has been React's pitch since 2013. What changed is how literally the machinery honors it:

| Era | What changed | What it did to the model |
| --- | --- | --- |
| 2013–2016 | Stack reconciler, class components | The model exists, but rendering is synchronous and uninterruptible; "UI = f(state)" is true with lifecycle-method footnotes |
| React 16 (2017) | Fiber rewrite | Rendering becomes incremental work that can be scheduled — the prerequisite for everything after |
| React 16.8 (2019) | Hooks | Components become *actual functions* — state and lifecycle expressed as function calls; `f` in `UI = f(state)` stops being a metaphor |
| React 18 (2022) | Concurrent rendering, automatic batching | Renders become interruptible and discardable, so purity graduates from best practice to load-bearing requirement |
| React 19 (2024) | Actions, `use()`, ref-as-prop | Async mutations and resources join the declarative model instead of living beside it |
| React Compiler 1.0 (2025) | Automatic memoization | The purity contract becomes machine-checked and machine-exploited — write the naive pure version, the Compiler makes it fast |

Each step made the original idea *more* true, not different. Code written against the mental model in 2015 still reads correctly today; code written against the implementation details of any single era did not age as well. That's the argument for learning the model first — it's the stable layer.

## Exercises

### 1. Minimal state: the temperature converter

Build two inputs — Celsius and Fahrenheit — that stay in sync: type in either, the other updates live. Decide the minimal state *before* writing code.

*Hint: it's one piece of state, not two. Store `{ value: string; scale: 'c' | 'f' }` — the raw text the user typed plus which box they typed it in — and derive the other box's display during render. The tempting two-states-plus-effect version desynchronizes on rounding (2.22 → 36.0 → 2.2…) and fires an extra render per keystroke; the single-source version can't.*

### 2. Kill the boolean soup

You inherit this from a form component:

```tsx
const [isSaving, setIsSaving] = useState(false);
const [isSaved, setIsSaved] = useState(false);
const [saveError, setSaveError] = useState<string | null>(null);
```

Refactor it into a single discriminated union so that "saving *and* saved" and "saved *with* an error" become uncompilable. Write the type and every transition (`start`, `succeed`, `fail`, `reset`).

*Hint: `type SaveState = { status: 'idle' } | { status: 'saving' } | { status: 'saved' } | { status: 'error'; message: string }` — then notice each transition is one `setSave({...})` call that can't be half-applied.*

### 3. Bug hunt: the list that won't update

```tsx
function handleAdd(item: Item) {
  items.push(item);
  setItems(items);
}
```

Explain *precisely* why the screen never changes — which step of the render loop bails, and on what comparison — then fix it two ways.

*Hint: `Object.is(items, items)` is `true`, so React skips the render at the trigger step. Fix with a new reference (`setItems([...items, item])`) or, better for updates based on previous state, the updater form (`setItems(prev => [...prev, item])`) — the distinction matters and is covered in [`../state/state-and-usestate.md`](../state/state-and-usestate.md).*

## Summary

You learned React's core model and the machinery that honors it: components are pure functions returning element descriptions; state is the single source of truth and the DOM is its projection; every interaction runs the trigger → render → commit loop, with state living on persistent fibers rather than in your closures. Purity is what lets React double-invoke, interrupt, discard, and auto-memoize your renders. The five-step workflow (hierarchy → static version → minimal state → state placement → inverse flow) turns the model into a design procedure, and the recurring disciplines — derive don't store, model states as unions, colocate state, name handlers for intent — are the model applied. Everything else in this resource is a deepening of some corner of this article.

## See also

- [`jsx-and-rendering.md`](jsx-and-rendering.md) — what those element descriptions physically are
- [`../state/state-and-usestate.md`](../state/state-and-usestate.md) — snapshots, batching, updater functions
- [`../rendering/how-react-renders.md`](../rendering/how-react-renders.md) — fiber, phases, lanes: the full machinery
- [`rules-of-react.md`](rules-of-react.md) — the purity contract, precisely stated
- [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md) — reconciliation from the user's seat
- [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) — what effects are actually for (and the derived-state anti-pattern in full)

## References

- [Thinking in React (react.dev)](https://react.dev/learn/thinking-in-react)
- [Render and Commit (react.dev)](https://react.dev/learn/render-and-commit)
- [State as a Snapshot (react.dev)](https://react.dev/learn/state-as-a-snapshot)
- [Keeping Components Pure (react.dev)](https://react.dev/learn/keeping-components-pure)
- [You Might Not Need an Effect (react.dev)](https://react.dev/learn/you-might-not-need-an-effect)
- [Choosing the State Structure (react.dev)](https://react.dev/learn/choosing-the-state-structure)
- [`StrictMode` API (react.dev)](https://react.dev/reference/react/StrictMode)
- [React Components, Elements, and Instances — Dan Abramov](https://legacy.reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.