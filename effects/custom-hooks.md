---
article_id: custom-hooks
concept_folder: effects
wave: 2
related:
  - effects/effects-and-synchronization
  - effects/useref-and-the-dom
  - foundations/rules-of-react
  - foundations/component-composition
  - rendering/memoization-and-the-compiler
react_baseline: "19.2"
status: { drafted: true, reviewed: false }
---

# Custom Hooks

> **Lead with this:** a custom hook is just a function whose name starts with `use` and that calls other hooks — no registration, no API, no runtime concept at all. What it buys is exactly what a good function buys: a *name* for an idea, a boundary that hides the gnarly synchronization details, and a unit you can test and reuse. What it does **not** buy is shared state — two components calling the same hook get two independent copies of everything inside. The craft is threefold: knowing *what* to extract (concepts, never lifecycles), designing *what comes back* (the return-shape conventions), and keeping the *contract* (reactive arguments stay reactive, event callbacks stay fresh, published hooks stay stable). This article locks all three, and graduates the hooks the rest of the library has been promising.

## What it is

```tsx
// A custom hook: a use-prefixed function that calls hooks. That's the whole definition.
export function useOnlineStatus(): boolean {
  const [online, setOnline] = useState(() => navigator.onLine);

  useEffect(() => {
    const goOnline = () => setOnline(true);
    const goOffline = () => setOnline(false);
    window.addEventListener("online", goOnline);
    window.addEventListener("offline", goOffline);
    return () => {
      window.removeEventListener("online", goOnline);
      window.removeEventListener("offline", goOffline);
    };
  }, []);

  return online;
}
```

Three properties define the species:

- **The `use` prefix is a contract, not magic.** It tells the linter to enforce the rules of hooks and exhaustive deps *inside* this function, and tells every reader that hook rules apply *at the call site*. Prefix a non-hook helper with `use` and you get phantom constraints; omit it from a real hook and the linter goes blind. Both directions matter.
- **Hooks share logic, not state.** Each call site's inner hooks land on *that component's* fiber. `useOnlineStatus()` in the header and in the sidebar means two subscriptions, two states, zero communication. When you catch yourself wanting two call sites to see the same value, the tool is [context](../state/context.md) or a store — a hook can *wrap* that sharing, but the hook itself doesn't create it.
- **They add zero capability.** Anything a custom hook does, the same code inlined in the component does identically. Extraction is purely an act of design — which is why the design rules below carry all the weight.

## How it works under the hood

Mechanically, a custom hook is *invisible to React*. There is no fiber entry, no wrapper, no boundary — the hooks called inside it land directly on the calling component's `memoizedState` list, in call order, exactly as if inlined ([how-react-renders](../rendering/how-react-renders.md) owns the list and its cursor). `useOnlineStatus` above contributes two nodes to its caller: one state hook, one effect hook. A component calling three custom hooks that each call three hooks has nine nodes on one flat list.

Everything else follows from that flattening:

- **The rules of hooks apply transitively.** Calling a custom hook inside a condition shifts the cursor for every hook after it — the violation lives at the call site, but the damage lands inside. The linter tracks this through the `use` prefix; this is the enforcement stack ([rules-of-react](../foundations/rules-of-react.md)) reaching through your abstraction.
- **Effects inside hooks are per-call-site.** Each caller gets its own effect instances, its own cleanups, its own StrictMode setup→cleanup→setup audit ([effects-and-synchronization](./effects-and-synchronization.md)). A hook with a leak leaks once per caller.
- **Reactive arguments are live wires.** Because the hook's body re-runs on every render of its caller, its parameters are re-supplied every render — they are *reactive values* and must appear in the deps of any effect that reads them. A hook that reads `roomId` in an effect but keys the effect on `[]` hasn't made `roomId` a config option; it's made a bug. The contract: **arguments a hook reads in effects are dependencies, and callers may change them at any time.**

That last point creates the central API-design tension of hooks — some inputs *should* re-synchronize (the room id), some *should not* (the `onMessage` callback, the retry count) — and React 19.2's `useEffectEvent` is the tool that resolves it, below.

## Basic usage

Extraction in its simplest honest form — two components with duplicated `matchMedia` plumbing become one hook and two one-liners:

```tsx
// use-media-query.ts
import { useEffect, useState } from "react";

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(
    () => window.matchMedia(query).matches,
  );

  useEffect(() => {
    const mq = window.matchMedia(query);
    const onChange = () => setMatches(mq.matches);
    setMatches(mq.matches); // re-sync if `query` changed between render and effect
    mq.addEventListener("change", onChange);
    return () => mq.removeEventListener("change", onChange);
  }, [query]); // reactive argument → dependency; callers may change it

  return matches;
}

// The specialization the effects article has been using:
export function useReducedMotion(): boolean {
  return useMediaQuery("(prefers-reduced-motion: reduce)");
}
```

Notes that generalize: the lazy `useState` initializer reads the *current* answer at mount (no flash of wrong value); `query` is honored as reactive; the return is a bare value because there's exactly one thing to return. One honest caveat for the record: "component state subscribed to an external source" is precisely the job `useSyncExternalStore` exists for — it closes a subtle update-before-subscribe gap and the SSR story. The effect version above is correct for a Vite SPA and matches the primitives you know; the upgrade is [escape-hatches-audit](./escape-hatches-audit.md)'s *(Wave 2, planned)* to own.

## Walkthrough: packaging the registration pattern

The brief: global keyboard shortcuts — `⌘K` opens the command palette. The raw version is the registration pattern from [effects-and-synchronization](./effects-and-synchronization.md), hand-rolled in every component that needs a key: a window listener, a fresh-callback problem, an input-focus guard, cleanup. Package it once, in two layers.

**Layer 1 — `useWindowEvent`: subscription and freshness, separated.**

```tsx
// use-window-event.ts
import { useEffect, useEffectEvent } from "react";

export function useWindowEvent<K extends keyof WindowEventMap>(
  type: K,
  handler: (event: WindowEventMap[K]) => void,
): void {
  // The handler is an EVENT, not a dependency: always call the latest one,
  // never re-subscribe because it changed.
  const onEvent = useEffectEvent(handler);

  useEffect(() => {
    const listener = (event: WindowEventMap[K]) => onEvent(event);
    window.addEventListener(type, listener);
    return () => window.removeEventListener(type, listener);
  }, [type]); // the subscription re-runs only when its identity — the type — changes
}
```

This is the whole `useEffectEvent` story in one hook. Split the inputs by role: `type` determines *what we're synchronized with* — reactive, in the deps, changing it tears down and resubscribes. `handler` is *what to do when it fires* — an event in the [effects article's](./effects-and-synchronization.md) sense; wrapping it in `useEffectEvent` gives a stable-identity function that always invokes the caller's latest render's version. No resubscribe churn when the caller passes an inline arrow (they will), no stale closure when the handler reads state (it does). It's the ref-mirror from [useref-and-the-dom](./useref-and-the-dom.md), inside React's own timing, with the plumbing deleted. The rules it keeps: effect-events are called from *inside effects or handlers*, never during render, and never listed as deps.

**Layer 2 — `useKeyboardShortcut`: policy on top of plumbing.**

```tsx
// use-keyboard-shortcut.ts
import { useWindowEvent } from "./use-window-event";

interface ShortcutOptions {
  /** Fire even when focus is inside an input/textarea/contentEditable. Default false. */
  allowInEditable?: boolean;
}

function isEditableTarget(target: EventTarget | null): boolean {
  if (!(target instanceof HTMLElement)) return false;
  return (
    target.isContentEditable ||
    target.tagName === "INPUT" ||
    target.tagName === "TEXTAREA" ||
    target.tagName === "SELECT"
  );
}

function matchesCombo(event: KeyboardEvent, combo: string): boolean {
  const parts = combo.toLowerCase().split("+");
  const key = parts.at(-1)!;
  const wantsMod = parts.includes("mod");
  const isMac = navigator.platform.toLowerCase().includes("mac");
  const modHeld = isMac ? event.metaKey : event.ctrlKey;

  if (wantsMod !== modHeld) return false;
  if (parts.includes("shift") !== event.shiftKey) return false;
  if (parts.includes("alt") !== event.altKey) return false;
  return event.key.toLowerCase() === key;
}

export function useKeyboardShortcut(
  combo: string, // "mod+k", "shift+/", "escape"
  onTrigger: (event: KeyboardEvent) => void,
  { allowInEditable = false }: ShortcutOptions = {},
): void {
  useWindowEvent("keydown", (event) => {
    if (!allowInEditable && isEditableTarget(event.target)) return;
    if (!matchesCombo(event, combo)) return;
    event.preventDefault();
    onTrigger(event);
  });
}
```

(One fence for the record: everything in this walkthrough reads `window` and `navigator` freely — correct for the Vite SPA baseline. The guards these hooks would need under server rendering are [ssr-and-hydration](../server/ssr-and-hydration.md)'s *(Wave 3, planned)* story.)

Hooks composing hooks, and the list stays flat: a component calling `useKeyboardShortcut` carries `useWindowEvent`'s hooks on its own fiber. Note the API split three ways — `combo` is reactive (change it, the *behavior* follows on the next event; no resubscribe needed since the check runs per-event), `onTrigger` rides the effect-event freshness from layer 1 automatically, and `allowInEditable` is policy config with a default. Each input's role is legible from the signature.

**Layer 3 — the consumer.**

```tsx
// CommandPalette.tsx
import { useState } from "react";
import { useKeyboardShortcut } from "./use-keyboard-shortcut";

export function CommandPalette() {
  const [open, setOpen] = useState(false);

  useKeyboardShortcut("mod+k", () => setOpen((prev) => !prev));
  useKeyboardShortcut("escape", () => setOpen(false));

  if (!open) return null;
  return <div role="dialog" aria-modal="true">{/* … */}</div>;
}
```

Thirty lines of listener plumbing per shortcut became one declarative line each — and the inline arrows are *correct*, not a compromise, because freshness was solved a layer down. This is what "hiding a synchronization contract behind a value-in/value-out signature" means in practice.

**Layer 4 — test it.** Two complementary styles:

```tsx
// use-keyboard-shortcut.test.tsx
import { renderHook } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { useKeyboardShortcut } from "./use-keyboard-shortcut";

it("fires on the combo and honors the latest handler", async () => {
  const user = userEvent.setup();
  const first = vi.fn();
  const second = vi.fn();

  const { rerender } = renderHook(({ fn }) => useKeyboardShortcut("mod+k", fn), {
    initialProps: { fn: first },
  });

  rerender({ fn: second }); // caller re-rendered with a new inline handler
  await user.keyboard("{Meta>}k{/Meta}");

  expect(first).not.toHaveBeenCalled(); // stale closure would fail here
  expect(second).toHaveBeenCalledTimes(1);
});

it("ignores keystrokes inside inputs", async () => {
  const user = userEvent.setup();
  const onTrigger = vi.fn();
  renderHook(() => useKeyboardShortcut("mod+k", onTrigger));

  const input = document.body.appendChild(document.createElement("input"));
  input.focus();
  await user.keyboard("{Meta>}k{/Meta}");

  expect(onTrigger).not.toHaveBeenCalled();
  input.remove();
});
```

`renderHook` mounts the hook inside a throwaway component; `rerender` with new props is how you assert the reactive/fresh contracts — the first test is *specifically* the useEffectEvent guarantee, pinned. For hooks whose behavior only makes sense with real UI (focus, refs, measurements), skip `renderHook` and test through a small probe component instead; the full toolchain stance is [testing](../ecosystem/testing.md)'s *(Wave 4, planned)*.

## Real-world patterns

**`useDebouncedValue`, graduated.** The [search-race-condition](../recipes/data-fetching/search-race-condition.md) recipe's debounce-the-derived stage, in its final packaged form:

```tsx
export function useDebouncedValue<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [value, delayMs]);

  return debounced;
}
```

Every line is a convention paying off. This is the **timer exception** from [effects-and-synchronization](./effects-and-synchronization.md) — an effect legitimately setting state, because the entire point is a *time-shifted copy*; equivalently, it's the **deliberate-lag exception** from [derive-vs-mirror](../state/usereducer-and-state-structure.md): a stored value that is *supposed* to disagree with its source for `delayMs`. The cleanup makes the whole timing correct — each new `value` cancels the previous timer, so only stillness flushes. Two boundary notes for the file's doc comment: on mount, `debounced` equals `value` immediately (no initial delay); and if the lag you need is *scheduling* lag ("update when React has time") rather than *wall-clock* lag ("update after the user pauses"), the tool is `useDeferredValue` — [concurrent-rendering](../concurrent/concurrent-rendering.md) *(Wave 3, planned)* draws that line precisely.

**Extraction rules.** When does inline effect code deserve a hook?

- **Extract concepts, never mechanisms.** The name test: if the best name describes *purpose* (`useOnlineStatus`, `useChatRoom`, `useKeyboardShortcut`), extract; if it describes *machinery* (`useMount`, `useUpdateEffect`, `useEffectOnce`), don't — lifecycle wrappers re-impose the mental model effects replaced, and they hide deps from the linter by design.
- **Extract to encapsulate a synchronization contract** — subscription + freshness + cleanup + edge cases behind a signature. The walkthrough is the template: the consumer can't misuse what it can't see.
- **Twice is a signal, once can be enough.** Duplication justifies extraction, but so does isolation: a component whose render logic is drowning in one gnarly effect reads better as `useAutosave(draft)` even with a single caller.
- **A hook's arguments are its deps contract.** Design signatures so every parameter's role is legible: reactive value (re-synchronizes), event callback (fresh via `useEffectEvent`), or one-shot config (read at mount — name it `initial*` or document it, per the [initial-contract](../state/state-and-usestate.md)). The worst hook APIs are the ones where the caller can't tell which is which.

**Return-shape conventions.**

| Shape | When | Example |
| --- | --- | --- |
| Bare value | One thing comes back | `useDebouncedValue`, `useReducedMotion` |
| Tuple `[value, action]` | One value + one action, and callers rename (multiple instances per component) | `useToggle` → `const [isOpen, toggleOpen] = …` |
| Object `{ … }` | 3+ members, or named capabilities callers pick from | `{ status, data, error, retry }` |
| Nothing (`void`) | Pure registration — the hook *is* its side effect | `useKeyboardShortcut`, `useDocumentTitle` |

Two rules that outrank the table. **Status comes back as a discriminated union**, never boolean soup — the return shape inherits all of [state-structure's](../state/usereducer-and-state-structure.md) discipline, because it *is* state, exported. And **invariants don't leak**: if the state has rules, return named actions (`retry`, `dismiss`) that enforce them — handing back a raw `setState` lets every caller break what the hook was built to guarantee.

**Published hooks are library boundaries.** Inside compiled app code, the Compiler stabilizes the callbacks and objects your hooks return. A hook shipped in a package cannot assume its consumers compile — so returned callbacks get `useCallback`, returned objects get `useMemo`, each with the boundary comment, and the identity guarantee goes in the docblock ("`retry` is stable across renders"). An unstable return from a library hook detonates in the *consumer's* deps — infinite effect loops they can't see the cause of. This is the same boundary case as context provider values ([context](../state/context.md)); [memoization-and-the-compiler](../rendering/memoization-and-the-compiler.md) *(Wave 2, planned)* owns verifying which side of the boundary you're on.

**Hook or component? The decision rule.** [component-composition](../foundations/component-composition.md) owns the history (mixins → HOCs → render props) and the escalation ladder; here's the modern boundary between its top rung and hooks. **If the abstraction produces *values*, it's a hook. If it owns *structure*, it's a component.** Hooks return data into the caller's render scope and impose zero markup — reuse of logic. Components (including render-prop components) earn their place when the abstraction must *render* something: establish a boundary (error boundaries, Suspense), mount a provider, own DOM (portals, virtualized list internals), or wrap children. The render-prop form specifically survives where the varying part is *what to render per item* inside structure the abstraction owns — `<VirtualList renderRow={…}>` can't be a hook, because the list owns the rows' existence. When both would work, the hook wins: flat call site, no tree noise, values in scope.

## Reference

| Contract | Rule |
| --- | --- |
| Naming | `use` prefix iff the function calls hooks — both directions enforced |
| State | Per call site, always; sharing is context/stores, which hooks may wrap |
| Rules of hooks | Apply transitively at every call site |
| Reactive args | Anything read by inner effects is a dep; callers may change it anytime |
| Event args | Wrap in `useEffectEvent`: stable identity, latest values, never a dep, never called in render |
| One-shot config | Read at mount; name `initial*` or document; changing it does nothing |
| Returns (app code) | Compiler stabilizes; write plainly |
| Returns (published) | Stabilize manually + comment; document identity guarantees |
| Rendering | Hooks never return JSX; structure-owning abstractions are components |

## Common mistakes

**1. Lifecycle wrappers.**

```tsx
function useMount(fn: () => void) {
  useEffect(fn, []); // 🔴 hides fn's deps from the linter, resurrects "on mount" thinking
}
```

Effects synchronize; they don't "run on mount." Every real use of `useMount` is either a subscription (write the effect with its real deps) or a one-time imperative act (often an event, not an effect at all).

**2. The `use` prefix pointing the wrong way.** `useFormatCurrency(n)` that calls no hooks drags hook constraints onto a plain function; `getOnlineStatus()` that calls `useState` hides a real hook from the linter *and* from readers who'd never suspect call-order rules apply. The prefix is load-bearing metadata — keep it truthful.

**3. Expecting two call sites to share.**

```tsx
// Header.tsx        → useCart() // 🔴 its own cart
// CheckoutPage.tsx  → useCart() // 🔴 a different cart
```

If `useCart` holds `useState`, these are strangers. Shared state needs a shared home — context or a store — with the hook as the *reading interface*: `useCart` wrapping `use(CartContext)` is the standard packaging.

**4. Conditional hook calls, transitively.** `if (enabled) useKeyboardShortcut(…)` shifts the cursor exactly like any conditional hook. Move the condition *inside* (`enabled` as a param the effect checks, or an early return in the handler) — the hook call itself stays unconditional.

**5. Freezing a reactive argument.**

```tsx
function useChatRoom(roomId: string) {
  useEffect(() => {
    const conn = connect(roomId);
    return () => conn.close();
  }, []); // 🔴 roomId changes; the connection doesn't — stale room forever
}
```

Arguments are live. If the caller can pass state, the hook must honor changes — deps, per the contract. (And if some argument genuinely *shouldn't* re-synchronize — it's an event or a one-shot — say so with `useEffectEvent` or `initial*`, not an empty array.)

**6. Options objects as accidental dynamite.** `useThing(options)` with `useEffect(…, [options])` and callers writing `useThing({ retries: 3 })` inline → new object every render → resubscribe storm. Take primitives (`useThing(retries)`), destructure to primitive deps inside, or split the object into reactive args + effect-event callbacks. Don't push a `useMemo` requirement onto every caller.

**7. Unstable returns from a published hook.** The consumer writes `useEffect(load, [load])` with your returned `load`, and their effect runs every render. At the library boundary this is your bug, not theirs — stabilize and document.

**8. `useFetch`, the thousandth reinvention.** A fetch hook without dedup, caching, invalidation, or abort discipline re-derives every bug in the [search-race-condition](../recipes/data-fetching/search-race-condition.md) arc, once per project. The abort pattern is locked in [effects-and-synchronization](./effects-and-synchronization.md); the real answer for server state is the query cache ([data-fetching-tanstack-query](../ecosystem/data-fetching-tanstack-query.md) *(Wave 4, planned)*) — per the placement rule, hooks don't exempt you.

**9. Returning JSX.** A `useModal` that returns `<dialog>` markup is a component wearing a prefix — it now owns structure, which is the component side of the decision rule, and it breaks every expectation (can't be composed as children, can't take a key). Return state + actions and let a component render, or commit to being `<Modal>`.

**10. Extracting the irrelevant, missing the concept.** A grab-bag `useHelpers()` returning six unrelated things is worse than inline code — it couples six change cadences and names nothing. Extraction earns its boundary when the inside is *cohesive*: one concept, one contract, one reason to change. (Same cohesion test as reducers and context channels — it's always the same test.)

## How this evolved

- **Mixins (`createClass` era):** the first reuse story — implicit state merging, name collisions, untraceable data sources. Officially regretted.
- **HOCs (2015–17):** `withRouter(withTheme(withData(Component)))` — reuse via wrapping, at the cost of wrapper-hell trees, prop-name collisions, and static typing pain.
- **Render props (2017–18):** explicit and typeable, but logic-reuse-as-JSX meant pyramid nesting for anything compositional.
- **Hooks (16.8):** the same reuse as flat function calls — composition became the call stack instead of the component tree. HOCs and render props retreated to the structural niches they still legitimately own.
- **React 19 / 19.2:** `use()` for conditional context/promise reads widens what hooks can express; **`useEffectEvent` lands as the missing primitive** — the reactive-vs-event split, previously hand-built with ref mirrors, becomes a one-liner with the right semantics.
- **Compiler era:** app-code hooks shed their `useCallback`/`useMemo` scaffolding; manual stabilization consolidates at the published-library boundary, exactly where human judgment still owns the contract.

## Exercises

**1. Extract by name.** Find (or write) two components that both track whether the tab is visible via `document.visibilityState` listeners. Extract `useDocumentVisible(): boolean`, choosing the return shape and the initial-read strategy. Write a `renderHook` test that dispatches a `visibilitychange` event and asserts the flip.
*Hint: `Object.defineProperty(document, "visibilityState", …)` or the `happy-dom`/`jsdom` equivalents let the test lie about visibility; the lazy `useState` initializer is what keeps the first render honest.*

**2. The interval that stays honest.** Implement `useInterval(callback: () => void, delayMs: number | null)` where the callback is always fresh, the interval never resets when the callback identity changes, changing `delayMs` *does* reset it, and `null` pauses. Then write the deps-only version and list what breaks.
*Hint: which parameter is reactive and which is an event? The deps-only version either resets the interval on every caller render (callback in deps) or freezes the callback (empty deps) — the exercise is proving there was no third option before `useEffectEvent`.*

**3. Draw the boundary.** Take a render-prop `<MouseTracker>{(pos) => …}</MouseTracker>` and convert it to `useMousePosition()`. Then take `<ErrorBoundary fallback={…}>` and explain — in one paragraph, in terms of the decision rule — why the same conversion is impossible.
*Hint: what does each abstraction own — values, or a place in the tree where rendering can be caught and replaced?*

## Summary

- A custom hook is a named, testable boundary around stateful logic — mechanically invisible (its hooks flatten onto the caller's fiber), transitively rule-bound, and per-call-site in everything it holds.
- Extract *concepts* with purpose names; never wrap lifecycles. One gnarly synchronization contract behind a clean signature is the canonical win.
- Design signatures by role: reactive args re-synchronize (they're deps), event callbacks stay fresh without resubscribing (`useEffectEvent`), one-shot config says so (`initial*`). The caller should read the contract off the signature.
- Return shapes: bare value → tuple → object as members grow; unions for status; named actions over raw setters; `void` for pure registration hooks.
- App code returns plainly (Compiler stabilizes); published hooks stabilize manually, comment why, and document identity guarantees — the library boundary is where manual memo lives now.
- Values → hook; structure → component. The render-prop form survives exactly where the abstraction owns the markup around a per-item hole.

## See also

- [effects-and-synchronization](./effects-and-synchronization.md) — the synchronization contract, timer exception, and registration pattern this article packages
- [useref-and-the-dom](./useref-and-the-dom.md) — the ref-mirror that `useEffectEvent` replaces, and the embassy hooks often wrap
- [rules-of-react](../foundations/rules-of-react.md) — transitive enforcement and why the `use` prefix is load-bearing
- [component-composition](../foundations/component-composition.md) — the escalation ladder, HOC/render-prop history, and the structural niches components keep
- [usereducer-and-state-structure](../state/usereducer-and-state-structure.md) — the shape discipline hook returns inherit
- [search-race-condition](../recipes/data-fetching/search-race-condition.md) — where `useDebouncedValue` came from, and why `useFetch` shouldn't exist
- [memoization-and-the-compiler](../rendering/memoization-and-the-compiler.md) *(Wave 2, planned)* — verifying the app-code/library-boundary split
- [escape-hatches-audit](./escape-hatches-audit.md) *(Wave 2, planned)* — the `useSyncExternalStore` upgrade for subscription hooks

## References

- [Reusing Logic with Custom Hooks — react.dev](https://react.dev/learn/reusing-logic-with-custom-hooks)
- [useEffectEvent — react.dev](https://react.dev/reference/react/useEffectEvent)
- [Separating Events from Effects — react.dev](https://react.dev/learn/separating-events-from-effects)
- [Rules of Hooks — react.dev](https://react.dev/reference/rules/rules-of-hooks)
- [renderHook — testing-library.com](https://testing-library.com/docs/react-testing-library/api/#renderhook)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.