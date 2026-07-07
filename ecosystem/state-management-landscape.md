---
article_id: state-management-landscape
concept_folder: ecosystem
wave: 4
related:
  - state/context
  - ecosystem/data-fetching-tanstack-query
  - state/usereducer-and-state-structure
  - concurrent/actions
  - rendering/how-react-renders
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this:** "Which state management library should I use?" is almost always the wrong first question. The right one is "what *kind* of state is this?" — because the state-placement rule already routes most of it away from any library at all. Server data goes to a query cache, form-shaped mutations go to Actions, static config goes to context, and local UI state stays in `useState`. What's left after those four take their share is a small residue — cross-cutting *client* state — and *that* is the only thing Zustand, Jotai, and Redux are competing for. This article is a decision table for that residue, not a tour of each library.

## What it is

### The placement funnel

State management is a **categorization** problem wearing a library problem's clothes. Before you compare tools, run the placement funnel — four questions, in order, that drain most "global state" away:

1. **Is it server data** (fetched, owned by a backend, can change without you)? → a query cache: [`TanStack Query`](./data-fetching-tanstack-query.md) (or RTK Query in a Redux shop). Never a client store. This one question removes 60–80% of what teams reach for Redux to hold.
2. **Is it a form-shaped mutation** (submit, pending, optimistic, rollback)? → [`Actions`](../concurrent/actions.md) (`useActionState`/`useOptimistic`).
3. **Is it static-ish config** (theme tokens, locale, the current user object, feature flags) that rarely changes? → [`context`](../state/context.md), which is transport, not a store.
4. **Is it local UI state** (this input's value, this accordion's open flag)? → `useState`/`useReducer`, colocated.

Only if the answer to all four is *no* do you actually have a store question. What survives the funnel is **cross-cutting client state**: client-owned (no server is the source of truth), read and written by *many distant* components (so lifting-and-passing is impractical), and changing often enough that context would re-render too much. A shopping cart's *client-side* draft before checkout, a multi-step wizard's shared state, canvas/editor selection, a client-only notification queue. That residue is smaller than people expect, and it's the entire subject of the libraries below.

## How it works under the hood

The libraries differ on *lots* of surface API, but there's exactly **one mechanical axis** that matters, and it's the reason they exist instead of context: **subscription granularity** — how narrowly a component can subscribe so it re-renders only when *its* slice changes.

[`context`](../state/context.md) has none. A Provider value change marks *every* consumer dirty and re-renders all of them, piercing the bailout check that would otherwise stop propagation (traced in [`how-react-renders`](../rendering/how-react-renders.md#bailouts-the-fast-paths)). Value identity is the whole ballgame; split-context is the manual escape, and it doesn't scale past a few axes. That limit — one value, all-or-nothing — is exactly what the [`context-rerenders-the-whole-tree` recipe](../recipes/state-management/context-rerenders-the-whole-tree.md) diagnoses.

The stores solve it by living **outside React** and exposing a selector:

- **Zustand** is one external store read through `useSyncExternalStore`. A component calls `useStore(selector)` and subscribes to the *selected slice*; when the store changes, each subscriber re-runs its selector and re-renders only if the result changed by the equality check. Re-renders are **selector-scoped, not store-wide** — the structural fix the megacontext recipe reaches for, built in.
- **Jotai** inverts it: state is split into **atoms**, and a component subscribes to specific atoms. Updating one atom notifies only that atom's dependents. Granularity is per-atom, bottom-up — "increment this counter, only this counter re-renders," no selector needed because the atom *is* the subscription unit.
- **Redux Toolkit** is one store read through `useSelector`, memoized selectors doing the slice-scoping, with the Flux discipline (actions describe what happened, reducers compute the next state) layered on top for traceability.

All three sit on `useSyncExternalStore`, which is also *why* they don't tear under concurrent rendering — the tearing problem and that hook's invariants are [`escape-hatches-audit`](../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely)'s territory. Context, being in-tree, never had a tearing problem to solve; the stores had to, and did.

That's the whole mechanical story. Everything else — middleware, devtools, persistence — is convenience on top of "subscribe to a slice of an external store."

## Minimal shapes

Just enough of each to recognize it — not a tutorial.

**Zustand** — one store, selector reads:

```tsx
import { create } from "zustand";
import { useShallow } from "zustand/react/shallow";

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  clear: () => void;
}

const useCartStore = create<CartStore>((set) => ({
  items: [],
  addItem: (item) => set((s) => ({ items: [...s.items, item] })),
  clear: () => set({ items: [] }),
}));

const count = useCartStore((s) => s.items.length); // atomic pick: strict equality, fine

// v5 GOTCHA: an object/array selector WITHOUT useShallow now *crashes* with
// "Maximum update depth exceeded" — it's a render error, not just extra renders.
const { addItem, clear } = useCartStore(useShallow((s) => ({ addItem: s.addItem, clear: s.clear })));
```

**Jotai** — atoms, subscribe per atom:

```tsx
import { atom, useAtom, useAtomValue } from "jotai";

const cartAtom = atom<CartItem[]>([]);
const cartCountAtom = atom((get) => get(cartAtom).length); // derived, read-only

function CartBadge() {
  const count = useAtomValue(cartCountAtom); // re-renders only when the count changes
  return <span>{count}</span>;
}
```

**Redux Toolkit** — slice + selector + dispatch:

```tsx
import { createSlice, configureStore, type PayloadAction } from "@reduxjs/toolkit";
import { useSelector, useDispatch } from "react-redux";

const cartSlice = createSlice({
  name: "cart",
  initialState: { items: [] as CartItem[] },
  reducers: {
    addItem: (state, action: PayloadAction<CartItem>) => {
      state.items.push(action.payload); // Immer makes the "mutation" immutable
    },
  },
});

export const store = configureStore({ reducer: { cart: cartSlice.reducer } });
// in a component:
const count = useSelector((s: RootState) => s.cart.items.length);
const dispatch = useDispatch();
// dispatch(cartSlice.actions.addItem(item));
```

Notice the shared skeleton under three syntaxes: define state, subscribe to a slice, dispatch a change. The differences are ergonomics and ceremony, not capability.

## The decision table

Reach for the **lowest** row that fits. Server state is deliberately *not* in this table — it never belongs in a client store.

| Tool | Mental model | Re-render unit | Reach for it when | Avoid when |
| --- | --- | --- | --- | --- |
| `useState` / `useReducer` | local, colocated | the component | the state has one owner and a small blast radius | many distant components need it |
| `context` | transport for a value | **all consumers** | static-ish config: theme, locale, current user, DI seams | the value changes often (re-renders everything) |
| **Zustand** | one external store, selector reads | the selected slice | the default cross-cutting client store; minimal ceremony, no provider | you want fine-grained derived graphs, or strict Flux auditing |
| **Jotai** | bottom-up atoms | the atom's dependents | fine-grained/derived-heavy state; per-item independence (grids, editors) | a few coarse global values (atoms are overkill) |
| **Redux Toolkit** | single store, actions → reducers | the selected slice | large teams, strict action-log/time-travel, existing Redux, middleware-heavy flows | a small app that just needs a shared value (boilerplate tax) |

Two rules of thumb make the table concrete: **most apps need zero rows below `context`** once server state is on Query; and when you *do* need a store, **Zustand is the low-ceremony default**, Jotai wins when the state is naturally atomic and derived-heavy, and Redux Toolkit earns its weight on large teams that value the strict action log and DevTools time-travel — not on "it's the industry standard."

## Walkthrough — routing one screen's state

A product dashboard had accumulated *all* its state in one megacontext: the product list, the logged-in user, the theme, the active filters, a cart draft, and the search box value. Every keystroke in search re-rendered all 200 product cards. We fix it not by optimizing the context but by running the funnel and sending each piece to a home whose re-render model fits its change cadence.

**Step 1 — Server data leaves for the query cache.** Products and the user are server-owned, so they were never client state to begin with:

```tsx
// queries.ts
export const productsQuery = (filters: Filters) =>
  queryOptions({ queryKey: ["products", filters], queryFn: ({ signal }) => fetchProducts(filters, signal) });
export const meQuery = queryOptions({ queryKey: ["me"], queryFn: fetchMe });
```

```tsx
function ProductList({ filters }: { filters: Filters }) {
  const { data } = useSuspenseQuery(productsQuery(filters));
  return <>{data.map((p) => <ProductCard key={p.id} product={p} />)}</>;
}
```

Two categories gone, and the list now refetches itself when the filter key changes — no manual sync.

**Step 2 — Config goes to context.** Theme changes rarely, so context's re-render-all consumers is a non-issue here — this is exactly what context is *for*:

```tsx
const ThemeContext = createContext<Theme>("light"); // <ThemeContext value={theme}> at the root
```

**Step 3 — The client residue goes to a selector store.** Filters and the cart draft are client-owned, read/written by many components, and change often — the genuine store case:

```tsx
interface DashboardStore {
  filters: Filters;
  cart: CartItem[];
  setFilters: (f: Filters) => void;
  addToCart: (item: CartItem) => void;
}

const useDashboard = create<DashboardStore>((set) => ({
  filters: defaultFilters,
  cart: [],
  setFilters: (filters) => set({ filters }),
  addToCart: (item) => set((s) => ({ cart: [...s.cart, item] })),
}));
```

Each component subscribes to just its slice, so changes don't cross-contaminate:

```tsx
function CartBadge() {
  const count = useDashboard((s) => s.cart.length); // re-renders only when cart length changes
  return <span>{count}</span>;
}

function FilterBar() {
  const filters = useDashboard((s) => s.filters);
  const setFilters = useDashboard((s) => s.setFilters);
  // changing filters re-renders FilterBar and re-keys ProductList's query — NOT CartBadge
  return <FilterControls value={filters} onChange={setFilters} />;
}
```

**Step 4 — Local stays local.** The search input's value is nobody else's business until it's committed:

```tsx
function Search({ onCommit }: { onCommit: (q: string) => void }) {
  const [q, setQ] = useState("");
  // debounce, then onCommit(q) → which calls setFilters on the store
  return <input value={q} onChange={(e) => setQ(e.target.value)} />;
}
```

The payoff, per interaction: a keystroke re-renders `Search` alone; committing a filter re-renders `FilterBar` and re-keys the products query; the cart badge re-renders only when the cart changes; the rare theme change re-renders its consumers harmlessly. The 200-card storm is gone — not because anything was memoized, but because each piece landed in a home whose re-render model matches how often it changes. That's the entire value of the funnel, made concrete.

## Real-world patterns

**Walk a real decision.** "We have a cart — put it in Zustand?" First run the funnel. If the cart is *server-persisted* (survives logout, syncs across devices), it's **server state** — it goes in the query cache, and putting it in Zustand is the cardinal miscategorization that makes it go stale (the `zustand-goes-stale` recipe, *planned*). If it's a *client-only* draft the user assembles before a checkout mutation, *then* it's cross-cutting client state and Zustand fits. Same noun, opposite answer, decided entirely by "who owns the source of truth."

**Server state in a Redux shop: RTK Query, not a slice.** "Server state → Query" doesn't mandate *TanStack* Query. If you're already on Redux Toolkit, **RTK Query** (`createApi`) is a server-state cache using the same store, middleware, and DevTools — the path of least resistance, and it keeps you from running two caches. Its `skipToken` is even type-safe in a way TanStack's `enabled` isn't. The rule is "server state belongs in *a* cache, never a hand-written slice or a client store" — [`data-fetching-tanstack-query`](./data-fetching-tanstack-query.md) is the standalone default; RTK Query is the same idea inside Redux. Pick one cache; never both.

**The escalation ladder from context.** The [`context-rerenders-the-whole-tree` recipe](../recipes/state-management/context-rerenders-the-whole-tree.md) ends at a structural ceiling: split-context buys you a few axes, then you're out of runway. A selector store *is* the next rung — Zustand's selector-scoped reads are precisely the "component subscribes to its slice" behavior context can't express. When you find yourself splitting a context into five, you've outgrown context; that's the signal, not a number.

**Derived state, and the Zustand-vs-Jotai call.** The table sends derived-heavy state to Jotai — here's the mechanism behind that. Computed values work differently per model: Zustand derives inside a selector (`useStore((s) => expensive(s.items))`, memoized if costly); Redux uses memoized selectors (`createSelector`/reselect); Jotai makes derivation *first-class* — a derived atom (`atom((get) => get(cartAtom).length)`) is itself a subscribable node, so a component reading it re-renders only when *that derived value* changes, and the dependency graph is tracked for you. The atomic model earns its keep when state is naturally *many independent pieces with derived relationships*: a spreadsheet where each cell is an atom (edit one cell → only it and its dependents re-render, no selector plumbing), a form builder, a canvas editor. For a handful of cohesive domains driven by actions, Zustand's single store is less ceremony. One caution that keeps the spine intact: if you find yourself writing async atoms or async store actions, stop and check — async data is almost always *server* state, which belongs in the query cache, not a client store.

**Outside-React access.** A genuine store advantage: Zustand's `getState()`/`subscribe()` and Redux's `store.dispatch` work *outside* the component tree — event handlers on `window`, a WebSocket message handler, an interceptor. Context can't be read outside a component. If non-React code needs to read or write the state, that alone can justify a store.

**Persistence, with the hydration caveat.** A frequent reason to reach for a store is surviving reloads — a cart draft, theme, or filters kept in `localStorage`. Both libraries have first-class support: Zustand's `persist` middleware wraps the store, Jotai's `atomWithStorage` wraps an atom:

```tsx
import { persist } from "zustand/middleware";

const useDashboard = create<DashboardStore>()(
  persist(
    (set) => ({ /* ...store as before... */ }),
    { name: "dashboard" }, // localStorage key
  ),
);
```

The caveat is SSR/hydration: the server renders with the default state (it has no `localStorage`), the client rehydrates from storage, and a naive read during render mismatches — the `getServerSnapshot`/two-pass problem from [`ssr-and-hydration`](../server/ssr-and-hydration.md). `persist` exposes `onRehydrateStorage`/`hasHydrated` to gate rendering until the store is ready; in an SSR app you render the persisted slice only after hydration. Client-only SPAs (this project's baseline) sidestep it, but know it's there before you take the store to a framework.

## Common mistakes

1. **Server state in a client store.** Caching `products`/`user`/`cart` in Zustand or a Redux slice re-signs the freshness promise you don't want — now *you* own invalidation and it goes stale. This is *the* architecture mistake every source names. Server state → a query cache, always.
2. **Reaching for a library before running the funnel.** Most "we need global state" turns out to be server state (→ Query) plus a bit of config (→ context). Apply the four questions first; you often need no store at all.
3. **Zustand v5 object selector without `useShallow`.** `useStore((s) => ({ a: s.a, b: s.b }))` returns a new object each call; in v5 that no longer just adds renders — it **crashes** with "Maximum update depth exceeded." Wrap multi-value selects in `useShallow`, or pick atomically.
4. **Frequently-changing state in context.** Context re-renders every consumer on any change; a value that ticks (cart, live filters) will jank the tree — the megacontext recipe's whole subject. That's the boundary where a selector store starts paying off.
5. **Redux "because it's the standard."** On a small app, the store/slice/provider/DevTools setup and the actions-and-reducers mental model are pure tax with no payoff. RTK removed ~60% of Redux's boilerplate, but zero boilerplate still beats a little.
6. **Two server-state caches.** Running RTK Query *and* TanStack Query means two caches, two mental models, two sources of truth to keep in sync. Pick one per app.
7. **Global-everything.** Putting an input's local value in a global store because the store is right there throws away colocation and turns every keystroke into a store update. Local state stays local; the store is for the residue.
8. **Selecting the whole store.** `useStore((s) => s)` (or an unselected `useSelector((s) => s)`) subscribes to *everything* and re-renders on every change — you've turned your selector store back into a megacontext. Select the narrowest slice you use.
9. **Jotai atoms created in render.** An `atom(...)` created inside a component body has a new identity each render; without `useMemo`/`useRef` it causes an infinite loop with `useAtom`. Define atoms at module scope (or memoize dynamic ones).
10. **Mirroring server/props into a store, synced by an effect.** `useEffect(() => store.setState(query.data))` re-creates the staleness and duplication the placement rule exists to prevent. Read from the cache directly; derive, don't mirror — the same rule from [`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md#derive-vs-mirror--the-deep-dive), at store scale.

## How this evolved

The through-line is **decomposition**. Flux (2014) and then Redux (2015) offered one answer to all shared state — a single store, actions, reducers — and for years "state management" *meant* Redux, boilerplate and all. Two shifts broke that monopoly. First, **server state got its own category**: React Query / SWR observed that most of what Redux held was cached backend data with entirely different needs (staleness, refetching, dedup), and pulled 60–80% of it out into a query cache — the split [`data-fetching-tanstack-query`](./data-fetching-tanstack-query.md) is built on. Second, with server state gone, the *client* residue was small enough that Redux's ceremony felt disproportionate, so **lightweight stores** (Zustand, Jotai, Valtio) filled the shrunken niche, and **Redux Toolkit** (with RTK Query) modernized Redux itself — `createSlice`/`configureStore` killing the boilerplate, RTK Query absorbing the server-state pattern back in for teams that want one ecosystem. Today "state management" isn't one thing: a query cache owns server state, a small client store owns the cross-cutting residue, context owns config, `useState` owns the local. The right answer is usually *several* tools, each holding only what it's shaped for — and RSC ([`server-components`](../server/server-components.md)) trims the client residue further by rendering more on the server to begin with.

## Exercises

1. **Categorize before you build.** Sort these into the four placement buckets: the logged-in user's profile (from `/api/me`), the theme toggle, a comment form's draft text, the list of products on screen, a modal's open flag, the active table filter. *Hint: three of the six never touch a store; one is a trap that looks like client state but is server state.*
2. **Context → store.** Take a megacontext bundling three unrelated client values and move it to a Zustand store; prove with the Profiler that a component reading one slice no longer re-renders when another slice changes. *Hint: selector-scoped subscriptions — and wrap any multi-value select in `useShallow`.*
3. **Justify or reject a store.** A teammate wants to "add Redux for the checkout flow." Decide, item by item, which pieces of that flow actually need a store and which are miscategorized server state or local form state. *Hint: run the funnel on each piece; the cart total from the server and the card-number field are not the same category.*

## Summary

- "Which library?" is downstream of "which *kind* of state?" Run the placement funnel first: server → query cache, form-shaped → Actions, config → context, local → `useState`. The store question is only about what survives.
- What survives is *cross-cutting client state* — client-owned, multi-consumer, frequently-changing. That residue is small, and it's all the libraries contest.
- The one mechanical axis that matters is subscription granularity: context re-renders all consumers; Zustand/Redux scope by selector; Jotai scopes by atom. All three ride `useSyncExternalStore` and so don't tear.
- Decision: Zustand is the low-ceremony default store; Jotai for atomic/derived-heavy state; Redux Toolkit for large teams, strict action logs, and existing Redux. Server state is never in this table.
- "Server state → Query" allows RTK Query in a Redux shop — same idea, same ecosystem — but never run two caches.
- The cardinal mistake is server state in a client store; the second is reaching for any store before the funnel says you need one.

## See also

- [`context`](../state/context.md) — transport-not-store, value-identity, split-context; the ceiling the stores exist to raise
- [`data-fetching-tanstack-query`](./data-fetching-tanstack-query.md) — the server-state category the funnel routes away first
- [`context-rerenders-the-whole-tree` recipe](../recipes/state-management/context-rerenders-the-whole-tree.md) — the concrete failure that ends at "escalate to a selector store"
- [`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md) — local complex client state, and derive-don't-mirror at scale
- [`actions`](../concurrent/actions.md) — form-shaped mutations, the category the funnel routes to Actions
- [`escape-hatches-audit`](../effects/escape-hatches-audit.md) — `useSyncExternalStore` and the tearing problem every store rides on

## References

- Zustand — [Migrating to v5](https://zustand.docs.pmnd.rs/reference/migrations/migrating-to-v5) (the `useShallow` / equality-fn change) and [README](https://github.com/pmndrs/zustand)
- Jotai — [Core `atom`](https://jotai.org/docs/core/atom) and [v2 API migration](https://jotai.org/docs/guides/migrating-to-v2-api)
- Redux Toolkit — [Overview](https://redux-toolkit.js.org/introduction/getting-started) and [RTK Query Overview](https://redux-toolkit.js.org/rtk-query/overview)
- React — [`useSyncExternalStore`](https://react.dev/reference/react/useSyncExternalStore)

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).