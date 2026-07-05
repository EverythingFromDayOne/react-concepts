---
article_id: components-and-props
concept_folder: foundations
wave: 1
related:
  - foundations/thinking-in-react
  - foundations/jsx-and-rendering
  - foundations/component-composition
  - foundations/rules-of-react
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Components and Props

> **Lead with this:** A component is a function; props are its arguments — read-only, flowing one direction, fully described by a TypeScript type. That type *is* your component's public API, and designing it is real API design: a well-shaped props contract makes invalid usage uncompilable, while a sloppy one (boolean soup, optional-everything, config objects three levels deep) pushes every mistake to runtime and every question to Slack. This article covers the mechanics — including the identity semantics that quietly govern `memo`, dependency arrays, and the Compiler — and then builds two real design-system components to practice the contract-design habits that separate a library people trust from prop soup people fear.

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

**`children` is just a prop** — one with syntax support. Whatever you nest between a component's tags arrives as `props.children`. It's the primitive composition is built on, and the first tool to reach for before inventing configuration props ([`component-composition.md`](component-composition.md) is the full toolkit).

**Conventions in this project:** typed props via a named `interface`, destructured in the signature, defaults via destructuring, `function` declarations with named exports, no `React.FC` (reasons below).

## How it works under the hood

### Every render, every props object is new

When a parent renders, it re-creates the element for each child — and the `props` object inside it — from scratch ([`jsx-and-rendering.md`](jsx-and-rendering.md) showed the objects). Two consequences:

**Children re-render when parents do, by default.** Not because "props changed" — React doesn't even check by default. Rendering the parent means evaluating its JSX, which means creating child elements, which means calling child functions. This is fine — renders are the cheap phase — until a subtree is genuinely expensive, at which point *skipping* becomes valuable, and skipping is governed entirely by identity.

**"Did props change?" is a shallow reference comparison.** `memo`, the Compiler's caches, `useState` bailouts, and dependency arrays all answer the question the same way: `Object.is`, per value. Know its exact behavior, because it is not `===` and it is not deep equality:

| Comparison | `Object.is` result | Consequence |
| --- | --- | --- |
| `'a'` vs `'a'`, `5` vs `5` | `true` | Primitives compare by value — safe |
| `{a:1}` vs `{a:1}` | `false` | Fresh literals always "changed" |
| `arr` vs same `arr` mutated | `true` | Mutations are invisible — the "list won't update" bug |
| `NaN` vs `NaN` | `true` | Unlike `===` — no NaN re-render loops |
| `+0` vs `-0` | `false` | Trivia until it isn't |

The middle two rows are the ones that bite: structurally identical fresh objects count as changed (memoization defeated), and mutated-in-place objects count as unchanged (updates swallowed). Both bugs are the same fact viewed from opposite sides.

### Children defeat `memo` — and composition beats both

Here's the identity fact almost nobody is taught, and it reorganizes how you think about performance. Wrap an expensive component in `memo` and give it children:

```tsx
const HeavyPanel = memo(function HeavyPanel({ children }: { children: ReactNode }) {
  // …expensive subtree…
});

function Dashboard() {
  const [tick, setTick] = useState(0);
  return (
    <div>
      <Clock tick={tick} onTick={setTick} />
      <HeavyPanel>
        <Reports />        {/* ← new element object on every Dashboard render */}
      </HeavyPanel>
    </div>
  );
}
```

The `memo` does nothing. Every `Dashboard` render creates a fresh `<Reports />` element, so `props.children` is a new reference every time, so the shallow compare says "changed." Paid for memoization, received none.

Now the counter-fact: **React automatically skips re-rendering any element whose reference is identical to the previous render.** No `memo` required — an unchanged element object is proof the subtree can't have new inputs. Composition exploits it by restructuring so the expensive elements are created by a component that *isn't* re-rendering:

```tsx
function Dashboard() {
  return (
    <ClockSection>               {/* the ticking state moves down here */}
      <HeavyPanel>
        <Reports />              {/* created by Dashboard — Dashboard isn't re-rendering */}
      </HeavyPanel>
    </ClockSection>
  );
}

function ClockSection({ children }: { children: ReactNode }) {
  const [tick, setTick] = useState(0);
  return (
    <div>
      <Clock tick={tick} onTick={setTick} />
      {children}                 {/* same element reference every tick → React bails out */}
    </div>
  );
}
```

Every tick re-renders `ClockSection` — but its `children` prop is the *same element object* Dashboard created, so React skips the entire `HeavyPanel` subtree. Zero `memo`, zero `useMemo`, structurally immune to the fresh-reference problem. This is the mechanism behind "push state down, pull content up," it's why [`thinking-in-react.md`](thinking-in-react.md)'s colocation pattern is a performance pattern and not just hygiene, and it's the first move in the typing-lag recipe — before any memoization API enters the conversation.

### Defaults live in destructuring now

`defaultProps` on function components was deprecated for years and **removed in React 19**. Default parameters are the mechanism:

```tsx
export function Avatar({ size = 40, shape = 'circle' }: AvatarProps) { /* … */ }
```

```tsx
// legacy: pre-19 defaultProps on function components — removed in React 19,
// modernized in the upgrade pass
Avatar.defaultProps = { size: 40, shape: 'circle' };
```

One subtlety: destructuring defaults apply on `undefined` only — not on `null`. If callers can pass `null` meaningfully ("explicitly no avatar" vs "unspecified"), handle it explicitly; the default won't.

### `ref` is a normal prop; `key` is not a prop at all

As of React 19, `ref` arrives in the props object like anything else — the `forwardRef` wrapper is legacy ceremony (details and imperative handles: [`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md)). `key`, by contrast, is stripped by React before your component runs; it belongs to reconciliation, and `props.key` is always `undefined`.

### Why not `React.FC`

The locked convention is plain `function` declarations with typed props. The reasons are practical: `React.FC` adds nothing since its implicit-`children` behavior was removed (React 18 types); it can't express generic components (`const List: React.FC<ListProps<T>>` has nowhere to declare `T` — and generics are exactly where good component APIs end up, as the walkthrough shows); and it types the *variable* rather than the function, giving worse inference and noisier declarations. There's also a mundane parser reason to prefer `function` over arrow components in `.tsx`: `const List = <T>(…)` parses as JSX and needs the `<T,>` comma hack. Named interface, plain function — better errors, real generics, one less wrapper to explain.

## Basic usage

The pattern that covers most real component APIs — including the one tutorials skip, **extending a native element**:

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

- `ComponentPropsWithoutRef<'button'>` inherits *every* legitimate button prop — `onClick`, `type`, `form`, `aria-*` — correctly typed, with no manual re-declaration and no "sorry, we didn't forward `onMouseEnter`" tickets. (In 19, plain `ComponentProps<'button'>` also types `ref` correctly if the component should accept one.)
- Custom props (`variant`, `loading`) are destructured *out* before the spread, so they never leak onto the DOM.
- Spread-then-override ordering is deliberate: callers can extend, but can't accidentally break the component's invariants (`className`, the `disabled || loading` rule).

## Walkthrough — two design-system components, contract first

The scenario: your app's forms are drifting — every feature hand-rolls its own label/error markup, half the inputs aren't associated with their labels, and there are four bespoke dropdowns. You're building the shared `TextField` and `Select`. The interesting work is entirely in the props contracts.

### Step 1 — `TextField`: native extension + accessibility as part of the contract

Requirements: label (required — unlabeled inputs shouldn't compile), optional hint, optional error, and everything a native `<input>` can do.

```tsx
// TextField.tsx
import { useId, type ComponentPropsWithoutRef, type ReactNode } from 'react';

interface TextFieldProps extends Omit<ComponentPropsWithoutRef<'input'>, 'id'> {
  label: string;
  hint?: ReactNode;
  error?: string;
}

export function TextField({
  label,
  hint,
  error,
  'aria-describedby': describedByFromCaller,
  ...rest
}: TextFieldProps) {
  const id = useId();
  const hintId = `${id}-hint`;
  const errorId = `${id}-error`;

  const describedBy =
    [hint ? hintId : null, error ? errorId : null, describedByFromCaller ?? null]
      .filter(Boolean)
      .join(' ') || undefined;

  return (
    <div className="field">
      <label htmlFor={id}>{label}</label>
      <input
        {...rest}
        id={id}
        aria-invalid={error ? true : undefined}
        aria-describedby={describedBy}
      />
      {hint && <p id={hintId} className="field__hint">{hint}</p>}
      {error && (
        <p id={errorId} className="field__error" role="alert">
          {error}
        </p>
      )}
    </div>
  );
}
```

Contract decisions worth narrating:

- **`Omit<…, 'id'>`** — the component *owns* the id, because the label/hint/error wiring depends on it. Letting callers pass `id` invites broken associations; removing it from the type makes the invariant structural. `useId` generates a unique, SSR-stable value — never `Math.random()` for this, which mismatches on hydration.
- **`label: string` is required.** The accessible version is the only version the type system permits. If a design genuinely needs no visible label, that's a *variant* to add deliberately (`labelHidden?: boolean` rendering a visually-hidden label), not an omission to allow silently.
- **`hint` is `ReactNode`, `error` is `string`.** Hints legitimately carry links and formatting; errors come from validation layers as strings, and keeping them strings keeps them serializable, loggable, and testable. Widen types when a real case demands it, not preemptively.
- **Caller's `aria-describedby` is merged, not clobbered** — extending a native element means cooperating with native usage, and this is the kind of detail that separates a design system from a pile of divs.

### Step 2 — `Select<T>`: generics are the payoff of plain functions

The bespoke dropdowns all do the same dance: array of domain objects, string value plumbing, find-by-id on change. Lift the dance into the component and let the domain type flow through:

```tsx
// Select.tsx
import { type ComponentPropsWithoutRef } from 'react';

interface SelectProps<T>
  extends Omit<ComponentPropsWithoutRef<'select'>, 'value' | 'onChange' | 'children'> {
  items: readonly T[];
  value: T | null;
  onChange: (value: T) => void;
  keyOf: (item: T) => string;
  labelOf: (item: T) => string;
  placeholder?: string;
}

export function Select<T>({
  items,
  value,
  onChange,
  keyOf,
  labelOf,
  placeholder = 'Select…',
  ...rest
}: SelectProps<T>) {
  return (
    <select
      {...rest}
      value={value !== null ? keyOf(value) : ''}
      onChange={(e) => {
        const next = items.find((item) => keyOf(item) === e.target.value);
        if (next !== undefined) onChange(next);
      }}
    >
      <option value="" disabled>
        {placeholder}
      </option>
      {items.map((item) => (
        <option key={keyOf(item)} value={keyOf(item)}>
          {labelOf(item)}
        </option>
      ))}
    </select>
  );
}
```

- **`T` flows from the call site.** `<Select items={countries} …/>` infers `T = Country`, so `onChange` hands back a `Country` — not a string id the caller re-finds in the array for the fourth time this sprint. This inference is exactly what `React.FC` can't express and plain functions get for free.
- **`value: T | null`** models "nothing selected" honestly instead of a sentinel string. The DOM boundary (string values) is handled once, inside, via `keyOf` — which doubles as the `key` for each option, keeping identity in one function.
- **`Omit` removes the native props this component re-defines** (`value`, `onChange`, `children`) — collide-by-accident is the classic native-extension bug, and the type system will tell you the moment a new custom prop shadows a native one, *if* you keep the `Omit` honest.

### Step 3 — the contracts in use

```tsx
<TextField
  label="Company name"
  hint={<>As registered. See <a href="/help/naming">naming rules</a>.</>}
  error={errors.company}
  value={company}
  onChange={(e) => setCompany(e.target.value)}
  autoComplete="organization"
/>

<Select
  items={countries}
  value={country}
  onChange={setCountry}          // (c: Country) => void — inferred
  keyOf={(c) => c.code}
  labelOf={(c) => c.name}
/>
```

Nothing here needed documentation to use correctly, and most ways to use it *incorrectly* don't compile. That's the bar.

## Real-world patterns

### Make invalid states unrepresentable — discriminated union props

When props are interdependent, encode the dependency in the type instead of documenting it in a comment:

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

`href` *and* `onClick` together? Compile error. Neither? Compile error. The body narrows on `kind`, so each branch knows exactly which fields exist. Compare the alternative — four optional props and a runtime `console.warn` — and this is the difference between a contract and a suggestion. (The same technique modeled *state* in [`thinking-in-react.md`](thinking-in-react.md); it earns its keep on both sides of the props boundary.)

### Boolean explosion → variant unions

Three booleans that can't legally coexist are one union in disguise:

```tsx
// ❌ 8 combinations, 3 valid, 0 compiler help — and a precedence rule someone must document
interface BadgeProps { isSuccess?: boolean; isWarning?: boolean; isError?: boolean }

// ✅ exactly the real states
interface BadgeProps { tone: 'success' | 'warning' | 'error' | 'neutral' }
```

Refactor the moment the second boolean of a family appears; the cost only compounds. The tell in review: any pair of boolean props where setting both is a bug.

### Elements as props — the slot pattern

`children` covers the main slot; named element props cover the rest. Pass *elements*, not configuration:

```tsx
interface PageHeaderProps {
  title: string;
  breadcrumb?: ReactNode;
  actions?: ReactNode;      // caller composes whatever belongs here
}

<PageHeader
  title="Invoices"
  breadcrumb={<Breadcrumb trail={trail} />}
  actions={<><ExportButton /><NewInvoiceButton /></>}
/>
```

This beats config props (`actionButtons: { label, onClick }[]`) decisively at scale: the caller keeps full control of the slot's contents — its own state, handlers, feature flags — and `PageHeader` stays ignorant of all of it. The moment you're adding `actionButtonVariant` and `actionButtonIcon` props, you needed a slot.

Know the distinction the pattern rests on: `actions={<ExportButton />}` passes an **element** (already-created description); `icon={ExportIcon}` passes a **component** (the function, for the receiver to render with props of its own — the registry pattern from [`jsx-and-rendering.md`](jsx-and-rendering.md)). Both are legitimate APIs; confusing them produces `Functions are not valid as a React child` at runtime, so pick per prop and type accordingly (`ReactNode` vs `ComponentType<P>`).

### Composition over drilling

When a prop's only job is to be handed down two more levels, restructure: let the component that *has* the data create the component that *needs* it, and pass the result through the middle as `children`. The middle layers stop knowing, stop re-typing, and stop appearing in every refactor diff — and per the identity mechanics above, they also stop re-rendering that content. Where restructuring genuinely can't reach — truly app-wide concerns — context exists ([`../state/context.md`](../state/context.md)), but composition should lose to it far less often than it does in the wild.

## TypeScript quick reference

The types that appear in every props file, and when each is the right one:

| Type | Means | Reach for it when |
| --- | --- | --- |
| `ReactNode` | Anything renderable: elements, strings, numbers, fragments, arrays, `null`… | Typing `children` and slot props — the default |
| `ReactElement` | A created element object, nothing else | You'll inspect/clone the element; stricter than usually needed — over-typing slots with it rejects strings and conditionals callers expect to pass |
| `ComponentType<P>` | A component (function or class) accepting `P` | Registries and `icon={TrashIcon}`-style component props |
| `PropsWithChildren<P>` | `P & { children?: ReactNode }` | Shorthand; an explicit `children` field is often clearer about optionality |
| `ComponentPropsWithoutRef<'button'>` / `ComponentProps<typeof Button>` | All props of an element or component | Native extension; reading another component's contract instead of re-declaring it |
| `Ref<HTMLInputElement>` | The type of a `ref` prop | Declaring 19-style `ref` props explicitly |
| `CSSProperties` | Typed inline-style object | `style`-shaped props |
| `Dispatch<SetStateAction<T>>` | A `useState` setter | Passing setters down — though an intent-named callback (`onQueryChange`) is usually the better contract |

## Common mistakes

**Mutating props.** Including the sneaky forms: `props.items.sort(…)` sorts the *caller's* array in place (`toSorted()` or copy first), and pushing into `props.children`. Purity violation; sometimes corrupts sibling renders.

**Leaking custom props to the DOM.** Spreading `{...props}` without destructuring your own props out first → `Warning: React does not recognize the 'variant' prop on a DOM element`, and sometimes a literal `variant="primary"` attribute in production HTML. Destructure out, then spread `rest`.

**The `isX`/`isY`/`isZ` pile.** Boolean props that are secretly variants. One union, the moment the pattern appears.

**Fresh-reference props defeating memoization.** Inline `{{…}}`, `[…]`, or arrow literals as props to memoized children — the `Object.is` table, row two. The Compiler eliminates most of this class in compiled app code; stay deliberate at boundaries it doesn't cover (external libraries, non-compiled packages).

**Expecting `memo` to survive `children`.** The mechanism section above. Composition first; `memo` for leaf-ish components with primitive-ish props.

**`defaultProps` on function components.** Removed in 19 — silently ignored, defaults vanish after upgrade. Destructuring defaults, always.

**Introspecting `children` with `React.Children`.** Walking and cloning children to inject props works until a consumer wraps an item in a Fragment, a `div`, or a feature-flag component — then it silently breaks. Explicit slots, or context between compound parts ([`component-composition.md`](component-composition.md)), are the sturdy versions of the idea.

**Optional-everything interfaces.** Twelve props, eleven `?`s, no discoverable minimal usage. Required-by-default; optional is a decision. If half the props apply only in one mode, that's a discriminated union asking to happen.

**One `config` object prop.** `<Table config={tableConfig} />` trades per-prop identity for one object that changes reference every render, hides the API from call-site autocomplete, and turns partial updates into spread rituals. Flat props unless a group genuinely travels together.

**Over-typing slots as `ReactElement`.** Callers reasonably pass `"Loading…"`, `{maybe && <X />}`, or fragments — all `ReactNode`, none `ReactElement`. Use the wide type unless you truly consume element internals.

## How this evolved

| Era | Change | Fallout today |
| --- | --- | --- |
| Class era | `this.props`, `propTypes` runtime validation | Both read as legacy markers now; validation moved to compile time |
| React 16.8 (2019) | Function components + hooks become the norm | Props become plain function parameters; TS interfaces replace `propTypes` |
| React 18 types (2022) | `React.FC` loses implicit `children` | The last argument *for* `React.FC` disappears |
| React 19 (2024) | `defaultProps` removed for FCs; `propTypes` ignored; `ref` becomes a normal prop | Destructuring defaults are the only defaults; `forwardRef` demoted to legacy |

The direction of travel is consistent: every prop mechanism that lived in runtime ceremony (`propTypes`, `defaultProps`, `forwardRef`) has collapsed into plain TypeScript and plain function parameters. Props design is now type design — which is why the union patterns above are core skills, not advanced tricks.

## Exercises

### 1. An `IconButton` that can't ship inaccessible

Build `IconButton`: renders a single icon, extends the native button, **requires** `aria-label`, and **forbids** `children` (the icon comes via an `icon: ComponentType<{ className?: string }>` prop).

*Hint: native `aria-label` is optional, so re-require it — `interface IconButtonProps extends Omit<ComponentPropsWithoutRef<'button'>, 'children' | 'aria-label'> { 'aria-label': string; icon: ComponentType<{ className?: string }>; children?: never }`. The `children?: never` makes `<IconButton>text</IconButton>` a compile error.*

### 2. Extend `Select<T>` without breaking its callers

Add `disabledOf?: (item: T) => boolean` (renders those options disabled) and an optional `groupOf?: (item: T) => string` that renders `<optgroup>`s when provided. Existing call sites must compile unchanged.

*Hint: both optional, both defaulting to "feature off." For grouping, `Object.groupBy(items, groupOf)` and remember `Object.entries` order — sort explicitly if the design requires it. Keep `keyOf` as the single source of option identity.*

### 3. Rescue this API

You inherit:

```tsx
interface CardProps {
  title: string;
  isCompact?: boolean;
  isHighlighted?: boolean;
  isInteractive?: boolean;
  config?: { showFooter?: boolean; footerText?: string; footerLinkUrl?: string; footerLinkText?: string };
}
```

Redesign the contract using this article's patterns; write the new interface and a migration note for callers.

*Hint: `isCompact`/`isHighlighted` → a `variant` or independent `density`/`tone` unions (decide whether they can co-occur — that decision IS the API design); `isInteractive` probably means "has `onClick`" — derive it, don't declare it; the entire `config.footer*` cluster is one `footer?: ReactNode` slot.*

## Summary

You learned props as they actually work: a fresh object per render, compared everywhere by `Object.is`, which makes reference identity — not structural equality — the currency of `memo`, dependency arrays, and the Compiler. The children-defeat-memo mechanism and its composition rescue (unchanged element references bail out automatically) explain why restructuring beats memoization sprinkling. On the contract side: extend native elements with `ComponentPropsWithoutRef` + destructure-then-spread, own invariants by `Omit`-ing what callers must not touch, encode interdependence as discriminated unions, replace boolean piles with variant unions, prefer slots over config, and let generics flow through plain functions — the `TextField` and `Select<T>` walkthrough put all of it in one place. React 19 finished moving props from runtime ceremony to type design; design accordingly.

## See also

- [`component-composition.md`](component-composition.md) — children, slots, and compound components in depth
- [`thinking-in-react.md`](thinking-in-react.md) — the one-way data flow these contracts live inside
- [`jsx-and-rendering.md`](jsx-and-rendering.md) — what a props object physically is, and element-vs-component props
- [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md) — identity, `memo`, and what the Compiler automates
- [`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md) — ref-as-prop and imperative handles
- [`../state/context.md`](../state/context.md) — when composition genuinely can't beat drilling

## References

- [Passing Props to a Component (react.dev)](https://react.dev/learn/passing-props-to-a-component)
- [Passing JSX as Children (react.dev)](https://react.dev/learn/passing-props-to-a-component#passing-jsx-as-children)
- [Using TypeScript (react.dev)](https://react.dev/learn/typescript)
- [`memo` API (react.dev)](https://react.dev/reference/react/memo)
- [`useId` API (react.dev)](https://react.dev/reference/react/useId)
- [React 19 Upgrade Guide — removals (react.dev)](https://react.dev/blog/2024/04/25/react-19-upgrade-guide)
- [Before You memo() — Dan Abramov](https://overreacted.io/before-you-memo/)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.