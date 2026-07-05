---
article_id: component-composition
concept_folder: foundations
wave: 1
related:
  - foundations/components-and-props
  - foundations/thinking-in-react
  - state/context
  - effects/custom-hooks
  - rendering/memoization-and-the-compiler
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Component Composition

> **Lead with this:** Composition is React's inheritance, its plugin system, and its performance model, all in one move: components that know *less* combine into more. The mechanical foundation is a single distinction — the **owner** who writes the JSX versus the **parent** who renders it — and on top of it sits an escalation ladder of API designs: `children`, named slots, compound components, render props. Climb only as far as the requirement forces you. This article names the owner/parent mechanics precisely, builds a production-grade compound `<Tabs>` (controlled *and* uncontrolled, keyboard-navigable, ARIA-correct), and locks in the two composition moves — push state down, lift content up — that the entire performance track is built on. Inheritance does not appear, because in React it never does.

## What it is

Composition is designing components so that *callers assemble behavior* instead of components anticipating it. The under-powered alternative is configuration: every new need becomes a prop (`showFooter`, `footerText`, `footerLinkUrl`…), the component accumulates knowledge about every caller, and its props file becomes a changelog of feature requests. The composed alternative hands the caller a *place to put things* — and suddenly the component never needs to know what things are.

The ladder, cheapest rung first:

1. **`children`** — one main slot. `<Card>{anything}</Card>`. Covers the majority of cases.
2. **Named slots** — several placed regions. `<PageHeader actions={…} breadcrumb={…} />` ([`components-and-props.md`](components-and-props.md) built these).
3. **Compound components** — a *family* of parts sharing implicit state: `<Tabs>`, `<Tabs.List>`, `<Tabs.Tab>`, `<Tabs.Panel>`. For when the parts must coordinate but the caller must control arrangement.
4. **Render props** — the caller supplies a *function* that the component invokes with data it owns: `<VirtualList renderRow={(item, style) => …} />`. For when the component owns per-item data or timing, but the caller owns the markup.

Each rung buys flexibility and costs API surface. The discipline is refusing to climb early: a `Card` with `children` should not become compound because one caller wanted a divider.

## How it works under the hood

### Owner versus parent — the distinction everything rests on

Two different components have a claim on any element:

- Its **owner** is the component whose render *created* it — whoever wrote the JSX. Owners supply the props, and the closures inside them.
- Its **parent** is the component directly above it in the rendered tree — whoever emitted it via `{children}` or a slot.

Usually they're the same component. Composition is precisely the technique of making them *different*:

```tsx
function Page() {                                  // OWNER of <Reports />
  return (
    <Shell>
      <Reports />
    </Shell>
  );
}

function Shell({ children }: { children: ReactNode }) {   // PARENT of <Reports />
  const [navOpen, setNavOpen] = useState(false);
  return (
    <div>
      <Sidebar open={navOpen} onToggle={() => setNavOpen((o) => !o)} />
      {children}
    </div>
  );
}
```

Now the mechanics from [`components-and-props.md`](components-and-props.md) get their proper names: **elements are re-created when their *owner* re-renders, and automatically skipped when only their *parent* does.** Toggle the sidebar — `Shell` re-renders, but `children` is the very same element object `Page` made, reference-identical, so React bails out of the entire `<Reports />` subtree without `memo` ever entering the picture. That bailout is not an optimization you apply; it's a structure you *choose*. React DevTools even surfaces the distinction — the "rendered by" list is the owner chain, not the parent chain — which is how you audit who's actually re-creating what.

### Slot content: owner's scope, render-position's context

A slotted element lives a double life. Its **props and closures come from the owner**; its **context comes from where it sits in the tree**:

```tsx
function Page() {
  const handleExport = () => exportReport(reportId);        // Page's closure
  return <Toolbar actions={<ExportButton onClick={handleExport} />} />;
}

function ExportButton(props: { onClick: () => void }) {
  const { density } = useToolbarContext();   // ← read at RENDER POSITION: inside Toolbar's provider
  return <button className={`btn btn--${density}`} onClick={props.onClick}>Export</button>;
}
```

`handleExport` closes over `Page`'s render; `density` resolves against `Toolbar`'s provider, because that's where the element ends up mounted. This is what makes slots powerful — content keeps the caller's data *and* inherits the host's environment — and it's the mechanism compound components are built on: the parts read the family's context because the caller placed them inside it.

### What "not rendering children" actually costs (and doesn't)

Because owners create elements eagerly, `<Tabs.Panel>` receives its `children` *element* whether or not the panel is active — element creation already happened in the owner. What a panel controls is **mounting**: returning `null` for an inactive panel means those child components are never called, no fibers, no state, no effects. So conditional slots are cheap where it matters (the render/mount work) and unavoidable where it doesn't (the object literals — which are the cheap phase, per [`thinking-in-react.md`](thinking-in-react.md)). Keep the two costs separate and half the "should I lazy-create this slot?" debates dissolve.

## Basic usage

The two rungs below compound, at production quality — a specialization wrapper and a layout component:

```tsx
// Specialization: composition where OO reaches for inheritance.
// A preset is a component that fills in decisions and forwards the rest.
import { type ComponentPropsWithoutRef } from 'react';
import { Button } from './Button';

type DangerButtonProps = Omit<ComponentPropsWithoutRef<typeof Button>, 'variant'>;

export function DangerButton(props: DangerButtonProps) {
  return <Button variant="danger" {...props} />;
}
```

```tsx
// Layout components: structure via children, variation via a few honest props.
interface StackProps {
  gap?: 'sm' | 'md' | 'lg';
  children: ReactNode;
}

export function Stack({ gap = 'md', children }: StackProps) {
  return <div className={`stack stack--${gap}`}>{children}</div>;
}

// Callers compose structure without Stack knowing anything:
<Stack gap="lg">
  <SearchBar … />
  <ResultsTable … />
</Stack>
```

The `Omit<…, 'variant'>` on the preset is the small move that separates a wrapper from a leak: the decision the preset *makes* is removed from the API, so `<DangerButton variant="ghost">` doesn't compile. Presets that re-expose what they decided aren't presets; they're suggestions.

## Walkthrough — a compound `<Tabs>`, from config prop to production API

The scenario: the design system needs tabs. Version 1 is the obvious config prop; the requirements that kill it arrive within a sprint. The walkthrough builds the replacement the way a library would: compound parts, implicit state via context, controlled *and* uncontrolled modes, keyboard navigation, correct ARIA.

### Step 1 — the config version, and its ceiling

```tsx
<Tabs
  items={[
    { id: 'overview', label: 'Overview', content: <Overview /> },
    { id: 'billing',  label: 'Billing',  content: <Billing /> },
  ]}
/>
```

Then: a badge on Billing, a disabled Beta tab, an icon in one label, analytics on one panel only, a right-aligned "Export" button *inside the tab strip*. Each is a new prop on the item object — `badge?`, `disabled?`, `labelRender?`, `onPanelView?` — the prop explosion [`components-and-props.md`](components-and-props.md) diagnosed, now nested inside an array. The component knows more every week and satisfies callers less. The ceiling is structural: **config describes what the component imagined; composition accepts what it didn't.**

### Step 2 — design the compound API first

```tsx
<Tabs defaultValue="overview">
  <Tabs.List>
    <Tabs.Tab value="overview">Overview</Tabs.Tab>
    <Tabs.Tab value="billing">Billing <Badge count={unpaid} /></Tabs.Tab>
    <Tabs.Tab value="beta" disabled>Beta</Tabs.Tab>
    <ExportButton />                      {/* arbitrary content in the strip — free */}
  </Tabs.List>

  <Tabs.Panel value="overview"><Overview /></Tabs.Panel>
  <Tabs.Panel value="billing" keepMounted><Billing /></Tabs.Panel>
  <Tabs.Panel value="beta"><BetaPane /></Tabs.Panel>
</Tabs>
```

Every killed requirement is now just JSX the caller writes. The parts coordinate through state the family shares implicitly — which is the one new mechanism this needs.

### Step 3 — the family's private channel: context

Context gets its full article later ([`../state/context.md`](../state/context.md)); compound components need exactly this much of it — a typed channel plus a guard hook:

```tsx
// tabs-context.ts
import { createContext, useContext } from 'react';

interface TabsContextValue {
  active: string;
  setActive: (value: string) => void;
  baseId: string;                        // for ARIA id wiring
}

const TabsContext = createContext<TabsContextValue | null>(null);

export function useTabsContext(part: string): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (ctx === null) {
    throw new Error(`<Tabs.${part}> must be rendered inside <Tabs>`);
  }
  return ctx;
}

export { TabsContext };
```

The `null` default plus the throwing guard is non-negotiable compound hygiene: a part rendered outside its family fails *loudly, at the part, with its name* — instead of as `Cannot read properties of null` three stack frames deep in someone's sprint.

### Step 4 — the root: controlled and uncontrolled, the forms pattern at component scale

`<Tabs>` should work both ways — self-managing (`defaultValue`) or driven by the caller (`value` + `onValueChange`, e.g. tabs synced to the URL). This is [`../forms/forms-controlled-and-uncontrolled.md`](../forms/forms-controlled-and-uncontrolled.md)'s dual-master design, one level up, including the same "presence of `value` selects the mode" test:

```tsx
// Tabs.tsx
import { useId, useState, type ReactNode } from 'react';
import { TabsContext, useTabsContext } from './tabs-context';

interface TabsProps {
  value?: string;                         // controlled
  onValueChange?: (value: string) => void;
  defaultValue?: string;                  // uncontrolled seed
  children: ReactNode;
}

export function Tabs({ value, onValueChange, defaultValue = '', children }: TabsProps) {
  const [internal, setInternal] = useState(defaultValue);
  const baseId = useId();

  const controlled = value !== undefined;
  const active = controlled ? value : internal;

  function setActive(next: string) {
    onValueChange?.(next);
    if (!controlled) setInternal(next);   // uncontrolled: own the state; controlled: only report
  }

  return (
    <TabsContext value={{ active, setActive, baseId }}>
      <div className="tabs">{children}</div>
    </TabsContext>
  );
}
```

(React 19 lets the context object render directly as the provider — `<TabsContext value={…}>`.) The mode rules mirror the input's exactly: controlled mode never writes internal state, uncontrolled mode never requires the callback, and switching modes mid-life is the same bug it was for inputs.

### Step 5 — the parts: ARIA and keyboard for free, forever

```tsx
function List({ children }: { children: ReactNode }) {
  return (
    <div role="tablist" className="tabs__list">
      {children}
    </div>
  );
}

interface TabProps {
  value: string;
  disabled?: boolean;
  children: ReactNode;
}

function Tab({ value, disabled = false, children }: TabProps) {
  const { active, setActive, baseId } = useTabsContext('Tab');
  const selected = active === value;

  function handleKeyDown(e: React.KeyboardEvent<HTMLButtonElement>) {
    // Roving tabindex, DOM-sibling flavor: arrows move focus; click/Enter activates.
    if (e.key !== 'ArrowRight' && e.key !== 'ArrowLeft') return;
    const dir = e.key === 'ArrowRight' ? 'nextElementSibling' : 'previousElementSibling';
    let el = e.currentTarget[dir] as HTMLElement | null;
    while (el && (el.getAttribute('role') !== 'tab' || el.hasAttribute('disabled'))) {
      el = el[dir] as HTMLElement | null;   // skip non-tabs (the ExportButton) and disabled tabs
    }
    el?.focus();
  }

  return (
    <button
      role="tab"
      id={`${baseId}-tab-${value}`}
      aria-selected={selected}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={selected ? 0 : -1}          // one tab stop for the whole strip
      disabled={disabled}
      onClick={() => setActive(value)}
      onKeyDown={handleKeyDown}
      className="tabs__tab"
    >
      {children}
    </button>
  );
}

interface PanelProps {
  value: string;
  keepMounted?: boolean;
  children: ReactNode;
}

function Panel({ value, keepMounted = false, children }: PanelProps) {
  const { active, baseId } = useTabsContext('Panel');
  const selected = active === value;

  if (!selected && !keepMounted) return null;               // unmount: clean teardown

  return (
    <div
      role="tabpanel"
      id={`${baseId}-panel-${value}`}
      aria-labelledby={`${baseId}-tab-${value}`}
      hidden={!selected}                                     // keepMounted: hide, keep warm
      className="tabs__panel"
    >
      {children}
    </div>
  );
}

Tabs.List = List;
Tabs.Tab = Tab;
Tabs.Panel = Panel;
```

What the family bought, permanently: `useId` in the root plus context gives every instance collision-free `aria-controls`/`aria-labelledby` wiring; roving tabindex (`0` on the active tab, `-1` elsewhere) makes the strip one Tab-key stop with arrow-key movement, skipping disabled tabs and non-tab content, per the WAI-APG tabs pattern (manual activation: arrows move focus, Enter/click selects — buttons give Enter/Space for free). `keepMounted` is [`conditional-rendering-and-events.md`](conditional-rendering-and-events.md)'s unmount-vs-hide table, surfaced as a per-panel decision the caller owns. Two honest footnotes: the DOM-sibling roving works because tabs are direct siblings — production libraries use a registration pattern (parts register via effects; you'll have the tools after [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)); and dot-notation (`Tabs.Tab`) reads beautifully but plays poorly with `React.lazy` and tree-shaking — libraries typically *also* export the parts as named exports, so do both.

## Real-world patterns

### The two performance moves — the groundwork

Every composition-based performance fix is one of two moves, and both are pure restructuring:

**Push state down.** State consumed by a small region shouldn't live above a big one:

```tsx
// ❌ Every keystroke re-renders the whole page (Page is the owner of everything)
function Page({ data }: { data: Data }) {
  const [query, setQuery] = useState('');
  return (
    <>
      <SearchInput value={query} onChange={setQuery} />
      <ExpensiveAnalyticsGrid data={data} />
    </>
  );
}

// ✅ The state and its consumers move into their own component
function Page({ data }: { data: Data }) {
  return (
    <>
      <Search />                          {/* owns query; its re-renders stop here */}
      <ExpensiveAnalyticsGrid data={data} />
    </>
  );
}
```

**Lift content up.** When the stateful component must *wrap* the expensive content, accept it as `children` so its owner is someone calmer:

```tsx
// ❌ ResizableSplit owns the panes — every drag pixel re-creates and re-renders them
<ResizableSplit left={<FileTree />} right={<Editor />} />   // fine IF Split doesn't own them…

function ResizableSplit({ left, right }: { left: ReactNode; right: ReactNode }) {
  const [ratio, setRatio] = useState(0.3);                  // updates 60×/sec while dragging
  return (
    <div className="split" style={{ ['--ratio' as string]: ratio }}>
      <div className="split__pane">{left}</div>
      <DragHandle onDrag={setRatio} />
      <div className="split__pane">{right}</div>
    </div>
  );
}
```

…and it *is* fine — because `left` and `right` were created by the caller. `ResizableSplit` re-renders per drag frame, but the pane elements are reference-identical, so React bails out of `FileTree` and `Editor` entirely. The anti-version is `ResizableSplit` importing and rendering `<FileTree />` itself: same screen, owner moved, bailout gone. **Owner placement is the optimization.** This pair — state down, content up — is what the typing-lag recipe reaches for before any memoization API, and why [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md) treats `memo` as the tool for the residue composition can't reach.

### Render props: where they still earn rent after hooks

Hooks replaced render props for *behavior* reuse. What hooks cannot do is let a component delegate **markup for data it owns per-instance** — because a hook runs in the caller, and the data lives in the callee's loop:

```tsx
// The list owns which rows exist, their measured positions, their virtualization —
// the caller owns what a row looks like. That inversion is a render prop's whole job.
<VirtualList
  items={tickets}
  rowHeight={48}
  renderRow={(ticket, style) => (
    <TicketRow key={ticket.id} ticket={ticket} style={style} />
  )}
/>
```

The decision rule: *reusing stateful logic in your own render* → custom hook ([`../effects/custom-hooks.md`](../effects/custom-hooks.md)); *a component invoking your rendering with its data* → render prop. Function-as-children is the same pattern with `children` as the function — fine, but named props (`renderRow`) read better once there's more than one.

### Slots make the middle layers disappear

The drilling refactor, named as a move: when `Layout → Sidebar → Nav` all forward `user` so `Nav` can render an avatar, the fix isn't context — it's re-owning. Let the component that *has* `user` create the element that *needs* it, and thread it through as content:

```tsx
// Before: three components typed and forwarded UserProps they never read
<Layout user={user} />

// After: zero middle-layer knowledge — Layout carries content, not data
<Layout nav={<Nav avatar={<Avatar user={user} />} />} />
```

The middles stop appearing in `user`-related diffs forever, and — owner mechanics again — stop re-rendering that content when their own state moves. Context remains the tool for *genuinely ambient* values ([`../state/context.md`](../state/context.md)); reaching for it to avoid two levels of forwarding is using a broadcast tower to pass a note.

## Typing quick reference

| Pattern | Type | Field notes |
| --- | --- | --- |
| Main slot | `children: ReactNode` | The default; `ReactNode` accepts elements, strings, arrays, conditionals |
| Named slot | `actions?: ReactNode` | Optional slots render `{actions}` — `undefined` is a fine child |
| Render prop | `renderRow: (item: T, style: CSSProperties) => ReactNode` | Return `ReactNode`, not `ReactElement` — let callers return strings/null |
| Function as children | `children: (state: S) => ReactNode` | Same as above; call it exactly once per render |
| Injected component | `icon: ComponentType<{ className?: string }>` | Caller passes the *function*; you render `<Icon />` — vs. slots, which pass elements |
| Compound context | `createContext<Value \| null>(null)` + throwing guard hook | The guard converts misuse into a named, actionable error |
| Preset wrapper | `Omit<ComponentPropsWithoutRef<typeof Base>, 'decidedProp'>` | Remove what the preset decides; forward the rest |

## Common mistakes

**`Children.map` + `cloneElement` to smuggle props into children.** The classic pre-context compound implementation — walk the children, clone each with injected props. It shatters the moment a caller wraps a part in a Fragment, a `div`, a feature flag, or their own component, and it can't reach children of children at all. Context is the sturdy channel; the official docs now list both APIs with "alternatives" sections for a reason. Same verdict for its cousin, runtime child-type *validation* ("only `Tabs.Tab` allowed here") — you'll reject legitimate wrappers; let the guard hook police usage instead.

**Compound parts with no guard.** Skipping the `null`-check hook means a stray `<Tabs.Tab>` fails as a property read on `null` somewhere inside. The two-line throwing guard turns it into `"<Tabs.Tab> must be rendered inside <Tabs>"` — the difference between a bug report and a self-service fix.

**Fresh provider `value` objects.** `<TabsContext value={{ active, setActive, baseId }}>` builds a new object per render, so every consumer re-renders whenever the root does — the `Object.is` rule from [`components-and-props.md`](components-and-props.md), applied to context. In compiled app code the Compiler stabilizes it; in *library* code (which compound components usually are), memoize the value deliberately — the full story is [`../state/context.md`](../state/context.md)'s, flagged here because the walkthrough's code is exactly where it bites.

**Config creep colonizing the compound API.** Six months in, `<Tabs variant centered fullWidth badgePlacement>` — the prop explosion reincarnated on the root. Appearance-of-a-part belongs on the part; arrangement belongs to JSX; the root carries state semantics (`value`, `onValueChange`) and little else.

**Re-drilling what composition already solved.** Adding a prop to three intermediate components to move data one owner already had — the slots pattern above was available. The review tell: a component whose props interface mentions types it never renders.

**Render props for behavior / hooks for delegation.** Both inversions of the decision rule: a `<MouseTracker>{(pos) => …}</MouseTracker>` wrapping half a page is a hook wearing a costume (and adds a tree layer for nothing); a `useVirtualRows()` hook that returns "please render these 30 items at these offsets" has reinvented a render prop with worse ergonomics.

**Dot-notation as the only export.** `Tabs.Tab` on the class of module systems: `React.lazy(() => import('./Tabs'))` can't pick parts off, and bundlers tree-shake attached properties poorly. Ship named exports alongside the dot-notation sugar.

**Calling render props conditionally-ish.** Invoking `children(state)` zero times ("nothing to show") or twice ("desktop and mobile layouts") surprises callers whose functions create elements with hooks-bearing components — fine — but whose *expectations* are once-per-render. Call exactly once; represent "nothing" in the argument.

## How this evolved

This article's patterns *are* the history of React code reuse — each era's answer to "share stateful behavior between components":

| Era | Pattern | What killed it |
| --- | --- | --- |
| `createClass` era (2013–15) | **Mixins** | Implicit name collisions, invisible data flow — officially "considered harmful" by 2016 |
| 2015–2018 | **Higher-order components** — `connect(withRouter(withTheme(Button)))` | Wrapper hell in DevTools, prop-name collisions, static typing pain, indirection per feature |
| 2017–2019 | **Render props** | Solved HOC opacity; produced render-prop pyramids for multi-behavior composition |
| React 16.3 (2018) | `createContext` ships | Compound components become practical without `cloneElement` |
| React 16.8 (2019) | **Hooks** | Behavior reuse becomes function calls; HOCs and most render props demoted overnight |
| React 19 (2024) | Context renders as its own provider | One less wrapper in every compound root |
| Compiler era (2025+) | Slot elements and provider values stabilized automatically in app code | The owner/parent bailout gets cheaper to rely on; composition wins by more |

```tsx
// legacy: HOC-era composition — wrapper hell, modernized in the upgrade pass
export default connect(mapState)(withRouter(withTheme(injectIntl(SettingsPanel))));
```

Render props and slots survived the hooks extinction because they answer a question hooks structurally can't: *who renders markup for data someone else owns.* Everything on the ladder today is post-selection survivorship — patterns that still earn their place.

## Exercises

### 1. Grow the family

Add `<Tabs.Actions>` — a right-aligned region inside the strip (the `ExportButton` from step 2 deserves a real home) — and make the arrow-key roving skip it *without* the `role` sniffing loop breaking. Then add an `orientation="vertical"` mode: `aria-orientation` on the list, Up/Down instead of Left/Right on the tabs.

*Hint: `Actions` is markup-only — it needs no context, which is itself a lesson (not every part of a compound family is stateful). The roving loop already skips non-`role="tab"` siblings; verify with the Actions in the middle of the strip, not just the end. Orientation belongs in the context value so `Tab` can pick its arrow keys — one field, not a prop drilled to every tab.*

### 2. Un-drill it

Three components forward `currentUser` so a deeply nested `<Avatar>` can render; nobody in between reads it. Refactor with slots so no intermediate component mentions `User`, then answer: which components now re-render when the avatar's owner updates `currentUser`, and which did before?

*Hint: work out both directions. Middle-update (the new behavior): an intermediate's own state change re-renders it, but the slot content it emits is the same element reference its owner created — React bails out of `<Avatar>` entirely. Owner-update: the owner re-creates the slot elements, so the avatar re-renders and the middles do too (they're in the render path) — same as before the refactor. That asymmetry is the owner/parent mechanics, and articulating it precisely is the exercise.*

### 3. Pick the rung

For each requirement, name the lowest ladder rung that solves it and one sentence why the rung below fails: (a) a `Modal` needing header/body/footer regions; (b) a `Combobox` where highlighted-option state must be shared between an input, a listbox, and option items the caller arranges; (c) a `DataGrid` that virtualizes 50k rows and lets callers define cell markup; (d) a `Tooltip` wrapping arbitrary content.

*Hint: (d) children, (a) named slots — children can't place three regions; (b) compound — slots can't share live state between caller-arranged parts; (c) render prop — compound parts can't be invoked per-virtualized-row by the grid's own loop. If you answered "context" for any of them, revisit the broadcast-tower line.*

## Summary

You learned composition as mechanics and as method. The mechanics: every element has an owner (who created it, whose closures it carries) and a parent (who emitted it, whose position gives it context) — elements re-create when owners render and bail out when only parents do, which makes owner placement itself the performance tool, expressed as the two moves: push state down, lift content up. The method: an escalation ladder — `children`, named slots, compound components, render props — climbed only under duress, with each rung's production form built here: presets that `Omit` what they decide, a compound `<Tabs>` with a typed context channel, a throwing guard, the controlled/uncontrolled duality inherited from forms, `useId`-wired ARIA, roving tabindex, and per-panel unmount-vs-hide. The reuse graveyard (mixins → HOCs → render props → hooks) explains what survived and why: slots and render props answer the one question hooks can't — who renders markup for data someone else owns. Configuration describes what a component imagined; composition accepts what it didn't.

## See also

- [`components-and-props.md`](components-and-props.md) — slots, element-vs-component props, and the identity mechanics this article names
- [`../state/context.md`](../state/context.md) — the full context story: value identity, splitting, when ambient beats assembled
- [`../effects/custom-hooks.md`](../effects/custom-hooks.md) — behavior reuse, the other half of the render-prop decision rule
- [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md) — `memo` and the Compiler as tools for composition's residue
- [`../forms/forms-controlled-and-uncontrolled.md`](../forms/forms-controlled-and-uncontrolled.md) — the dual-master pattern the Tabs root inherits
- [`conditional-rendering-and-events.md`](conditional-rendering-and-events.md) — unmount vs hide, the table behind `keepMounted`

## References

- [Passing JSX as Children (react.dev)](https://react.dev/learn/passing-props-to-a-component#passing-jsx-as-children)
- [Passing Data Deeply with Context (react.dev)](https://react.dev/learn/passing-data-deeply-with-context)
- [`cloneElement` — and its Alternatives section (react.dev)](https://react.dev/reference/react/cloneElement)
- [`Children` — and its Alternatives section (react.dev)](https://react.dev/reference/react/Children)
- [`useId` (react.dev)](https://react.dev/reference/react/useId)
- [WAI-ARIA Authoring Practices — Tabs pattern](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/)
- [Higher-Order Components (legacy docs — historical)](https://legacy.reactjs.org/docs/higher-order-components.html)
- [Render Props (legacy docs — historical)](https://legacy.reactjs.org/docs/render-props.html)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.