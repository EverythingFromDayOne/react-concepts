---
article_id: accessibility-in-react
concept_folder: ecosystem
wave: 4
related:
  - effects/useref-and-the-dom
  - rendering/portals-and-the-event-system
  - ecosystem/testing
  - forms/forms-controlled-and-uncontrolled
  - forms/forms-at-scale
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Accessibility in React

> React renders exactly the DOM you describe — including inaccessible DOM. Accessibility isn't a React feature you switch on; it's a discipline React makes easy to skip, because a `<div onClick>` compiles and ships just like a `<button>`. The good news: almost every win is *not fighting the platform*. Use the real element, associate labels, put focus where the user expects it, and announce what changes silently. This article is the three things React adds on top of HTML semantics — ARIA-in-JSX, focus management via refs, and portals for dialogs — and the production judgment for when to stop hand-rolling and reach for a headless library.

## What it is

Accessible React is DOM that assistive technology (AT) — screen readers, switch devices, voice control — can operate. Three pillars carry almost all of it:

1. **Semantics** — ship real elements (`<button>`, `<nav>`, `<a>`, `<label>`), and use ARIA only to fill the gaps HTML can't express. The first rule of ARIA is *don't use ARIA*: a native `<button>` beats `<div role="button" tabIndex={0}>` on every axis.
2. **Focus management** — decide where keyboard focus goes when the DOM changes: on modal open/close, on a client route change, when a template swaps. React removes and replaces DOM freely, and focus doesn't follow unless you move it.
3. **Announcements** — for content that changes *without* a focus move (a toast, a search-result count, a form error), a live region tells AT what happened.

React's specific surface on top of this is small: the JSX naming for ARIA attributes, `useId` for stable label/description associations, refs for imperative focus, and portals for dialogs. React does *not* audit your accessibility — that's on you and your tests. Baseline here is React 19.2, targeting WCAG 2.2 AA.

## How it works under the hood

### The accessibility tree, and why real elements win

Browsers compute an **accessibility tree** from your DOM and ARIA — a parallel structure AT reads. A `<div role="button">` puts a "button" node in that tree, so a screen reader *announces* a button — but the div still isn't focusable, doesn't fire on Enter/Space, and has no pressed state. You'd have to add `tabIndex={0}`, keydown handlers for Enter and Space, and manage `aria-pressed` yourself, and you'd still miss platform details (Windows high-contrast, forced-colors). A `<button>` gives you all of it for free because the browser already wired the DOM node to the a11y tree. Every "just use the right element" recommendation is really "let the browser populate the a11y tree correctly."

### ARIA in JSX

JSX diverges from its usual camelCase for accessibility attributes, and the exceptions are load-bearing: `aria-*` and `data-*` stay **hyphenated and lowercase** (`aria-labelledby`, not `ariaLabelledBy`); `role` passes straight through; but `for` becomes **`htmlFor`** and `tabindex` becomes **`tabIndex`** (these two are camelCased because `for`/`class` are JS reserved-ish). Get the `aria-*` casing wrong and React silently renders a no-op attribute AT never sees.

### `useId`, and why not a counter

Associations need matching IDs on both nodes (`htmlFor`/`id`, `aria-describedby`/`id`). Hardcoding an ID breaks when a component renders twice; a module-level `nextId++` counter or `Math.random()` breaks under SSR, because the server and client must emit *identical* IDs for hydration to match — a mismatch either throws a hydration warning or silently unlinks the label ([the ssr-and-hydration article](../server/ssr-and-hydration.md#hydration-mismatch) owns why). `useId` generates IDs that are stable across renders and identical across the server/client boundary, and it's portal-safe. It is the only correct way to mint an ID for an ARIA association.

### `inert`: the focus-trap primitive

React 19 supports **`inert` as a boolean prop** (pre-19 needed a string-attribute workaround). `inert` removes an entire subtree from focus, clicks, tabbing, *and* the accessibility tree in one attribute — strictly more than `aria-hidden`, which hides from AT but leaves the subtree clickable and tabbable. When a modal opens, marking the rest of the page `inert` is the whole focus trap: focus physically cannot leave the dialog because everything else is gone from the tab order. Refs remain the escape hatch for the imperative `.focus()` call itself ([owned by the useref article](../effects/useref-and-the-dom.md#the-containment-embassy)).

## Basic usage

Right element first, ARIA only to fill gaps:

```tsx
import { useId } from "react";

// A real button — keyboard, focus, and role for free.
export function SaveButton({ onSave }: { onSave: () => void }) {
  return <button type="button" onClick={onSave}>Save</button>;
}

// A labelled, described input — associations via useId.
export function EmailField() {
  const id = useId();
  const hintId = useId();
  return (
    <div>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" aria-describedby={hintId} />
      <p id={hintId}>We'll only use this to sign you in.</p>
    </div>
  );
}

// An icon-only control needs an accessible name.
export function CloseButton({ onClose }: { onClose: () => void }) {
  return (
    <button type="button" aria-label="Close" onClick={onClose}>
      <XIcon aria-hidden="true" />
    </button>
  );
}
```

A live region announces changes that don't move focus. Render it **empty first**, then update it — a region inserted already-populated is often not announced:

```tsx
export function StatusAnnouncer({ message }: { message: string }) {
  // role="status" == aria-live="polite"; visually hidden but read by AT.
  return (
    <div role="status" aria-live="polite" className="sr-only">
      {message}
    </div>
  );
}
```

The `sr-only` class is load-bearing and easy to get wrong: `display: none` or `visibility: hidden` would hide the region from screen readers too, silently killing the announcement. Use the clip pattern, which removes it visually while leaving it in the accessibility tree:

```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

## Walkthrough — an accessible modal, end to end

The modal is the canonical hard case: it ties together focus movement, trapping, labelling, restore-on-close, Escape, and the a11y tree. We'll build it three ways — native first (the 2026 default), then the custom portal version for when you need control, then the honest production answer.

### Step 1 — Why the naive modal fails

```tsx
// ❌ Renders, demos fine, fails five WCAG criteria.
function BadModal({ open, children }: { open: boolean; children: React.ReactNode }) {
  if (!open) return null;
  return <div className="overlay"><div className="panel">{children}</div></div>;
}
```

No role (AT doesn't know it's a dialog), no name, focus stays on the trigger *behind* the overlay, Tab walks straight out into the page underneath, and Escape does nothing. Every one of those is a real barrier.

### Step 2 — The native `<dialog>` (your default)

The platform now does the hard parts. `showModal()` gives you a focus trap, Escape-to-close, top-layer rendering (no z-index wars, no portal needed), and a `::backdrop` — for free:

```tsx
// ConfirmDialog.tsx
import { useEffect, useId, useRef } from "react";

interface ConfirmDialogProps {
  open: boolean;
  onClose: () => void;
  onConfirm: () => void;
}

export function ConfirmDialog({ open, onClose, onConfirm }: ConfirmDialogProps) {
  const ref = useRef<HTMLDialogElement>(null);
  const titleId = useId();

  useEffect(() => {
    const dialog = ref.current;
    if (!dialog) return;
    if (open) dialog.showModal(); // native focus trap + Esc + top layer
    else dialog.close();
  }, [open]);

  return (
    <dialog ref={ref} aria-labelledby={titleId} onClose={onClose}>
      <h2 id={titleId}>Delete this item?</h2>
      <p>This action can't be undone.</p>
      <button type="button" onClick={onClose}>Cancel</button>
      <button type="button" onClick={onConfirm}>Delete</button>
    </dialog>
  );
}
```

`showModal()` (not `show()`) is what makes it modal and traps focus; the native `onClose` fires on Escape too, so one handler covers both close paths. For most confirm/alert dialogs, stop here.

### Step 3 — The custom portal version (when you need control)

When you need a dialog the native element can't express — bespoke animation, nested layering, a design-system API — render through a [portal](../rendering/portals-and-the-event-system.md#portals) and reproduce the four behaviors by hand:

```tsx
// Modal.tsx
import { useEffect, useId, useRef, type ReactNode } from "react";
import { createPortal } from "react-dom";

interface ModalProps {
  open: boolean;
  onClose: () => void;
  title: string;
  children: ReactNode;
}

export function Modal({ open, onClose, title, children }: ModalProps) {
  const panelRef = useRef<HTMLDivElement>(null);
  const restoreRef = useRef<HTMLElement | null>(null);
  const titleId = useId();

  useEffect(() => {
    if (!open) return;
    // 1. Remember what to restore focus to on close.
    restoreRef.current = document.activeElement as HTMLElement;
    // 2. Move focus into the dialog.
    panelRef.current?.focus();
    // 3. Escape closes.
    function onKey(e: KeyboardEvent) {
      if (e.key === "Escape") onClose();
    }
    document.addEventListener("keydown", onKey);
    return () => {
      document.removeEventListener("keydown", onKey);
      // 4. Restore focus to the trigger on close.
      restoreRef.current?.focus();
    };
  }, [open, onClose]);

  if (!open) return null;

  return createPortal(
    <div className="overlay" onClick={onClose}>
      <div
        ref={panelRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        tabIndex={-1}
        className="panel"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id={titleId}>{title}</h2>
        {children}
      </div>
    </div>,
    document.body,
  );
}
```

The focus **trap** comes from making the rest of the page `inert` while the modal is open — the React 19 boolean prop does it declaratively, and it beats a hand-rolled Tab-cycling handler because it also covers mouse, screen-reader virtual cursor, and dynamically-added elements:

```tsx
// App.tsx — background goes inert while any modal is open
export function App({ modalOpen }: { modalOpen: boolean }) {
  return (
    <>
      <main inert={modalOpen}>{/* the rest of the app */}</main>
      {/* Modal renders into document.body via the portal, outside <main> */}
    </>
  );
}
```

That's `role="dialog"` + `aria-modal` + a name (`aria-labelledby`), focus moved in, focus trapped via `inert`, Escape wired, and focus restored on close — the five things `BadModal` missed.

### Step 4 — Accessible errors for the form inside

A form in the dialog needs its errors *announced* and *associated*, not just colored red. Wire `aria-invalid`, point `aria-describedby` at the error's id, and give the error `role="alert"` so it's read the moment it appears (the [forms-at-scale](../forms/forms-at-scale.md#field-errors) and [forms-controlled](../forms/forms-controlled-and-uncontrolled.md) articles own the validation flow; here we make its output accessible):

```tsx
import { useId } from "react";

interface FieldProps {
  label: string;
  error?: string;
  value: string;
  onChange: (v: string) => void;
}

export function Field({ label, error, value, onChange }: FieldProps) {
  const id = useId();
  const errorId = useId();
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        aria-invalid={error ? true : undefined}
        aria-describedby={error ? errorId : undefined}
      />
      {error && (
        <p id={errorId} role="alert">
          {error}
        </p>
      )}
    </div>
  );
}
```

On submit, move focus to the first invalid field — a color change alone is invisible to keyboard and AT users.

### Step 5 — The production answer

Even done correctly, the modal has edge cases that will bite: iOS VoiceOver's rotor, focus returning to a trigger that unmounted, background scroll-lock jumping on iOS. In production, reach for a **headless a11y library** — [React Aria](https://react-spectrum.adobe.com/react-aria/) (Adobe) or [Radix Primitives](https://www.radix-ui.com/primitives) — which ship the behavior and ARIA unstyled, so you own the CSS and they own the corner cases. Understand the mechanism (this article) so you can wire and debug them; don't hand-roll a combobox or a menu from scratch. And whatever you ship, **test with a real screen reader** — VoiceOver, NVDA, TalkBack each interpret dialogs differently, and no amount of correct ARIA substitutes for hearing it.

## Real-world patterns

**Move focus on client route changes.** A full-page load moves focus to the top; a client-side [route change](../ecosystem/routing-react-router.md) does not — the user's focus is stranded on the link they clicked, and screen-reader users get no signal the page changed. On navigation, move focus to the new page's `<h1>` (give it `tabIndex={-1}` so it's programmatically focusable) or a skip target, and announce the new title in a live region. This single fix is the most-missed SPA a11y bug.

**Live regions, precisely.** `role="status"` / `aria-live="polite"` waits for a pause (use for toasts, saved-confirmations, result counts); `role="alert"` / `aria-live="assertive"` interrupts (use sparingly, for errors). Two gotchas: the region must exist in the DOM *before* it's populated (insert empty, then set text), and `aria-atomic="true"` makes AT re-read the whole region rather than just the diff. Pair a polite region with [Suspense](../concurrent/suspense.md) fallbacks and action pending states to narrate async UI.

**Semantics are orthogonal to styling.** A Tailwind-styled `<div>` that *looks* like a button is not a button — `className` never touches the accessibility tree ([the styling article](../ecosystem/styling-approaches.md#semantics) owns this boundary). Style the real element: `<button className="…">`. This is why "use a component library's Button" is good a11y advice and "copy the button's CSS onto a div" is a trap.

**Tests are an accessibility check.** Querying by role in RTL isn't just a testing style — `getByRole("button", { name: /save/i })` *fails* if your control has no accessible role or name, so a passing role-based test is a passing a11y smoke test ([the testing article](../ecosystem/testing.md#the-query-priority-ladder) established the query ladder; here's the payoff). Add `jest-axe` for a structural audit:

```tsx
import { axe } from "jest-axe";

test("dialog has no a11y violations", async () => {
  const { container } = render(<Modal open title="Settings">…</Modal>);
  expect(await axe(container)).toHaveNoViolations();
});
```

`axe` catches missing names, bad contrast in markup, and invalid ARIA — the mechanical failures — while role queries catch the interactive ones. Neither replaces a real screen reader, but together they gate regressions in CI.

**Never kill the focus ring blindly.** `outline: none` with no replacement makes the keyboard invisible. Use `:focus-visible` so mouse users don't see a ring but keyboard users always do.

**Landmarks and skip links.** Wrap regions in `<header>`, `<nav>`, `<main>`, `<footer>` so AT users can jump between them, and put a "Skip to content" link as the first focusable element. Keep heading order sane (one `<h1>`, no skipped levels) — screen-reader users navigate by heading.

**Respect motion preferences.** Gate non-essential animation behind `prefers-reduced-motion` (a CSS media query or `matchMedia` in JS) — vestibular disorders make large transitions genuinely painful.

## API / attribute reference

| In JSX | Renders / purpose |
| --- | --- |
| `htmlFor={id}` | `for` — links `<label>` to an input. |
| `aria-labelledby` / `aria-describedby` | Reference another element's `id` for name / description. |
| `aria-label` | Inline accessible name (icon-only controls). |
| `role="dialog"` + `aria-modal="true"` | Marks a modal dialog for AT. |
| `aria-invalid` / `role="alert"` | Field error state / announced error. |
| `aria-live` (`polite`/`assertive`), `aria-atomic`, `aria-busy` | Live-region behavior. |
| `tabIndex={0 \| -1}` | `0` = in tab order; `-1` = focusable only via `.focus()`. Never `> 0`. |
| `inert={boolean}` | Removes a subtree from focus/clicks/tabbing/a11y tree (React 19 boolean). |
| `useId()` | SSR- and portal-safe unique IDs for the associations above. |

## Common mistakes

**1. `<div onClick>` as a button.** No role, no keyboard, no focus. Every clickable thing is a `<button>` (or `<a>` if it navigates). If you catch yourself adding `role="button"` + `tabIndex` + key handlers, you're rebuilding `<button>` worse.

**2. Placeholder as label.** `placeholder="Email"` vanishes on input and isn't a reliable name. Use a real `<label htmlFor>`; keep hints in `aria-describedby`.

**3. `Math.random()` or a counter for association IDs.** Breaks under SSR (hydration mismatch) and on re-render. Use `useId`.

**4. Live region populated at mount.** A region inserted already-containing text often isn't announced. Render it empty, then set the message.

**5. Modal with no focus trap or restore.** Focus escapes to the page behind, and on close it's lost to `<body>`. Trap with `inert` (or native `<dialog>`), and restore to the trigger.

**6. Redundant or fighting ARIA.** `<button role="button">`, `aria-label` on an element with a visible text label, `role="list"` on a `<ul>` — ARIA that duplicates or overrides native semantics usually makes things worse. Prefer native; add ARIA only where HTML can't express the pattern.

**7. Removing the focus outline.** `outline: none` with no `:focus-visible` replacement strands keyboard users. Never remove focus indication without providing one.

**8. Positive `tabIndex`.** `tabIndex={3}` hijacks the natural tab order and creates a maze. Only `0` and `-1` are correct.

**9. Icon-only control with no name.** An SVG button reads as "button" with no label. Add `aria-label` on the control and `aria-hidden="true"` on the decorative icon.

**10. Errors shown only in color.** Red text with no `role="alert"`, no `aria-invalid`, and no focus move is silent for AT and invisible to colorblind users. Announce, associate, and move focus.

## How this evolved

Early React inherited HTML's accessibility model but the ecosystem drifted into `<div>`-soup with click handlers, because JSX made a styled div as easy to ship as a semantic element. Focus management was awkward in the class era and stayed clumsy through the `forwardRef` years, since moving focus into a child component meant threading refs by hand. React 18's `useId` fixed SSR-safe associations — the single most common a11y wiring, finally correct across hydration. React 19 removed more friction: [ref-as-prop](../effects/useref-and-the-dom.md#ref-as-a-prop) dropped the `forwardRef` ceremony around focusable components, and first-class `inert` turned focus-trapping from a MutationObserver hack into one declarative prop. In parallel the platform matured — `:focus-visible`, `inert`, `prefers-reduced-motion`, and a robust native `<dialog>` all landed and React just passes them through — and headless libraries (React Aria, Radix) emerged as the production answer to "don't author ARIA widgets by hand." The arc: from bolting ARIA onto divs, to leaning on native semantics and platform primitives while React stays out of the way.

## Exercises

**1. De-div a control.** Take a `<div onClick role="button" tabIndex={0}>` with hand-written Enter/Space handlers and replace it with a `<button>`. Confirm in the accessibility tree (DevTools) that you *lost no behavior* and deleted code. *Hint:* the keydown handlers and `role` all disappear; only `onClick` remains.

**2. Make a toast announce.** Add a live region that reads out "Saved" when a save succeeds, and prove it announces on the *second* save too. *Hint:* the region must be mounted empty before the message changes; if you conditionally render the whole region only when there's a message, it won't announce reliably — keep it mounted and change its text.

**3. Trap and restore focus.** Starting from the naive `BadModal`, add: focus moved into the panel on open, background `inert`, Escape-to-close, and focus restored to the trigger on close. *Hint:* capture `document.activeElement` in the open effect and call `.focus()` on it in the cleanup; test it by tabbing — you should never reach the page behind.

## Summary

React renders the DOM you write, accessible or not, so accessibility is a discipline, not a feature. Lead with real semantic elements and let the browser populate the accessibility tree; add ARIA only to fill genuine gaps. Manage focus explicitly — into dialogs, back to triggers, onto the new heading after a client route change — because React won't do it for you. Announce silent changes through live regions rendered empty-then-filled. Use `useId` for every ARIA association, `inert` (or native `<dialog>`) for focus traps, and reach for React Aria or Radix in production rather than hand-authoring widgets. Gate regressions with role-based tests and `jest-axe`, and confirm with a real screen reader. The through-line: don't fight the platform — React's job is to stay out of the way while you use it well.

## See also

- [useref and the DOM](../effects/useref-and-the-dom.md#the-containment-embassy) — the ref mechanics behind focus management.
- [Portals and the event system](../rendering/portals-and-the-event-system.md#portals) — where the custom modal renders.
- [Testing](../ecosystem/testing.md#the-query-priority-ladder) — role queries and `jest-axe` as the a11y regression gate.
- [Forms at scale](../forms/forms-at-scale.md#field-errors) / [controlled forms](../forms/forms-controlled-and-uncontrolled.md) — the validation flow whose output this wires for AT.
- [Styling approaches](../ecosystem/styling-approaches.md#semantics) — why `className` never touches the accessibility tree.
- [Routing (React Router)](../ecosystem/routing-react-router.md) — route-change focus management.

## References

- React docs — [`useId`](https://react.dev/reference/react/useId)
- MDN — [ARIA](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA) · [`inert`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert)
- WAI-ARIA — [Authoring Practices Guide (APG)](https://www.w3.org/WAI/ARIA/apg/)
- web.dev — [The `inert` attribute](https://web.dev/articles/inert)
- [React Aria](https://react-spectrum.adobe.com/react-aria/) · [Radix Primitives](https://www.radix-ui.com/primitives)

## Demo source

*Demo pending — see the roadmap's demo-hosting decision (StackBlitz vs. CodeSandbox vs. local `demos/`).*