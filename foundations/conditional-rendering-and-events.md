---
article_id: conditional-rendering-and-events
concept_folder: foundations
wave: 1
related:
  - foundations/jsx-and-rendering
  - state/state-and-usestate
  - rendering/rendering-lists-and-keys
  - effects/effects-and-synchronization
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Conditional Rendering and Events

> **Lead with this:** These are the two verbs of interactivity: *branch* — choose which description to return for the current state — and *respond* — turn user input into state changes that produce the next description. Both look trivial in a demo and both hide production-grade machinery: conditionals raise real architecture questions (who owns the condition? does hiding mean unmounting? does a `null`-returning component actually cost nothing?), and React's event system is a delegation layer with its own semantics — `onChange` that isn't the DOM's `change`, focus events that bubble when native ones don't, passive listeners that silently eat `preventDefault`, and an event-ordering trap (`blur` before `click`) that has broken more Cancel buttons than any other single fact in this article. The pieces already introduced — the `&&`/`0` bug, position-holding, snapshots in handlers — get cross-referenced, not re-taught; this article owns the rest.

## What it is

**Conditional rendering** is nothing more than JavaScript control flow deciding which elements your function returns. There's no directive, no template syntax — early returns, ternaries, `&&`, extracted variables, `switch` — because a render *is* a function call ([`thinking-in-react.md`](thinking-in-react.md)) and elements are values ([`jsx-and-rendering.md`](jsx-and-rendering.md)). The craft is not in the mechanics but in three decisions: **which construct** reads best for the state being expressed, **where the condition lives** (the parent's call site or inside the component), and **whether "not shown" should mean unmounted or merely hidden** — three decisions with genuinely different behavior, covered below.

**Event handlers** are functions you pass (never call) as `on*` props. They are the *only* place in a component that's supposed to cause things: state updates, mutations, navigation, analytics. Renders describe; handlers act. When you're deciding where a side effect belongs, the first question is always "is there a user interaction this belongs to?" — and if yes, it goes in the handler, not an effect ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) formalizes this as the event-first rule). Handlers close over the render's snapshot ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)), receive a **synthetic event** — React's typed, normalized wrapper around the native one — and participate in a delegation system whose mechanics are worth ten minutes of your attention exactly once.

## How it works under the hood

### One listener at the root: delegation

React does not call `addEventListener` on your buttons. At `createRoot(container)`, it attaches **one listener per event type on the container**, lets native events bubble up to it, works out which fiber the event targeted, and then simulates the capture and bubble phases *through your component tree*, calling your `onClickCapture`/`onClick` props along the way.

Consequences that matter in real apps:

- **Propagation follows the React tree, not the DOM tree.** A click inside a portal bubbles to the portal's React *parent*, even though its DOM sits in `document.body` — the behavior your modals rely on ([`../rendering/portals-and-the-event-system.md`](../rendering/portals-and-the-event-system.md) owns the details).
- **`e.stopPropagation()` in a React handler stops the native event too** — nothing above the root (e.g., a `document`-level listener some analytics or outside-click library installed) will hear it. Powerful, and exactly why "stop propagation everywhere" is an architecture smell (mistakes list).
- Pre-17, delegation targeted `document` instead of the root — the source of a generation of "two React apps / jQuery interop" bugs. The root change (React 17) made multiple React versions on one page coexist; if you ever debug a legacy embed, this is the fact you need.

### The synthetic event

The object your handler receives wraps the native event with normalized, camelCased, fully typed properties, and exposes the original as `e.nativeEvent`. Since React 17 it's a plain per-event object — no pooling, safe to read after `await`:

```tsx
// legacy: pre-17 event pooling — the event object was reused across events,
// so async access required e.persist(). Removed in React 17; modernized in the upgrade pass.
onClick={(e) => {
  e.persist();
  setTimeout(() => track(e.currentTarget), 0);
}}
```

Two properties carry most of the confusion budget:

```tsx
function onCardClick(e: React.MouseEvent<HTMLDivElement>) {
  e.currentTarget; // HTMLDivElement — the element this handler is attached to. Always.
  e.target;        // EventTarget — whatever was actually hit: the <svg>, a <span>, anything inside
}
```

`currentTarget` is what the generic types; `target` is untyped `EventTarget` because delegation means it can be any descendant. The one deliberate exception: `ChangeEvent<HTMLInputElement>` types `target` as the input — which is why `e.target.value` type-checks in `onChange` and almost nowhere else.

### Semantics that differ from the DOM — on purpose

- **`onChange` fires on every input**, like the native `input` event — *not* like the native `change` event (which waits for commit/blur). React unified them because per-keystroke is what controlled inputs need ([`../forms/forms-controlled-and-uncontrolled.md`](../forms/forms-controlled-and-uncontrolled.md)). When you want commit semantics — validate on leave, save on done — that's `onBlur` or form submission, not `onChange`.
- **`onFocus`/`onBlur` bubble.** Native `focus`/`blur` don't; React implements them with `focusin`/`focusout` semantics, so one handler on a container can track focus for a whole group — and `FocusEvent.relatedTarget` tells you where focus went (or came from), which the walkthrough will need. `relatedTarget` is `null` when focus leaves the document entirely.
- **`onMouseEnter`/`onMouseLeave` don't bubble** (matching native) — container-level hover tracking wants `onMouseOver`/`onMouseOut` or, better, `onPointerEnter` on the right element.
- **Some root listeners are passive.** React registers `wheel`, `touchstart`, and `touchmove` as passive listeners for scroll performance — which means `e.preventDefault()` inside `onWheel`/`onTouchMove` **cannot block scrolling** and fails with a console warning at best. Blocking scroll/zoom (canvas tools, custom map panning) requires a native non-passive listener attached via a ref in an effect — the escape hatch is four lines, and it lives in [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) where its cleanup can be done justice.

## Basic usage

### The conditional toolbox, and when each earns its place

```tsx
function OrderPanel({ order }: { order: Order | null }) {
  // 1. Guard clauses — whole-component branching, plus TypeScript narrowing:
  if (!order) return <EmptyState label="No order selected" />;
  // below this line, `order` is Order — every access is narrowed, free

  // 2. Extracted variable — when a branch is too big to inline but too local to extract:
  const eta = order.express
    ? <strong className="eta eta--express">{order.eta}</strong>
    : <span className="eta">{order.eta}</span>;

  return (
    <section>
      <h2>Order {order.number}</h2>
      {/* 3. Ternary — a real either/or, both branches meaningful */}
      {order.paid ? <PaidBadge /> : <PayNowButton orderId={order.id} />}
      {/* 4. && — genuinely optional extras; comparison when the left side can be a number */}
      {order.notes.length > 0 && <NotesList notes={order.notes} />}
      <p>Arriving {eta}</p>
    </section>
  );
}
```

One placement rule is non-negotiable: **guard clauses go after every hook call.** Hooks are matched by call order on the fiber ([`thinking-in-react.md`](thinking-in-react.md)), so a `return` above a `useState` makes the hook conditional and corrupts the whole list. Hooks first, guards second, JSX last — muscle memory worth building in week one.

For state machines, the conditional *is* a `switch` over the union, exhaustively — the same discipline as the activity feed in [`jsx-and-rendering.md`](jsx-and-rendering.md):

```tsx
switch (search.status) {
  case 'idle':    return <SearchHint />;
  case 'loading': return <Spinner label={`Searching “${search.query}”…`} />;
  case 'success': return <Results items={search.results} />;
  case 'error':   return <SearchError message={search.message} />;
  default:        return search satisfies never;
}
```

### Handlers: pass, wrap, type

```tsx
function Row({ item, onArchive }: { item: Item; onArchive: (id: string) => void }) {
  // Local handlers: handleX. Handler props: onX, named for intent (thinking-in-react).
  function handleKeyDown(e: React.KeyboardEvent<HTMLLIElement>) {
    if (e.key === 'Delete') onArchive(item.id);
  }

  return (
    <li tabIndex={0} onKeyDown={handleKeyDown}>
      {item.label}
      <button onClick={() => onArchive(item.id)}>Archive</button>  {/* wrap to pass args */}
      {/* onClick={onArchive(item.id)} would CALL it during render — the loop from
          state-and-usestate; pass functions, never invocations */}
    </li>
  );
}
```

Inline arrows are the idiomatic way to bind arguments, and under this project's Compiler-first stance they're also *free* — the Compiler stabilizes them in compiled code, so there's no `useCallback` tax to pay for readability ([`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md)).

## Walkthrough — an editable title, and the event-ordering bug that eats Cancel buttons

The scenario: a document title that's a heading until you click **Edit**, then becomes an input with **Save**/**Cancel**, Enter-to-save, Escape-to-cancel, save-on-blur, and an inline validation error. Small feature, and it exercises every sharp edge in this article — including one genuine war story.

### Step 1 — model the modes as a union

```tsx
type Mode =
  | { name: 'viewing' }
  | { name: 'editing'; draft: string; error: string | null };
```

The draft lives *inside* the editing mode — it doesn't exist while viewing, so the type says so. One `setMode` per transition keeps every switch atomic ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)).

### Step 2 — branch per mode, hooks above the branch

```tsx
import { useState } from 'react';

interface EditableTitleProps {
  value: string;
  onSave: (next: string) => void;
}

export function EditableTitle({ value, onSave }: EditableTitleProps) {
  const [mode, setMode] = useState<Mode>({ name: 'viewing' });   // hooks FIRST

  if (mode.name === 'viewing') {                                  // guards after
    return (
      <h2 className="title">
        {value}
        <button onClick={() => setMode({ name: 'editing', draft: value, error: null })}>
          Edit
        </button>
      </h2>
    );
  }

  // From here down, TypeScript knows mode is the editing member — draft and error exist.
  /* …editing branch, built in steps 3–4… */
}
```

Note what makes this branch-swap *safe*: viewing renders an `<h2>`, editing renders a `<form>` — a type change at the same position, which [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md) taught you tears the subtree down. That's fine here **because the state that matters (`mode`, and the draft inside it) lives above the swap** — the subtrees being destroyed are stateless projections. Swap types freely when the state is above the swap; keep the type stable when it isn't.

### Step 3 — commit and cancel, through the right events

```tsx
  function commit() {
    const trimmed = mode.draft.trim();
    if (trimmed.length === 0) {
      setMode({ ...mode, error: 'Title cannot be empty' });
      return;
    }
    onSave(trimmed);
    setMode({ name: 'viewing' });
  }

  function cancel() {
    setMode({ name: 'viewing' });
  }

  return (
    <form
      className="title title--editing"
      onSubmit={(e) => {
        e.preventDefault();   // it's a real form: Enter submits, and we own navigation
        commit();
      }}
    >
      <input
        autoFocus
        value={mode.draft}
        onChange={(e) =>
          setMode({ ...mode, draft: e.target.value, error: null })  // typed: ChangeEvent special-cases target
        }
        onKeyDown={(e) => {
          if (e.key === 'Escape') cancel();
        }}
        aria-invalid={mode.error ? true : undefined}
        aria-describedby={mode.error ? 'title-error' : undefined}
      />
      <button type="submit">Save</button>
      <button type="button" onClick={cancel}>Cancel</button>
      {mode.error && (
        <p id="title-error" role="alert" className="title__error">
          {mode.error}
        </p>
      )}
    </form>
  );
```

Deliberate choices: a real `<form>` so Enter-to-save comes from the platform (`onSubmit` + `preventDefault`) instead of a hand-rolled keydown check; `type="button"` on Cancel because inside a form the default is `submit` — omit it and Cancel *saves*; Escape via `e.key` (never `keyCode`); the error rendered conditionally with `role="alert"` and wired to the input via `aria-describedby`/`aria-invalid` — the conditional's a11y half.

### Step 4 — save-on-blur, and the trap

Add the requirement "clicking anywhere else saves the draft":

```tsx
<input onBlur={commit} /* … */ />
```

Ship it, and QA files: **"Cancel button saves instead of canceling."** Here's the sequence when a user clicks Cancel while the input has focus:

```
pointerdown/mousedown (on Cancel)  →  blur (input)  →  mouseup  →  click (Cancel)
```

`blur` fires **before** `click`. Your `onBlur` commits the draft and flips to viewing; by the time the click would run `cancel`, the form is gone — Cancel never had a chance. This ordering has broken autocomplete option-clicks, toolbar buttons on rich-text editors, and every "save on blur, but also have buttons" form ever written. Two honest fixes:

```tsx
// Fix A (chosen): keep focus from ever leaving — no blur, click proceeds normally.
// preventDefault on mousedown suppresses the focus transfer, nothing else.
<button type="button" onMouseDown={(e) => e.preventDefault()} onClick={cancel}>
  Cancel
</button>

// Fix B: let blur fire, but detect WHERE focus went (React's focusout gives relatedTarget)
<input
  onBlur={(e) => {
    const next = e.relatedTarget as Node | null;   // null when focus left the document
    if (next && e.currentTarget.form?.contains(next)) return;  // our own buttons — they decide
    commit();
  }}
/>
```

Fix A is simpler and self-contained per button; Fix B centralizes the rule on the input and survives new buttons being added — pick per situation, but pick *knowingly*. (Submit doesn't need either fix here — `requestSubmit` semantics fire submit even as blur commits first? No: Save going through `commit` twice is idempotent by construction, which is its own quiet lesson — make commit safe to double-fire.)

## Real-world patterns

### Who owns the condition: the call site or the component?

A component that returns `null` **still mounts**: its hooks run, its effects fire, its subscriptions open — it's a full citizen of the tree that happens to render nothing:

```tsx
// Self-hiding: usePriceStream connects even when invisible
function LivePriceBadge({ visible, symbol }: { visible: boolean; symbol: string }) {
  const price = usePriceStream(symbol);   // ← runs regardless
  if (!visible) return null;
  return <span className="badge">{formatPrice(price)}</span>;
}

// Parent-decides: nothing mounts at all
{visible && <LivePriceBadge symbol={symbol} />}
```

Default to **parent-decides** — it's honest about cost and keeps components single-purpose. Self-hiding earns its place when the component *owns the knowledge* (a permission, a feature flag) and is cheap when hidden; and the best version of that is making ownership explicit with a wrapper whose entire job is the condition:

```tsx
<RequireRole role="admin">
  <DangerZone />
</RequireRole>
// function RequireRole({ role, children }) { return hasRole(role) ? children : null; }
```

### Unmount, hide, or inert — three different "not shown"s

| | Unmount — `{cond && …}` | Hide — `hidden` attr / `display:none` | `inert` attribute |
| --- | --- | --- | --- |
| Component state | destroyed | preserved | preserved |
| Effects/subscriptions | cleaned up | keep running | keep running |
| DOM & layout cost | rebuilt on show | paid up front, kept warm | paid, and still *painted* |
| Visibility | gone | invisible, no layout | **visible** |
| Accessibility | gone from a11y tree | removed (for `display:none`/`hidden`) | in the tree but unfocusable, uninteractive |

The tab panel is the canonical decision: a cheap settings pane → unmount, take the clean teardown. A heavy editor tab with scroll position, undo history, and an initialized third-party widget → hide, keep it warm, and remember its effects are still running (pause what should pause). `inert` is the third tool for a different job — content that must stay *visible* but non-interactive: the page behind a modal, a form section pending an upgrade. One caveat with `hidden`: any CSS that sets `display` on the element overrides it — pair it with `[hidden] { display: none !important }` in a codebase with utility classes, or you'll debug a "hidden" element that isn't.

### Handlers are where writes live

The unifying discipline: renders compute, **handlers cause**. Saving, navigating, tracking, opening sockets on demand, writing storage — if it happens *because the user did something*, it belongs in the handler for that something, synchronously and directly. The anti-pattern is laundering interactions through state: set a `shouldSave` flag in the handler, "react" to it in an effect, reset the flag — three moving parts, a double-fire in StrictMode, and a race, replacing one function call. When you reach [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md), this pattern is half the article; internalizing it now makes that one easy.

### Derived handlers stay in one place

When several controls share commit-ish logic (the walkthrough's Enter, blur, and Save all call `commit`), define the verbs once inside the component and let events be thin adapters onto them. The test: reading the JSX should tell you *what* each interaction does (`commit`, `cancel`) without scrolling; reading the verbs should tell you *how*, without knowing which event triggered it. Handlers that inline twenty lines of business logic fail the first reading; verbs that take a `React.SyntheticEvent` fail the second.

## Event types quick reference

| Handler | Event type | Field notes |
| --- | --- | --- |
| `onClick`, `onPointerDown`… | `React.MouseEvent<T>` / `React.PointerEvent<T>` | Prefer pointer events for anything touch-facing — one API for mouse/touch/pen |
| `onChange` | `React.ChangeEvent<T>` | The exception where `e.target` is typed as `T`; fires per input, not per commit |
| `onSubmit` | `React.FormEvent<HTMLFormElement>` | Pair with `e.preventDefault()`; submitting is the platform's Enter handling for free |
| `onKeyDown`/`onKeyUp` | `React.KeyboardEvent<T>` | Branch on `e.key` (`'Escape'`, `'Enter'`, `'ArrowDown'`); `keyCode` is dead |
| `onFocus`/`onBlur` | `React.FocusEvent<T>` | Bubble (focusin/out semantics); `relatedTarget` = the other side of the transfer, `null` at document exit |
| `onDragOver`/`onDrop` | `React.DragEvent<T>` | Drop targets **must** `preventDefault()` in `onDragOver` or `onDrop` never fires |
| `onWheel`, `onTouchMove` | `React.WheelEvent<T>` / `React.TouchEvent<T>` | Passive at the root — `preventDefault` can't block scrolling; go native via ref+effect |
| `onCopy`/`onPaste` | `React.ClipboardEvent<T>` | `e.clipboardData` for custom paste handling |
| Any of the above | `ComponentPropsWithoutRef<'input'>['onChange']` | The zero-import way to type a handler prop identically to the native one |

Every bubbling handler has a capture twin (`onClickCapture`, …) that runs on the way *down* — rarely needed in app code; the legitimate uses (analytics that must see everything, closing layers before children react) live in [`../rendering/portals-and-the-event-system.md`](../rendering/portals-and-the-event-system.md).

## Common mistakes

**Calling instead of passing.** `onClick={save()}` runs during render; with a setter inside, it's the infinite loop from [`../state/state-and-usestate.md`](../state/state-and-usestate.md). `onClick={save}` or `onClick={() => save(id)}`.

**Guard clauses above hooks.** `if (!user) return null;` placed before `useState` makes every hook after it conditional — "Rendered fewer hooks than expected" on the first render where the guard flips. Hooks first, always; the lint rule catches most, not all.

**Treating `onChange` as commit.** Validating or persisting on every keystroke because "change" sounded like the DOM's change event. Per-keystroke work belongs in `onChange`; commit-time work belongs in `onBlur`/`onSubmit` — the walkthrough used all three deliberately.

**`e.target` when you meant `e.currentTarget`.** Click the icon inside the button and `e.target` is the `<svg>` — the `dataset` you wanted isn't there, and TypeScript warned you by typing it `EventTarget`. `currentTarget` is the element you attached to, and it's typed.

**`preventDefault` in `onWheel`/`onTouchMove`.** Silently useless — passive root listeners. The canvas/map/carousel that must block scrolling needs a native non-passive listener via ref + effect.

**Save-on-blur eating button clicks.** The ordering war story. If a blur handler commits state that removes or disables buttons, every button in the neighborhood needs Fix A or the input needs Fix B — and commit needs to be idempotent regardless.

**`null`-returning components that still pay rent.** The "hidden" badge holding a WebSocket open, the collapsed panel polling every 10s. Returning `null` unmounts nothing; move the condition to the parent, or gate the subscription on visibility.

**`stopPropagation` as architecture.** One inner handler stopping propagation to "fix" an outer one silently breaks outside-click closing, analytics delegation, and future listeners — including native ones above the root. Prefer checking `e.target` in the outer handler (`if (ref.current?.contains(e.target as Node)) return`); stop propagation only for a boundary you *own* and document.

**Ternary soup.** Nested ternaries three levels deep in JSX is control flow wearing camouflage. The fixes are the toolbox in order: early returns for whole-component branches, extracted variables for local ones, a `switch` over the union when it's secretly a state machine, a child component when it's secretly two components.

## How this evolved

| Era | Change | What it means now |
| --- | --- | --- |
| Class era | Handlers needed `this` — `.bind` in constructors or class-field arrows | The entire binding genre is legacy; function components close over what they need |
| React 16 | Synthetic events pooled; async access required `e.persist()` | Gone — but the pattern lingers in old code and old answers |
| React 17 (2020) | Delegation moves `document` → root; pooling removed; `onFocus`/`onBlur` reimplemented via `focusin`/`focusout` | Multi-app pages work; events are safe after `await`; focus bubbles |
| React 18 (2022) | Handlers batch like everything else | One render per handler regardless of setter count ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)) |
| Compiler era (2025+) | Inline handlers stabilized automatically | The `useCallback`-every-handler reflex is retired in compiled code; write the readable arrow |

```tsx
// legacy: class-era handler binding — modernized in the upgrade pass
constructor(props) {
  super(props);
  this.handleClick = this.handleClick.bind(this);
}
```

## Exercises

### 1. Event-order detective

Add `console.log` for `onMouseDown`, `onBlur` (input), `onMouseUp`, and `onClick` (button) to the walkthrough component *without* Fix A/B applied. Predict the exact log order for "user clicks Cancel while editing," then explain — in one sentence each — why Fix A produces a different log and Fix B produces the same log but different behavior.

*Hint: `mousedown → blur → mouseup → click`. Fix A removes the `blur` line entirely (focus never transfers). Fix B keeps the order but the blur handler early-returns — same logs, `cancel` finally wins the race it was losing.*

### 2. Tabs, both ways

Build `<Tabs>` with a `keepMounted` prop: `false` unmounts inactive panels (`{active === id && …}`), `true` hides them (`hidden={active !== id}`). Put a `<textarea>` draft and a ticking `console.log` interval (cheat a `useEffect` from the next article, or a `setInterval` in a child you write for this) inside one panel, then write down what differs between the modes for: the draft, the ticking, and switch-back speed.

*Hint: your observations should reproduce the unmount/hide table row by row — draft survives only with `keepMounted`, ticking continues only with `keepMounted` (that's the rent), and remount cost is the price of the clean teardown. There's no right default; there's a right default per panel.*

### 3. The dropdown that closes too soon

A search box shows a suggestions list while focused; `onBlur` hides the list; `onClick` on a suggestion fills the box. Users report clicks "do nothing" — the list vanishes first. Fix it twice: once on the suggestions (Fix A style), once on the input (Fix B style), and state which one survives adding a "clear" button next week without edits.

*Hint: this is the walkthrough's bug wearing autocomplete clothing — it's the same four-event sequence. Fix B (the `relatedTarget` containment check on a wrapping element) is the one that absorbs new interactive children for free; Fix A must be re-applied per child.*

## Summary

You learned the two verbs and their machinery. Branching is plain JavaScript over element values — guard clauses (after hooks, always) with free TypeScript narrowing, ternaries for either/ors, comparison-guarded `&&`, exhaustive switches over state unions — plus the three decisions that outrank syntax: condition at the call site vs. self-hiding (a `null` return still mounts, still subscribes), unmount vs. `hidden` vs. `inert` (state, effects, and a11y differ per column), and safe type-swaps when state lives above the swap. Responding runs through React's delegation layer: one root listener per type, propagation along the React tree, a post-pooling synthetic event with `currentTarget` typed and `target` not, `onChange` meaning per-input, focus events that bubble with `relatedTarget` attached, and passive root listeners that ignore `preventDefault`. The walkthrough's blur-before-click ordering is the one sequence to memorize — it will find you — and the discipline that ties the article to the rest of the resource: renders describe, handlers cause.

## See also

- [`jsx-and-rendering.md`](jsx-and-rendering.md) — `&&`/`0`, elements as values, the conditional constructs' compile story
- [`../state/state-and-usestate.md`](../state/state-and-usestate.md) — snapshots in handlers, batching, the setter-during-render loop
- [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md) — position-holding, type swaps, when "not shown" needs identity thinking
- [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) — the native-listener escape hatch, and the event-first/effect-last rule in full
- [`../forms/forms-controlled-and-uncontrolled.md`](../forms/forms-controlled-and-uncontrolled.md) — where `onChange`-per-keystroke becomes the controlled-input model
- [`../rendering/portals-and-the-event-system.md`](../rendering/portals-and-the-event-system.md) — React-tree propagation, capture phase, portals

## References

- [Conditional Rendering (react.dev)](https://react.dev/learn/conditional-rendering)
- [Responding to Events (react.dev)](https://react.dev/learn/responding-to-events)
- [Common components — props and events reference (react.dev)](https://react.dev/reference/react-dom/components/common)
- [Separating Events from Effects (react.dev)](https://react.dev/learn/separating-events-from-effects)
- [React v17 RC: changes to event delegation (React blog)](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html)
- [`<input>` — controlled inputs and change semantics (react.dev)](https://react.dev/reference/react-dom/components/input)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.