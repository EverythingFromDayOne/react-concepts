---
article_id: context
concept_folder: state
wave: 2
related:
  - foundations/component-composition
  - rendering/how-react-renders
  - state/state-and-usestate
  - state/usereducer-and-state-structure
  - ecosystem/state-management-landscape
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Context

> **Lead with this:** context is a transport, not a store. It carries one value from a provider to every component that reads it, however deep — no drilling, no middlemen props. That's the entire feature. It has no selectors, no partial subscriptions, no update scheduling of its own: when the value changes, **every reader re-renders**, and nothing you wrap around a reader can prevent it. Used for what it's built for — ambient, slow-moving values like theme, locale, the session, a compound component's coordination channel — it's clean and nearly free. Used as an app-state manager, it becomes a broadcast tower: one frequency, every tuned receiver re-rendering on every transmission. This article covers the mechanics that make both halves of that sentence true, and the patterns that keep you on the right side of it.

## What it is

Three pieces:

```tsx
// theme-context.ts
import { createContext, use } from "react";

export type Theme = "light" | "dark";

const ThemeContext = createContext<Theme | null>(null);

export function ThemeProvider({
  theme,
  children,
}: {
  theme: Theme;
  children: React.ReactNode;
}) {
  return <ThemeContext value={theme}>{children}</ThemeContext>;
}

export function useTheme(): Theme {
  const theme = use(ThemeContext);
  if (theme === null) {
    throw new Error("useTheme must be used inside <ThemeProvider>");
  }
  return theme;
}
```

- **`createContext(defaultValue)`** creates the channel. The default is used *only* when a reader has no provider above it — it is not a fallback for a provider whose value happens to be `null`.
- **The provider** — since 19, the context object itself renders as one: `<ThemeContext value={...}>`. It announces "everything below me reads this value for this channel."

  ```tsx
  // legacy: written for React <19 — modernized in the upgrade pass
  <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>
  ```

- **Reading** — `use(ThemeContext)` (19) or the equivalent `useContext(ThemeContext)`. They resolve identically; `use` additionally works inside conditions and loops, which `useContext` — bound by the hook cursor ([how-react-renders](../rendering/how-react-renders.md)) — cannot. The guard-hook wrapper above is the standard packaging: a domain-named hook that fails loudly at the read site instead of `null`-crashing three components later.

What context is *for* maps to one word: **ambient**. A value is ambient when every component in a region of the tree would give the same answer to "which one do you want?" — the theme, the locale, the authenticated user, the router's location, the Tab group you're inside. Values that are *assembled* — this list, that entity, the data for this specific card — are not ambient; they belong to props, composition, or the server cache, and dragging them into context is how broadcast towers get built.

## How it works under the hood

### Reading resolves by position, not by owner

Reading a context walks *up the fiber tree* — `return` pointer by `return` pointer — to the nearest matching provider. Nearest wins; nesting overrides. The consequence [component-composition](../foundations/component-composition.md) set up as scope-vs-position: the value a component reads is determined by **where it renders**, not by who created its element. A card passed as `children` into a `<DarkPanel>` was *created* by an owner standing outside the panel, but it *renders* under the panel's provider — so it reads dark. Slots, compound children, and portal content all inherit the providers at their render position — a portal's fiber keeps its React-tree parent even though its DOM output lands elsewhere ([portals-and-the-event-system](../rendering/portals-and-the-event-system.md) *(Wave 2, planned)* owns that split), so a modal portaled to `document.body` still reads the theme of the component that rendered it. This is what makes theme islands and scoped overrides work: wrap any subtree in a nearer provider and every read inside it re-resolves.

### How a value change finds its readers

Each fiber that reads a context records that fact on its `dependencies` list. When a provider re-renders, React compares the new `value` prop to the old with `Object.is`. Same → nothing special happens. Different → React walks the provider's subtree, and every fiber whose `dependencies` include this context gets its lanes marked — the same breadcrumb mechanism any `setState` uses ([how-react-renders](../rendering/how-react-renders.md)).

Two precise consequences:

1. **Context pierces bailouts.** The bailout check is *same props AND no pending lanes AND no context change* — a changed context fails check 3 by construction. `memo`, stable elements, Compiler-stabilized props: none of it can save a reader from a context it reads. The only ways to render less are to read less (split contexts) or change the value less often (stabilize identity).
2. **Only readers re-render.** The propagation walk marks *dependents*, not descendants. A 500-component subtree under a provider where 6 components read it: a value change renders those 6 (plus whatever cascades below them by normal rules — which composition's stable-`children` moves can still contain). The folk belief that "everything under the provider re-renders" gets the mechanism backwards — when that appears to happen, the real cause is almost always the next section.

### Value identity is the whole ballgame

The provider's change detection is `Object.is` on the `value` prop. So this classic:

```tsx
function SessionProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  return (
    <SessionContext value={{ user, setUser }}> {/* 🔴 fresh object every render */}
      {children}
    </SessionContext>
  );
}
```

…re-broadcasts on **every render of the provider component**, whether `user` changed or not. Any state change in an ancestor that re-renders `SessionProvider` mints a new `{ user, setUser }`, fails `Object.is`, and marks every reader in the app. This is the mechanism behind "my whole app re-renders when anything happens" — not context being slow, but a value whose identity churns.

In Compiler-compiled app code, that object literal gets stabilized automatically — same inputs, same reference — and the problem largely evaporates. But **provider value memoization remains one of the legitimate manual-memoization sites at library boundaries**: if you ship a provider in a package, or the provider lives outside the compiled graph, stabilize by hand and say why:

```tsx
// Manual memo: library boundary — consumers may not run the Compiler,
// and this value fans out to every reader in their tree.
const value = useMemo(() => ({ user, setUser }), [user]);
return <SessionContext value={value}>{children}</SessionContext>;
```

[memoization-and-the-compiler](../rendering/memoization-and-the-compiler.md) *(Wave 2, planned)* owns the verification workflow that proves which case you're in.

## Basic usage

The full minimal loop — typed channel, guard hook, 19 provider syntax, a reader:

```tsx
// session-context.tsx
import { createContext, use, useMemo, useState } from "react";
import type { User } from "./types";

interface Session {
  user: User | null;
  login: (user: User) => void;
  logout: () => void;
}

const SessionContext = createContext<Session | null>(null);

export function SessionProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const value = useMemo<Session>(
    () => ({
      user,
      login: (next) => setUser(next),
      logout: () => setUser(null),
    }),
    [user], // manual memo: library boundary — see memoization article
  );

  return <SessionContext value={value}>{children}</SessionContext>;
}

export function useSession(): Session {
  const session = use(SessionContext);
  if (session === null) {
    throw new Error("useSession must be used inside <SessionProvider>");
  }
  return session;
}
```

```tsx
// Anywhere below the provider — no drilling, no prop plumbing
import { useSession } from "./session-context";

export function AccountMenu() {
  const { user, logout } = useSession();
  if (!user) return <a href="/login">Sign in</a>;
  return (
    <nav aria-label="Account">
      <span>{user.name}</span>
      <button onClick={logout}>Log out</button>
    </nav>
  );
}
```

And the one read shape only `use` can do — conditional:

```tsx
function Price({ amount, localized }: { amount: number; localized: boolean }) {
  if (localized) {
    const locale = use(LocaleContext); // ✅ legal with use(); illegal with useContext
    return <span>{formatPrice(amount, locale)}</span>;
  }
  return <span>{amount.toFixed(2)}</span>;
}
```

The mechanics of why `use` escapes the hook-order rule — and its bigger job of reading promises — belong to [use-and-promises](../concurrent/use-and-promises.md) *(Wave 3, planned)*.

## Walkthrough: from broadcast tower to tuned channels

The scenario: a dashboard where ~40 components read the session — most only to *display* the user, a handful only to *call* `logout` (header button, idle-timeout modal, error screens). One context carries `{ user, login, logout }`. The symptom: renaming the user in account settings re-renders all 40; worse, the Profiler shows all 40 re-rendering when a *sidebar toggle* — unrelated state in a shared ancestor — flips.

**Stage 1 — read the evidence.** Profiler, "Record why each component rendered," toggle the sidebar: all 40 session readers say *context changed*. But the user didn't change. The provider component re-rendered (its parent did), the inline `value={{ user, login, logout }}` minted a new object, `Object.is` failed, broadcast. The tower is transmitting static.

**Stage 2 — stabilize identity.** The `useMemo` from Basic usage (or the Compiler, in compiled app code — verify, don't assume). Re-profile: the sidebar toggle now renders zero session readers. The genuine user change still renders all 40 — correct so far as it goes: 40 components *do* read `user`… except the logout-only ones don't. They read the object that *contains* it.

**Stage 3 — split the contexts.** Data on one channel, actions on another. Actions are born stable — `setUser` from `useState` never changes identity — so the actions channel *never re-broadcasts*:

```tsx
// session-context.tsx — split form
import { createContext, use, useMemo, useState } from "react";
import type { User } from "./types";

interface SessionActions {
  login: (user: User) => void;
  logout: () => void;
}

const UserContext = createContext<User | null | undefined>(undefined);
const SessionActionsContext = createContext<SessionActions | null>(null);

export function SessionProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const actions = useMemo<SessionActions>(
    () => ({
      login: (next) => setUser(next),
      logout: () => setUser(null),
    }),
    [], // closes only over the stable setter — created once
  );

  return (
    <SessionActionsContext value={actions}>
      <UserContext value={user}>{children}</UserContext>
    </SessionActionsContext>
  );
}

export function useUser(): User | null {
  const user = use(UserContext);
  if (user === undefined) {
    throw new Error("useUser must be used inside <SessionProvider>");
  }
  return user;
}

export function useSessionActions(): SessionActions {
  const actions = use(SessionActionsContext);
  if (actions === null) {
    throw new Error("useSessionActions must be used inside <SessionProvider>");
  }
  return actions;
}
```

Note the `undefined`-vs-`null` split on `UserContext`: `null` is a legitimate value (logged out), so the guard sentinel has to be something a provider can never legally supply — `undefined` as "no provider," `null` as "no user." Sloppy sentinels here turn "forgot the provider" into "silently renders logged-out UI."

**Stage 4 — re-profile.** Rename the user: the ~35 display readers render (they must — they show the name); the logout-only components render **zero times**, because their channel never transmitted. Sidebar toggle: zero and zero. The audit trail: 40 renders per unrelated ancestor render → 0; 40 per user change → 35, all of them genuine readers of the changed data. Nothing was memoized at a consumer; the *channels* were tuned. Blast radius fixed structurally — the same doctrine as [component-composition](../foundations/component-composition.md)'s two performance moves, applied to broadcast instead of props.

When the state graduates from one field to a real object with rules — think auth status unions, multi-step flows — the state half of this pattern typically becomes a reducer with `dispatch` on the actions channel (`dispatch` is stable for free, no `useMemo` at all); that pairing is [usereducer-and-state-structure](./usereducer-and-state-structure.md)'s *(next article)* to own.

## Real-world patterns

**Ambient beats assembled — the placement test.** Before creating a context, ask the region-of-tree question: *would every component under this provider give the same answer to "which one?"* Theme, locale, density, session, feature flags, the current route — yes; ambient; context is right. "The products for this table," "the form's values," "the selected row" — no; each consumer wants *its* data; that's assembled, and it travels by props and composition, or lives in the right store. The broadcast tower is the failure mode of ignoring this test: assembled data on an ambient channel means every reader hears every other reader's updates. If different listeners genuinely need different slices of one changing object, context is the wrong radio — that's a selector-based store (Zustand — [state-management-landscape](../ecosystem/state-management-landscape.md) *(Wave 4, planned)* owns the decision table) or, for anything fetched, the server cache. **Server state never rides in context**: a provider that fetches and broadcasts is a hand-rolled cache with no invalidation, no dedup, no staleness policy — [data-fetching-tanstack-query](../ecosystem/data-fetching-tanstack-query.md) *(Wave 4, planned)* exists so you don't build that.

**Compound-component channels.** The other first-class use: a *private* context coordinating a compound family — [component-composition](../foundations/component-composition.md)'s Tabs runs its selection state through exactly this channel, 19 provider syntax, guard hook and all. The channel isn't ambient app data; it's the family's internal wiring, invisible in the public API. Scoping rule of thumb: module-private context object, exported compound parts, guard hook that names the family ("`<Tabs.Panel>` must be used inside `<Tabs>`").

**DI-ish config for swappability.** Context earns its "dependency-injection-ish" reputation with service objects: an `apiClient`, an analytics sink, a clock. Provide the real one in `main.tsx`, provide a fake in tests — consumers can't tell. This is the *static config* row of the state-placement rule: the value changes never or once (environment boot), so the broadcast cost is zero and the seam is pure profit. Keep the injected surface an interface, not a concrete class, and the test story writes itself — in RTL, the provider *is* the harness:

```tsx
function renderWithSession(ui: React.ReactElement, user: User | null = null) {
  return render(ui, {
    wrapper: ({ children }) => <FakeSessionProvider user={user}>{children}</FakeSessionProvider>,
  });
}
```

One `wrapper` per test file replaces module mocking for everything the channel carries.

**The provider pyramid, tamed.** Real apps accumulate providers — session, theme, query client, router, i18n — and `main.tsx` grows a six-deep indentation staircase. Collapse it into one composition component so the app has a single place where ambient scope is declared:

```tsx
export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      <SessionProvider>
        <ThemeProvider theme="light">{children}</ThemeProvider>
      </SessionProvider>
    </QueryClientProvider>
  );
}
```

Order matters exactly when one provider *reads* another (a theme that depends on user preferences must sit inside the session); otherwise it's alphabetical-by-taste. The same component, parameterized with fakes, becomes the integration-test harness — the pyramid pays for itself twice.

**Scoped overrides.** Nearest-provider resolution makes providers composable as *scopes*: a `<ThemeProvider theme="dark">` around one marketing panel creates a dark island in a light app; a compact-density provider around a data grid tightens spacing without a prop touching a cell. Overrides nest arbitrarily; readers always get the nearest. This is also the escape valve when a library's provider sits too high — wrap the subtree that needs different values rather than lifting a prop through it.

**Default values as test fixtures — sparingly.** `createContext(realisticDefault)` lets leaf components render in isolation (unit tests, Storybook) without provider scaffolding. The cost: a forgotten provider *silently works* with the default instead of failing loudly. House stance: guard hooks with `null`/`undefined` sentinels for anything stateful or app-critical; meaningful defaults only for cosmetic ambience (theme, density) where a silent fallback is genuinely harmless.

## Reference

| API | What it does | Status in 19.2 |
| --- | --- | --- |
| `createContext(default)` | Creates the channel; default applies only with no provider above | Baseline |
| `<MyContext value={v}>` | Provider — the context object rendered directly | Baseline (19+) |
| `use(MyContext)` | Read nearest value; legal in conditions/loops | Baseline (19+) |
| `useContext(MyContext)` | Read nearest value; hook-order rules apply | Baseline |
| `<MyContext.Provider value>` | Legacy provider spelling | Works; write the 19 form |
| `<MyContext.Consumer>` | Render-prop reader (two generations old) | Avoid |

**Placement rule (the standing table, context column highlighted):**

| The value is… | It belongs in… |
| --- | --- |
| Local UI state | `useState` / `useReducer` in the component |
| Assembled data a parent has | Props / composition |
| **Ambient, slow-moving, one-answer-per-region** | **Context** |
| Fast-changing client state with per-consumer slices | Store with selectors (Zustand) |
| Anything fetched from a server | TanStack Query — never a client store, never context |

## Common mistakes

**1. Inline provider values at boundaries the Compiler doesn't cover.**

```tsx
<AppConfigContext value={{ flags, api }}> {/* 🔴 in a published package: broadcasts on every provider render */}
```

Stabilize with `useMemo` and a boundary comment, per the convention. Inside compiled app code, verify the Compiler handled it before adding hand-written memo — don't do both.

**2. Fast state on the channel.** Keystrokes, cursor positions, scroll offsets in context means every reader renders per event — 60Hz broadcast to the whole subscriber list. That data is local state, or a selector store if it's genuinely shared. (The dashboard-scale version of this is a queued recipe in the `state-management/` track.)

**3. Wrapping readers in `memo` to dodge a context change.** Can't work — context is bailout check 3; the reader's own subscription forces it. Render less by *reading less*: split channels, or move the read lower so the re-rendering component is smaller. Memoizing around context is treating the receiver for a transmitter problem.

**4. Believing the provider re-renders "everything under it."** Only readers get marked; the rest bail out normally. Chasing this myth leads to both over-memoization and fear of legitimately broad providers. When "everything" *does* render, look for value-identity churn (mistake 1) or readers that pass unstable props downward — the Profiler's "why" column distinguishes them in seconds.

**5. Silent default-value fallbacks.**

```tsx
const CartContext = createContext<Cart>({ items: [], total: 0 }); // 🔴 forgot-the-provider renders an empty cart
```

A plausible default turns a wiring bug into a data bug. Stateful contexts get an impossible sentinel plus a guard hook that throws with the provider's name in the message.

**6. One mega-context object.** `{ user, theme, cart, notifications, flags }` on a single channel couples every reader to every field's change cadence. Channels are cheap — one per concern, one per change *frequency*. The split in the walkthrough is the two-channel case of a general rule.

**7. Server data distributed via context.**

```tsx
// 🔴 a cache with no invalidation, no dedup, no staleness, no retries
function ProductsProvider({ children }) {
  const [products, setProducts] = useState<Product[]>([]);
  useEffect(() => { fetchProducts().then(setProducts); }, []);
  return <ProductsContext value={products}>{children}</ProductsContext>;
}
```

Placement rule, hardest form: server state never in client stores — context included. The fetch belongs to the server-cache layer; components read the cache.

**8. Context for two levels of drilling.** Passing a prop through one honest intermediary is not a problem to solve — it's legible data flow. The escalation runs props → composition moves (lift, `children`, slots) → context, and context wins only when the value is genuinely ambient. Reaching for it to avoid typing a prop twice trades visible wiring for invisible coupling. ([component-composition](../foundations/component-composition.md) owns the ladder.)

**9. Mutating the value in place.**

```tsx
settings.density = "compact"; // 🔴 same reference — Object.is passes, no reader ever hears it
setSettings(settings);
```

The provider's change detection is identity, like every other comparison in the pipeline. New value, new object — the immutable-recipes table in [state-and-usestate](./state-and-usestate.md) applies verbatim.

**10. `useContext` inside a condition.** Hook-order violation, and the linter says so. In 19 the fix is often just `use(MyContext)` — but reach for a conditional read because the *logic* is conditional (a variant that opts into ambience), not to paper over a component doing two unrelated jobs.

## How this evolved

- **Legacy context (pre-16.3):** `contextTypes` and `getChildContext` — undocumented-then-discouraged, and broken in a deep way: `shouldComponentUpdate` returning false *blocked propagation* to everything below. Context that couldn't reliably arrive.
- **`createContext` (16.3):** the modern channel, with propagation that pierces bailouts by design — fixing exactly the legacy flaw. Reading meant `<Ctx.Consumer>{value => …}</Ctx.Consumer>` render-prop pyramids.
- **`useContext` (16.8):** reading became one line; Consumer pyramids collapsed; context usage exploded — including into the state-management role this article spends half its length talking you back out of.
- **React 19:** `<Ctx>` as its own provider; `use(Ctx)` with conditional reads; `Consumer` fully vestigial.
- **Compiler era:** automatic stabilization of provider values in compiled code removes the most common footgun in app code — leaving identity discipline as a *boundary* concern, which is where the manual-memo convention now points.

## Exercises

**1. Static on the tower.** Build a provider with an inline object value, 20 reader components, and an unrelated `useState` in the provider's parent. Profile a parent re-render; count reader renders. Stabilize the value; re-count. Then split data/actions channels and count again for a data change. You should record 20 → 0 → (readers-of-data only).
*Hint: "Record why each component rendered" distinguishes *context changed* from *parent rendered* — watch the reason flip between stages.*

**2. Nearest wins.** Compose `<ThemeProvider theme="light">` wrapping a page, `<ThemeProvider theme="dark">` wrapping one card, and a `ThemeBadge` component used in four spots: page level, inside the card, passed as `children` *into* the card from page level, and inside a portal rendered from the card. Predict all four readings before running.
*Hint: position, not owner — where does each badge's fiber sit when the read walks up `return` pointers? The portal's fiber parent is where it was *rendered*, not where it lands in the DOM.*

**3. The seam.** Define an `ApiClient` interface, provide a real client at the root, and write a Vitest test that renders a consumer inside a provider carrying a fake. No MSW, no module mocking — the context *is* the seam. Then add a guard hook and assert (with an error boundary or `expect(…).toThrow`) that rendering without a provider fails with a useful message.
*Hint: the fake satisfies the interface with canned promises; the test wraps `render(<Provider value={fake}>…)`. Which of the two tests just documented your DI story?*

## Summary

- Context is transport for **ambient** values — one answer per region of the tree. Assembled data travels by props and composition; sliced shared state wants a selector store; server data wants the server cache. The placement table settles it before any code exists.
- Reads resolve to the **nearest provider by render position** — the mechanism behind theme islands, scoped overrides, and slots reading the environment they're placed into.
- A value change marks **readers only**, pierces every bailout, and is detected by `Object.is` on the provider's `value` — so identity discipline is everything: Compiler-stabilized in app code, `useMemo` with a stated reason at library boundaries.
- The **split-context pattern** tunes channels by change frequency — data on one, born-stable actions on another — and fixes blast radius structurally, with zero consumer-side memoization.
- 19 spelling throughout: `<MyContext value>`, `use(MyContext)` (conditionally legal), guard hooks with impossible sentinels; `Consumer` and `.Provider` are reading knowledge only.

## See also

- [component-composition](../foundations/component-composition.md) — scope vs position, the escalation ladder context sits atop, and the Tabs channel this article generalizes
- [how-react-renders](../rendering/how-react-renders.md) — the `dependencies`/lanes machinery and bailout check 3
- [state-and-usestate](./state-and-usestate.md) — identity, immutability, and why `Object.is` rules every comparison
- [usereducer-and-state-structure](./usereducer-and-state-structure.md) *(next)* — the reducer + dispatch-through-context pairing for state with rules
- [memoization-and-the-compiler](../rendering/memoization-and-the-compiler.md) *(Wave 2, planned)* — verifying what the Compiler stabilized, including provider values
- [state-management-landscape](../ecosystem/state-management-landscape.md) *(Wave 4, planned)* — the decision table for when a selector store replaces the channel

## References

- [createContext — react.dev](https://react.dev/reference/react/createContext)
- [use — react.dev](https://react.dev/reference/react/use)
- [Passing Data Deeply with Context — react.dev](https://react.dev/learn/passing-data-deeply-with-context)
- [Scaling Up with Reducer and Context — react.dev](https://react.dev/learn/scaling-up-with-reducer-and-context)
- [useContext — react.dev](https://react.dev/reference/react/useContext)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.