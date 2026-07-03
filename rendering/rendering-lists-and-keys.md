---
article_id: rendering-lists-and-keys
concept_folder: rendering
wave: 1
related:
  - thinking-in-react
  - jsx-and-rendering
  - state-and-usestate
  - how-react-renders
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Rendering Lists and Keys

> **Lead with this:** A `key` is not a React formality to silence a console warning — it's the *identity* of an item across renders, and identity decides everything that persists: component state, DOM nodes, focus, scroll position, in-flight CSS transitions, uncontrolled input values. Get keys right and lists just work; get them wrong and you ship the strangest bug class in React — checkboxes that jump to the wrong row, drafts that attach to the wrong ticket, the wrong item animating out — all invisible in demos and all guaranteed the first time a real user deletes, filters, or prepends. This article shows exactly how React matches list children, traces the index-key corruption step by step, and locks the conventions that make it unrepresentable.

## What it is

When a component re-renders, React receives a brand-new element tree and must answer one question per position: **is this the same thing as last time, or a different thing?** Three heuristics decide it:

1. **Different `type` → different thing.** `<div>` became `<span>`, or `ArticleView` became `VideoView`: React tears the old subtree down (state destroyed, effects cleaned up, DOM removed) and builds the new one fresh. It never attempts to morph across types.
2. **Same `type` → same thing.** Keep the existing instance — fiber, state, DOM node — and update its props. This is the cheap, common path.
3. **For dynamic children (arrays), matching is by `key`.** Position is meaningless in a list that can insert, remove, filter, or reorder — so React matches old children to new ones by their keys, and only falls back to position (index) when you didn't provide any.

A key is any stable string uniquely identifying an item **among its siblings** — not globally, not across the app, just within one array of children. React reads it, uses it for matching, and strips it: it never reaches your component as a prop (`props.key` is `undefined` — [`../foundations/jsx-and-rendering.md`](../foundations/jsx-and-rendering.md) showed where it physically lives on the element).

The one-sentence model to keep: **same key + same type = "update this item in place"; new key = "mount a new item"; missing key = "delete that item."** Everything else in this article is that sentence, applied.

## How it works under the hood

### The matching algorithm, honestly

React's array reconciliation is a pragmatic single pass, not a general diff:

1. **Walk old and new children in lockstep** while keys match position-for-position — the fast path that makes "append to the end" and "update in place" nearly free.
2. **On the first mismatch**, dump the *remaining* old children into a map by key.
3. **For each remaining new child**, look up its key: hit → reuse that fiber (state and DOM survive, props update); miss → mount fresh.
4. **Whatever's left in the map** gets deleted — unmount, effect cleanup, DOM removal.

One honest limitation worth knowing at scale: React tracks a `lastPlacedIndex` and only ever marks nodes to *move forward*. Append is optimal; **moving the last item to the front is the degenerate case** — React moves the other N−1 nodes instead of the one that actually moved. Irrelevant for a 30-row table; measurable for a 5,000-row reorder or a list with expensive layout per move. (Some frameworks run a longest-increasing-subsequence diff to minimize moves; React deliberately keeps the single pass. The full traversal story is [`how-react-renders.md`](how-react-renders.md).)

Two compile-time notes that connect back: arrays from `.map()` go through `jsx()` and get the missing-key warning, while statically written siblings compile to `jsxs()` and don't — the compiler vouches static siblings can't reorder ([`../foundations/jsx-and-rendering.md`](../foundations/jsx-and-rendering.md)). And with no keys at all, React silently uses the index — meaning "I didn't add keys" and "I used index keys" are the *same program*, with the same bugs minus the warning.

### What "same item" preserves — and what a new key destroys

Matching a key means the **fiber survives**, and the fiber is where everything lives:

| Survives when key + type match | Destroyed when the key changes |
| --- | --- |
| `useState` / `useReducer` values (the hook list rides the fiber — [`../state/state-and-usestate.md`](../state/state-and-usestate.md)) | State reset to initializers |
| The DOM node itself — focus, selection, scroll position, media playback | Fresh DOM: focus lost, scroll reset, video restarts |
| Uncontrolled input values (they live in the DOM, not in React) | Drafts vanish |
| Effect instances (deps unchanged → not re-run) | Cleanup runs, effects re-fire on the new mount |
| In-flight CSS transitions/animations | Cut dead |

Read that table twice, because it's both the bug and the feature. The bug: with index keys, *position* inherits all of that, not the item — delete row one and every row below inherits its predecessor's state. The feature: change a key **on purpose** and you get a guaranteed, total reset — the `key={user.id}` pattern from [`../state/state-and-usestate.md`](../state/state-and-usestate.md), now with its mechanism visible.

### The corruption, traced

Three rows, index keys, one piece of local state per row (`expanded`), row B expanded:

| Fiber (key) | State | Renders data |
| --- | --- | --- |
| `0` | `expanded: false` | Ticket A |
| `1` | `expanded: true` | Ticket B |
| `2` | `expanded: false` | Ticket C |

User deletes Ticket A. The new element array is `[B → key 0, C → key 1]`. React matches by key:

| New element (key) | Matched fiber | Fiber's state | Result on screen |
| --- | --- | --- | --- |
| B (`0`) | old fiber `0` | `expanded: false` | **B collapsed** — it "lost" its expansion |
| C (`1`) | old fiber `1` | `expanded: true` | **C expanded** — it "stole" B's |
| — | old fiber `2` | — | deleted |

No exception, no warning, nothing in the console. The state stayed loyal to *positions*; the data slid one notch underneath it. Every index-key bug — scrambled checkboxes, drafts under the wrong row, the wrong row animating — is this table with different nouns.

### A key match doesn't override a type mismatch

The heuristics apply in order: a matched key reuses the fiber **only if the `type` also matches**. Same position, same key, but a different component — heuristic 1 wins, teardown and remount. The production shape of this trap is the inline-edit toggle:

```tsx
// ❌ Toggling mode remounts the row — different type at the same keyed position
{rows.map((r) =>
  editingId === r.id
    ? <EditRow key={r.id} row={r} />
    : <ViewRow key={r.id} row={r} />
)}
```

Enter edit mode: `ViewRow` unmounts (its state, its DOM, any in-flight transition — gone), `EditRow` mounts cold. Exit: same again in reverse, so the edit draft can't survive an accidental toggle either. The symptom reads as "flicker" or "my row forgets things when I toggle," and the console is silent because nothing is wrong — you asked for a different component.

If state should survive the toggle, keep the type stable and branch *inside*:

```tsx
// ✅ One type per keyed position; the mode is data
{rows.map((r) => (
  <Row key={r.id} row={r} mode={editingId === r.id ? 'edit' : 'view'} />
))}
```

And when the reset *is* the desired behavior — a fresh edit form every time — the first version is legitimate; just choose it on purpose. (This also resolves an apparent paradox with heterogeneous lists like the activity feed in [`../foundations/jsx-and-rendering.md`](../foundations/jsx-and-rendering.md): each item renders a different component per `kind`, which is fine because any given item's kind never changes — the type at each *key* is stable, and that's all the heuristic asks.)

### Identity outside arrays: position — and the ternary that shares state

Keys govern arrays; everywhere else, identity is **position among siblings**. Two consequences show up in production:

**Conditionals hold their slot.** `{showBanner && <Banner />}` renders `false` when hidden — and `false` is still a child, occupying that position — so the `<Composer />` after it keeps its own position either way. Toggling the banner never disturbs its siblings' identity. This is *why* `null`/`false`/`undefined` render as nothing instead of being stripped from the children list: they're placeholders that keep positional identity stable.

**Ternaries share it.** Same position, same type → same fiber — and the ternary doesn't care that *you* think of the branches as two different things:

```tsx
// One fiber wearing two hats — internal state survives the player switch
{isPlayerA
  ? <Scoreboard player={playerA} />
  : <Scoreboard player={playerB} />}
```

Switching players just updates props on the *same* `Scoreboard` fiber. Whatever it tracks internally — streak counters, an expanded stats panel — carries straight across, and player B opens wearing player A's numbers. Nothing is wrong by React's rules: position matched, type matched, instance preserved. When each player should get their own scoreboard, say so with identity:

```tsx
// ✅ Idiomatic: one render site, explicit identity — switch = reset
<Scoreboard
  key={(isPlayerA ? playerA : playerB).id}
  player={isPlayerA ? playerA : playerB}
/>

// Also valid: two positions = two fibers (each one's state is destroyed while toggled away)
{isPlayerA && <Scoreboard player={playerA} />}
{!isPlayerA && <Scoreboard player={playerB} />}
```

So the identity rules unify into one line: **position by default, key when you say otherwise, and type always able to veto.** Lists are just the place where position stops being trustworthy, which is why they demand keys.

## Basic usage

The canonical forms, with the placement rule that trips people:

```tsx
// The key goes on the element the map RETURNS — the outermost one
{tickets.map((t) => (
  <TicketRow key={t.id} ticket={t} />
))}

// NOT buried inside the component's own JSX — this key does nothing for the list:
function TicketRow({ ticket }: { ticket: Ticket }) {
  return <li key={ticket.id}>…</li>;   // ❌ wrong layer — React already matched by then
}
```

```tsx
// Multi-element items: the long-form Fragment carries the key
{terms.map((t) => (
  <Fragment key={t.id}>
    <dt>{t.term}</dt>
    <dd>{t.definition}</dd>
  </Fragment>
))}
```

```tsx
// Nested lists: keys are scoped per sibling set — reusing values across levels is fine
{groups.map((g) => (
  <section key={g.id}>
    <h3>{g.title}</h3>
    <ul>
      {g.items.map((item) => (
        <li key={item.id}>{item.label}</li>   // only needs uniqueness among g.items
      ))}
    </ul>
  </section>
))}
```

## Walkthrough — a triage queue that corrupts, then doesn't

The scenario: a support triage queue. Each ticket row has local UI state (`expanded`) and an uncontrolled notes `<textarea>` (agents jot before committing). Tickets can be dismissed, and new tickets **prepend** from polling. This is the exact shape where index keys detonate.

### Step 1 — the natural first draft (with the landmine)

```tsx
// ticket.ts
export interface Ticket {
  id: string;
  subject: string;
  from: string;
}
```

```tsx
// TicketRow.tsx
import { useState } from 'react';
import type { Ticket } from './ticket';

interface TicketRowProps {
  ticket: Ticket;
  onDismiss: () => void;
}

export function TicketRow({ ticket, onDismiss }: TicketRowProps) {
  const [expanded, setExpanded] = useState(false);

  return (
    <li className="ticket">
      <button onClick={() => setExpanded((e) => !e)} aria-expanded={expanded}>
        {ticket.subject} — {ticket.from}
      </button>
      <button onClick={onDismiss} aria-label={`Dismiss ${ticket.subject}`}>✕</button>
      {expanded && (
        <div className="ticket__detail">
          <textarea placeholder="Triage notes…" defaultValue="" />
        </div>
      )}
    </li>
  );
}
```

```tsx
// TriageQueue.tsx — first draft
{tickets.map((t, index) => (
  <TicketRow
    key={index}                              // ← the landmine
    ticket={t}
    onDismiss={() => dismiss(t.id)}
  />
))}
```

It demos perfectly: expand, collapse, type notes, dismiss the *last* row — all fine, because none of those moves data relative to position.

### Step 2 — reproduce the corruption

Expand ticket #2, type half a note into it, then dismiss ticket #1. On screen: **ticket #3 is now the expanded one, and your half-written note sits under it.** Two separate corruptions with one cause:

- `expanded` is React state on the fiber — fibers matched by index, data shifted underneath (the trace table above, live).
- The note draft is worse: it's an **uncontrolled** textarea, so the text lives in the *DOM node*, and the DOM node also belongs to the fiber. Position keeps the node; the ticket under it changed.

The polling prepend is the same bug without anyone deleting anything: a new ticket arrives at the top, every element shifts down one index, and the row you're mid-typing in now "means" the previous ticket — plus every row's props changed, so every row re-renders for one insertion.

### Step 3 — the fix is one attribute

```tsx
// TriageQueue.tsx — fixed
{tickets.map((t) => (
  <TicketRow key={t.id} ticket={t} onDismiss={() => dismiss(t.id)} />
))}
```

Re-run the reconciliation in your head with ids: dismissing ticket #1 removes exactly the fiber keyed `t1` — every other fiber matches its own key and is untouched; #2 stays expanded, the draft stays under #2. The prepend inserts exactly one new fiber at the top; existing fibers (and the textarea you're typing in, and its focus) don't move in identity, only in position — React relocates the DOM nodes and everything rides along.

```tsx
// TriageQueue.tsx — complete
import { useState } from 'react';
import type { Ticket } from './ticket';
import { TicketRow } from './TicketRow';

export function TriageQueue({ initialTickets }: { initialTickets: Ticket[] }) {
  const [tickets, setTickets] = useState(initialTickets);

  function dismiss(id: string) {
    setTickets((prev) => prev.filter((t) => t.id !== id));
  }

  function receive(ticket: Ticket) {
    setTickets((prev) => [ticket, ...prev]);   // prepend — safe now
  }

  return (
    <section aria-label="Triage queue">
      <ul>
        {tickets.map((t) => (
          <TicketRow key={t.id} ticket={t} onDismiss={() => dismiss(t.id)} />
        ))}
      </ul>
    </section>
  );
}
```

### Step 4 — spend a key on purpose: the detail pane

Add a master-detail layout: clicking a row selects it; a `TicketDetail` pane shows a reply composer with its own draft state. Requirement: **switching tickets resets the composer** — a draft for #2 must never appear under #5.

```tsx
{selected && <TicketDetail key={selected.id} ticket={selected} />}
```

Same mechanism, opposite intent: a changed key is a *guaranteed* teardown of every piece of state inside `TicketDetail`, however deeply nested — no reset effects, no imperative `clear()` calls, no forgotten field three components down. The destroy-column of the table above, used as a feature.

## Real-world patterns

### Where keys come from — the decision ladder

| Source | Example | Verdict |
| --- | --- | --- |
| Server/database id | `key={ticket.id}` | The default. Stable, unique, already there |
| Natural composite | `key={`${row.userId}-${row.date}`}` | Fine when genuinely unique-per-sibling and stable — beware composites built on *mutable* fields |
| Generated **at creation time** | `id: crypto.randomUUID()` in the "add item" handler | Correct for client-created rows that lack server ids — the id is born with the data and never changes |
| Generated **at render time** | `key={crypto.randomUUID()}` in the map | Never. New key every render = remount every render: state wiped, focus lost, N× the work |
| Array index | `key={index}` | Last resort, and only when *all* hold: the list never reorders/filters/prepends, items carry no state, and length changes only at the end. Leave a comment, because the requirements will change before the comment does |

### Editable content is never the key

Tempting and catastrophic: `key={todo.text}` on a list where the text is editable. Every keystroke changes the key → new identity → **remount mid-keystroke** → the input you're typing in is destroyed and recreated, dropping focus each character. The key must be the part of the item that *doesn't* change when the item changes.

### Sort and filter are derived views — keys are what make that safe

Client-side sorting and filtering should be computed during render, per the derive-don't-store discipline from [`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md):

```tsx
const [sortBy, setSortBy] = useState<'newest' | 'sender'>('newest');
const [query, setQuery] = useState('');

const visible = tickets
  .filter((t) => t.subject.toLowerCase().includes(query.toLowerCase()))
  .toSorted(sortBy === 'newest' ? byCreatedDesc : bySender);   // toSorted, never sort — no mutating the source

{visible.map((t) => <TicketRow key={t.id} ticket={t} />)}
```

Notice what the keys are doing here: every keystroke and every sort toggle produces a *differently ordered, differently sized* array, and none of it is scary — fibers follow ids, so expansion state, drafts, and focus stay glued to their tickets while the projection reshuffles around them. This is the pairing to internalize: **derived views make reordering free to express; stable keys make it free to render.** With index keys, this exact code is the walkthrough's corruption on every keystroke — which is why "we can't add client-side sorting, it breaks the rows" is a bug report about keys, not about sorting.

### Stable keys are what make reordering features possible

Drag-and-drop, sortable columns, "move to top" — all of them are reorders, and reorders are exactly where position-based identity lies maximally. With stable ids, a reorder is pure relocation: state, focus, and animations travel with the row. This is also the contract exit-animation libraries build on — they detect "this key disappeared" to animate an item out; index keys tell them the *last* item always leaves, so the wrong row animates every time.

### Keys are sibling-scoped — stop globally prefixing

`key={`ticket-${t.id}`}` everywhere, "to be safe," is noise: uniqueness is only checked among one element's children. The same `t.id` can key a row in the queue, a chip in the filter bar, and an option in a dropdown simultaneously. Prefixes earn their place in exactly one case: heterogeneous lists merged from sources whose ids can collide (`key={`${item.source}:${item.id}`}` for a feed mixing tickets and alerts).

### Keys are invisible to the DOM

`key` never renders as an attribute and can't be targeted by CSS or e2e selectors. When tests or styling need per-item hooks, that's `data-testid={ticket.id}` / `data-ticket-id` — deliberate, visible attributes. Reaching for `key` to "tag" DOM is a category error; it lives entirely in the reconciler.

## Common mistakes

**Index keys on a dynamic list.** The headliner; the walkthrough is the autopsy. The lint rule catches literal `key={index}`; it can't catch `key={i}` laundered through a variable — review for the pattern, not the token.

**Random keys per render.** `key={Math.random()}` or `uuid()` inside the map "fixes" the duplicate-key warning by remounting the universe every render. If the warning fired, the *data* lacks identity — fix that at creation time.

**Duplicate keys.** Two siblings sharing a key is undefined behavior wearing a console error: updates skip rows, deletes take the wrong one. Usual causes: keying on a non-unique field (name, date) or double-inserting the same record — the key did its job by *revealing* the data bug.

**Key on the wrong layer.** The key belongs on the element the `.map()` returns. Keying something inside the component does nothing for list matching; keying a wrapper *and* the inner element does nothing extra.

**`key` from mutable/edited fields.** The remount-per-keystroke bug above. Also its quieter cousin: composite keys including a status field, so changing status "deletes" one row and "creates" another — state gone, exit/enter animations firing on an update.

**Reading `props.key`.** Always `undefined`, plus a warning. Need the id inside? Pass it twice: `<Row key={t.id} id={t.id} />` — one for the reconciler, one for you.

**Spreading a key in.** `<Row {...props} />` where `props` contains `key` — React 19 warns; the transform can't extract what it can't see. `<Row key={id} {...rest} />` ([`../foundations/jsx-and-rendering.md`](../foundations/jsx-and-rendering.md)).

**The `?? index` fallback.** `key={item.id ?? index}` looks defensive and quietly splits the list into two identity regimes — the id-less subset carries every index-key bug, now unfindable because the code "uses ids." If some items lack ids, mint them at creation.

**Expecting keys to skip re-renders.** Keys are identity, not memoization — a keyed row still re-renders whenever its parent does (fresh element, fresh props object, per [`../foundations/components-and-props.md`](../foundations/components-and-props.md)). Keys decide *which fiber* receives a render, never *whether* one happens; skipping is the Compiler's and `memo`'s department. The reverse confusion also ships: slapping `memo` on rows while the keys are broken fixes nothing, because remounts bypass memoization entirely — identity first, then skipping.

**Keying to force updates that props should drive.** Changing a key to make a child "notice" new data is a remount papering over a component that wrongly copied props into state ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)). Key-resets are for *discarding* state on subject change — not a substitute for rendering from props.

## How this evolved

| Era | Change | Why it matters now |
| --- | --- | --- |
| 2013 | Keys ship with the original reconciler | The heuristics — type match, key match — predate almost everything else and haven't changed |
| React 16 (2017) | Fiber: keys become fiber identity | The "what survives" table becomes literally true — state, DOM, and effects all ride the keyed fiber |
| React 16.2 (2017) | Keyed `<Fragment>` | Multi-element list items without wrapper `div`s |
| React 18 (2022) | Concurrent rendering | Interrupted/replayed renders make stable identity *more* load-bearing, not less |
| React 19 (2024) | Spread-`key` deprecated; `key` stays outside `props` | Keys must be statically visible to the transform; the reserved-word status is now total |

The design has been boring-stable for a decade — which is the point. Keys are the piece of React you learn once and never relearn; the only thing that changes is how expensive it gets to ignore them as apps grow interactive.

## Exercises

### 1. Trace the checkboxes

Five rows `[A, B, C, D, E]` with index keys; each row holds `useState(false)` for a checkbox. The user checks B and D, then the app removes C. Write the resulting table (fiber key → state → data shown) and state which rows *appear* checked.

*Hint: new elements are `[A→0, B→1, D→2, E→3]`; fibers 0–3 survive with their state `[✗, ✓, ✗, ✓]`; fiber 4 (unchecked) is the one deleted. On screen: **B checked** (correct, by luck — its position didn't shift) and **E checked** (it inherited fiber 3, where D's check lives); **D appears unchecked**. Nothing was "lost" — every check is exactly where position left it, which is the whole disease.*

### 2. The filter is a delete in disguise

A searchable checklist filters rows as the user types, rows keyed by index, each row expandable. Explain — using the matching algorithm, not vibes — why typing in the search box scrambles which rows are expanded, then fix it. Bonus: why does *clearing* the search appear to "randomly" expand rows the user never touched?

*Hint: filtering re-indexes the survivors every keystroke, so expansion state re-attaches to whatever now occupies each position; clearing restores the full list and the state map is now shifted garbage. Stable ids make filter/unfilter a pure add/remove of fibers whose state never moved.*

### 3. Duplicate-a-row, done right

Add a "Duplicate" button to the triage queue: inserts a copy of the ticket directly below the original. Make it correct with respect to identity, and state what must *not* be copied.

*Hint: the copy needs a **new id minted in the handler** — `{ ...t, id: crypto.randomUUID() }` — created with the data, not at render. Copying `t.id` gives duplicate keys; generating in the map gives remount storms. The row's *UI state* (expanded, drafts) doesn't copy either — it lives on the fiber, not the data, and the new fiber starts fresh. Is that the UX you want? That question is the exercise.*

## Summary

You learned how React decides identity: type match keeps an instance, type change destroys a subtree, and dynamic children match by `key` through a single-pass, map-assisted diff (append-optimal, move-to-front degenerate). Identity is what persistence hangs on — state, DOM nodes, focus, uncontrolled values, effects, animations all ride the keyed fiber — so index keys don't merely warn, they re-attach all of that to positions while data slides underneath, exactly as the triage-queue walkthrough reproduced. The conventions that make the bug unrepresentable: keys from stable identity (server id → natural composite → minted-at-creation UUID), never from indexes on dynamic lists, never from editable fields, never generated during render; keys on the mapped element, sibling-scoped, invisible to the DOM. And the same mechanism, pointed forward, is a tool: changing a key on purpose is React's one-attribute total reset.

## See also

- [`../foundations/jsx-and-rendering.md`](../foundations/jsx-and-rendering.md) — where `key` lives on the element, `jsx` vs `jsxs`, keyed fragments
- [`../state/state-and-usestate.md`](../state/state-and-usestate.md) — the key-reset pattern from the state side; why state rides the fiber
- [`how-react-renders.md`](how-react-renders.md) — the full reconciliation and commit machinery this article's algorithm lives inside
- [`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md) — the render loop that produces the element arrays being matched
- [`../foundations/components-and-props.md`](../foundations/components-and-props.md) — reference identity and bailouts, the sibling discipline to key identity

## References

- [Rendering Lists (react.dev)](https://react.dev/learn/rendering-lists)
- [Preserving and Resetting State (react.dev)](https://react.dev/learn/preserving-and-resetting-state)
- [Reconciliation (legacy docs — still the canonical algorithm description)](https://legacy.reactjs.org/docs/reconciliation.html)
- [`Fragment` API — keyed fragments (react.dev)](https://react.dev/reference/react/Fragment)
- [`useState` — resetting state with a key (react.dev)](https://react.dev/reference/react/useState#resetting-state-with-a-key)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.