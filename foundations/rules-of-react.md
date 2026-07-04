---
article_id: rules-of-react
concept_folder: foundations
wave: 1
related:
  - thinking-in-react
  - state-and-usestate
  - effects-and-synchronization
  - memoization-and-the-compiler
  - react-compiler-deep-dive
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# The Rules of React

> **Lead with this:** React's rules are not etiquette. They are the load-bearing assumptions of four machines you already rely on: the fiber's hook list (which matches hooks *by call order*), StrictMode (which double-invokes to *audit* purity), concurrent rendering (which discards and replays renders it may never commit), and the React Compiler (which memoizes your code *on the assumption* the rules hold — and silently skips components where they don't). Every previous article made you feel a rule mechanically: the mutation that swallowed an update, the hook after an early return, the inline component that remounted per keystroke. This capstone states the contract precisely, shows the enforcement stack that checks it, runs a clinic on the violations that compile fine and break subtly — and, just as important, marks the *legal exceptions* people wrongly avoid. Following the rules is no longer just correctness; in the Compiler era, it's where the performance comes from.

## What it is

Three rule families, each guarding a different machine:

**1. Components and hooks must be pure.** During render — the component body, plus everything React calls during it (`useState` initializers, updater functions, `useMemo` callbacks) — code must be *idempotent* (same inputs, same output, every time) and *externally unobservable* (no mutations that outlive the call, no I/O, no side effects). Concretely:

- **Props, state, and hook arguments/returns are immutable.** They are shared history — other components hold references to them ([`../state/state-and-usestate.md`](../state/state-and-usestate.md) showed the swallowed update; [`components-and-props.md`](components-and-props.md) the frozen props).
- **Values freeze at handoff.** Once you've passed a value to JSX, to `setState`, or returned it from a hook, it's been *seen* — mutating it afterward is mutating someone else's snapshot. Before handoff, it's yours (the crucial exception below).
- **Local mutation is completely legal.** Purity is about external observability, not the `push` keyword. Creating a fresh array/object during render and mutating it *before* handing it over is fine, idiomatic, and often clearest:

```tsx
// ✅ Pure. `rows` was born in this render and mutated only before handoff.
const rows = [];
for (const item of items) {
  if (item.visible) rows.push(<Row key={item.id} item={item} />);
}
return <ul>{rows}</ul>;
```

- **No non-determinism in render.** `Math.random()` and `Date.now()` in the body make the component non-idempotent — StrictMode's double render literally shows you two different answers. Randomness and time belong in event handlers, in effects, or in a lazy initializer when "sampled once at mount" is genuinely the semantics.
- **Refs are not readable or writable during render** (initialization excepted). `ref.current` is the designated mutable cell — which is exactly why touching it in render breaks idempotence ([`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md)).

**2. React calls components and hooks — you don't.** `<Card />`, never `{Card(props)}` ([`jsx-and-rendering.md`](jsx-and-rendering.md) traced the fiber-identity corruption); hooks are called by name at the top of components and custom hooks, never stored in variables, passed around, or invoked dynamically. React owns the call so it can attach state, identity, and DevTools presence to the right fiber.

**3. The Rules of Hooks.** Hooks are matched to their fiber slots purely by call order ([`thinking-in-react.md`](thinking-in-react.md) showed the linked list), so the order must be identical every render: top level only — not in conditions, loops, or nested functions; not after a conditional early return; not in event handlers or effects' bodies; and — the one that surprises seniors — **not inside `try`/`catch`/`finally` blocks**, because a throw between hook calls desynchronizes the list just like a skipped branch does. Callable from exactly two places: component functions and custom hooks ([`../effects/custom-hooks.md`](../effects/custom-hooks.md)).

## How it works under the hood

### What each machine buys with your compliance

The rules read as restrictions until you see the purchase order:

| Machine | What it does | The rule it spends |
| --- | --- | --- |
| Fiber hook list | Hands each `useState`/`useEffect` its slot by call position | Stable call order (Rules of Hooks) |
| StrictMode | Double-invokes render, initializers, updaters; audit-cycles effects | Idempotence — a pure component renders twice invisibly ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)) |
| Concurrent rendering | Pauses, discards, and replays renders that never commit | External unobservability — a discarded render must leave no trace ([`../concurrent/concurrent-rendering.md`](../concurrent/concurrent-rendering.md)) |
| Server prerendering | Renders components off-DOM, possibly never hydrating them | Same — render can't assume it "happened" for the user |
| **React Compiler** | Memoizes components and values automatically | *All of it* — same inputs → same output is the entire legal basis for caching your code |

The Compiler line is the era-defining one. Pre-Compiler, breaking purity produced weird bugs *sometimes*. Now it also produces a concrete, measurable cost *always*: the Compiler statically analyzes each component, and **where it can't prove the rules hold, it bails out and compiles nothing** — your component silently forfeits automatic memoization while its rule-following neighbors get it free. The rules stopped being advice and became the optimization boundary ([`../rendering/react-compiler-deep-dive.md`](../rendering/react-compiler-deep-dive.md) shows the bailout diagnostics).

### The enforcement stack, bottom to top

You don't hold the rules in your head; you run the checkers:

1. **`eslint-plugin-react-hooks` (recommended config)** — the floor. Call-order violations, missing deps, and (since the Compiler-powered rules landed in the plugin) a static purity analysis that flags mutation-during-render and friends *at edit time*. In this project it's non-negotiable, and `eslint-disable`-ing it is treated as deleting a test.
2. **Development-mode freezing** — React `Object.freeze`s props and state objects in dev, so many mutations throw immediately instead of corrupting quietly.
3. **StrictMode** — the runtime auditor: double-invocation makes non-idempotence *visible* (two different timestamps, doubled logs) and the effect audit cycle makes missing cleanup visible. Always on in this project; "it works without StrictMode" is a bug report, not a defense.
4. **The Compiler itself** — the final examiner. Its bailouts are queryable (lint diagnostics; DevTools badges compiler-optimized components), turning "am I compliant?" into something you can grep for rather than argue about.

### Where every kind of code belongs

The rules, inverted into a placement guide — most "rule violations" are just code in the wrong room:

| Code | Home | Why |
| --- | --- | --- |
| Computing values from props/state | Render body | Pure derivation is what render *is* |
| Building throwaway structures | Render body (local mutation) | Born and handed off in one call |
| Responding to interaction; writes, navigation, analytics | Event handlers | The event-first rule ([`conditional-rendering-and-events.md`](conditional-rendering-and-events.md)) |
| Synchronizing external systems | Effects, with cleanup | The survivors of the decision tree ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)) |
| Randomness, timestamps, ids | Handlers/effects; lazy initializer if "once at mount" is the real semantic | Non-determinism can't live where idempotence is required |
| Reading mutable external sources | `useSyncExternalStore` | Render-time reads of changing globals tear under concurrency ([`../effects/escape-hatches-audit.md`](../effects/escape-hatches-audit.md)) |
| Mutable instance-ish values (timers, connections, DOM handles) | Refs, touched from handlers/effects | The sanctioned mutable cell, outside render's jurisdiction |

## Walkthrough — the violation clinic

One realistic component, five violations. Every one compiles, every one "works" in a happy-path demo, and each is caught by a different layer of the enforcement stack. The patient:

```tsx
// ❌ InvoicePanel — do not ship
let renderTally = 0;                                             // (E)

export function InvoicePanel({ invoice, items }: InvoicePanelProps) {
  renderTally++;                                                 // (E)
  if (!invoice) return null;                                     // (A)

  const [expanded, setExpanded] = useState(false);               // (A)

  const sorted = items.sort((a, b) => b.amount - a.amount);      // (B)
  const generatedAt = new Date().toLocaleTimeString();           // (C)

  return (
    <section>
      <h3>Invoice {invoice.number} — generated {generatedAt}</h3>
      {StatusBadge({ status: invoice.status })}                  {/* (D) */}
      <button onClick={() => setExpanded((e) => !e)}>Details</button>
      {expanded && <ul>{sorted.map((i) => <li key={i.id}>{i.label}: {i.amount}</li>)}</ul>}
    </section>
  );
}
```

**(A) Hook after a conditional return** — caught by the lint plugin instantly (`React Hook "useState" is called conditionally`). The first render where `invoice` is null skips the hook; the next render's hook list is off by one, and the crash message blames a component three files away. Fix: hooks first, guards after — swap the two lines. Felt before in: [`conditional-rendering-and-events.md`](conditional-rendering-and-events.md)'s placement rule.

**(B) Mutating a prop** — `Array.prototype.sort` sorts *the caller's array* in place. Symptom in the wild: a sibling component rendering the same `items` mysteriously re-orders when this panel mounts — action at a distance, the defining smell of shared-history mutation. Dev-mode freezing throws for direct prop-object mutation; array methods on nested references often slip past it, which is why the Compiler-powered lint rule flags it statically. Fix: `items.toSorted(...)` — the non-mutating set from [`../state/state-and-usestate.md`](../state/state-and-usestate.md).

**(C) `new Date()` during render** — non-idempotent. StrictMode makes it undeniable: the double render produces two timestamps, and any memoization (manual or Compiler) would legitimately freeze an arbitrary one. The fix is a design question, not a syntax one: if it's "when the invoice was generated," it's *data* — comes from the invoice. If it's "when the user opened this," it's an event — sampled in a handler or a lazy initializer (`useState(() => new Date())` — pure-at-call, once semantics explicit).

**(D) Calling a component as a function** — `StatusBadge`'s hooks (today or after next quarter's refactor) register on *this* fiber; its conditional appearance elsewhere becomes a hook-order crash here. Lint catches it; the fix is two characters of JSX: `<StatusBadge status={invoice.status} />`. Felt before in: [`jsx-and-rendering.md`](jsx-and-rendering.md).

**(E) The render tally** — the popular "count my renders" snippet is a genuine rules violation twice over: a module-level write during render (external mutation — two mounted panels now share one corrupted counter), and it's exactly the kind of observable side effect that makes a render non-discardable. StrictMode shows it counting double; concurrent replays make the number fiction. If render counts matter, the React DevTools Profiler measures them without lying; if the *value* is needed at runtime, the honest homes are an effect (commits, not attempts) — and usually the need evaporates under questioning.

The discharged patient:

```tsx
// ✅ InvoicePanel — fully compilable, StrictMode-invisible
export function InvoicePanel({ invoice, items }: InvoicePanelProps) {
  const [expanded, setExpanded] = useState(false);
  const [openedAt] = useState(() => new Date());        // "when opened" — sampled once, on purpose

  if (!invoice) return null;

  const sorted = items.toSorted((a, b) => b.amount - a.amount);

  return (
    <section>
      <h3>Invoice {invoice.number} — opened {openedAt.toLocaleTimeString()}</h3>
      <StatusBadge status={invoice.status} />
      <button onClick={() => setExpanded((e) => !e)}>Details</button>
      {expanded && <ul>{sorted.map((i) => <li key={i.id}>{i.label}: {i.amount}</li>)}</ul>}
    </section>
  );
}
```

Same rendered pixels, and now: lint-clean, identical under StrictMode's double render, safe to discard and replay — and eligible for the Compiler's memoization, which the original forfeited on three separate counts.

## Real-world patterns

### Containing rule-breaking dependencies

Real apps embed libraries that predate the rules — imperative map SDKs, charting engines, editors whose objects mutate constantly. The pattern is **containment**: the mutable instance lives in a ref, all interaction with it happens in effects and handlers, and what crosses into render is an immutable projection (state you set from the library's events). The rules apply to *your render*, not to the world; the ref/effect boundary is the embassy where the two jurisdictions meet ([`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md) builds one).

### `"use no memo"` — the labeled exception

When a component genuinely can't be made compliant *yet* (mid-migration, a library interop you haven't contained), the Compiler accepts an explicit opt-out directive at the top of the function: `"use no memo"`. Treat it exactly like a lint-suppress with a ticket number: a flag planted on debt, visible in review and greppable, never a lifestyle. A codebase's `"use no memo"` count is a health metric ([`../rendering/react-compiler-deep-dive.md`](../rendering/react-compiler-deep-dive.md)).

### Idempotence as the test-writing cheat code

A side benefit worth naming: rule-following components are trivially testable — render with props, assert on output, no mocking of module state, no "run it twice and the second differs." When a component is hard to test, the friction is usually a rules violation wearing a different hat: hidden inputs (module state, `Date.now()`) or hidden outputs (mutations). The rules and testability are the same property viewed from two sides — a thread [`../ecosystem/testing.md`](../ecosystem/testing.md) picks up.

### Reading the Compiler's verdicts

Practical compliance auditing, in order: run the lint with the recommended (Compiler-powered) config and treat its purity diagnostics as build failures; keep StrictMode on and investigate *any* dev/prod behavioral difference as a purity suspect; check DevTools for the compiler-optimization badge on hot components — a hot component *without* it is either opted out or bailing, and the lint diagnostics will say which rule it tripped.

## The rules, mapped

The capstone table — each rule, its canonical violation, its enforcer, and where this resource already made you feel it:

| Rule | Canonical violation | First enforcer | Felt in |
| --- | --- | --- | --- |
| Render is idempotent | `Date.now()` / `Math.random()` in body | StrictMode double render | Clinic (C) |
| No side effects in render | fetch/log/track in body | StrictMode doubling; Compiler bailout | [`thinking-in-react.md`](thinking-in-react.md) |
| Props/state immutable | `items.sort()`, `item.qty++` | Dev freeze; lint; swallowed updates | [`../state/state-and-usestate.md`](../state/state-and-usestate.md) |
| Frozen after handoff | mutating an object already `setState`-ed or rendered | Stale memoized children | [`components-and-props.md`](components-and-props.md) |
| Local mutation OK | — (the legal exception) | — | This article |
| No ref access in render | `ref.current++` render tally | Lint (compiler rules); replay fiction | Clinic (E) |
| React calls components | `{Card(props)}` | Lint; hook-order crash | [`jsx-and-rendering.md`](jsx-and-rendering.md) |
| Hooks: top level, stable order | hook after early return; in `try`/`catch` | Lint; "fewer hooks than expected" | Clinic (A) |
| Hooks: components/custom hooks only | hook in a handler or plain util | Lint | [`../effects/custom-hooks.md`](../effects/custom-hooks.md) |
| Initializers/updaters pure | analytics inside `setX(prev => …)` | StrictMode double-invokes them | [`../state/state-and-usestate.md`](../state/state-and-usestate.md) |

## Common mistakes

**The render-count ref.** Clinic (E), singled out because it's *taught* in tutorials. It's a purity violation whose output is fiction under StrictMode and concurrency; the Profiler exists.

**Diagnosing StrictMode as the bug.** "It renders twice / fetches twice / logs twice — React is broken" and the fix applied is disabling StrictMode. The doubling is the auditor's probe; every "fix" that removes the auditor ships the leak it found. The correct response is always symmetry ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)) or purity, never silence.

**Suppressing the hooks lint.** `// eslint-disable-next-line react-hooks/exhaustive-deps` is the single most-shipped rules violation in the ecosystem — a frozen closure with a paper trail. The deps are derived from the code; if they're wrong, the *code* is wrong ([`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md)'s stale-closure section is the bill).

**Memoizing over impurity.** "It changed every render, so I wrapped it in `useMemo`" — caching a non-idempotent computation doesn't make it pure; it makes it *arbitrarily stale*, and hands the Compiler a component it must refuse. Fix the input (move the randomness/time to its right room), then memoization becomes both unnecessary and legal.

**Shallow-spread laundering.** `draft.items[0].qty = 2; setDraft({ ...draft })` — the top-level spread buys a new reference (so the render happens), while the nested mutation corrupts shared history (so memoized children holding `items[0]` see an unchanged reference and skip). Passes casual testing, fails precisely where memoization exists — the nastiest mutation variant because it *looks* immutable.

**Hooks in `try`/`catch`.** Wrapping `useContext` or a custom hook in `try` "in case it throws" — a throw between hooks desynchronizes the list exactly like a conditional. Error strategy for render lives in error boundaries ([`../rendering/error-boundaries.md`](../rendering/error-boundaries.md)); hooks stay bare at the top.

**Hook factories and dynamic hooks.** `const useThing = flags.x ? useA : useB` or storing hooks in objects and calling `hooks[name]()` — call-order stability can't survive indirection React can't see. Branch *inside* one hook, or render different components.

**"Pure enough" module caches.** A module-level `let cache` written during render "because it's idempotent-ish." Sometimes genuinely fine (write-once, same-value), always Compiler-hostile and concurrency-suspect; the sanctioned versions are `useState` lazy init, a ref initialized once, or real memoization keyed outside render. When in doubt, it goes in the containment embassy.

## How this evolved

| Era | Change | What it means now |
| --- | --- | --- |
| 2013–2018 | Purity as *convention* — docs said "props are read-only," nothing checked | The folklore era; violations shipped and mostly worked, synchronously |
| React 16.8 (2019) | Hooks make call order load-bearing; Rules of Hooks + `eslint-plugin-react-hooks` ship | The first machine-checked rule; "hooks are magic" was always "hooks are a positional list" |
| React 18 (2022) | Concurrent rendering; StrictMode gains double-invoke + effect audit | Renders become discardable — purity graduates from courtesy to soundness requirement, with a runtime auditor |
| React 19 era (2024) | The Rules of React formalized as a docs section — pure/calls/hooks, precisely worded | The contract gets a canonical text; "is this allowed?" gains a citation |
| Compiler 1.0 (2025) | Rules become machine-*consumed*: lint gains compiler-powered purity analysis; compliant code is auto-memoized, violations bail out | Convention → documentation → verification → **compilation**. The rules are now where performance comes from |

```tsx
// legacy: the suppression culture — a frozen closure with a paper trail,
// modernized in the upgrade pass
// eslint-disable-next-line react-hooks/exhaustive-deps
}, []);
```

## Exercises

### 1. Violation hunt

Four violations, four different rules, four different enforcers — name each:

```tsx
export function useLastSeen(user: User) {
  if (!user.trackable) return null;
  const [seen, setSeen] = useState(user.lastSeen);
  try {
    const locale = useContext(LocaleContext);
    user.checked = true;
    return formatSeen(seen, locale, Date.now());
  } catch {
    return String(seen);
  }
}
```

*Hint: (1) hook after conditional return — lint; (2) hooks inside `try`/`catch` — lint; (3) mutating the `user` argument — dev freeze / compiler-lint, and it's a hook argument, same immutability class as props; (4) `Date.now()` in a pure computation path — StrictMode shows the drift. Bonus judgment call: as designed, can this hook be rules-compliant at all while returning early? Restructure it (compute `trackable` handling *after* the hooks) and say what changed.*

### 2. Win the Compiler back

This component bails out of compilation. Identify why, then refactor two ways — once keeping the cache (make it idempotent and render-safe) and once deleting it (derive + let the Compiler memoize) — and argue which ships:

```tsx
let lastResult: Report | null = null;

export function ReportView({ rows }: { rows: Row[] }) {
  if (!lastResult || lastResult.rowCount !== rows.length) {
    lastResult = buildReport(rows);          // expensive
  }
  return <ReportTable report={lastResult} />;
}
```

*Hint: module-level write during render (shared across instances, wrong under replay) and a cache key (`rowCount`) that's wrong anyway — same length, different rows, stale report. The delete-it version is three lines: `const report = buildReport(rows)` and let the Compiler cache it against `rows`' identity. That's the ship: the cache was hand-rolled memoization with a corrupted key, which is what the Compiler exists to replace.*

### 3. Defend the legal three

Each of these gets flagged in review as "impure." Write the one-paragraph defense for each, citing the precise rule boundary:

```tsx
// (a)
const groups = new Map<string, Item[]>();
for (const item of items) {
  const list = groups.get(item.kind) ?? [];
  list.push(item);
  groups.set(item.kind, list);
}

// (b)
const [connectionId] = useState(() => crypto.randomUUID());

// (c)
if (playerRef.current === null) {
  playerRef.current = new AudioPlayer();
}
```

*Hint: (a) local mutation — every mutated object was born this render and handed off after; externally unobservable, therefore pure. (b) lazy initializers may be non-deterministic **across mounts** — the contract is purity per call and once-per-mount semantics, which is exactly what "an id for this instance" means; the smell would be `useState(crypto.randomUUID())` without the thunk (new value computed and discarded every render). (c) the documented ref-init exception: a guarded, idempotent, first-render-only write — it converges to the same state under double-invoke and replay, which is the property all three share and the property the rules actually protect.*

## Summary

You learned the contract that everything else was secretly teaching: render (and everything React calls during it) is pure — idempotent, externally unobservable, treating props, state, and hook values as frozen shared history, with local pre-handoff mutation explicitly legal and refs explicitly off-limits; React calls components and hooks, never you; and hooks keep a stable, top-level, un-`try`-wrapped call order because they are a positional list on the fiber. Each rule is purchased by a machine — the hook list, StrictMode's audit, concurrent discard-and-replay, server prerendering, and the Compiler, which converted compliance from correctness into performance: compliant components get memoized free, violations bail out silently. The enforcement stack (compiler-powered lint → dev freezing → StrictMode → Compiler verdicts) means you check compliance, not memorize it; the placement table means most violations are just code in the wrong room; and the clinic's five patients — the conditional hook, the prop sort, the render timestamp, the called component, the render tally — are the ones you'll actually meet. Wave 1 closes here: the mental model, the element, the contract. Everything after this is machinery and application.

## See also

- [`thinking-in-react.md`](thinking-in-react.md) — why purity is the design, not a constraint on it
- [`../state/state-and-usestate.md`](../state/state-and-usestate.md) — immutability's mechanics: `Object.is`, swallowed updates, pure updaters
- [`../effects/effects-and-synchronization.md`](../effects/effects-and-synchronization.md) — where side effects legally live; StrictMode's other audit
- [`../rendering/memoization-and-the-compiler.md`](../rendering/memoization-and-the-compiler.md) — what compliance buys, mechanically
- [`../rendering/react-compiler-deep-dive.md`](../rendering/react-compiler-deep-dive.md) — bailout diagnostics, `"use no memo"`, the compiled output
- [`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md) — the sanctioned mutable cell and the containment embassy

## References

- [Rules of React (react.dev)](https://react.dev/reference/rules)
- [Components and Hooks must be pure (react.dev)](https://react.dev/reference/rules/components-and-hooks-must-be-pure)
- [React calls Components and Hooks (react.dev)](https://react.dev/reference/rules/react-calls-components-and-hooks)
- [Rules of Hooks (react.dev)](https://react.dev/reference/rules/rules-of-hooks)
- [Keeping Components Pure (react.dev)](https://react.dev/learn/keeping-components-pure)
- [`StrictMode` API (react.dev)](https://react.dev/reference/react/StrictMode)
- [React Compiler (react.dev)](https://react.dev/learn/react-compiler)
- [`eslint-plugin-react-hooks` (npm)](https://www.npmjs.com/package/eslint-plugin-react-hooks)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.