---
article_id: jsx-and-rendering
concept_folder: foundations
wave: 1
related:
  - foundations/thinking-in-react
  - foundations/components-and-props
  - rendering/how-react-renders
  - rendering/rendering-lists-and-keys
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# JSX and Rendering

> **Lead with this:** JSX is not HTML and it is not a template language — it's syntax sugar for function calls that return plain JavaScript objects. Once `<Card title={x} />` reads as `jsx(Card, { title: x })` in your head, an entire class of confusion evaporates: why `{}` takes expressions but not `if` statements, why components must be capitalized, why `class` is `className`, why defining a component inside another component destroys its state, and why rendering `<Foo />` is not the same as calling `Foo()`. This article makes the compile target and the element model concrete — including the parts the tutorials skip (what `jsxs` is, where `key` actually goes, why dev and prod stack traces differ) — because every later topic in this resource operates on these objects.

## What it is

JSX is an XML-like syntax extension to JavaScript, compiled away entirely at build time. There is no JSX at runtime — no HTML strings, no `innerHTML`, no parsing. Your build tool (Vite's Oxc transform, in this project's baseline) rewrites it into calls to the **automatic JSX runtime**:

```tsx
// What you write
const el = <button className="primary" onClick={save}>Save</button>;

// What ships (React 17+ automatic runtime — the import is injected for you)
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

Three rules fall directly out of "it's just function calls":

**The curly braces take expressions, not statements.** `{count + 1}`, `{items.map(...)}`, `{isAdmin ? <Panel /> : null}` all work because each is an expression with a value. `{if (x) ...}` cannot work — `if` has no value. This isn't a JSX limitation to memorize; it's JavaScript.

**Capitalization is semantic, not stylistic.** `<button>` compiles to `jsx('button', ...)` — a *string* type, a host element. `<Button>` compiles to `jsx(Button, ...)` — the *function itself*, passed by reference. Lowercase means "DOM tag name as data"; capitalized means "identifier in scope." Rename a component to lowercase and React will earnestly try to create an HTML element called `<mycomponent>`.

**Attribute names are JavaScript property names.** `className` and `htmlFor` exist because `class` and `for` are reserved words in the language these objects live in; events are camelCased because they're object keys, not HTML attributes.

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

- **`$$typeof` is a security measure.** It's a `Symbol`, and symbols don't survive `JSON.parse`. If an attacker smuggles a crafted object through your API into something you render, React refuses it because the brand can't be forged from JSON. This is why "we render server data" is not automatically an injection hole — the actual holes are narrower and covered in the walkthrough below.
- **`key` is extracted at compile/creation time, not passed through.** The transform emits it as a separate argument — `jsx(Item, props, key)` — and React stores it on the element, never in `props`. That's why reading `props.key` inside a component logs a warning and yields `undefined`: the key belongs to the reconciler, and by the time your function runs, reconciliation already spent it.
- **Elements are immutable and disposable.** React freezes them in development. Every render creates a brand-new tree of them — which sounds wasteful and isn't: they're small object literals, and creating them is the cheap phase. The expensive phase (DOM mutation) is what the diffing exists to minimize ([`thinking-in-react.md`](thinking-in-react.md)).

### `jsx`, `jsxs`, `jsxDEV` — why there are three

Look at real compile output and you'll see variants. Each exists for a reason:

```tsx
// One or dynamic children → jsx()
<ul>{items.map(renderItem)}</ul>
// → jsx('ul', { children: items.map(renderItem) })

// Multiple children written statically in source → jsxs()
<ul><li>a</li><li>b</li></ul>
// → jsxs('ul', { children: [jsx('li',{children:'a'}), jsx('li',{children:'b'})] })
```

The `s` means **static children**: the compiler is vouching that *it* built this array from your source, not your runtime code. Static siblings can never reorder, so React suppresses the missing-`key` warning for them — that's the entire feature. Dynamic arrays (anything from `.map`) go through `jsx()` and get the key check, because those *can* reorder ([`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md)).

Development builds compile to a third form, `jsxDEV(type, props, key, isStaticChildren, source, self)` from `react/jsx-dev-runtime` — it carries the filename/line/column of each JSX expression. That's what powers the component stack in error overlays and DevTools "open in editor," and it's why a production stack trace is so much poorer than the dev one: the source metadata was never compiled in.

None of the three is ever called by hand. But recognizing them turns bundle output and stack traces from noise into information.

### Children normalization — and the `0` that ships to production

`children` can be an element, an array, a string, a number, or the "render nothing" family: `null`, `undefined`, `true`, `false`. Nested arrays are flattened; the nothing-family disappears. That last group is the entire reason `&&` works as a conditional:

```tsx
{isAdmin && <AdminPanel />}   // false → renders nothing
```

But **numbers render**. `0` is falsy in JavaScript and visible in the DOM:

```tsx
{cart.items.length && <CartBadge />}   // empty cart → the page shows "0"
```

This one genuinely ships to production because it passes every test with a non-empty cart. The convention in this project: comparisons, not truthiness, whenever the left side can be a number — `{cart.items.length > 0 && <CartBadge />}`. (`NaN` has the same failure mode with worse optics.)

### Element ≠ component ≠ instance

Three words that get used interchangeably and mean different things:

- A **component** is the function — the blueprint. `function Card(props) {...}`
- An **element** is a description — the object. `<Card title="Hi" />` produces one. **The function has not been called yet.**
- An **instance** is what React maintains internally per rendered position (a fiber, plus the real DOM node for host elements). You never construct these; React does, during render.

The gap between "element created" and "function called" is where two production bugs live:

**Calling a component instead of rendering it.** `{Card({ title })}` runs the function *inside the parent's render*, right now. Any hooks inside `Card` register against the *parent's* fiber — hook order corrupts the moment the call becomes conditional, and you get the infamous "Rendered fewer hooks than expected" crash. `<Card title={...} />` defers the call to React, which gives `Card` its own instance, own state, own place in DevTools. The two are never interchangeable.

**Defining a component inside a component.** Every parent render creates a *new function identity* for the inner component. Reconciliation matches positions by `type` reference; a new reference means "different component" → unmount the old, mount the new, **state destroyed**. The symptom in the wild is weirdly specific: *"my input loses focus on every keystroke"* — the keystroke updates parent state, parent re-renders, redefines the inner component, React remounts it, focus dies. The fix is always the same: hoist the component to module scope and pass data via props.

### One more DOM reality: the browser edits your markup

Browsers auto-correct invalid HTML nesting — a `<div>` inside `<p>` closes the paragraph; a `<tr>` outside `<tbody>` gets reparented. Client-only, you might never notice. Under SSR, the server streams the markup you *described*, the browser *corrects* it, and hydration then compares React's description against a DOM that no longer matches — mismatch errors that look haunted until you know the cause. Valid nesting isn't pedantry; it's a hydration requirement ([`../server/ssr-and-hydration.md`](../server/ssr-and-hydration.md)).

## Basic usage

The forms you'll write daily, annotated:

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

Two small mechanics worth locking in early: comments inside JSX must ride in expression holes — `{/* like this */}` — because `<!-- -->` is HTML syntax and JSX isn't HTML; and JSX collapses whitespace across line breaks, so when a space between two expressions matters, write it explicitly as `{' '}`.

Spread props exist and compile to object spread — `<Input {...fieldProps} />` → `jsx(Input, { ...fieldProps })`. Powerful for wrapper components, easy to abuse ([`components-and-props.md`](components-and-props.md) draws the lines).

## Walkthrough — an activity feed, from data model to safe rendering

The scenario: a project dashboard shows a feed of heterogeneous events — comments (with rich-text bodies from the server), deploys, and alerts — grouped by day, with an unread badge in the header. This exercises everything above: element construction, conditional rendering, keys, fragments, and the one real injection surface.

### Step 1 — model the events as a discriminated union

The data shape decides how clean the rendering can be. One union, one `kind` discriminant:

```tsx
// activity.ts
export type ActivityEvent =
  | { id: string; kind: 'comment'; author: string; bodyHtml: string; at: string }
  | { id: string; kind: 'deploy'; service: string; version: string; at: string }
  | { id: string; kind: 'alert'; severity: 'warn' | 'critical'; message: string; at: string };
```

### Step 2 — one renderer per kind, dispatched exhaustively

Each renderer takes exactly its narrowed member — `Extract` does the narrowing at the type level:

```tsx
// renderers.tsx
import type { ActivityEvent } from './activity';
import { RichText } from './RichText';

type EventOf<K extends ActivityEvent['kind']> = Extract<ActivityEvent, { kind: K }>;

function CommentEvent({ event }: { event: EventOf<'comment'> }) {
  return (
    <article className="event event--comment">
      <strong>{event.author}</strong> commented:
      <RichText html={event.bodyHtml} />
    </article>
  );
}

function DeployEvent({ event }: { event: EventOf<'deploy'> }) {
  return (
    <p className="event event--deploy">
      <code>{event.service}</code> deployed <code>{event.version}</code>
    </p>
  );
}

function AlertEvent({ event }: { event: EventOf<'alert'> }) {
  return <p className={`event event--alert-${event.severity}`}>{event.message}</p>;
}

export function renderEvent(event: ActivityEvent) {
  switch (event.kind) {
    case 'comment': return <CommentEvent event={event} />;
    case 'deploy':  return <DeployEvent event={event} />;
    case 'alert':   return <AlertEvent event={event} />;
    default:        return event satisfies never; // adding a kind breaks the build here
  }
}
```

The `switch` gives full narrowing and exhaustiveness with zero casts — when someone adds a `pr_merged` kind next quarter, the `satisfies never` line turns the forgotten renderer into a compile error instead of a blank feed item. At plugin scale (kinds registered from many modules) you'd graduate to a registry object keyed by `kind`; TypeScript can't correlate a union across an object lookup, so that version buys extensibility at the cost of one localized cast. Start with the switch; earn the registry.

### Step 3 — the feed: keys, grouping, and keyed fragments

Group by day and render. Two key decisions here, literally:

```tsx
// ActivityFeed.tsx
import { Fragment } from 'react';
import type { ActivityEvent } from './activity';
import { renderEvent } from './renderers';

interface ActivityFeedProps {
  events: ActivityEvent[]; // newest first
  unread: number;
}

export function ActivityFeed({ events, unread }: ActivityFeedProps) {
  const byDay = Object.groupBy(events, (e) => e.at.slice(0, 10)); // '2026-07-03'

  return (
    <section aria-label="Activity">
      <h2>
        Activity {unread > 0 && <span className="badge">{unread}</span>}
      </h2>
      <ol className="feed">
        {Object.entries(byDay).map(([day, dayEvents]) => (
          <Fragment key={day}>
            <li className="feed__separator" aria-hidden>{day}</li>
            {dayEvents!.map((event) => (
              <li key={event.id}>{renderEvent(event)}</li>
            ))}
          </Fragment>
        ))}
      </ol>
    </section>
  );
}
```

- Each day produces *multiple siblings* (separator + items) with no legal wrapper inside `<ol>` — exactly the job of the long-form `<Fragment key={day}>`; the shorthand `<>` can't carry a key.
- Event keys come from `event.id` — stable server identity, never the array index; the feed prepends new events, and index keys under prepending re-associate every row's state with the wrong item.
- The unread badge uses `> 0`, because `unread` is a number and this article already told you where that story ends.

### Step 4 — the rich-text boundary

`{event.author}` and `{event.message}` are safe by construction — JSX auto-escapes text children, so a `<script>` in a username renders as inert text. The one place markup must pass through *as markup* is the comment body, and that goes through exactly one audited component:

```tsx
// RichText.tsx
import DOMPurify from 'dompurify';

export function RichText({ html }: { html: string }) {
  return (
    <div
      className="rich-text"
      dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }}
    />
  );
}
```

One component, greppable name, sanitizer in one place — an XSS review of this app is a review of this file. The moment `dangerouslySetInnerHTML` appears in feature code directly, that guarantee is gone; six months later someone pastes the pattern without the sanitizer, and the hole reopens. The second, quieter injection surface is attacker-controlled URLs (`href={userInput}` carrying a `javascript:` payload) — React 19 blocks the blatant forms, but treat URLs as untrusted input anyway.

## Real-world patterns

### Elements are values — treat them like values

Because an element is just an object, everything you do with values works: assign to variables, return early, store in Maps, pass as props.

```tsx
export function OrderStatus({ order }: { order: Order }) {
  if (order.cancelled) return <CancelledNotice reason={order.cancelReason} />;

  const eta = order.express ? <strong>{order.eta}</strong> : <span>{order.eta}</span>;
  return <p>Arriving {eta}</p>;
}
```

Early returns beat nested ternaries for whole-component branching; ternaries beat `&&` when there's a real either/or; `&&` is for genuinely optional fragments. Pick per case — the convention is *readability of the state being expressed*, not one operator everywhere.

### Config-driven rendering with a component map

Component references are values too. A capitalized local variable renders as a component — the standard pattern for data-driven UI:

```tsx
import { CheckCircle, AlertTriangle, XOctagon, type LucideIcon } from 'lucide-react';

type Status = 'ok' | 'warn' | 'error';

const STATUS_ICON = {
  ok: CheckCircle,
  warn: AlertTriangle,
  error: XOctagon,
} as const satisfies Record<Status, LucideIcon>;

export function StatusIcon({ status }: { status: Status }) {
  const Icon = STATUS_ICON[status];   // must be Capitalized to render as a component
  return <Icon className={`icon icon--${status}`} />;
}
```

The `satisfies` keeps the map exhaustive — add a `Status` member and TypeScript flags the missing icon at the map, not as a runtime crash three screens away. (Same principle as the walkthrough's `satisfies never`, applied to data instead of control flow.)

### Slots preview: elements as props

`children` is one slot; any prop can carry an element — `<PageHeader actions={<ExportButton />} />`. The pattern, its typing, and when it beats configuration props belong to [`components-and-props.md`](components-and-props.md); it's flagged here because it's nothing more than "elements are values" pointed at API design.

## API quick reference

| API | What it's for | Field notes |
| --- | --- | --- |
| `<>…</>` / `Fragment` | Children without a wrapper DOM node | Only the long form accepts `key` (and nothing else) |
| `isValidElement(x)` | Runtime "is this an element?" check | Library-code territory; app code rarely needs it |
| `cloneElement(el, props)` | Copy an element with merged props | A smell in app code — restructure so the right props exist at creation; breaks silently when children get wrapped |
| `createElement(type, props, …children)` | Manual element creation | The legacy transform target; still legitimate for fully dynamic `type` + `children` built outside JSX |
| `jsx` / `jsxs` / `jsxDEV` | Compiler output | Never call directly; recognize them in bundles and stack traces |

## Common mistakes

**Rendering `0` (or `NaN`) via `&&`.** Use `> 0` when the left operand can be a number. Invisible in happy-path testing; enable the lint rule.

**Defining components inside components.** The remount-per-render bug. Presents as focus loss, animation restarts, or state resetting "randomly." Hoist to module scope, always — even for six-line helpers.

**Calling components as functions.** `{renderItem()}` where `renderItem` contains hooks is a fiber-identity bug waiting for its first conditional. If it has hooks, it's a component: render it as `<Item />`.

**Mutating an element after creating it.** Elements are frozen in development; the mutation throws or silently no-ops in production builds. If you need a modified copy, `cloneElement` exists but usually means the structure is wrong.

**Rendering plain objects.** `Objects are not valid as a React child` — usually a `Date`, a Decimal, or a whole record where a field was meant. Format at the leaf: `{order.createdAt.toLocaleDateString()}`.

**String "booleans".** `<Input disabled="false" />` is a *truthy string* — the input is disabled. Booleans go through expression holes: `disabled={false}`, or bare presence for `true`.

**HTML comments.** `<!-- note -->` inside JSX is a syntax error or, worse, stray text. Comments ride in braces: `{/* note */}`.

**Spreading a `key` in.** `<Item {...props} />` where `props` contains `key` draws a React 19 warning — the transform can't extract a key it can't see statically. Pass it explicitly: `<Item key={id} {...rest} />`.

**Invalid HTML nesting.** Fine until SSR turns it into hydration mismatches. `<p>` can't contain block elements; table parts belong in table structure. The HTML content model is part of correctness, not styling.

## How this evolved

| Era | Change | Why it matters now |
| --- | --- | --- |
| Pre-17 | Classic transform: `React.createElement`, `import React` required everywhere | You'll still see both in older codebases; the element object was the same idea |
| React 17 (2020) | Automatic runtime: `jsx`/`jsxs`/`jsxDEV` from `react/jsx-runtime`, imports injected | Today's baseline; enabled smaller output and build-tool-level optimization |
| React 19 (2024) | `ref` becomes a normal prop inside `props`; `element.ref` removed; element brand renamed; spread-`key` deprecated | `forwardRef` demoted to legacy ceremony ([`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md)); element internals simplified |
| Compiler era (2025+) | React Compiler analyzes and caches element creation | The "new element tree every render" cost model gets cheaper still — automatically |

The element model itself — plain objects describing UI, diffed by type and key — has not changed since 2013. It's the stable substrate; learn it once.

## Exercises

### 1. Compile it by hand

Without a build tool, write the runtime calls this compiles to — which calls are `jsx` vs `jsxs`, and where `key` goes:

```tsx
<section className="panel">
  <h3>Alerts</h3>
  <ul>{alerts.map((a) => <li key={a.id}>{a.text}</li>)}</ul>
</section>
```

*Hint: `section` has static children → `jsxs`. The `ul` has one dynamic child (the array from `.map`) → `jsx`. Each `li` is `jsx('li', { children: a.text }, a.id)` — key as the third argument, never in props.*

### 2. Extend the feed — let the compiler drive

Add a `{ kind: 'pr_merged'; repo: string; number: number; … }` event to the walkthrough. Add *only* the union member first, then follow the compile errors to done.

*Hint: exactly one error should appear — the `satisfies never` in `renderEvent`. If nothing broke, your switch has a `default` swallowing unknown kinds, which is precisely the bug this pattern exists to prevent.*

### 3. Four bugs, one component

Find and fix all four:

```tsx
function Inbox({ user, messages }: InboxProps) {
  return (
    <div>
      <!-- unread count -->
      <p>Signed in as {user}</p>
      {messages.length && <ul>…</ul>}
      <button disabled="false">Refresh</button>
    </div>
  );
}
```

*Hint: HTML comment syntax; rendering an object (`user` is a record — render `user.name`); `0` from `&&`; truthy string on `disabled`.*

## Summary

You learned what JSX actually is: build-time sugar over `jsx`/`jsxs`/`jsxDEV` calls that produce frozen, disposable element objects — `type`, `props`, and a `key` that lives outside props because it belongs to the reconciler. The element/component/instance distinction explains the two classic identity bugs (calling components as functions, defining them inline), children normalization explains both why `&&` works and why `0` leaks to the screen, and the browser's markup auto-correction explains a whole family of hydration mysteries. The walkthrough turned the theory into a feed: a discriminated union dispatched exhaustively, stable keys with keyed fragments for grouped siblings, and rich text passing through exactly one sanitized boundary. Elements are values; the rest of React is what gets done with them.

## See also

- [`thinking-in-react.md`](thinking-in-react.md) — the render loop these elements flow through
- [`components-and-props.md`](components-and-props.md) — designing the props side of the element
- [`../rendering/how-react-renders.md`](../rendering/how-react-renders.md) — what React does with the element tree (fiber, phases)
- [`../rendering/rendering-lists-and-keys.md`](../rendering/rendering-lists-and-keys.md) — `key`, the reconciliation half of the element object
- [`../server/ssr-and-hydration.md`](../server/ssr-and-hydration.md) — why the DOM must match the description exactly

## References

- [Writing Markup with JSX (react.dev)](https://react.dev/learn/writing-markup-with-jsx)
- [JavaScript in JSX with Curly Braces (react.dev)](https://react.dev/learn/javascript-in-jsx-with-curly-braces)
- [Conditional Rendering (react.dev)](https://react.dev/learn/conditional-rendering)
- [Rendering Lists (react.dev)](https://react.dev/learn/rendering-lists)
- [`Fragment` API (react.dev)](https://react.dev/reference/react/Fragment)
- [`createElement` API (react.dev)](https://react.dev/reference/react/createElement)
- [`cloneElement` API (react.dev)](https://react.dev/reference/react/cloneElement)
- [Introducing the New JSX Transform — React blog](https://legacy.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)
- [React Components, Elements, and Instances — Dan Abramov](https://legacy.reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.