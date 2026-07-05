---
article_id: portals-and-the-event-system
concept_folder: rendering
wave: 2
related:
  - foundations/conditional-rendering-and-events
  - rendering/how-react-renders
  - state/context
  - rendering/error-boundaries
  - effects/useref-and-the-dom
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Portals and the Event System

> **Lead with this:** `createPortal` renders children into a different DOM container while keeping them in the *same React tree* — one component, two trees, deliberately out of sync. Every portal question reduces to one test: **which tree governs this behavior?** Context, state, error boundaries, and — the part that surprises everyone — *event propagation* follow the **React tree**. CSS inheritance, stacking, focus order, and the accessibility tree follow the **DOM tree**. Hold that split and portals stop being spooky: a click inside a modal portaled to `<body>` still bubbles to `onClick` on the card that opened it, while the same modal stubbornly refuses to inherit the card's CSS custom properties. This article grounds both trees mechanically, pays off the event-system debts — synthetic propagation through portals and the legitimate uses of the capture phase — and takes an honest 2026 position on when the native `<dialog>`/popover top layer means you don't need a portal at all.

## What it is

```tsx
import { createPortal } from "react-dom";

function ConfirmLayer({ children }: { children: React.ReactNode }) {
  return createPortal(children, document.getElementById("overlay-root")!);
}
```

`createPortal(children, domNode, key?)` returns a special element: React reconciles `children` as ordinary children of the *component* that rendered the portal, but commits their DOM output into `domNode` instead of the parent's DOM position. The classic motivations are all escape acts — out of `overflow: hidden` clipping, out of `z-index` stacking-context wars, out of a transformed ancestor that would hijack `position: fixed`.

The governing split, as a table you'll use weekly:

| Behavior | Governed by | Consequence for portals |
| --- | --- | --- |
| Context reads | React tree | Portal content reads the providers of the component that rendered it ([context](../state/context.md)) |
| State, re-renders, props | React tree | The portal is an ordinary subtree to the reconciler |
| Error boundaries | React tree | A throw in portal content hits the *rendering component's* nearest boundary ([error-boundaries](./error-boundaries.md)) |
| **Synthetic event propagation** | **React tree** | Clicks inside the portal bubble to React handlers on React-tree ancestors |
| CSS inheritance, custom properties, stacking | DOM tree | Ancestor classes/variables don't reach portal content |
| Focus order, `Tab` sequence, a11y tree | DOM tree | Focus falls out of the portal into whatever's DOM-adjacent |
| Native (non-React) listeners, `event.composedPath()` | DOM tree | A `document` click listener sees the portal's *DOM* path only |

Everything in this article is a working-out of one row or a collision between two.

## How it works under the hood

### The fiber side

A portal is a fiber like any other — a `HostPortal`-tagged node sitting in its parent fiber's `child`/`sibling` chain, carrying a `containerInfo` pointer at the foreign DOM node. The render phase treats it as a normal subtree: reconciliation, context propagation ([the `return`-chain walk](../state/context.md)), lanes, bailouts — all standard, because they all traverse fibers ([how-react-renders](./how-react-renders.md)). The divergence happens in **one place**: the commit mutation phase. When React inserts the portal's host children, it appends them into `containerInfo` instead of the parent fiber's `stateNode`. That single redirected `appendChild` is the entire trick — which is why *only* DOM-derived behaviors (the bottom half of the table) diverge.

### The event side: dispatch walks fibers, not DOM

[conditional-rendering-and-events](../foundations/conditional-rendering-and-events.md) owns the delegation model: React attaches its listener set at the root container and dispatches synthetic events by collecting handlers along the **fiber path** from the target upward — two passes, capture (root → target) then bubble (target → root). Portals stress-test exactly this design:

- **Getting the native event into React.** A portal container outside the app root isn't covered by the root's listeners — the native event bubbling up from the portal's DOM never passes through `#root`. React handles this by wiring its delegated listener set onto the portal's container too, the first time content portals there. Mechanically invisible to you; the takeaway is that portal events are *not* a special case you must plumb.
- **Then propagation follows fibers.** Once dispatched, the synthetic event walks the React tree: target fiber, `return`, `return`… collecting `onClickCapture`s downward and `onClick`s upward. The portal fiber is on that path like any parent — so **React-tree ancestors receive events from DOM-detached descendants.** A menu portaled to `<body>` from inside a clickable table row *will* fire the row's `onClick`. By design: the component tree is the unit of composition, and a modal conceptually *belongs to* the thing that opened it.
- **Native listeners never see this.** `document.addEventListener("click", …)` receives the browser's event with the browser's path: portal node → `overlay-root` → `body` → `document`. No row, no `#root`. Every "outside click" bug in the mistakes section is this row of the table colliding with the one above it.
- **`stopPropagation`, both flavors.** Synthetic `e.stopPropagation()` halts the *fiber* walk — React ancestors go quiet, but a native `document` listener still fires (its DOM path was never React's business). Conversely, a **native** listener inside the tree calling `stopPropagation()` before the event reaches React's container kills the synthetic dispatch entirely — the third-party-widget footgun: an embedded non-React datepicker that stops propagation makes your React `onClick`s on ancestors silently dead.

### The capture phase, and when it's legitimate

Every React event has a `on<Event>Capture` twin that fires on the root-to-target pass — before any child sees the event, and **before any child's `stopPropagation` can eat it**. The legitimate uses, catalogued (this is a short list on purpose — capture as a default is spaghetti):

1. **Observing non-bubbling events from a container.** `scroll` doesn't bubble (React 17 aligned with the DOM on this) — so a panel that must react to *any descendant* scrolling uses `onScrollCapture`. The anchored-popover walkthrough below needs exactly this:

```tsx
// close-on-any-descendant-scroll — the only clean option
<div className="table-scroller" onScrollCapture={() => setOpen(false)}>
  {rows}
</div>
```

2. **Dismissal that must survive `stopPropagation`.** A global overlay manager closing menus on any interaction can't rely on bubble handlers — any widget legitimately stopping propagation would exempt itself. A capture handler at the shell sees everything first:

```tsx
// Shell.tsx — closes all menus before any child's bubble handler fires
<div id="app-shell" onPointerDownCapture={closeAllMenus}>
  {children}
</div>
```

3. **Instrumentation.** Analytics/interaction logging at the app shell via `onClickCapture` observes every click with zero cooperation from feature code — and must never call `stopPropagation` itself, only observe:

```tsx
<div onClickCapture={(e) => analytics.click(e.target)} id="app-shell">
  {children}
</div>
```

4. **Interaction shields.** During a destructive pending state, a capture handler that `preventDefault()`s and stops propagation turns a region inert cheaply (the *proper* tool is the `inert` attribute — [conditional-rendering-and-events](../foundations/conditional-rendering-and-events.md)'s table — but capture shields survive in older codebases and you'll read them):

```tsx
<form onClickCapture={isPending ? (e) => e.stopPropagation() : undefined}>
  {/* all buttons inert while submitting */}
</form>
```

Outside these, prefer bubble handlers: capture inverts the "closest handler wins" intuition every other handler in the codebase follows.

## Basic usage

The mount-node pattern plus the minimal honest portal:

```html
<!-- index.html — a dedicated sibling keeps overlay stacking trivially inspectable -->
<body>
  <div id="root"></div>
  <div id="overlay-root"></div>
</body>
```

```tsx
// Tooltip.tsx
import { useState } from "react";
import { createPortal } from "react-dom";

export function Tooltip({ label, children }: { label: string; children: React.ReactNode }) {
  const [anchor, setAnchor] = useState<DOMRect | null>(null);

  return (
    <span
      onPointerEnter={(e) => setAnchor(e.currentTarget.getBoundingClientRect())}
      onPointerLeave={() => setAnchor(null)}
    >
      {children}
      {anchor !== null &&
        createPortal(
          <span
            role="tooltip"
            className="tooltip"
            style={{ position: "fixed", top: anchor.bottom + 6, left: anchor.left }}
          >
            {label}
          </span>,
          document.getElementById("overlay-root")!,
        )}
    </span>
  );
}
```

**And the honest 2026 caveat before you build a modal this way:** the platform caught up. Native `<dialog>.showModal()` and the popover API render into the browser's **top layer** — above every stacking context, no portal, no `z-index`, with focus trapping, `Escape`, `::backdrop`, and light-dismiss increasingly free. A `<dialog>` rendered *inline* in your component keeps React-tree context and events trivially (it never left the tree) *and* wins the stacking war. **When NOT to use a portal:** a standard modal or popover in a single-root app — reach for `<dialog>`/popover first. **Portals remain the tool for:** toast/notification layers (many concurrent items, app-controlled stacking), rendering into DOM you don't own (CMS slots, micro-frontend shells, `<td>`-forbidden markup escapes), custom overlay systems with cross-layer coordination, and anything anchored that must escape `overflow` clipping while staying richly interactive.

## Walkthrough: an anchored row menu, and both trees fighting back

The scenario: a 300-row admin table inside `overflow: auto`; each row has a "⋯" menu with Duplicate/Archive/Delete. Rendered inline, the menu clips at the table edge — the bottom rows' menus are unusable. Portal it, then fix the three bugs the two trees hand you.

**Step 1 — portal + anchor positioning.**

```tsx
// RowMenu.tsx
import { useLayoutEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";

interface RowMenuProps {
  onAction: (action: "duplicate" | "archive" | "delete") => void;
}

export function RowMenu({ onAction }: RowMenuProps) {
  const [open, setOpen] = useState(false);
  const buttonRef = useRef<HTMLButtonElement>(null);
  const menuRef = useRef<HTMLDivElement>(null);
  const [pos, setPos] = useState({ top: 0, left: 0 });

  // Measure-then-place: layout effect so the menu never paints at (0,0) first.
  useLayoutEffect(() => {
    if (!open || !buttonRef.current) return;
    const rect = buttonRef.current.getBoundingClientRect();
    setPos({ top: rect.bottom + 4, left: rect.right - 160 });
  }, [open]);

  return (
    <>
      <button
        ref={buttonRef}
        aria-haspopup="menu"
        aria-expanded={open}
        onClick={() => setOpen((v) => !v)}
      >
        ⋯
      </button>
      {open &&
        createPortal(
          <div
            ref={menuRef}
            role="menu"
            className="row-menu"
            style={{ position: "fixed", ...pos }}
          >
            <button role="menuitem" onClick={() => { onAction("duplicate"); setOpen(false); }}>Duplicate</button>
            <button role="menuitem" onClick={() => { onAction("archive"); setOpen(false); }}>Archive</button>
            <button role="menuitem" onClick={() => { onAction("delete"); setOpen(false); }}>Delete</button>
          </div>,
          document.getElementById("overlay-root")!,
        )}
    </>
  );
}
```

Clipping solved — `position: fixed` in `overlay-root`, immune to the table's overflow and any transformed ancestor. (The measure-in-layout-effect move is [useref-and-the-dom](../effects/useref-and-the-dom.md)'s timing table applied; the layout-vs-passive rules are [escape-hatches-audit](../effects/escape-hatches-audit.md)'s *(next article)*.)

**Step 2 — bug: opening the menu selects the row.** The row has `onClick={() => selectRow(id)}`. Click "Archive" and the row *also* gets selected — the synthetic event walked the fiber path: menuitem → menu → portal → `RowMenu` → `<tr>`. React-tree propagation, working as designed, against you. Two fixes, choose deliberately: stop the bubble at the menu boundary —

```tsx
<div role="menu" onClick={(e) => e.stopPropagation()} /* the menu is an interaction island */>
```

— which is honest here (nothing above a row menu has a legitimate claim on its clicks), or, where ancestors *do* need some events, guard at the receiver instead: `if (e.target instanceof HTMLElement && e.target.closest('[role="menu"]')) return;`. House lean: `stopPropagation` at genuine interaction islands (menus, modals, drag handles), target-guards when the ancestor's handler must stay omniscient.

**Step 3 — bug: the outside-click closer closes on inside clicks.** The standard dismiss listener:

```tsx
useEffect(() => {
  if (!open) return;
  function onPointerDown(e: PointerEvent) {
    const t = e.target as Node;
    if (buttonRef.current?.contains(t)) return; // the trigger toggles itself
    if (menuRef.current?.contains(t)) return;   // 🔑 the PORTAL node — without this line,
    setOpen(false);                             // clicking "Archive" closes before onClick fires
  }
  document.addEventListener("pointerdown", onPointerDown);
  return () => document.removeEventListener("pointerdown", onPointerDown);
}, [open]);
```

This is the two trees colliding in one function: the *native* listener sees the DOM path, where the menu is a stranger to the row — `rowEl.contains(target)` is `false` for menu clicks. The fix is checking containment against the **portal node itself** (and the trigger). Checking only the anchor is the single most common portal bug in the wild, and it presents as "my dropdown closes when I click the dropdown."

**Step 4 — bug: the menu doesn't follow its row on scroll.** `position: fixed` froze the menu to the viewport; the table scrolls under it. `scroll` doesn't bubble, so an `onScroll` on some ancestor never hears the table's scroll — this is capture use #1: on the table container (or a shell-level wrapper), `onScrollCapture={() => setOpen(false)}` — close-on-scroll, the pragmatic pattern for row-anchored menus (continuous repositioning is a job for a positioning library, and a different article's trade-off table). Add `Escape` at the menu (`onKeyDown`) and focus return to the trigger on close, and the interaction is honest; the full focus-trap treatment is [accessibility-in-react](../ecosystem/accessibility-in-react.md)'s *(Wave 4, planned)*.

**Step 5 — prove the React-tree row of the table.** Wrap each row's cells — including `RowMenu` — in the widget-level [error boundary](./error-boundaries.md) from article 17, and make a menu item throw. The fallback renders **in the row**, not in `overlay-root` chaos: the throw unwound the *fiber* tree. Same experiment with theme: the menu reads `useTheme()` correctly (React tree) but does *not* pick up the table's `--row-accent` CSS variable (DOM tree) — set theme-derived styles on the menu root explicitly.

## Real-world patterns

**The overlay layer.** One `#overlay-root` sibling, one `OverlayProvider` owning an ordered list of open overlays, portals appending in order — stacking becomes *mount order*, inspectable in the Elements panel, no z-index arms race. Toasts are the canonical tenant: `useToast()` (context — actions channel, [context](../state/context.md)) dispatches into the provider; the provider renders the stack through one portal. Per-toast timers follow the effects contract; per-toast errors hit the provider's boundary, not the page's.

**Choosing portal vs top layer.** The decision in one pass: *Is this a modal dialog or a simple popover in DOM you own?* → `<dialog>`/popover, inline, no portal. *Is it a stack the app orchestrates (toasts), DOM you don't own, markup-validity escape (`<div>` out of `<tbody>`), or an anchored surface needing custom interaction?* → portal. Mixed systems are fine — a design system's `Modal` on `<dialog>` and its `Toaster` on a portal are each using the right primitive.

**Native-listener interop, diagnosed.** When React handlers mysteriously stop firing inside some region, suspect a native `stopPropagation` between the target and React's container — embedded widgets are the usual culprit. Diagnosis: a temporary *capture-phase* native listener on `document` logs the event (capture beats their bubble-stop); fix by isolating the widget (its own wrapper handling its events at the boundary) rather than fighting it.

**Testing portal UIs.** RTL renders into `document.body`, and `screen` queries search the whole document — portal content is found with zero ceremony, `userEvent` clicks it like anything else. The two real gotchas: snapshot/`container` assertions only cover the render *container* (use `baseElement` when the portal must appear in a snapshot), and your test HTML needs the mount node. A `beforeEach` keeps suites honest:

```tsx
// vitest setup or describe-level hook
import { afterEach, beforeEach } from "vitest";

let overlayRoot: HTMLDivElement;
beforeEach(() => {
  overlayRoot = document.createElement("div");
  overlayRoot.id = "overlay-root";
  document.body.appendChild(overlayRoot);
});
afterEach(() => overlayRoot.remove());

// In the test itself — portal content visible to screen queries immediately:
it("shows the menu via portal", async () => {
  const user = userEvent.setup();
  render(<RowMenu onAction={vi.fn()} />);
  await user.click(screen.getByRole("button", { name: /⋯/i }));
  // menu lives in #overlay-root, but screen.getByRole finds it in document:
  expect(screen.getByRole("menu")).toBeInTheDocument();
});
```

## Reference

| API | Notes |
| --- | --- |
| `createPortal(children, domNode, key?)` | react-dom; children reconcile in place, commit into `domNode`; `key` matters when portaling in lists |
| `on<Event>Capture` | Root→target pass; fires before children and before their `stopPropagation` |
| Synthetic `e.stopPropagation()` | Halts the fiber walk; native document listeners unaffected |
| Native `stopPropagation()` (inside tree) | Can starve React's delegation entirely — interop footgun |
| `<dialog>.showModal()` / popover API | Top layer, no portal needed; prefer for standard modals/popovers |

(The two-trees governance table in **What it is** is the other half of this reference — between them they answer every "why did the portal do that?")

## Common mistakes

**1. Expecting DOM-style bubbling.** Surprise in both directions: the modal click that triggers the opener card's `onClick` (fiber walk reaches it), and the developer adding a React handler on `#overlay-root`'s *div in index.html* expecting to intercept portal events (it's not a React element; nothing is dispatched "through" it). Propagation is fibers; the DOM container is scenery.

**2. Outside-click checks against the anchor.** `anchorEl.contains(e.target)` is false for every click inside the portaled surface → "clicking my dropdown closes my dropdown." Containment checks run against the portal node (and trigger), per the walkthrough.

**3. Fighting stacking contexts instead of leaving them.**

```tsx
// 🔴 z-index war: the parent has transform: translateZ(0) — a new stacking context
// z-index: 9999 inside it still loses to z-index: 1 outside the context
<div style={{ transform: "translateZ(0)" }}>        {/* stacking context root */}
  <DropdownMenu style={{ zIndex: 9999 }} />         {/* capped inside context */}
</div>

// ✅ structural escape — the menu leaves the context entirely
{createPortal(<DropdownMenu />, document.getElementById("overlay-root")!)}
```

Any of `transform`, `filter`, `opacity < 1`, `will-change: transform`, `isolation: isolate`, or `contain: layout` creates a stacking context that caps z-indexes below it. The number is irrelevant; the escape is structural (portal or top layer), not numeric.

**4. Ignoring focus and the a11y tree.** Tab order follows DOM: focus walks out of a portaled modal into whatever follows `overlay-root`. Modals need trap + return-to-trigger + backgrounded content marked `inert` — or, better, `<dialog>` which ships all three. Screen-reader context (what "contains" the popup) also follows DOM — `aria-haspopup`/`aria-expanded` on the trigger carry the relationship the DOM lost.

**5. Expecting CSS to follow the React tree.** The themed card's `--accent` variable, its `font-size`, its `.dark` ancestor class — none reach portal content. Theme through context (React tree) and apply it as classes/inline styles at the portal's own root; a design system's portal components should each re-anchor the theme.

**6. Minting containers during render.** `document.createElement("div")` in the component body is a render-phase side effect (impure, StrictMode-hostile) and leaks nodes. Static mount node in `index.html`, or lazy-create in an effect/ref-callback with symmetric removal ([useref-and-the-dom](../effects/useref-and-the-dom.md)'s cleanup contract).

**7. `onScroll` on an ancestor to track descendants.** Doesn't bubble; the handler never fires; the anchored menu drifts. `onScrollCapture` (or a window/container listener) — capture use #1.

**8. Portaling list items without keys.** Multiple portals into one container from a list reconcile like any children — unkeyed, they inherit each other's state on reorder exactly as [rendering-lists-and-keys](./rendering-lists-and-keys.md) warns; `createPortal`'s third argument exists for this.

**9. Global `stopPropagation` hygiene.** An interaction island stopping *all* pointer events also silences the shell's capture-phase analytics… no — capture already ran (that's the point); what it silences is every *bubble* ancestor: drag-and-drop managers, outside-click closers of *other* overlays, keyboard shortcut scopes listening on bubble. Stop propagation at the narrowest honest boundary, and prefer target-guards where the ancestor ecosystem is rich.

## How this evolved

- **Pre-16:** the escape hatch was `unstable_renderSubtreeIntoContainer` — a second render root with hand-carried context. Genuinely unstable.
- **React 16:** `createPortal` lands with the defining decision — one React tree, events propagating through it regardless of DOM location. The two-trees model is born.
- **React 17:** delegation moves from `document` to the root container (multiple Reacts on one page stop fighting), portal containers get wired into the listener system, and `onScroll` stops incorrectly bubbling — the event system's portal story reaches its modern shape.
- **React 18–19:** portals unchanged; the *platform* moves — `<dialog>` top layer everywhere, then the popover API — which is why the modern portal story is smaller and more honest than the 2019 one: the browser took the modal use-case back, and portals kept the rest.

## Exercises

**1. Two-trees lab.** Portal a badge out of a themed, `onClick`-instrumented card into `#overlay-root`. Test five behaviors and record which tree governed each: `useTheme()` context, an inherited CSS variable, click bubbling to the card's handler, `Tab` order from the card's button, and a throw caught by the card's error boundary. Score yourself against the governance table before running.
*Hint: exactly two of the five follow the DOM. If you got three, re-check which handler you gave the card — native or React?*

**2. Fix the dropdown, fully.** Start from step 1 of the walkthrough (portal + positioning only) and introduce all three bugs' conditions: row `onClick` selection, an anchor-only outside-click closer, and a scrollable table. Fix each, then add `Escape`-to-close with focus return. Write the RTL test that clicks a menu item and asserts the row was *not* selected.
*Hint: the test is the `stopPropagation` decision made falsifiable — assert `onAction` fired AND the row-select spy didn't.*

**3. Boundary jurisdiction.** Build the toast layer (provider + one portal) and give the *page* and the *provider* separate error boundaries. Throw from inside one toast's render. Which fallback appears, where does the fallback *display*, and what did `onCaughtError` receive? Explain all three in terms of a single tree walk.
*Hint: the unwind climbs fibers — the portal fiber's `return` chain runs through the provider, not through `overlay-root`. Where the fallback then paints is just… where the boundary's children commit.*

## Summary

- One redirected `appendChild` at commit is the whole mechanism: portals are ordinary fibers whose host output lands in `containerInfo`. Everything fiber-driven — context, state, boundaries, reconciliation, **synthetic events** — ignores the relocation; everything DOM-driven — CSS, stacking, focus order, native listeners — honors it.
- Synthetic dispatch walks the fiber path in two passes; portal containers are wired into delegation automatically. React ancestors hear portal events; `document` listeners see only the DOM path — design your `stopPropagation`s and containment checks knowing which audience each has.
- Capture phase is a scalpel with four honest jobs: non-bubbling observation (`onScrollCapture`), stop-proof dismissal, shell instrumentation, and legacy interaction shields. Everything else stays on bubble.
- The 2026 division of labor: `<dialog>`/popover top layer for standard modals and popovers; portals for toast stacks, foreign DOM, markup escapes, and custom anchored systems. Both are fine; drifting z-indexes are not.
- Portal a11y is a DOM problem — trap, return, `inert`, and trigger ARIA carry the relationship the DOM tree dropped.

## See also

- [conditional-rendering-and-events](../foundations/conditional-rendering-and-events.md) — root delegation, synthetic events, and the ordering lore this article builds on
- [how-react-renders](./how-react-renders.md) — the fiber structures and commit phases behind the two-trees split
- [context](../state/context.md) — read-at-render-position, now mechanically grounded for portals
- [error-boundaries](./error-boundaries.md) — jurisdiction follows the fiber unwind, portals included
- [useref-and-the-dom](../effects/useref-and-the-dom.md) — anchor measurement and container-lifecycle cleanup
- [rendering-lists-and-keys](./rendering-lists-and-keys.md) — keyed portals in lists
- [accessibility-in-react](../ecosystem/accessibility-in-react.md) *(Wave 4, planned)* — focus traps, `inert`, and the full modal a11y contract

## References

- [createPortal — react.dev](https://react.dev/reference/react-dom/createPortal)
- [Responding to Events — react.dev](https://react.dev/learn/responding-to-events)
- [<dialog>: The Dialog element — MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/dialog)
- [Popover API — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Popover_API)
- [Changes to Event Delegation — React 17 release notes](https://legacy.reactjs.org/blog/2020/10/20/react-v17.html#changes-to-event-delegation)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.