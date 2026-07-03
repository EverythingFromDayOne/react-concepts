---
article_id: components-and-props
concept_folder: foundations
related:
  - thinking-in-react
  - jsx-and-rendering
  - component-composition
  - rules-of-react
react_baseline: "19.2"
---

# Components and Props

> **Lead with this:** A component is a function; props are its arguments — read-only, flowing one direction, fully described by a TypeScript type. That type *is* your component's public API, and designing it is real API design: a well-shaped props contract makes invalid usage uncompilable, while a sloppy one (boolean soup, optional-everything, config objects three levels deep) pushes every mistake to runtime and every question to Slack. This article covers the mechanics — and the contract-design habits that separate a component library people trust from prop soup people fear.

## What it is

A component is a function that accepts a single object — the props — and returns elements:

```tsx
interface PriceTagProps {
  amount: number;
  currency: string;
  discounted?: boolean;
}

export function PriceTag({ amount, currency, discounted = false }: PriceTagProps) {
  return (
    <span className={discounted ? 'price price--discounted' : 'price'}>
      {new Intl.NumberFormat(undefined, { style: 'currency', currency }).format(amount)}
    </span>
  );
}
```

The load-bearing rules:

**Props are read-only.** A component never modifies its props — not the object, not arrays or objects nested inside it. React freezes the props object in development, but the deeper reason is the purity contract from [`thinking-in-react.md`](thinking-in-react.md): a render must be a pure computation, and mutating inputs makes it neither repeatable nor discardable. Data changes flow *up* through callbacks; new data flows back *down* as new props.

**`children` is just a prop** — one with syntax support. What you nest between a component's tags arrives as `props.children`. It's the primitive that composition is built on, and it's the first tool to reach for before inventing configuration props (the full composition toolkit is [`component-composition.md`](component-composition.md)).

**Conventions in this project:** typed props via a named `interface`, destructured in the signature, defaults via destructuring, named exports, no `React.FC` (reasons below).

## How it works under the hood

### Every render, every props object is new

When a parent renders, it re-creates the element for each child — and the `props` object inside it — from scratch. Two consequences:

**Children re-render when parents do, by default.** Not because "props changed" — React doesn't even check by default. Rendering the parent means evaluating its JSX, which means creating child elements, which means calling child functions. This is fine (renders are cheap; see [`thinking-in-react.md`](thinking-in-react.md)) until a subtree is genuinely expensive — at which point *skipping* becomes valuable, and skipping is where identity starts to matter.

**"Did props change?" is a shallow reference comparison.** `memo`, the React Compiler, and dependency arrays all answer that question the same way: `Object.is` per prop. Two structurally identical objects with different references count as "changed":

```tsx
<Chart options={{ animate: true }} />   // new object literal every render
```

A memoized `Chart` receiving that literal re-renders every time regardless — the memoization is paid for and defeated in the same line. Under the project's Compiler-first stance you rarely hand-manage this (the Compiler hoists and caches stable values automatically), but the *model* — identity, not structure — explains dependency-array behavior, memo behavior, and half of React's performance folklore. Full treatment: [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md).

### Defaults live in destructuring now

`defaultProps` on function components was deprecated for years and **removed in React 19**. Default parameters are the mechanism:

```tsx
export function Avatar({ size = 40, shape = 'circle' }: AvatarProps) { /* ... */ }
```

```tsx
// legacy: pre-19 defaultProps on function components — removed in React 19,
// modernized in the upgrade pass
Avatar.defaultProps = { size: 40, shape: 'circle' };
```

One subtlety worth knowing: destructuring defaults apply on `undefined`, not on `null`. If a caller can pass `null` meaningfully, handle it explicitly.

### `ref` is a normal prop; `key` is not a prop at all

As of React 19, `ref` arrives in the props object like anything else — the `forwardRef` wrapper is legacy ceremony (details and the imperative-handle story: [`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md)). `key`, by contrast, is stripped by React before your component runs; it belongs to reconciliation, and `props.key` is always undefined.

### Why not `React.FC`

The locked convention is plain functions with typed props, and the reasons are practical, not aesthetic: `React.FC` adds nothing since the implicit-`children` behavior was removed from its type (React 18 types), it makes generic components awkward (`React.FC` can't express `<ItemList<T>>` cleanly), and it types the *variable* rather than the function — worse inference, noisier declarations. A named interface plus a plain function gives better errors, better generics, and one less wrapper to explain.

## Basic usage

The pattern that covers 90% of real component APIs — including the one most tutorials skip, **extending a native element**:

```tsx
import { type ComponentPropsWithoutRef } from 'react';

interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'secondary' | 'ghost';
  loading?: boolean;
}

export function Button({
  variant = 'secondary',
  loading = false,
  disabled,
  children,
  ...rest                                   // every other valid <button> prop
}: ButtonProps) {
  return (
    <button
      {...rest}                             // spread first…
      className={`btn btn--${variant}`}     // …so component-owned props win
      disabled={disabled || loading}
      aria-busy={loading}
    >
      {loading ? <Spinner size="sm" /> : children}
    </button>
  );
}
```

Why this shape is the workhorse:

- `ComponentPropsWithoutRef<'button'>` inherits *every* legitimate button prop — `onClick`, `type`, `form`, `aria-*` — with correct types, no manual re-declaration, no "sorry, we didn't forward `onMouseEnter`" tickets.
- Custom props (`variant`, `loading`) are destructured *out* before the spread, so they never leak onto the DOM (React warns about unknown attributes; the warning means your abstraction is leaking).
- Spread-then-override ordering is deliberate: callers can extend, but can't accidentally break the component's invariants.

## Real-world patterns

### Make invalid states unrepresentable — discriminated union props

The senior move in props design: when props are interdependent, encode the dependency in the type instead of documenting it in a comment.

```tsx
type ActionButtonProps =
  | { kind: 'link'; href: string; onClick?: never }
  | { kind: 'action'; onClick: () => void; href?: never };

export function ActionButton(props: ActionButtonProps & { label: string }) {
  if (props.kind === 'link') {
    return <a className="btn" href={props.href}>{props.label}</a>;
  }
  return <button className="btn" onClick={props.onClick}>{props.label}</button>;
}
```

`href` *and* `onClick` together? Compile error. Neither? Compile error. The component body narrows on `kind` and TypeScript knows exactly which fields exist in each branch. Compare the alternative — four optional props and a runtime `console.warn` — and this is the difference between a contract and a suggestion. (The same technique modeled *state* instead of props in [`thinking-in-react.md`](thinking-in-react.md); it earns its keep on both sides.)

### Boolean explosion → variant unions

Three booleans that can't legally coexist are one union in disguise:

```tsx
// ❌ 8 combinations, 3 valid, 0 compiler help
interface BadgeProps { isSuccess?: boolean; isWarning?: boolean; isError?: boolean }

// ✅ exactly the real states
interface BadgeProps { tone: 'success' | 'warning' | 'error' | 'neutral' }
```

The refactor also future-proofs the API: adding a fourth tone is one union member, not a fourth boolean and a precedence rule.

### Elements as props — the slot pattern

`children` covers the main slot; named element props cover the rest. Pass *elements*, not render instructions:

```tsx
interface PageHeaderProps {
  title: string;
  actions?: ReactNode;      // caller composes whatever belongs here
  breadcrumb?: ReactNode;
}

<PageHeader
  title="Invoices"
  breadcrumb={<Breadcrumb trail={trail} />}
  actions={<><ExportButton /><NewInvoiceButton /></>}
/>
```

This beats config props (`actionButtons: {label, onClick}[]`) decisively at scale: the caller keeps full control of what renders in the slot — its own state, handlers, feature flags — and `PageHeader` stays ignorant of it all. The moment you find yourself adding `actionButtonVariant` and `actionButtonIcon` props, you needed a slot.

Know the distinction it rests on: `actions={<ExportButton />}` passes an **element** (an instantiated description); `icon={ExportIcon}` passes a **component** (the function, for the receiver to render, possibly with props of its own — the `STATUS_ICON` map pattern from [`jsx-and-rendering.md`](jsx-and-rendering.md)). Both are valid APIs; mixing them up produces the runtime error `Functions are not valid as a React child`.

### Composition over drilling

When a prop's only job is to be handed down two more levels, restructure so the component that *has* the data renders the component that *needs* it, and pass the result through the middle as `children`. The middle layers stop knowing, stop re-typing, and stop appearing in every refactor diff. Where restructuring genuinely can't reach — truly app-wide concerns — context exists ([`../state/context.md`](../state/context.md)), but composition should lose to it far less often than it does in the wild.

## Common mistakes

**Mutating props.** Including the sneaky forms: `props.items.sort(...)` sorts the caller's array in place (`toSorted` or copy first), and pushing into `props.children`. All of it violates purity; some of it corrupts sibling renders.

**Leaking custom props to the DOM.** Spreading `{...props}` without destructuring your own props out first → `Warning: React does not recognize the 'variant' prop on a DOM element`, and sometimes an actual attribute `variant="primary"` in production HTML. Destructure out, then spread `rest`.

**The `isX`/`isY`/`isZ` pile.** Boolean props that are secretly variants. Refactor to a union the second the third boolean appears — the cost only grows.

**Fresh-reference props defeating memoization.** Inline `{{...}}`, `[...]`, or arrow functions as props to memoized children. The Compiler eliminates most of this class automatically in compiled app code; be deliberate at the boundaries it doesn't cover (external libraries, non-compiled packages).

**`defaultProps` on function components.** Removed in 19; silently ignored → defaults vanish after upgrade. Destructuring defaults, always.

**Introspecting `children` with `React.Children`.** Walking and cloning children to inject props works until a consumer wraps an item in a Fragment, a div, or a feature-flag component — then it silently breaks. Explicit slots or context between compound parts ([`component-composition.md`](component-composition.md)) are the sturdy versions of the same idea.

**Optional-everything interfaces.** Twelve props, eleven `?`s, and no way to know the minimal valid usage. Required-by-default; optional is a decision, not a reflex. If half the props only apply in one mode, that's the discriminated-union pattern asking to happen.

**One `config` object prop.** `<Table config={tableConfig} />` trades React's per-prop identity model for one object that changes reference every render, hides the API from autocomplete-at-the-callsite, and turns every partial update into a spread ritual. Flat props unless a group genuinely travels together.

## How this evolved

| Era | Change | Fallout today |
| --- | --- | --- |
| Class era | `this.props`, `propTypes` runtime validation | Both patterns read as legacy markers now; types moved to compile time |
| React 16.8 (2019) | Function components + hooks become the norm | Props become plain function parameters; TS interfaces replace `propTypes` |
| React 18 types (2022) | `React.FC` loses implicit `children` | The last argument *for* `React.FC` disappears; explicit `children: ReactNode` |
| React 19 (2024) | `defaultProps` removed for FCs; `propTypes` ignored; `ref` becomes a normal prop | Destructuring defaults are the only defaults; `forwardRef` demoted to legacy |

The direction of travel is consistent: every prop mechanism that lived in runtime ceremony (`propTypes`, `defaultProps`, `forwardRef`) has collapsed into plain TypeScript and plain function parameters. Props design is now type design — which is exactly why the union patterns above are core skills, not advanced tricks.

## See also

- [`component-composition.md`](component-composition.md) — children, slots, and compound components in depth
- [`thinking-in-react.md`](thinking-in-react.md) — the one-way data flow these contracts live inside
- [`jsx-and-rendering.md`](jsx-and-rendering.md) — what a props object physically is
- [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md) — reference identity, memo, and what the Compiler automates
- [`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md) — ref-as-prop and imperative handles
- [`../state/context.md`](../state/context.md) — when composition genuinely can't beat drilling