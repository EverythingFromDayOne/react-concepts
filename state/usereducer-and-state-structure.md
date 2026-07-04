---
article_id: usereducer-and-state-structure
concept_folder: state
wave: 2
related:
  - state/state-and-usestate
  - effects/effects-and-synchronization
  - state/context
  - foundations/thinking-in-react
  - concurrent/actions
react_baseline: "19.2"
status: { drafted: true, reviewed: false }
---

# useReducer and State Structure

> **Lead with this:** `useState` scales fine until your state grows *rules* — fields that must move together, transitions that are only legal from certain statuses, one user event with four consequences. At that point the choice isn't really "useState vs useReducer"; it's *where the rules live*. Scattered across call sites and patched together with effects, they produce render cascades and impossible states. Centralized in a reducer — one pure function that folds events into next-states — they become testable, exhaustive, and enforceable. This article covers the reducer half and the deeper half underneath it: state **shape**. Most reducer pain is shape pain wearing a disguise, so derive-vs-mirror, normalization, and the smells table live here too.

## What it is

```tsx
const [state, dispatch] = useReducer(reducer, initialArg, init?);
```

- **`reducer(state, action) => nextState`** — a pure function. Given the current state and a description of *what happened*, it returns the next state. No fetches, no mutation, no reading clocks; same inputs, same output.
- **`dispatch(action)`** — how events enter. Its identity is **stable for the life of the component** — pass it anywhere, no memoization, ever.
- **`init?`** — optional lazy initializer: `useReducer(reducer, props.saved, createDraft)` runs `createDraft(props.saved)` once at mount, not per render.

When it beats `useState`, concretely: several values that update *together*; next state depending on previous state in non-trivial ways; updates dispatched from many call sites that must all obey the same rules; state you want to unit-test without rendering anything. When it doesn't: independent values with no rules — a reducer wrapping one boolean is ceremony.

The frame that makes reducers click: **actions are events, not setters.** `{ type: "filters-changed", filters }` describes what the user did; the *reducer* decides that this also resets the page and clears the selection. Write actions as `{ type: "set-page" }`, `{ type: "set-selection" }` and you've rebuilt `useState` with extra steps — the consequences scatter back to the call sites, and the whole point evaporates.

## How it works under the hood

**`useState` *is* `useReducer`.** Internally, `useState` mounts the same hook with a built-in reducer (`basicStateReducer`: "if the action is a function, call it with state; otherwise the action *is* the next state"). Everything [state-and-usestate](./state-and-usestate.md) traced — the hook node on `memoizedState`, the update queue, batching, snapshot semantics — applies verbatim. What `useReducer` changes is only *what sits in the queue*: **actions**, not values. Dispatch appends an action; when the render drains the queue, React folds the actions through your reducer in order to produce the next state. Three dispatches in one handler ([how-react-renders](../rendering/how-react-renders.md): same lane) → one render, reducer runs three times, one commit.

**Why `dispatch` is born stable.** It's bound to the fiber and its queue at mount, not recreated per render — the same reason `setState`'s setter is stable. This is the free lunch [context](./context.md) cashed in: an actions channel carrying `dispatch` never re-broadcasts, no `useMemo` involved.

**The eager-bailout wrinkle.** When a dispatch arrives and the fiber has no other pending work, React may run your reducer *immediately, at dispatch time*, to check whether the result is `Object.is`-identical to current state — if so, it skips scheduling a render entirely. Two consequences worth internalizing: returning the *same state reference* for a no-op action is a real optimization, not just style ("reject illegal transition" below returns `state`, and React doesn't even render); and your reducer must be pure **at dispatch time too** — it can run outside render, inside render, twice under StrictMode (which double-invokes reducers in dev precisely to surface impurity), and any count in between.

**Where the state lives:** on the hook node, one shape, one owner. Which is the segue — because what you put *in* that shape decides whether the reducer is twenty clean lines or a swamp.

## Basic usage

A cart with typed events, an exhaustive switch, and immutable updates:

```tsx
// cart-reducer.ts
import type { CartItem } from "./types";

export interface CartState {
  items: CartItem[];
}

export type CartAction =
  | { type: "item-added"; item: CartItem }
  | { type: "item-removed"; id: string }
  | { type: "quantity-changed"; id: string; quantity: number };

export function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "item-added": {
      const existing = state.items.find((i) => i.id === action.item.id);
      if (existing) {
        return {
          items: state.items.map((i) =>
            i.id === action.item.id
              ? { ...i, quantity: i.quantity + action.item.quantity }
              : i,
          ),
        };
      }
      return { items: [...state.items, action.item] };
    }
    case "item-removed":
      return { items: state.items.filter((i) => i.id !== action.id) };
    case "quantity-changed": {
      if (action.quantity < 1) return state; // illegal → same reference → no render
      return {
        items: state.items.map((i) =>
          i.id === action.id ? { ...i, quantity: action.quantity } : i,
        ),
      };
    }
    default: {
      const exhaustive: never = action; // typo'd or new action types fail to compile
      return exhaustive;
    }
  }
}
```

```tsx
// Cart.tsx
import { useReducer } from "react";
import { cartReducer } from "./cart-reducer";

export function Cart() {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });

  const total = state.items.reduce((sum, i) => sum + i.price * i.quantity, 0); // derived — never stored

  return (
    <section aria-label="Cart">
      {state.items.map((item) => (
        <div key={item.id}>
          {item.name} × {item.quantity}
          <button onClick={() => dispatch({ type: "item-removed", id: item.id })}>
            Remove
          </button>
        </div>
      ))}
      <output>Total: {total.toFixed(2)}</output>
    </section>
  );
}
```

Three conventions on display: events not setters (`item-added` carries intent; the merge-or-append rule lives in *one* place), the `never`-typed default (rename an action and every unhandled switch fails at compile time), and `total` derived during render — the first sighting of this article's second theme.

## Walkthrough: collapsing the effect chain

The scenario: a product browser — filter panel, paginated results, row selection with bulk actions. The `useState` version stores `filters`, `page`, `selectedIds`, and keeps them consistent with effects:

```tsx
// ProductBrowser.tsx — the "before": rules smeared across effects
export function ProductBrowser({ products }: { products: Product[] }) {
  const [filters, setFilters] = useState<Filters>(EMPTY_FILTERS);
  const [page, setPage] = useState(1);
  const [selectedIds, setSelectedIds] = useState<ReadonlySet<string>>(new Set());

  // 🔴 rule 1 as an effect: new filters reset the page
  useEffect(() => {
    setPage(1);
  }, [filters]);

  // 🔴 rule 2 as an effect: page or filter changes clear selection
  useEffect(() => {
    setSelectedIds(new Set());
  }, [filters, page]);

  /* … render … */
}
```

**Stage 1 — read the symptom.** Change a filter and trace it ([how-react-renders](../rendering/how-react-renders.md) machinery): render 1 commits new `filters` with the *old* page and selection — a painted frame where page 7 of the new filter set flashes, stale checkboxes included; the passive flush runs effect 1, `setPage(1)` → render 2; its flush runs effect 2 → render 3. **Three renders, two inconsistent painted frames, per keystroke on a filter.** This is branch 6 of the [effects decision tree](../effects/effects-and-synchronization.md) — *state chained to state through effects* — and it's the branch with the worst symptom-to-cause distance: the flicker is three hops from the rule that caused it. Effects synchronize with *external* systems; these two synchronize state with state, which is the reducer's job description.

**Stage 2 — name the events.** What can actually *happen* here? Not "page gets set" — that's a consequence. The events: the user changed filters; the user navigated to a page; the user toggled a row; the user toggled everything.

```tsx
// browser-reducer.ts
export interface BrowserState {
  filters: Filters;
  page: number;
  selectedIds: ReadonlySet<string>;
}

export type BrowserAction =
  | { type: "filters-changed"; filters: Filters }
  | { type: "page-changed"; page: number }
  | { type: "row-toggled"; id: string }
  | { type: "all-toggled"; visibleIds: string[] };
```

**Stage 3 — one event, all consequences, one place.**

```tsx
export function browserReducer(state: BrowserState, action: BrowserAction): BrowserState {
  switch (action.type) {
    case "filters-changed":
      // The rules, adjacent and atomic: new filters ⇒ first page, empty selection
      return { filters: action.filters, page: 1, selectedIds: new Set() };

    case "page-changed":
      if (action.page === state.page) return state;
      return { ...state, page: action.page, selectedIds: new Set() };

    case "row-toggled": {
      const next = new Set(state.selectedIds);
      next.has(action.id) ? next.delete(action.id) : next.add(action.id);
      return { ...state, selectedIds: next };
    }

    case "all-toggled": {
      const allSelected = action.visibleIds.every((id) => state.selectedIds.has(id));
      return { ...state, selectedIds: allSelected ? new Set() : new Set(action.visibleIds) };
    }

    default: {
      const exhaustive: never = action;
      return exhaustive;
    }
  }
}
```

Re-trace the filter change: one dispatch, one render, one committed frame — already consistent. The audit: **3 renders → 1; inconsistent frames 2 → 0; the two rules moved from two effects (18 lines apart) to two adjacent lines.** And a rule you couldn't cleanly express before falls out free: "changing page clears selection, but re-selecting the same page doesn't" — a guard returning `state`, which the eager bailout turns into *no render at all*.

**Stage 4 — derive the rest.** Notice what the reducer *doesn't* store:

```tsx
export function ProductBrowser({ products }: { products: Product[] }) {
  const [state, dispatch] = useReducer(browserReducer, undefined, initBrowser);

  // All derived, all during render, all incapable of being stale:
  const visible = applyFilters(products, state.filters);
  const totalPages = Math.max(1, Math.ceil(visible.length / PAGE_SIZE));
  const pageItems = visible.slice((state.page - 1) * PAGE_SIZE, state.page * PAGE_SIZE);
  const allSelected = pageItems.length > 0 && pageItems.every((p) => state.selectedIds.has(p.id));

  /* … render, dispatching events … */
}
```

Every one of those four is a value a `useState` version *might* have stored — and every stored copy would need its own synchronization effect, rebuilding the chain we just demolished. Which is the deep dive this article owes:

### Derive vs mirror — the deep dive

The test is one question: **can this value be computed from state and props I already have?** If yes, storing it creates a *mirror* — a second copy of a fact — and mirrors have exactly one new capability: **disagreeing with the source.** Keeping them agreeing requires synchronization; synchronization means effects; effects mean extra renders and a window of inconsistency between them. Deriving during render has none of these — a derived value can't be stale because it doesn't *persist*; it's recomputed from truth every time truth changes.

The mirror taxonomy, with the fix for each:

| Mirror | Smell in the wild | Fix |
| --- | --- | --- |
| Prop → state | `const [items, setItems] = useState(props.items)` — "why doesn't it update?" | Derive; if reset-on-identity is the goal, `key`; the full [props-change escalation ladder](./state-and-usestate.md) |
| State → state (aggregate) | `totalPages`, `isValid`, `count` in state, plus effects updating them | Compute during render; the Compiler makes re-derivation cheap |
| Object instead of identity | `selectedProduct: Product` in state — edits to the list leave a stale ghost selected | Store `selectedId`, derive the object: `products.find(p => p.id === selectedId)` |
| State → state (cascade) | Effects setting state when other state changes | This walkthrough — reducer collapse |

And the honest exceptions, because "never store derived data" is a slogan, not a rule: **point-in-time snapshots** (the order total *at purchase* is intentionally frozen — deriving it from current prices would be a bug); **user-editable copies of a starting value** (a draft initialized from a prop is a fork, not a mirror — the `initial*` naming contract from [state-and-usestate](./state-and-usestate.md) marks it); and **deliberately lagging values** for scheduling (`useDeferredValue` — machinery in [concurrent-rendering](../concurrent/concurrent-rendering.md) *(Wave 3, planned)*). What each exception shares: the stored copy is *supposed* to differ from the live derivation. If yours isn't, derive.

## Real-world patterns

**Transitions in one place.** When state is a *process* — submission, upload, connection — the shape is a discriminated union (the status-union discipline from [thinking-in-react](../foundations/thinking-in-react.md), now with an enforcement arm) and the reducer is the legality authority:

```tsx
type SubmitState =
  | { status: "idle" }
  | { status: "submitting"; startedAt: number }
  | { status: "error"; message: string; attempts: number }
  | { status: "success"; receiptId: string };

type SubmitAction =
  | { type: "submitted"; at: number }
  | { type: "failed"; message: string }
  | { type: "succeeded"; receiptId: string }
  | { type: "retried"; at: number };

function submitReducer(state: SubmitState, action: SubmitAction): SubmitState {
  switch (action.type) {
    case "submitted":
      if (state.status !== "idle") return state;        // double-fire → no-op, no render
      return { status: "submitting", startedAt: action.at };
    case "failed":
      if (state.status !== "submitting") return state;   // late failure after success → ignored
      return { status: "error", message: action.message, attempts: 1 };
    case "retried":
      if (state.status !== "error") return state;         // retry is only legal FROM error
      return { status: "submitting", startedAt: action.at };
    case "succeeded":
      if (state.status !== "submitting") return state;
      return { status: "success", receiptId: action.receiptId };
    default: {
      const exhaustive: never = action;
      return exhaustive;
    }
  }
}
```

Every guard is a bug that now can't exist: no double-submit from a re-clicked button, no success screen overwritten by a laggard error, no retry from nowhere. With `useState` these guards live at every call site or not at all. Note the division of labor: the reducer decides *legality and next state*; the async work itself — the fetch, the abort — stays in handlers and effects, dispatching results as events. (React 19's `useActionState` is exactly this shape productized for form mutations — pending state, folded results — owned by [actions](../concurrent/actions.md) *(Wave 3, planned)*; the double-submit guard as a full recipe is [double-submit-and-optimistic-like](../recipes/forms-and-ux/double-submit-and-optimistic-like.md).)

**Normalize deep state.** Nested state makes every update a spread pyramid and every entity a potential duplicate. The fix predates React — it's how databases think. Store each entity type flat, keyed by id; store *relations as ids*:

```tsx
// 🔴 nested: updating one comment's text, two levels deep
interface ThreadStateNested {
  posts: { id: string; title: string; comments: { id: string; author: User; text: string }[] }[];
}

// ✅ normalized: every entity one hop away
interface ThreadState {
  posts: Record<string, { id: string; title: string; commentIds: string[] }>;
  comments: Record<string, { id: string; authorId: string; text: string }>;
  users: Record<string, User>;
}

case "comment-edited":
  return {
    ...state,
    comments: {
      ...state.comments,
      [action.id]: { ...state.comments[action.id], text: action.text },
    },
  };
```

The nested version of that case is a `posts.map` containing a `comments.map` containing a spread — and it still leaves `author` duplicated in every comment, so a username change means hunting copies. Normalized, the user exists once; every rename is one record; every component derives its view (`post.commentIds.map(id => comments[id])`) during render. The threshold in practice: entities referenced from more than one place, or updates reaching 3+ levels deep — normalize. A settings object two levels deep — don't bother.

**The honest Immer take.** Immer lets you write `draft.comments[id].text = action.text` and get a correctly immutable update out, via proxies. Where it genuinely earns its spot: reducers over state that is *irreducibly* deep — a rich-text document model, a canvas scene graph, a third-party payload whose shape you don't own — where native updates are unreadable spread pyramids no matter how disciplined you are. `useImmerReducer` there is a legibility win worth the dependency. The honest costs: logging a draft prints a Proxy, not your data (debugging tax); it exempts your team from immutability *fluency*, which still gets exercised everywhere Immer isn't; and — the big one — **it anesthetizes the pain that's telling you to normalize.** A spread pyramid is a shape smell; Immer treats the symptom. House order of operations: flatten first, native spreads and the `toSorted`-class methods for what remains (they cover most array cases in one call), Immer only for the genuinely-deep residue — with a comment saying which residue.

**Dispatch through context.** When the tree between state and events is deep, `dispatch` rides the actions channel from [context](./context.md) — stable identity, zero re-broadcasts, and leaf components that can *report events* without knowing what the consequences are. That last property is the architectural payoff: a `<FilterChip>` dispatching `filters-changed` doesn't know pages reset. The rule knows.

**Test the reducer, not the render.** A reducer is a pure function; its tests need no DOM, no RTL, no async:

```tsx
// browser-reducer.test.ts
import { describe, expect, it } from "vitest";
import { browserReducer } from "./browser-reducer";

it.each([
  ["filters reset page", { ...base, page: 7 }, filtersChanged, 1],
  ["same page is a no-op", base, { type: "page-changed", page: base.page }, base.page],
])("%s", (_, state, action, expectedPage) => {
  const next = browserReducer(state, action as BrowserAction);
  expect(next.page).toBe(expectedPage);
});

it("rejects retry from success", () => {
  const state = { status: "success", receiptId: "r1" } as const;
  expect(submitReducer(state, { type: "retried", at: 0 })).toBe(state); // same reference
});
```

Note `toBe` on the rejection test — asserting the *same reference* pins down both the legality rule and the no-render bailout. Table-driven tests over a reducer are the cheapest high-value tests in a React codebase; the component tests that remain only need to prove events get dispatched.

## Reference

| | Reach for `useState` | Reach for `useReducer` |
| --- | --- | --- |
| Values | Independent, no rules | Move together, have rules |
| Update logic | Trivial (`setX(y)`) | Depends on previous state in structured ways |
| Call sites | One or two | Many, all needing the same rules |
| Illegal states | N/A | Guarded centrally, unions make them unrepresentable |
| Testing | Via component | Pure function, table-driven |

**State-shape smells:**

| Smell | Symptom | Fix |
| --- | --- | --- |
| Boolean soup (`isLoading`, `isError`, `isSuccess`) | Impossible combinations reachable | Discriminated status union |
| Stored derived value | Drifts from source; sync effects appear | Derive during render |
| Object stored where id suffices | Stale ghost after list updates | Store id, derive object |
| Effect chains between state | Multi-render cascades, flicker frames | Reducer; consequences in one event |
| Spread pyramids | Deep nesting | Normalize by id; Immer for the residue |
| Setter-shaped actions | Rules scattered at call sites | Event-shaped actions |

## Common mistakes

**1. Mutating in the reducer.**

```tsx
case "row-toggled":
  state.selectedIds.add(action.id); // 🔴 same references throughout
  return { ...state };              // top-level spread doesn't launder the nested mutation
```

Sometimes it even *appears* to work — until a bailout compares the mutated reference, or the eager dispatch evaluation sees "unchanged," or StrictMode's double reducer run adds twice. New objects along the changed path, every time.

**2. Side effects in the reducer.** Fetching, dispatching, logging analytics, `Date.now()` inside a case — reducers run during render, at dispatch time, and doubled in dev. Timestamps and ids enter *through the action* (`{ type: "submitted", at: Date.now() }`, minted in the handler); async work lives outside and reports back as events.

**3. Setter actions.** An action vocabulary of `set-filters`, `set-page`, `set-selection` means every call site re-decides the consequences — the exact disease, relocated. If a code review can't tell what *happened* from the action type, it's a setter.

**4. Boolean soup.** `{ isSubmitting: true, isError: true }` is a state your UI has no answer for, and it's reachable. Unions make it unrepresentable, and give the reducer's guards something to switch on.

**5. Storing what you can compute.** `totalPages` in state works until a delete makes it lie. Any field whose comment would be "kept in sync with X" is a mirror; delete it and derive.

**6. Selected object, not selected id.** `selected: product` captures a snapshot that survives edits, deletions, and refetches of the real product. Identity in state, object at render.

**7. Rebuilding the chain on top of the reducer.** The hybrid failure: a clean reducer *plus* a `useEffect` watching `state.filters` to dispatch `page-changed`. If a state change implies another state change, that implication is a reducer case, not an effect — branch 6 doesn't stop applying because a reducer exists.

**8. A `default` that silently returns state.** Without the `never` check, a typo'd `dispatch({ type: "filter-changed" })` (singular) is a no-op with no error, no render, no clue. Exhaustiveness turns it into a compile failure.

**9. Reading state right after dispatch.**

```tsx
dispatch({ type: "item-added", item });
trackCartSize(state.items.length); // 🔴 the snapshot — still the old length
```

Dispatch queues; the render hasn't happened; `state` is this render's value ([state-and-usestate](./state-and-usestate.md) owns the trace). Compute the next value locally for the side channel, or move the tracking to where the new state is in scope.

**10. One mega-reducer for unrelated concerns.** Cart logic, modal visibility, and a form draft in one reducer couples their types, their tests, and their renders. Reducers are cheap — one per *cohesive* state machine; unrelated booleans can stay `useState` next door. The split heuristic is the same as context channels: group by rules and change cadence, not by "it's all state."

## How this evolved

- **Redux (2015):** popularized the reducer pattern for React at app scale — single store, actions, time-travel devtools — and taught a generation event-driven state, along with a lot of ceremony.
- **`useReducer` (16.8):** the pattern absorbed into React at *component* scale, no store, no middleware — right-sized for the majority of cases that never needed global state.
- **The great unbundling (18 era):** server state moved to query caches, ambient state to context, the leftover client state shrank — and mostly fit in `useState`/`useReducer`. Redux Toolkit remains the answer at a scale most apps don't reach ([state-management-landscape](../ecosystem/state-management-landscape.md) *(Wave 4, planned)* has the decision table).
- **React 19:** `useActionState` ships a reducer-shaped API for async mutations — `(prevState, formData) => newState` — the pattern grown a pending flag and form integration.
- **Compiler era:** re-deriving during render got cheaper to *not think about* — the Compiler memoizes the `applyFilters` call when inputs are stable — which removes the last performance excuse for mirrors.

## Exercises

**1. Collapse a chain.** Take the "before" `ProductBrowser` (or any component you own with a `setX`-inside-`useEffect` watching other state) and profile one filter change — count commits. Refactor to the event reducer and re-profile. You're looking for 3 → 1, and the disappearance of the intermediate frame where selection was visibly stale.
*Hint: React DevTools Profiler shows each commit separately; the "before" shows three commits with different "what changed" causes chained like dominoes.*

**2. Legality by construction.** Design the state union and action set for a seat-booking flow: browsing → holding (with a 5-minute expiry) → paying → confirmed | expired. Write the reducer with guards, then table-driven Vitest tests asserting every illegal transition returns the same reference. Where does the expiry *timer* live?
*Hint: not in the reducer — the timeout is an effect that dispatches `{ type: "hold-expired" }`; the reducer decides whether that event still matters (it doesn't if you're already `confirmed`).*

**3. Normalize and feel the difference.** Model a post with nested comments (each embedding its author) both ways. Implement `comment-edited` and `user-renamed` in each shape. Count the lines, then answer: in the nested shape, what does the UI show after a rename — and how many places did you have to touch to fix it?
*Hint: the nested rename either misses embedded author copies (stale names) or maps across every comment of every post. The normalized rename is three lines, and every derived view is correct at the next render for free.*

## Summary

- Reducers centralize *rules*: actions are events describing what happened; the reducer alone decides consequences and legality. Setter-shaped actions forfeit the entire benefit.
- Mechanically it's `useState`'s machinery with actions in the queue: stable `dispatch` for free, batching as usual, eager bailout rewarding same-reference returns, StrictMode doubling reducer runs to catch impurity.
- Effect chains that synchronize state with state are reducer cases in disguise — collapsing them turns N-render cascades with inconsistent frames into one atomic transition.
- Derive, don't mirror: if it's computable from what you store, computing it is the only version that can't be stale. The exceptions — snapshots, forks, deliberate lag — are defined by *wanting* divergence.
- Shape decides everything downstream: unions kill impossible states, ids kill stale ghosts, normalization kills spread pyramids — and Immer is for the deep residue that survives normalization, not a substitute for it.
- Reducers are the cheapest tests you'll write: pure, table-driven, reference-equality assertions on rejected transitions.

## See also

- [state-and-usestate](./state-and-usestate.md) — the update queue, snapshots, and immutability recipes this article builds on
- [effects-and-synchronization](../effects/effects-and-synchronization.md) — the decision tree whose branch 6 this article's walkthrough deletes
- [context](./context.md) — the actions channel that carries `dispatch` without re-broadcasts
- [thinking-in-react](../foundations/thinking-in-react.md) — status-union discipline and derive-don't-store, first principles form
- [double-submit-and-optimistic-like](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) — the transition-guard problem as a production recipe
- [actions](../concurrent/actions.md) *(Wave 3, planned)* — `useActionState`, the reducer shape productized for async mutations
- [state-management-landscape](../ecosystem/state-management-landscape.md) *(Wave 4, planned)* — when component-scale reducers stop being enough

## References

- [useReducer — react.dev](https://react.dev/reference/react/useReducer)
- [Extracting State Logic into a Reducer — react.dev](https://react.dev/learn/extracting-state-logic-into-a-reducer)
- [Choosing the State Structure — react.dev](https://react.dev/learn/choosing-the-state-structure)
- [Scaling Up with Reducer and Context — react.dev](https://react.dev/learn/scaling-up-with-reducer-and-context)
- [You Might Not Need an Effect — react.dev](https://react.dev/learn/you-might-not-need-an-effect)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.