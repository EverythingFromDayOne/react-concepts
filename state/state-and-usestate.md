---
article_id: state-and-usestate
concept_folder: state
wave: 1
related:
  - foundations/thinking-in-react
  - state/usereducer-and-state-structure
  - effects/effects-and-synchronization
  - rendering/rendering-lists-and-keys
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# State and useState

> **Lead with this:** `useState` looks like a variable and behaves like nothing else in JavaScript, and every classic React confusion — "I set it and logged it and it's the old value", "I called it three times and it went up by one", "the timeout shows a stale number" — comes from reading it as a variable. It isn't. State is a **snapshot**: each render receives fixed values, setters don't change anything in the current render, they *queue* work for the next one, and React batches the queue. Internalize the snapshot + queue model once and this entire bug family becomes predictable; this article builds that model down to the actual queue mechanics, then applies it to a cart feature where the bugs are real ones.

## What it is

`useState` declares a value that React remembers between renders of this component instance, plus the function that requests a change to it:

```tsx
const [query, setQuery] = useState('');
```

Four properties define its behavior, and each is the antidote to a specific bug class:

**State persists across renders; variables don't.** A plain `let count = 0` in the component body resets to `0` on every render — your function runs top-to-bottom fresh each time ([`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md)). `useState` stores the value *outside* your function, on the component's fiber, and hands it back each render.

**State is a snapshot per render.** The `query` your render reads — including inside every handler and every closure that render creates — is a constant. It will never change. "Updating state" means asking React for a *next render* with a *next snapshot*; the current one is already immutable history.

**Setters schedule, they don't assign.** `setQuery('a')` enqueues an update and asks React to re-render. Nothing about the current execution changes — which is precisely why the log on the next line shows the old value.

**Setting state replaces the value.** Unlike class-era `this.setState`, there is no merging. Whatever you pass (or return from an updater) *is* the whole next value — the reason immutable-update patterns matter for objects and arrays.

**State is per-instance and private.** Render `<Cart />` in two places and you have two independent carts — each fiber carries its own hook list; there is no "the component's state" in the singular. And no parent can reach in from outside: props flow down, change requests flow up ([`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md)), and the only external lever that touches a child's state is identity itself — the `key` reset covered later in this article.

And one property people rely on without knowing it's guaranteed: **the setter function's identity is stable for the life of the component.** `setQuery` from render 1 and render 50 are the same function — safe to pass down, safe to omit from dependency arrays.

## How it works under the hood

### Where state actually lives

Your function's locals die on return; state lives on the **fiber** — the persistent node React keeps per component instance. `fiber.memoizedState` points at a linked list of hook nodes, one per hook call, matched purely by **call order**:

```ts
// Simplified hook node for one useState
{
  memoizedState: 'current value',
  queue: {
    pending: /* circular list of queued updates */,
    dispatch: /* the stable setter */,
  },
  next: /* the next hook in this component */,
}
```

On **mount**, each `useState` call creates a node and stores the initial value. On **update**, React walks the existing list in order: first call gets the first node, second gets the second. That's the whole reason hooks can't sit inside conditionals — skip one call and every subsequent hook reads its neighbor's node ([`../foundations/rules-of-react.md`](../foundations/rules-of-react.md)). The initial argument is only *read* on mount; on every later render it's evaluated and thrown away — which is why `useState(expensive())` pays the cost every render, and lazy initialization exists (below).

### The update queue, traced

Every setter call pushes an update object onto the hook's queue. Two kinds exist, and the difference is the most practically important mechanic in this article:

- **Replace update** — `setLikes(5)`: "the next value is 5."
- **Updater function** — `setLikes(l => l + 1)`: "compute the next value from whatever the previous update produced."

During the next render, React drains the queue **in order**, feeding each updater the running result:

```tsx
// Snapshot this render: likes = 0
function handleClick() {
  setLikes(likes + 1);   // enqueue: replace with 1   (likes is 0 in this snapshot)
  setLikes(likes + 1);   // enqueue: replace with 1   (still the same snapshot!)
  setLikes(likes + 1);   // enqueue: replace with 1
}
// Queue: [replace 1, replace 1, replace 1] → next render: likes = 1
```

```tsx
function handleClick() {
  setLikes(l => l + 1);  // enqueue: fn(prev)
  setLikes(l => l + 1);
  setLikes(l => l + 1);
}
// Queue: [fn, fn, fn] → 0 → 1 → 2 → 3 → next render: likes = 3
```

Mixing them follows the same rule — each step sees the running result:

| Queue (starting from `0`) | Processing | Next render |
| --- | --- | --- |
| `set(5)`, `set(l => l + 1)` | 5 → 6 | `6` |
| `set(l => l + 1)`, `set(42)` | 1 → **42** | `42` — a replace stomps everything before it |
| `set(likes + 5)`, `set(l => l + 1)` | 5 → 6 | `6` |

Two contract clauses come with updaters: they must be **pure** (React's StrictMode double-invokes them in development to prove it — an updater that also fires analytics fires it twice), and they receive the *queue-processed* previous value, not the render snapshot.

### Batching: one render per event, everywhere

React collects all state updates from one synchronous run — an event handler, a promise resolution, a timeout callback — and performs **one** render pass at the end. Three setters in a handler, across three different state variables, still means one render, one commit, one paint. Since React 18 this holds *everywhere*; it isn't limited to React event handlers.

```tsx
// legacy: pre-18 — only React event handlers batched; this timeout caused TWO renders,
// one per setter. Modernized by automatic batching in the upgrade pass.
setTimeout(() => {
  setCount(c => c + 1);   // render 1 (pre-18)
  setFlag(f => !f);       // render 2 (pre-18)
}, 1000);
```

The rare case that genuinely needs the DOM updated *mid-handler* (measure after a state change, imperative scroll math) has an escape hatch, `flushSync` — deliberately covered in [`../effects/escape-hatches-audit.md`](../effects/escape-hatches-audit.md), because reaching for it is usually a design smell.

### The snapshot, caught in a closure

The sharpest version of the snapshot rule:

```tsx
function handleSend() {
  setMessages(m => [...m, draft]);
  setTimeout(() => {
    alert(`Sent. You now have ${messages.length} messages.`);  // ← the OLD length
  }, 3000);
}
```

The timeout fires after three seconds, long after the re-render — and still reports the old count. The closure captured this render's `messages`, and this render's `messages` is a constant forever. State doesn't behave like a variable that handlers "look up later"; every closure holds the snapshot of the render that created it. When code genuinely needs the *latest* value at a future moment, that's a synchronization job — [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) and refs own that story. Most of the time, the honest fix is computing the message from values you already have at schedule time.

### The bailout

If the next value is `Object.is`-equal to the current one, React bails: no children re-render, no effects fire. (React may still invoke *this* component's function once more before deciding — don't be surprised by one extra log line; the subtree stays untouched.) The bailout is a gift with a sharp edge: it's why setting the same primitive is free, and it's exactly why **mutate-then-set does nothing** — the mutated array is the same reference, `Object.is` says "unchanged," and the update is swallowed whole. The `Object.is` semantics table lives in [`../foundations/components-and-props.md`](../foundations/components-and-props.md); it governs here identically.

## Basic usage

The daily forms, in one component — including the typing decisions that don't fit on one line:

```tsx
import { useState } from 'react';

interface RsvpFormProps {
  initialGuests?: number; // "initial" in the name = read once, by design
}

export function RsvpForm({ initialGuests = 1 }: RsvpFormProps) {
  const [name, setName] = useState('');                              // inferred: string
  const [guests, setGuests] = useState(initialGuests);               // inferred: number
  const [attending, setAttending] = useState<'yes' | 'no' | null>(null); // explicit — see below
  const [invitees] = useState(() => buildInviteeIndex());            // lazy init: runs once

  return (
    <form>
      <label>
        Name
        <input value={name} onChange={(e) => setName(e.target.value)} />
      </label>

      <label>
        Guests
        <input
          type="number"
          value={guests}
          onChange={(e) => setGuests(Math.max(1, e.target.valueAsNumber || 1))}
        />
      </label>

      <fieldset>
        <button type="button" onClick={() => setAttending('yes')}>Attending</button>
        <button type="button" onClick={() => setAttending('no')}>Can't make it</button>
        {attending !== null && <p>Marked as: {attending}</p>}
      </fieldset>
    </form>
  );
}
```

Three typing notes that save real debugging time: `useState('yes')` infers `string`, not the literal — union states need the explicit generic (`useState<'yes' | 'no' | null>(null)`), or the whole discriminated-union discipline from [`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md). "Nothing yet" is modeled as an explicit `null` in the generic, not by omitting the argument (`useState<User>()` quietly makes every read `User | undefined`). And `useState(() => buildInviteeIndex())` passes the *function* — `useState(buildInviteeIndex())` calls it on every render and discards the result on all but the first.

## Walkthrough — a cart with the classic state bugs, fixed properly

The scenario: a mini cart — add products, step quantities, remove lines, live subtotal — plus the feature that flushes out the real bug: **"Buy again"**, which re-adds every item from a past order at once.

### Step 1 — model the minimal state

```tsx
// cart.ts
export interface Product {
  id: string;
  name: string;
  price: number;
}

export interface CartItem {
  id: string;
  name: string;
  unitPrice: number;
  qty: number;
}

export function toCartItem(p: Product): CartItem {
  return { id: p.id, name: p.name, unitPrice: p.price, qty: 1 };
}
```

State inventory, per the five-step workflow: the `items` array is the *only* state. Subtotal and item count change over time but are computable — derive them. One source of truth means no synchronization jobs later.

### Step 2 — add to cart, immutably

```tsx
const [items, setItems] = useState<CartItem[]>([]);

function addToCart(product: Product) {
  setItems((prev) => {
    const existing = prev.find((i) => i.id === product.id);
    if (existing) {
      return prev.map((i) =>
        i.id === product.id ? { ...i, qty: i.qty + 1 } : i,
      );
    }
    return [...prev, toCartItem(product)];
  });
}
```

Both branches build a **new array** (and, for the bumped line, a new item object) — the bailout section explained why mutation would be silently swallowed. The updater form isn't decoration here; the next step is where it becomes load-bearing.

### Step 3 — "Buy again": the loop that adds one item

The first implementation everyone writes:

```tsx
// ❌ Order had 3 items; cart gains… 1
function buyAgain(order: Product[]) {
  for (const p of order) {
    setItems([...items, toCartItem(p)]);
  }
}
```

Run the queue model on it: every iteration spreads **the same `items` snapshot** — this render's constant — so the queue is `[replace: old+A, replace: old+B, replace: old+C]`, and the last replace wins. Three products, one survives, and it reproduces only for multi-item orders, which is why it escapes the demo and ships.

Two correct shapes — prefer the first:

```tsx
// ✅ One update built from the whole batch
function buyAgain(order: Product[]) {
  setItems((prev) => [...prev, ...order.map(toCartItem)]);
}

// ✅ Also correct: updater per iteration — each fn receives the running result
for (const p of order) {
  setItems((prev) => [...prev, toCartItem(p)]);
}
```

(The clean version also wants "merge quantities for items already in the cart" — that's exercise 2.)

### Step 4 — quantity stepper with a clamp, deletion by zero

```tsx
function changeQty(id: string, delta: number) {
  setItems((prev) =>
    prev
      .map((i) =>
        i.id === id ? { ...i, qty: Math.max(0, i.qty + delta) } : i,
      )
      .filter((i) => i.qty > 0),
  );
}
```

The convention this project locks: **any update computed from the previous value uses the updater form** — it's immune to snapshot staleness no matter where the call ends up (loops, `await` continuations, callbacks passed to children). Stepping to zero removes the line in the same pass: one update, one render, no "set then clean up in an effect" two-step.

### Step 5 — derive, render, done

```tsx
// Cart.tsx
import { useState } from 'react';
import { type CartItem, type Product, toCartItem } from './cart';

export function Cart({ catalog, lastOrder }: { catalog: Product[]; lastOrder: Product[] }) {
  const [items, setItems] = useState<CartItem[]>([]);

  // …addToCart / buyAgain / changeQty from the steps above…

  const subtotal = items.reduce((sum, i) => sum + i.unitPrice * i.qty, 0);
  const count = items.reduce((sum, i) => sum + i.qty, 0);

  return (
    <section aria-label="Cart">
      <ul>
        {catalog.map((p) => (
          <li key={p.id}>
            {p.name} — ${p.price.toFixed(2)}{' '}
            <button onClick={() => addToCart(p)}>Add</button>
          </li>
        ))}
      </ul>

      <button onClick={() => buyAgain(lastOrder)}>Buy again ({lastOrder.length} items)</button>

      <h2>Cart {count > 0 && <span className="badge">{count}</span>}</h2>
      <ul>
        {items.map((i) => (
          <li key={i.id}>
            {i.name} × {i.qty}
            <button onClick={() => changeQty(i.id, -1)} aria-label={`Decrease ${i.name}`}>−</button>
            <button onClick={() => changeQty(i.id, +1)} aria-label={`Increase ${i.name}`}>+</button>
            <span>${(i.unitPrice * i.qty).toFixed(2)}</span>
          </li>
        ))}
      </ul>
      <p>Subtotal: <strong>${subtotal.toFixed(2)}</strong></p>
    </section>
  );
}
```

Audit against the model: one state variable; every projection (`subtotal`, `count`, line totals, the badge) derived fresh per render; every prev-derived update in updater form; every update a new reference; keys from stable ids. Nothing to keep in sync, nothing that goes stale.

## Real-world patterns

### Reset state with `key`, not with effects

A form component keeps its draft state while the user edits — but must reset when the *subject* changes (switch users, switch documents). The reflex is a synchronization effect (`useEffect(() => setDraft(user.draft), [user.id])`); the idiomatic tool is **identity**:

```tsx
<ProfileForm key={user.id} user={user} />
```

A changed `key` tells reconciliation "different instance" — React unmounts and remounts, and *every* piece of state inside resets to its initializers, including state in grandchildren you'd otherwise chase with effect cascades. One attribute replaces a category of sync bugs. (Why keys control identity: [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md).)

### When props change mid-life: the escalation ladder

Sometimes a component must *adjust* its own state when a prop changes — "clear the selection when the items list changes" is the archetype. Reach for these strictly in order:

**1. Derive it.** Most "adjust" wishes are derived state in disguise — the selection doesn't need clearing if invalid selections are simply never *shown*:

```tsx
const selection = items.some((i) => i.id === selectedId) ? selectedId : null;
```

**2. `key` it.** If *everything* in the component should reset, the prop change is really a subject change — the pattern above.

**3. Guarded set during render.** The narrow, documented exception to "never set state while rendering" — for a genuine partial adjustment that can't be derived:

```tsx
const [prevItems, setPrevItems] = useState(items);
const [selectedId, setSelectedId] = useState<string | null>(null);

if (items !== prevItems) {     // the guard is what makes this terminate
  setPrevItems(items);
  setSelectedId(null);
}
```

React discards this render's output and immediately re-renders with the new state — *before* rendering children and before anything commits, so no frame ever shows the stale selection (the effect version paints one, then corrects it). The guard is load-bearing: this exact shape without the condition is the "Too many re-renders" loop from the mistakes list. Use it rarely; when you find yourself here often, the state design is fighting you.

**4. An effect.** Last on the ladder, and only when the adjustment must involve the outside world — the plain state-from-props sync effect remains the anti-pattern it always was ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)).

### Immutable update recipes — the modern set

Arrays got ergonomic non-mutating methods in ES2023; use them and retire the copy-then-mutate dance:

| Intent | Use | Not |
| --- | --- | --- |
| add | `[...prev, item]` | `push` |
| remove | `prev.filter(i => i.id !== id)` | `splice` |
| update one | `prev.map(i => i.id === id ? { ...i, done: true } : i)` | `find` + assign |
| replace at index | `prev.with(idx, next)` | `prev[idx] = next` |
| sort / reverse | `prev.toSorted(cmp)` / `prev.toReversed()` | `sort` / `reverse` — **these mutate** |
| nested object field | `{ ...prev, address: { ...prev.address, city } }` | reach-in assignment |

When updates go two-plus levels deep routinely, the state shape is the problem — flatten/normalize it, or graduate to a reducer where an immutability helper can earn its keep ([`usereducer-and-state-structure.md`](usereducer-and-state-structure.md) covers both, including the honest take on Immer).

### Group what changes together; split what doesn't

Neither "one mega-object" nor "twelve atoms" is a rule — the boundary is *co-change*. Position `{x, y}` updates as one thing: one state. `name` and `email` edited independently: two states, and no `setForm({ ...form, name })` spread ritual that goes stale the first time it runs after an `await` (use the updater spread `setForm(f => ({ ...f, name }))` if they must live together). When several variables always transition *in concert* — status, data, error — that's not a grouping problem, that's a state machine: one discriminated union, per [`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md), and its transitions are one honest `set` each.

### `initialX` props: read-once by contract

`useState(props.initialGuests)` ignores later prop changes — which is a *feature* when named honestly (`initialValue`, `defaultTab`): the parent seeds, the component owns. The bug is the same code with the prop named `value`, where the caller expects it to track. Pick a side per prop: fully controlled (parent owns, no state here) or uncontrolled-with-initial (state here, `initial*` naming) — the full decision framework is [`../forms/forms-controlled-and-uncontrolled.md`](../forms/forms-controlled-and-uncontrolled.md), and the "sync it with an effect" middle ground is the bug factory to refuse.

## API quick reference

| Form | Type | Notes |
| --- | --- | --- |
| `useState(initial)` | `[T, Dispatch<SetStateAction<T>>]` | `T` inferred from `initial`; literals widen (`'idle'` → `string`) — use the explicit generic for unions |
| `useState<T \| null>(null)` | explicit generic | The standard "nothing yet" modeling; avoid bare `useState<T>()` unless `undefined` is truly meaningful |
| `useState(() => compute())` | lazy initializer | Runs once, on mount; the place for expensive seeds. StrictMode double-invokes it in dev — keep it pure |
| `setX(value)` | replace update | Whole-value replacement — no merging, ever |
| `setX(prev => next)` | updater | Receives the queue-processed previous value; must be pure; the default for prev-derived updates |
| `SetStateAction<T>` | `T \| ((prev: T) => T)` | What setter props accept — the type to use when passing setters down (though intent-named callbacks are usually the better contract, per [`../foundations/components-and-props.md`](../foundations/components-and-props.md)) |
| setter identity | stable | Same function across renders; safe in dependency arrays and props |

## Common mistakes

**Mutate, then set.** `item.qty++; setItems(items)` — same reference, `Object.is` bailout, update swallowed. New references at every level you changed.

**Replace form for prev-derived updates.** `setCount(count + 1)` in a loop, after an `await`, or several times per handler collapses to one increment — the queue holds N copies of "replace with snapshot+1." Updater form.

**Reading state right after setting it.** `setQuery(next); search(query)` searches the *old* value — the setter scheduled a render, it didn't assign. Use the value you already have (`search(next)`); "react to committed state" is an effect's job, and usually a design smell even there.

**Calling the setter during render.** `<button onClick={setOpen(true)}>` *calls* the setter while rendering → render → call → render → "Too many re-renders." Pass a function: `onClick={() => setOpen(true)}`. (The one sanctioned exception — the *guarded* compare-and-set for adjusting state on prop change — is step 3 of the escalation ladder above; the guard is the difference between a pattern and the loop.)

**Storing a function in state.** Both slots treat bare functions as *instructions*, not values: `useState(buildValidator)` calls it as a lazy initializer; `setValidator(myValidator)` calls it as an updater and stores its return. To store the function itself, wrap it — `useState(() => buildValidator)`, `setValidator(() => myValidator)`. The same trap fires for anything callable (stored components, callbacks in config objects) — and hitting it is often a hint the design wants a ref or a discriminated union instead of a naked function in state.

**Expecting closures to see fresh state.** The timeout/subscription/`await`-continuation that reports stale values — snapshots, captured. Compute at schedule time, or treat "latest value later" as the synchronization problem it is ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)).

**Eager expensive initialization.** `useState(buildIndex(data))` runs `buildIndex` on *every* render for a value used once. `useState(() => buildIndex(data))`.

**Impure updaters or initializers.** Firing analytics, mutating shared objects, or reading `Date.now()`-style nondeterminism inside `setX(prev => …)` — StrictMode double-invokes both in development precisely to surface this. Updaters compute; effects and handlers act.

**Widened literal types.** `useState('idle')` infers `string`, and six weeks later `setStatus('loadnig')` compiles. Explicit union generics for state machines.

**Stale object merges.** `setForm({ ...form, email })` after an `await` writes a snapshot of `form` from before the await — clobbering whatever changed meanwhile. `setForm(f => ({ ...f, email }))`, or split states that don't co-change.

**Props copied into state, accidentally.** The component ignores prop updates and nobody knows why. If it should track: render from the prop. If it should seed: name it `initial*`. If it should reset per subject: `key`.

## How this evolved

| Era | Change | What it means now |
| --- | --- | --- |
| Class era | `this.setState(partial, callback?)` — shallow **merge**, updater as `(state, props) =>`, post-update callback | The merge instinct and the callback instinct are both legacy; `useState` replaces wholesale and has no callback — "after it commits" is effect territory |
| React 16.8 (2019) | `useState` ships | State becomes per-concern values on a hook list instead of one class-owned object |
| React 18 (2022) | Automatic batching everywhere; StrictMode double-invokes initializers/updaters | Timeouts/promises stop causing render-per-setter; impure updaters get caught on day one |
| React 19 (2024) | Actions: `useActionState`, `useOptimistic` | *Async* state transitions (pending/error/optimistic) get first-class primitives — `useState` stays the synchronous core ([`../concurrent/actions.md`](../concurrent/actions.md)) |
| Compiler era (2025+) | Automatic memoization | Derived-during-render values become cached-by-default — the "derive, don't store" discipline gets cheaper, not obsolete |

```tsx
// legacy: class-era state — merge semantics + callback, modernized in the upgrade pass
this.setState({ qty: this.state.qty + 1 }, () => this.scrollToCart());
```

## Exercises

### 1. Predict the render log

Without running it, write the exact console output of one click, and the button label after:

```tsx
function Widget() {
  const [n, setN] = useState(0);
  console.log('render:', n);

  function onClick() {
    setN(n + 1);
    setN(n + 1);
    setN((x) => x + 1);
    console.log('handler:', n);
    setTimeout(() => console.log('timeout:', n), 500);
  }

  return <button onClick={onClick}>{n}</button>;
}
```

*Hint: queue is `[replace 1, replace 1, fn]` → 1, 1, 2. Snapshot rule for both logs in the handler and the timeout. Expected: `render: 0` (mount) → click → `handler: 0` → `render: 2` → `timeout: 0`. Label: `2`. StrictMode in dev will add a second `render:` line per render — explain why that's fine.*

### 2. Merge quantities in "Buy again"

The walkthrough's fixed `buyAgain` appends blindly — re-ordering something already in the cart creates a duplicate line. Rewrite it so existing lines get their `qty` bumped and only genuinely new products append. One `setItems` call.

*Hint: inside the single updater, build a `Map<string, CartItem>` from `prev`, fold the order into it (`existing ? { ...existing, qty: existing.qty + 1 } : toCartItem(p)`), return `[...map.values()]`. The whole merge is one pure function of `prev` — which is exactly why it belongs in the updater.*

### 3. Delete the sync effect

Refactor this so the effect disappears and behavior is preserved (draft resets when the selected user changes, edits don't otherwise reset):

```tsx
function ProfileEditor({ user }: { user: User }) {
  const [draft, setDraft] = useState(user.bio);
  useEffect(() => {
    setDraft(user.bio);
  }, [user.id]); // reset when switching users
  /* … */
}
```

*Hint: the component's state should live and die with the user it's editing — `<ProfileEditor key={user.id} user={user} />` at the call site, `useState(user.bio)` inside, effect deleted. Note what else the `key` version fixes that the effect version didn't: the one-render flash of the old draft, and nested state you forgot to reset.*

## Summary

You learned `useState` as it actually executes: values live on the fiber's hook list (matched by call order), each render reads an immutable snapshot, and setters enqueue replace-or-updater entries that React drains in order and batches into one render per synchronous run. That model makes the bug families mechanical — triple-set-increments-once is three replaces of one snapshot; the stale timeout is a captured constant; the swallowed update is an `Object.is` bailout on a mutated reference. The working disciplines follow directly: updater form for anything prev-derived, lazy initializers for expensive seeds, whole-value immutable updates with the ES2023 non-mutating methods, explicit union generics, `key` for subject resets instead of sync effects, and grouping state by co-change. `useState` is the synchronous core of React's state story; reducers structure it, effects synchronize it outward, and Actions extend it into async — each in their own article.

## See also

- [`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md) — minimal state, derive-don't-store, states-as-unions
- [`usereducer-and-state-structure.md`](usereducer-and-state-structure.md) — when transitions outgrow scattered setters
- [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) — reacting to committed state, and the sync-effect anti-patterns
- [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md) — why `key` resets state: identity and reconciliation
- [`../forms/forms-controlled-and-uncontrolled.md`](../forms/forms-controlled-and-uncontrolled.md) — the controlled/uncontrolled decision the `initial*` pattern feeds into
- [`../concurrent/actions.md`](../concurrent/actions.md) — async state transitions with `useActionState`/`useOptimistic`

## References

- [`useState` API (react.dev)](https://react.dev/reference/react/useState)
- [State: A Component's Memory (react.dev)](https://react.dev/learn/state-a-components-memory)
- [State as a Snapshot (react.dev)](https://react.dev/learn/state-as-a-snapshot)
- [Queueing a Series of State Updates (react.dev)](https://react.dev/learn/queueing-a-series-of-state-updates)
- [Updating Objects in State (react.dev)](https://react.dev/learn/updating-objects-in-state)
- [Updating Arrays in State (react.dev)](https://react.dev/learn/updating-arrays-in-state)
- [Preserving and Resetting State (react.dev)](https://react.dev/learn/preserving-and-resetting-state)
- [Automatic batching in React 18 — Working Group discussion](https://github.com/reactwg/react-18/discussions/21)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.