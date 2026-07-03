---
article_id: jsx-and-rendering
concept_folder: foundations
related:
  - thinking-in-react
  - components-and-props
  - how-react-renders
  - rendering-lists-and-keys
react_baseline: "19.2"
---

# JSX and Rendering

> **Lead with this:** JSX is not HTML and it is not a template language — it's syntax sugar for function calls that return plain JavaScript objects. Once `<Card title={x} />` reads as `jsx(Card, { title: x })` in your head, an entire class of confusion evaporates: why `{}` takes expressions but not `if` statements, why components must be capitalized, why `class` is `className`, why defining a component inside another component destroys its state, and why rendering `<Foo />` is not the same as calling `Foo()`. This article makes the compile target and the element model concrete, because every later topic — reconciliation, keys, memoization, the Compiler — operates on these objects.

## What it is

JSX is an XML-like syntax extension to JavaScript, compiled away entirely at build time. There is no JSX at runtime — no HTML strings, no `innerHTML`, no parsing. Your build tool (Vite's Oxc transform, in this project's baseline) rewrites it into calls to the **automatic JSX runtime**:

```tsx
// What you write
const el = <button className="primary" onClick={save}>Save</button>;

// What ships (React 17+ automatic runtime)
import { jsx } from 'react/jsx-runtime';
const el = jsx('button', { className: 'primary', onClick: save, children: 'Save' });
```

```tsx
// legacy: pre-17 classic transform — every file needed `import React` in scope
const el = React.createElement(
  'button',
  { className: 'primary', onClick: save },
  'Save',
);
```

Two rules fall directly out of "it's just function calls":

**The curly braces take expressions, not statements.** `{count + 1}`, `{items.map(...)}`, `{isAdmin ? <Panel /> : null}` all work because each is an expression with a value. `{if (x) ...}` cannot work — `if` has no value. This isn't a JSX limitation to memorize; it's JavaScript.

**Capitalization is semantic, not stylistic.** `<button>` compiles to `jsx('button', ...)` — a *string* type, a host element. `<Button>` compiles to `jsx(Button, ...)` — the *function itself* passed by reference. Lowercase means "DOM tag name as data"; capitalized means "component in scope." Rename a component to lowercase and React will try to create an HTML element called `<mycomponent>`.

And the attribute differences (`className`, `htmlFor`, camelCased events) exist because these are JavaScript property names on an object literal, where `class` and `for` are reserved words.

## How it works under the hood

### Anatomy of a React element

The object `jsx()` returns is a **React element** — the atomic unit everything else consumes:

```ts
{
  $$typeof: Symbol.for('react.transitional.element'), // brand (React 19 name)
  type: 'button',        // string → host element; function → component
  key: null,             // reconciliation identity — deliberately NOT in props
  props: {
    className: 'primary',
    onClick: save,
    children: 'Save',
    // ref lives here too as of React 19 — it's a normal prop now
  },
}
```

Worth knowing at a principal level:

- **`$$typeof` is a security measure.** It's a `Symbol`, and symbols don't survive `JSON.parse`. If an attacker smuggles a crafted object through your API into something you render, React refuses it because the brand can't be forged from JSON. This is why "we render server data" is not automatically an injection hole — the holes are elsewhere (see the XSS pattern below).
- **`key` is extracted, not passed through.** React pulls `key` off before building `props`, which is why reading `props.key` inside a component logs a warning and gives you nothing. It belongs to the reconciler, not to you.
- **Elements are immutable and disposable.** Every render creates a brand-new tree of them. That sounds wasteful and isn't — they're small object literals, and creating them is the cheap phase. The expensive phase (DOM mutation) is what the diffing exists to minimize.

### Children normalization — and the `0` that ships to production

`children` can be an element, an array of elements, a string, a number, or the "render nothing" family: `null`, `undefined`, `true`, `false`. That last group is the entire reason `&&` works as a conditional:

```tsx
{isAdmin && <AdminPanel />}   // false → renders nothing
```

But **numbers render**. `0` is falsy in JavaScript and visible in the DOM:

```tsx
{cart.items.length && <CartBadge />}   // empty cart → the page shows "0"
```

This one genuinely ships to production because it passes every test with a non-empty cart. The convention in this project: comparisons, not truthiness, when the left side can be a number — `{cart.items.length > 0 && <CartBadge />}`.

### Element ≠ component ≠ instance

Three words that get used interchangeably and mean different things:

- A **component** is the function — the blueprint. `function Card(props) {...}`
- An **element** is a description — the object. `<Card title="Hi" />` produces one. **The function has not been called yet.**
- An **instance** is what React maintains internally per rendered position (a fiber, plus the real DOM node for host elements). You never construct these; React does, during render.

The gap between "element created" and "function called" is where two production bugs live:

**Calling a component instead of rendering it.** `{Card({ title })}` runs the function *inside the parent's render*, right now. Any hooks inside `Card` register against the *parent's* fiber — hook order corrupts the moment the call becomes conditional, and you get the infamous "Rendered fewer hooks than expected" crash. `<Card title={...} />` defers the call to React, which gives `Card` its own instance, own state, own place in DevTools. The two are never interchangeable.

**Defining a component inside a component.** Every parent render creates a *new function identity* for the inner component. Reconciliation matches positions by `type` reference; a new reference means "different component" → unmount the old, mount the new, **state destroyed**. The symptom in the wild is weirdly specific: *"my input loses focus on every keystroke"* — the keystroke updates parent state, parent re-renders, redefines the inner component, React remounts it, focus dies. The fix is always the same: hoist the component to module scope and pass data via props.

### One more DOM reality: the browser edits your markup

Browsers auto-correct invalid HTML nesting — a `<div>` inside `<p>` closes the paragraph; a `<tr>` outside `<tbody>` gets reparented. Client-only, you might never notice. Under SSR, the server streams the markup you *described*, the browser *corrects* it, and hydration then compares React's description against a DOM that no longer matches — mismatch errors that look haunted until you know the cause. Valid nesting isn't pedantry; it's a hydration requirement (full story in [`../server/ssr-and-hydration.md`](../server/ssr-and-hydration.md)).

## Basic usage

The forms you'll write daily, annotated with what they compile to conceptually:

```tsx
interface ProfileCardProps {
  user: { name: string; unread: number; avatarUrl: string };
  onOpen: () => void;
}

export function ProfileCard({ user, onOpen }: ProfileCardProps) {
  return (
    <>                                          {/* Fragment: children without a wrapper node */}
      <img src={user.avatarUrl} alt="" />       {/* expression holes for any attribute */}
      <h2>{user.name}</h2>                      {/* text child */}
      {user.unread > 0 && (                     {/* comparison, not truthiness */}
        <span className="badge">{user.unread}</span>
      )}
      <button onClick={onOpen}>Open</button>    {/* handler passed by reference, not called */}
    </>
  );
}
```

Spread props exist and compile to object spread — `<Input {...fieldProps} />` → `jsx(Input, { ...fieldProps })`. Powerful for wrapper components, easy to abuse (covered in [`components-and-props.md`](components-and-props.md)).

## Real-world patterns

### Elements are values — treat them like values

Because an element is just an object, everything you do with values works: assign to variables, return early, put them in Maps, pass them as props.

```tsx
export function OrderStatus({ order }: { order: Order }) {
  if (order.cancelled) return <CancelledNotice reason={order.cancelReason} />;

  const eta = order.express ? <strong>{order.eta}</strong> : <span>{order.eta}</span>;
  return <p>Arriving {eta}</p>;
}
```

Early returns beat nested ternaries for whole-component branching; ternaries beat `&&` when there's a real either/or; `&&` is for genuinely optional fragments. Pick per case — this project's convention is *readability of the state being expressed*, not one operator everywhere.

### Config-driven rendering with a component map

Component references are values too. A capitalized local variable renders as a component — the standard pattern for data-driven UI:

```tsx
const STATUS_ICON = {
  ok: CheckCircle,
  warn: AlertTriangle,
  error: XOctagon,
} as const satisfies Record<Status, ComponentType<{ className?: string }>>;

export function StatusIcon({ status }: { status: Status }) {
  const Icon = STATUS_ICON[status];   // must be Capitalized to render as a component
  return <Icon className={`icon-${status}`} />;
}
```

The `satisfies` keeps the map exhaustive — add a `Status` variant and TypeScript flags the missing icon at the map, not at a runtime crash three screens away.

### Keyed fragments for grouped list items

When each list item produces multiple siblings with no wrapper, the long-form `Fragment` takes the key:

```tsx
{terms.map((t) => (
  <Fragment key={t.id}>
    <dt>{t.term}</dt>
    <dd>{t.definition}</dd>
  </Fragment>
))}
```

### The one true injection hole: `dangerouslySetInnerHTML`

JSX auto-escapes text children — `{userInput}` renders `<script>` as harmless text. The XSS surface in a React app is narrow and specific: raw HTML insertion, and attacker-controlled URLs (`href={userInput}` with a `javascript:` payload — React 19 blocks the worst of these, but treat URLs as untrusted anyway). When you must render rich HTML (CMS content, markdown output), sanitize at the boundary and centralize it:

```tsx
import DOMPurify from 'dompurify';

export function RichText({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />;
}
```

One audited component, greppable name, sanitizer in exactly one place. Scattered `dangerouslySetInnerHTML` across feature code is how the hole reopens six months later.

## Common mistakes

**Rendering `0` (or `NaN`) via `&&`.** Covered above — use `> 0` when the left operand can be a number. The lint rule for this is worth enabling; the bug is invisible in happy-path testing.

**Defining components inside components.** The remount-per-render bug. Presents as focus loss, animation restarts, or state resetting "randomly." Hoist to module scope, always — even for tiny helpers.

**Calling components as functions.** `{renderItem()}` where `renderItem` contains hooks is a fiber-identity bug waiting for its first conditional. If it has hooks, it's a component: render it as `<Item />`.

**Mutating an element after creating it.** Elements are frozen in development; the mutation silently no-ops or throws. If you need a modified copy, `cloneElement` exists but is a smell — restructure so the right props are passed at creation.

**Rendering plain objects.** `Objects are not valid as a React child` — usually a `Date`, a Decimal, or a whole record where a field was meant. Format at the leaf: `{order.createdAt.toLocaleDateString()}`.

**String "booleans".** `<Input disabled="false" />` is a *truthy string* — the input is disabled. Booleans go through expression holes: `disabled={false}`, or bare presence for `true`.

**Invalid HTML nesting.** Fine until SSR turns it into hydration mismatches. `<p>` can't contain block elements; table parts belong in table structure. Treat the HTML content model as part of correctness, not styling.

## How this evolved

| Era | Change | Why it matters now |
| --- | --- | --- |
| Pre-17 | Classic transform: `React.createElement`, `import React` required everywhere | You'll still see both in older codebases; the element object was the same idea |
| React 17 (2020) | Automatic runtime (`react/jsx-runtime`): no React import needed, `jsx`/`jsxs` calls | The baseline everywhere today; enabled build-tool-level optimizations |
| React 19 (2024) | `ref` becomes a normal prop inside `props`; `element.ref` removed; element brand renamed | `forwardRef` becomes legacy ceremony (see [`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md)); element internals simplified |
| Compiler era (2025+) | React Compiler analyzes and memoizes element creation | The "new element tree every render" cost model gets even cheaper — automatically |

The element model itself — plain objects describing UI, diffed by type and key — has not changed since 2013. It's the stable substrate; learn it once.

## See also

- [`thinking-in-react.md`](thinking-in-react.md) — the render loop these elements flow through
- [`components-and-props.md`](components-and-props.md) — designing the props side of the element
- [`../rendering/how-react-renders.md`](../rendering/how-react-renders.md) — what React does with the element tree (fiber, phases)
- [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md) — `key`, the reconciliation half of the element object
- [`../server/ssr-and-hydration.md`](../server/ssr-and-hydration.md) — why the DOM must match the description exactly