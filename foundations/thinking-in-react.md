---
article_id: thinking-in-react
concept_folder: foundations
related:
  - jsx-and-rendering
  - state-and-usestate
  - how-react-renders
  - rules-of-react
react_baseline: "19.2"
---

# Thinking in React

> **Lead with this:** React's core bet is that your UI is a function of your state: `UI = f(state)`. You never edit the screen — you edit data, and React re-projects the screen from it. A component is not a thing on the page; it's a function that *describes* what the page should look like for the current data, and React's job is to make the real DOM match that description as cheaply as possible. Every React API — hooks, Suspense, Actions, the Compiler — falls out of this one idea, and the majority of React bugs come from fighting it: mutating data in place, touching the DOM behind React's back, or storing things that should be computed.

## What it is

React is a **declarative** UI library. You describe each possible state of the interface once, and React figures out the transitions between them. The alternative — the imperative model — is what UI code looks like without it: find a node, toggle a class, update a counter label, remember to also update the badge in the header, remember to undo all of it when the user clears the filter. With N pieces of state and M places they show up, imperative code has to hand-write every combination. Declarative code writes N descriptions and lets the library handle the M×N bookkeeping.

Three ideas make the model work:

**Components are functions that return descriptions.** A component takes props, reads state, and returns React *elements* — lightweight objects describing what should exist. It does not create DOM nodes and does not return them. Calling the same component twice with the same inputs must produce the same description; that purity requirement is not a style preference, it's what the entire machine is built on (see [`rules-of-react.md`](rules-of-react.md)).

**State is the source of truth; the DOM is a projection.** If something on screen can change, there is a piece of state it derives from. The rendered output is disposable — React can throw it away and recompute it at any time, because your state holds everything needed to rebuild it.

**Data flows one way.** Props flow down the tree; changes flow up through callbacks. A child never reaches over and edits its parent's data — it asks the parent to change it, the parent updates state, and the new data flows back down. This is what makes a large React tree debuggable: to find why something rendered wrong, you walk *up* to the state that produced it, never sideways.

## How it works under the hood

### What a component actually returns

JSX compiles to plain function calls that produce plain objects. This:

```tsx
<button className="primary" onClick={handleSave}>
  Save
</button>
```

becomes, roughly:

```ts
{
  type: 'button',
  props: {
    className: 'primary',
    onClick: handleSave,
    children: 'Save',
  },
}
```

That object is a **React element** — a cheap, immutable description. Creating thousands of them per render costs almost nothing; they're just objects. No DOM has been touched yet. (The exact compile output and the element/component/instance distinction get the full treatment in [`jsx-and-rendering.md`](jsx-and-rendering.md).)

### The render loop

Everything React does is one loop with three steps:

**1. Trigger.** There are exactly two triggers: the initial mount, and a state update. Calling a state setter does not re-run anything on the spot — it *schedules* a render. (This is why reading state right after setting it gives you the old value; state behaves like a snapshot per render — full mechanics in [`../state/state-and-usestate.md`](../state/state-and-usestate.md).)

**2. Render.** React calls your component functions, top-down from wherever the update originated, and collects the element tree they return. "Render" in React vocabulary means *calling your functions* — nothing more. No DOM work happens in this phase. That's also why this phase is allowed to be interrupted, thrown away, or run twice: it's supposed to have no observable side effects.

**3. Commit.** React diffs the new element tree against the previous one (reconciliation — it compares by element `type` and `key`, which is why keys matter so much in lists; see [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md)) and computes the minimal set of real DOM mutations. Then it applies them in one synchronous pass, and the browser paints.

The performance implication is the part most people get backwards: **renders are cheap, commits are expensive — and React already minimizes commits for you.** A component re-rendering 30 times that produces the same description each time results in zero DOM work. When React apps feel slow, the cause is almost never "too many renders" in the abstract; it's specific expensive work happening *during* renders, or commits thrashing layout. Which is why the fix is rarely "prevent renders" and usually "make renders pure and cheap" — the stance the React Compiler now automates (see [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md)).

### What purity buys

Because the render phase is a pure computation over state, React is free to:

- **Run it twice in development.** `<StrictMode>` deliberately double-invokes renders to surface impurity early. If your component misbehaves under StrictMode, the component is wrong, not StrictMode.
- **Interrupt and discard it.** Concurrent rendering (React 18+) can pause a render mid-tree to handle something urgent, then restart. A render that mutated something outside itself can't safely be discarded — pure ones can.
- **Skip it entirely.** The React Compiler memoizes components and values automatically, on the assumption that same inputs → same output. The purity rules are the contract that makes this legal.

This is the deepest version of the mental model: your components describe; React decides *when*, *whether*, and *how often* to ask for the description.

## Basic usage

A minimal but complete example of the model — state, a derived projection, and an event flowing back up:

```tsx
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

Read it through the model:

- **One piece of state** (`query`). The filtered list and the "Showing X of Y" count are both *projections* of it, computed fresh each render. They cannot drift apart, because neither is stored.
- **The event handler doesn't update the UI.** It updates state. The input's displayed value, the list, and the count all follow automatically on the next render.
- **Nothing reads or writes the DOM.** No `querySelector`, no manual class toggling, no "remember to also update the counter." The description is the whole job.

## Real-world patterns

### The five-step design workflow

For anything bigger than one component, this sequence keeps you from designing state backwards:

1. **Break the mock into a component hierarchy.** One component per thing that has a single responsibility or repeats.
2. **Build the static version first — props only, no state.** Render the whole tree from hardcoded data top-down. If you can't, your hierarchy is wrong; fix it now while it's cheap.
3. **Find the minimal state.** For every piece of data ask: does it change over time? Is it passed in via props? Can it be computed from something else? Only "changes over time, not passed in, not computable" survives as state. In the `ProductList` above, `query` is state; `visible` and the count are not.
4. **Decide where each piece of state lives.** Find every component that renders from it, then put the state in their lowest common parent. Not higher "just in case."
5. **Add inverse data flow.** Pass handlers down so children can request changes to state they don't own.

Steps 3 and 4 are where real apps go wrong, which is why they get their own deep dives ([`../state/usereducer-and-state-structure.md`](../state/usereducer-and-state-structure.md)).

### Derive, don't store

If a value can be computed from existing state or props, compute it during render. Storing it creates a second source of truth that you now have to keep synchronized — usually with an effect, which is the single most common self-inflicted React bug (the full anti-pattern catalog lives in [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)):

```tsx
// ❌ Two sources of truth, synced by hand
const [products, setProducts] = useState<Product[]>([]);
const [visibleCount, setVisibleCount] = useState(0);
useEffect(() => {
  setVisibleCount(products.filter((p) => p.inStock).length);
}, [products]);

// ✅ One source of truth, one projection
const [products, setProducts] = useState<Product[]>([]);
const visibleCount = products.filter((p) => p.inStock).length;
```

The second version is shorter, cannot desynchronize, and renders once instead of twice per change.

### Model states, not booleans

Enumerate the states your UI can actually be in, and store *which one you're in* — not a pile of flags that can combine into nonsense:

```tsx
// ❌ 2⁴ = 16 combinations, most of them impossible
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

With the discriminated union, `search.results` doesn't even *exist* unless `status === 'success'` — a whole category of "loading spinner and error toast showing at the same time" bugs becomes uncompilable.

### Colocate state; lift only when shared

State should live in the lowest component that needs it. Every level you lift it "for later" widens the subtree that re-renders on each change and couples components that didn't need coupling. Lift when two components genuinely render from the same data — and no further than their lowest common parent. (This one habit prevents most of the re-render problems people reach for memoization to solve.)

## Common mistakes

**Mutating state in place.** React decides whether anything changed by comparing references. Mutating an array or object keeps the reference identical, so React sees "no change" and skips the render:

```tsx
// ❌ Same array reference — no re-render
items.push(newItem);
setItems(items);

// ✅ New reference
setItems([...items, newItem]);
```

**Expecting a state setter to update the variable immediately.** `query` is a snapshot for the current render; `setQuery` schedules the *next* render. Code that sets state and reads it on the next line is reading the old snapshot. (Full mechanics: [`../state/state-and-usestate.md`](../state/state-and-usestate.md).)

**Storing derived data and syncing it with an effect.** Covered above in *Derive, don't store* — it deserves its spot in both lists because it is that common. The tell is `useEffect` + `setState` where the effect's only inputs are other state/props.

**Touching the DOM behind React's back.** `document.querySelector(...).classList.add(...)` works until React's next commit overwrites it — or worse, doesn't, and now the screen shows something no state can explain. If you need the change, it belongs in state; if you need direct DOM access (focus, measurement, third-party widgets), that's what refs are for ([`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md)).

**Side effects during render.** Fetching, subscribing, writing to storage, or mutating anything in the component body breaks the contract that renders are discardable. StrictMode's double-invoke will expose it in development — twice the fetches, twice the log lines. Effects belong in event handlers or `useEffect`, in that order of preference.

**Copying props into state.** `useState(props.initialValue)` reads the prop *once*, on mount; later prop changes are silently ignored. Sometimes that's exactly what you want (a true "initial" value) — but if the component should follow the prop, render from the prop directly, no state at all.

**Treating re-renders as the enemy.** Wrapping everything in manual memoization before measuring anything optimizes the cheap phase (render) while ignoring the model: pure, fast renders are the design, and the Compiler now handles the memoization mechanically. Reach for the profiler before reaching for `memo` — and see [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md) for when manual control still earns its keep.

## How this evolved

The mental model — UI as a function of state — has been React's pitch since 2013. What changed is how literally the machinery honors it:

| Era | What changed | What it did to the model |
| --- | --- | --- |
| 2013–2016 | Stack reconciler, class components | The model exists, but rendering is synchronous and uninterruptible; "UI = f(state)" is true with lifecycle-method footnotes |
| React 16 (2017) | Fiber rewrite | Rendering becomes incremental work that can be scheduled — the prerequisite for everything after |
| React 16.8 (2019) | Hooks | Components become *actual functions* — state and lifecycle expressed as function calls instead of class ceremony; `f` in `UI = f(state)` stops being a metaphor |
| React 18 (2022) | Concurrent rendering, automatic batching | Renders become interruptible and discardable, so purity graduates from best practice to load-bearing requirement |
| React 19 (2024) | Actions, `use()`, ref-as-prop | Async mutations and resources join the declarative model instead of living beside it |
| React Compiler 1.0 (2025) | Automatic memoization | The purity contract becomes machine-checked and machine-exploited — write the naive pure version, the Compiler makes it fast |

Each step made the original idea *more* true, not different. Code written against the mental model in 2015 still reads correctly today; code written against the implementation details of any single era did not age as well. That's the argument for learning the model first — it's the stable layer.

## See also

- [`jsx-and-rendering.md`](jsx-and-rendering.md) — what JSX compiles to; elements vs. components vs. instances
- [`../state/state-and-usestate.md`](../state/state-and-usestate.md) — snapshots, batching, updater functions
- [`../rendering/how-react-renders.md`](../rendering/how-react-renders.md) — fiber, render/commit phases, the full machinery
- [`rules-of-react.md`](rules-of-react.md) — the purity contract, precisely stated
- [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md) — reconciliation from the user's seat
- [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) — what effects are actually for