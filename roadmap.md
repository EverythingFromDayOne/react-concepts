# reactjs-concepts — Roadmap

> Status: **draft for approval** · Planning date: July 2026
> Companion to angular-concepts. Phase 1 is pure React — no Angular references anywhere in content.

---

## 1. Target versions

| What | Version | Notes |
|---|---|---|
| **React / React DOM** | **19.2** (latest stable line, currently 19.2.x) | Baseline for all content. Actions, `use()`, ref-as-prop, `useOptimistic`, `useActionState` are all baseline-stable — teach them as the default, not as "new features." |
| React Compiler | 1.0 (stable since Oct 2025) | Editorial stance in §5. Assumed ON for app code. |
| TypeScript | 5.9+ strict | TS 7 (`tsgo` native port) is in beta — mention in tooling article, don't baseline it. |
| Vite | 8.x + `@vitejs/plugin-react` v6 | v6 dropped Babel for Oxc. React Compiler now wires in via `@rolldown/plugin-babel` + `reactCompilerPreset` — this exact gotcha becomes a recipe (most online tutorials show the dead `react({ babel: {...} })` form). |
| Node | 22 LTS | Vite 8 floor is 20.19+/22.12+. |
| Vitest + Testing Library | latest | Vitest, not Jest — no translation layer against Vite. |

**Version-evolution narrative** (the React equivalent of Zone.js → signals → zoneless):

> class components → hooks (16.8) → concurrent rendering + automatic batching (18) → Actions, `use()`, ref-as-prop (19) → React Compiler (auto-memoization, 19.x era)

Every article whose API changed across this arc gets a "How this evolved" section anchored to it.

---

## 2. Meta-framework stance (decision)

**Phase 1 baseline = React + Vite SPA. RSC/Next.js is a fenced-off wave at the end of Phase 1, not interleaved.**

Rationale:

1. **Fundamentals are framework-agnostic; frameworks blur them.** Learning `useEffect`, reconciliation, or Suspense *inside* Next.js means constantly disentangling React semantics from framework conventions (server/client boundary, router cache, `"use client"` directives). A learner can't tell which behavior belongs to whom. Client-side React first gives a clean mental model.
2. **The client paradigm still dominates** real codebases and interviews. RSC is important but it's a genuinely different mental model — it deserves its own coherent arc (articles 27–28 + the `ssr-and-rsc` recipe track) once client React is solid, rather than leaking into every foundational article as caveats.
3. **Vite 8 is the de facto non-framework baseline** — even the anti-SPA crowd concedes this is what you reach for when you want React-the-library.

Adjacent stances locked with it:

- **Routing baseline: React Router v7 in library/data mode.** Most common in the wild; framework mode (the Remix lineage) is out of scope for Phase 1. TanStack Router gets its own ecosystem article as the type-safe alternative — it's winning greenfield mindshare and deserves honest coverage, but two routers in walkthroughs doubles every example.
- **Data fetching baseline: TanStack Query.** Raw `fetch`-in-`useEffect` is taught *once*, as the failure mode (race conditions, no dedup, no cache) that motivates the library — mirroring the symptom-first philosophy.
- **RSC coverage happens via Next.js App Router** when we get there, since that's where RSC is actually usable in production.

---

## 3. Concept articles — priority order

Numbering = reading order. Waves gate delivery (finish a wave before starting the next).

### Wave 1 — Foundations (the mental model)

1. `thinking-in-react` — UI as a function of state; render vs. commit; why "re-render" doesn't mean "re-DOM"
2. `jsx-and-rendering` — what JSX compiles to; elements vs. components vs. instances
3. `components-and-props` — typed props, children, composition as the primitive
4. `state-and-usestate` — state as a snapshot; batching; updater functions; why mutation breaks everything
5. `rendering-lists-and-keys` — reconciliation from the user's seat; why index keys corrupt state
6. `conditional-rendering-and-events` — event handlers, synthetic events, common `&&` footguns
7. `forms-controlled-and-uncontrolled` — the controlled-input model; when uncontrolled wins
8. `component-composition` — lifting state, colocation, children-as-slots, compound components
9. `effects-and-synchronization` — effects synchronize with external systems; cleanup; StrictMode double-invoke; "you might not need an effect"
10. `rules-of-react` — rules of hooks + purity rules, framed as the contract the Compiler enforces

### Wave 2 — Intermediate (the machinery)

11. `how-react-renders` — fiber, render/commit phases, why StrictMode exists (the "under the hood" anchor article)
12. `useref-and-the-dom` — refs vs. state; ref-as-prop (19); `useImperativeHandle`; legacy `forwardRef` with marker
13. `context` — what it's for (DI-ish config), what it's not for (state management); split-context performance pattern
14. `usereducer-and-state-structure` — reducers, deriving vs. mirroring state, state-shape smells
15. `custom-hooks` — extraction rules, return-shape conventions, testing them
16. `memoization-and-the-compiler` — `memo`/`useMemo`/`useCallback` as mechanism; Compiler as the default; when manual still matters (library code, non-compiled deps)
17. `error-boundaries` — the one remaining class use-case; `react-error-boundary` in practice
18. `portals-and-the-event-system` — modals, event bubbling through portals
19. `escape-hatches-audit` — `useSyncExternalStore`, `useLayoutEffect`, `flushSync`: what each is actually for

### Wave 3 — Advanced (concurrent + 19-era)

20. `concurrent-rendering` — transitions, `useDeferredValue`, interruptible rendering, tearing
21. `suspense` — data + code suspense, boundaries, fallback design
22. `use-and-promises` — the `use()` hook, promises as first-class render inputs
23. `actions` — form actions, `useActionState`, `useOptimistic`, `useFormStatus` — the 19 mutation story
24. `react-compiler-deep-dive` — what it compiles, the rules it assumes, the ESLint plugin, opting out
25. `ssr-and-hydration` — SSR fundamentals, hydration, streaming, mismatch mechanics (framework-agnostic)
26. `server-components` — the RSC model: server/client boundary, serialization, what dies at the boundary

### Wave 4 — Ecosystem (priority order)

27. `data-fetching-tanstack-query` — server state as a distinct category; caching, invalidation, mutations
28. `routing-react-router` — data mode: loaders, actions, deferred data; code splitting per route
29. `state-management-landscape` — Zustand / Jotai / Redux Toolkit / context: decision table, not a tour
30. `forms-at-scale` — React Hook Form + Zod; where form actions end and RHF begins
31. `testing` — Vitest + RTL + user-event + MSW; testing behavior, not implementation
32. `styling-approaches` — CSS Modules / Tailwind / the CSS-in-JS retreat; decision table
33. `nextjs-and-rsc-in-practice` — App Router, server actions, caching model (builds on 25–26)
34. `performance-profiling` — DevTools Profiler, React Scan, Core Web Vitals workflow
35. `tanstack-router` — the type-safe alternative; when it beats React Router
36. `accessibility-in-react` — focus management, live regions, the ARIA-in-JSX gotchas

Wave 4 items 33–36 are flexible in order; 27–32 are not (recipes depend on them).

---

## 4. Recipe tracks — priority order

| # | Track | Why this priority | Sample symptoms |
|---|---|---|---|
| 1 | `data-fetching/` | The foundational track — race conditions, dedup, and abort get cross-referenced by everything else (same role race-condition recipes played in angular-concepts) | Stale search results overwrite fresh ones; duplicate requests on double-mount; spinner never resolves after unmount |
| 2 | `performance/` | Re-render storms are *the* React failure mode; Compiler changes the answers, which makes this track timely | Typing lags at 200ms/keystroke; 6s freeze on tab switch; 900KB initial bundle |
| 3 | `forms-and-ux/` | Highest density of real-world bugs per line of code | Double-submit creates two orders; optimistic Like flickers back; validation fires on first keystroke |
| 4 | `state-management/` | Where architectures go wrong | Server data cached in Zustand goes stale; prop drilling 6 levels; context re-renders the whole tree |
| 5 | `auth/` | Every real app; token refresh + protected routes are perennial | Refresh storm on 401; flash of protected content; logout leaves stale cache |
| 6 | `routing/` | Deep links, scroll restoration, route-level code splitting | Back button loses scroll; lazy route flashes blank; params out of sync with state |
| 7 | `testing/` | Async testing pain is universal | Flaky `waitFor` tests; `act()` warnings; MSW handler leaks between tests |
| 8 | `real-time/` | WebSocket/SSE lifecycle vs. StrictMode is a genuinely hard problem | Double socket connections in dev; missed reconnection; stale closure in message handler |
| 9 | `ssr-and-rsc/` | Deferred until articles 25–26 + 33 exist | Hydration mismatch on dates; `"use client"` sprawl; server action returns stale data |
| 10 | `pwa-offline/` | Long tail | Cached shell serves stale app; offline mutation queue |

Recipes begin after Wave 1 articles are complete, then run in parallel with Waves 2–4 (mirrors the angular-concepts pattern that worked).

---

## 5. Locked editorial conventions

### Code conventions

- **Function components only.** Classes appear once: in the error-boundaries article, immediately wrapped by `react-error-boundary`.
- **TypeScript strict throughout. No `React.FC`** — plain function with a typed props interface; discriminated unions for variant props.
- **Effects contract** (the `takeUntilDestroyed()` equivalent): every subscription-like effect returns a cleanup; every fetch inside an effect uses `AbortController` and ignores aborts; effects synchronize with external systems — never used to derive state from props/state.
- **Derive, don't mirror.** Computed values happen during render. `useEffect`-sets-state-from-props is always shown as the anti-pattern with the fix.
- **Keys are stable IDs, never array index.** (`key={item.id}`, the `track item` equivalent.)
- **`ref` as a prop (19+) over `forwardRef`.** Legacy form shown with marker comment.
- **Compiler-first memoization stance:** write pure components; React Compiler is assumed enabled for app code; `memo`/`useMemo`/`useCallback` are taught as *mechanism* and reserved for library code or non-compiled boundaries. No cargo-cult memoization in examples.
- **State placement rule:** local UI state → `useState`/`useReducer`; server state → TanStack Query (never in a client store); cross-cutting client state → Zustand; static config/DI → context. Violations of this table are recipe material.
- **Mutations:** form actions + `useActionState`/`useOptimistic` where forms are involved; TanStack Query mutations elsewhere.
- **Testing:** Vitest + React Testing Library + user-event + MSW. Query by role first; `data-testid` as fallback only.
- **Named exports**, co-located test files (`Component.test.tsx`).
- **Legacy marker:** `// legacy: written for React <19 — modernized in the upgrade pass` (same convention, React-flavored).

### Conventions deliberately dropped from angular-concepts

- No `inject()`-style DI framing — React has no DI container; context is taught as composition, not injection.
- No zoneless/change-detection framing — replaced by the render/commit + Compiler narrative.
- No NgModule-vs-standalone dual examples — React has one component model; dual examples only appear where React itself has old/new forms (forwardRef, class error boundaries, pre-Actions form handling).

### Article template — **adopted as-is**

frontmatter (`article_id`, `related`, `react_baseline: "19.2"`) → lead-with-this callout → what it is → how it works under the hood → basic usage → real-world patterns → common mistakes → how this evolved (when applicable) → see also

### Recipe template — **adopted with one rename**

frontmatter (`recipe_id`, `primary_concept`, `difficulty`, `react_baseline`) → "What you'll build" callout → the scenario (concrete failure mode with numbers) → walkthrough → variations → trade-offs and common pitfalls (incl. "when NOT to use this") → see also (≥3) → references → demo source

All the angular-concepts recipe qualities carry over unchanged: symptom-first framing, concrete numbers, 10–15 pitfalls per recipe, decision tables when patterns compete, dense inline cross-referencing.

---

## 6. Phase structure (starting from zero — no translation source)

- **Phase 1a — Foundations:** Waves 1–2 concept articles, in order. Approval checkpoint after articles 1–3, then rolling delivery.
- **Phase 1b — Recipes:** starts once Wave 1 is done; runs in parallel with everything after. Tracks in §4 order.
- **Phase 1c — Advanced + Ecosystem:** Waves 3–4, interleaved with recipes.
- **Phase 2 — Angular bridge layer:** deferred, additive. Bridge articles/annotations only; zero rewrites of Phase 1 content.

Working directory: `E:\linh tinh\EverythingFromDayOne\experimental-projects\reactjs-concepts\docs\` with `concepts/` + `recipes/<track>/`, plus `roadmap.md` and `progress.md` at root. Cursor handles bulk ops; Claude generates content; fresh chats per phase/session.

---

## 7. First deliverables (post-approval)

### First 3 concept articles

1. **`thinking-in-react`** — the mental-model anchor. UI = f(state); render vs. commit; a re-render is a function call, not a DOM rebuild. Everything else references this.
2. **`state-and-usestate`** — snapshots and batching. Scenario hooks: "I set state and logged it — it's the old value"; "I called setCount three times, it went up by one."
3. **`effects-and-synchronization`** — the most misused API in React. Cleanup, StrictMode double-invoke, the "you might not need an effect" decision tree, AbortController pattern locked here.

### First 3 recipes

1. **`data-fetching/search-race-condition`** — user types "rea", then "react"; the "rea" response lands last and stale results overwrite fresh ones. Fix arc: ignore-flag → AbortController → why TanStack Query makes this disappear. *The* foundational recipe every later one cross-references.
2. **`performance/typing-lag-rerender-storm`** — controlled input in a 300-row dashboard; every keystroke re-renders the table, 180ms per keystroke. Fix arc: state colocation → composition (children don't re-render) → Compiler verification with Profiler. Deliberately *not* "sprinkle useMemo."
3. **`forms-and-ux/double-submit-and-optimistic-like`** — user clicks Buy, nothing happens for 800ms, clicks again, two orders created; sibling scenario: Like count flickers back on failure. Fix arc: form actions + `useActionState` pending state → `useOptimistic` with rollback.

---

## Approval checklist

- [ ] §1 versions (React 19.2 / Vite 8 / TS strict / Vitest)
- [ ] §2 meta-framework stance (Vite SPA baseline, RSC fenced at end of Phase 1, React Router v7 + TanStack Query baselines)
- [ ] §3 article order and wave gating
- [ ] §4 recipe track order
- [ ] §5 conventions (esp. Compiler-first memoization stance and state placement rule)
- [ ] §6 phase structure
- [ ] §7 first three articles + first three recipes

On approval: start `thinking-in-react`.