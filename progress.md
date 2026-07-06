# reactjs-concepts — progress

Maintainer-facing tracking document. See [`README.md`](README.md) for the reader-facing entry point and [`roadmap.md`](roadmap.md) for the planning rationale.

## Legend

- ✅ Complete and reviewed
- 🟢 Drafted (React 19.2 idioms applied; awaiting review)
- 🟡 In progress
- ⚪ Queued
- ❌ Dropped / out of scope

---

## Roadmap (🟢 drafted)

`roadmap.md` drafted July 2026. Locks React 19.2 baseline, Vite 8 SPA stance (RSC fenced to end of Phase 1), article priority order (4 waves, 36 articles), recipe track order (10 tracks), and editorial conventions. **Approval checklist at the bottom of `roadmap.md` still open.**

---

## Phase 1a — foundation articles (🟡 in progress)

Waves gate delivery: finish a wave before starting the next. Approval checkpoint after the first 3 articles, then rolling delivery.

### Wave 1 — Foundations

| # | Article | Status | Notes |
| --- | --- | --- | --- |
| 1 | `foundations/thinking-in-react.md` | 🟢 | **First deliverable.** Mental-model anchor — everything references it. Opus tier |
| 2 | `foundations/jsx-and-rendering.md` | 🟢 | |
| 3 | `foundations/components-and-props.md` | 🟢 | |
| 4 | `state/state-and-usestate.md` | 🟢 | Second deliverable — snapshots, batching. Opus tier |
| 5 | `rendering/rendering-lists-and-keys.md` | 🟢 | Locks the stable-key convention |
| 6 | `foundations/conditional-rendering-and-events.md` | 🟢 | |
| 7 | `forms/forms-controlled-and-uncontrolled.md` | 🟢 | |
| 8 | `foundations/component-composition.md` | 🟢 | Composition-beats-memoization groundwork for the performance track |
| 9 | `effects/effects-and-synchronization.md` | 🟢 | Third deliverable — cleanup, StrictMode double-invoke, AbortController pattern locked here. Opus tier |
| 10 | `foundations/rules-of-react.md` | 🟢 | Framed as the contract the Compiler enforces |

### Wave 2 — Intermediate

| # | Article | Status | Notes |
| --- | --- | --- | --- |
| 11 | `rendering/how-react-renders.md` | 🟢 | Fiber, render/commit — the "under the hood" anchor. Opus tier |
| 12 | `effects/useref-and-the-dom.md` | 🟢 | ref-as-prop baseline; legacy `forwardRef` with marker |
| 13 | `state/context.md` | 🟢 | What it's for / not for; split-context pattern |
| 14 | `state/usereducer-and-state-structure.md` | 🟢 | Derive-don't-mirror lives here |
| 15 | `effects/custom-hooks.md` | 🟢 | |
| 16 | `rendering/memoization-and-the-compiler.md` | 🟢 | Compiler-first stance. Opus tier |
| 17 | `rendering/error-boundaries.md` | 🟢 | The one class use-case; `react-error-boundary` in practice |
| 18 | `rendering/portals-and-the-event-system.md` | 🟢 | |
| 19 | `effects/escape-hatches-audit.md` | 🟢 | `useSyncExternalStore`, `useLayoutEffect`, `flushSync` |

---

## Phase 1b — recipes (🟡 in progress — Wave 1 complete)

Starts once Wave 1 articles are complete, then runs in parallel with Waves 2–4. Track order per `roadmap.md` §4.

### First 3 recipes (locked)

| # | Recipe | Status | Scenario |
| --- | --- | --- | --- |
| 1 | `recipes/data-fetching/search-race-condition.md` | 🟢 | Stale "rea" response lands after "react" response and overwrites it. Fix arc: ignore-flag → AbortController → TanStack Query. The foundational recipe — heaviest expected in-degree |
| 2 | `recipes/performance/typing-lag-rerender-storm.md` | 🟢 | Controlled input in a 300-row dashboard; 180ms per keystroke. Fix arc: colocation → composition → Compiler verification. Deliberately not "sprinkle useMemo" |
| 3 | `recipes/forms-and-ux/double-submit-and-optimistic-like.md` | 🟢 | Two orders from a double-click; Like count flickers back on failure. Fix arc: form actions + `useActionState` → `useOptimistic` with rollback |

### Recipe #4 — state-management track opener

| # | Recipe | Status | Scenario |
| --- | --- | --- | --- |
| 4 | `recipes/state-management/context-rerenders-the-whole-tree.md` | 🟢 | 200-card product grid; megacontext bundling cart/wishlist/filters re-renders all 200 cards on any change, 220ms/interaction, INP 90ms→310ms. Fix arc: split by change cadence → leaf-extraction → honest structural ceiling + `useSyncExternalStore` escalation variation |

### Recipe #5 — data-fetching dedup (three-sessions-overdue clear)

| # | Recipe | Status | Scenario |
| --- | --- | --- | --- |
| 5 | `recipes/data-fetching/strictmode-double-mount.md` | 🟢 | Product page fires 2× GET and 2× POST-views per load; teammate deletes `<StrictMode>`, ships the missing-cleanup bug; prod view counts run ~1.9× actual. Fix arc: read the dev double as a fire drill → wrong fix (disable StrictMode) → abort+ignore for reads → classify writes (idempotency/relocation, `sendBeacon` in a loader) → key-dedup handover to TanStack Query. Sibling to `search-race-condition` (dedup half vs. ordering half) |

### Recipe #6 — data-fetching waterfall (unblocked by 27 + 28)

| # | Recipe | Status | Scenario |
| --- | --- | --- | --- |
| 6 | `recipes/data-fetching/request-waterfall.md` | 🟢 | Nested product-page components each fetch on render; Suspense boundaries accidentally serialize independent requests into a staircase — 2.8s on a throttled phone (vs ~900ms parallel). Escaped QA: localhost latency ~2ms hides it, skeletons masked it. Fix taxonomy: see/name (throttle) → parallelize independent (`useSuspenseQueries`) → real-dependency (collapse round-trips at the API) → route-loader flatten (`Promise.all` prefetch) → create-high/read-low → verify-the-loop. Cashes 27's prevention + 28's loader flatten + suspense serial-suspend from the bug side |

### Queued recipe candidates by track

| Track | Candidates |
| --- | --- |
| `data-fetching/` | ~~Duplicate requests on StrictMode double-mount~~ (✅ landed as recipe #5); ~~request waterfall from nested components~~ (✅ landed as recipe #6); spinner never resolves after unmount |
| `performance/` | 900KB initial bundle (route splitting); 6s freeze on tab switch (virtualization); LCP regression from client-only rendering |
| `forms-and-ux/` | Validation fires on first keystroke; debounced search with loading states; multi-step wizard state survival |
| `state-management/` | Server data cached in Zustand goes stale; prop drilling 6 levels deep |
| `auth/` | Refresh storm on 401; flash of protected content before redirect; logout leaves stale query cache |
| `routing/` | Back button loses scroll position; lazy route flashes blank; URL params out of sync with component state |
| `testing/` | Flaky `waitFor` tests; `act()` warnings; MSW handler leaks between tests |
| `real-time/` | Double WebSocket connections in dev (StrictMode); missed reconnection; stale closure in message handler |
| `ssr-and-rsc/` | Hydration mismatch on dates/random values; `"use client"` sprawl; server action returns stale data |
| `pwa-offline/` | Cached shell serves stale app; offline mutation queue |

### Planned composition hubs

Dense cross-referencing is a locked philosophy (composition IS content). Expected heaviest in-degree, mirroring what worked before:

- **`search-race-condition`** — every async recipe references it
- **`typing-lag-rerender-storm`** — hub for the performance track
- **`effects-and-synchronization` (concept)** — referenced by every recipe touching lifecycle
- **`data-fetching-tanstack-query` (concept)** — referenced by data-fetching, auth, and state-management tracks

---

## Phase 1c — advanced + ecosystem articles (⚪ queued)

### Wave 3 — Advanced

| # | Article | Status | Notes |
| --- | --- | --- | --- |
| 20 | `concurrent/concurrent-rendering.md` | 🟢 | Transitions, `useDeferredValue`, tearing. Opus tier |
| 21 | `concurrent/suspense.md` | 🟢 | Pending-is-a-boundary; ErrorBoundary-outside/Suspense-inside triad; stale-chunk reset |
| 22 | `concurrent/use-and-promises.md` | 🟢 | `use()` promise-tracking trace; stable-promise rule; await-vs-use; `.catch()` graceful degradation |
| 23 | `concurrent/actions.md` | 🟢 | The 19 mutation story; pairs with the double-submit recipe |
| 24 | `rendering/react-compiler-deep-dive.md` | 🟢 | Includes the Vite 8 / `@rolldown/plugin-babel` wiring gotcha. Opus tier |
| 25 | `server/ssr-and-hydration.md` | 🟢 | Framework-agnostic fundamentals. Opus tier |
| 26 | `server/server-components.md` | 🟢 | The RSC model. Opus tier |

### Wave 4 — Ecosystem

| # | Article | Status | Notes |
| --- | --- | --- | --- |
| 27 | `ecosystem/data-fetching-tanstack-query.md` | 🟢 | Wave 4 opener (428 lines). v5 API verified against official docs before writing. Paid ALL owed debts: server-cache spine, `useSuspenseQuery` as production stable-promise source, `useSuspenseQueries` waterfall prevention, `QueryErrorResetBoundary`+react-error-boundary, server-persisted-cart (optimistic `onMutate`/`onError`/`onSettled`), client-cache-vs-RSC contrast, raw-fetch-in-useEffect failure mode. Gates the data-fetching recipe track's library stage |
| 28 | `ecosystem/routing-react-router.md` | 🟢 | Data mode (395 lines). v7 APIs verified against official docs (defer-wrapper removed → bare-promise deferral, granular route `lazy`, `react-router` package consolidation, two error channels). Paid: `errorElement`/loader-error integration + the errorElement-vs-React-boundary distinction (error-boundaries), `<Await errorElement>` deferred-rejection (suspense), loaders/actions/deferred data, route-`lazy` code-split vs React.lazy (suspense). Bonus: RR `<Form>` vs React-19 `<form action>` disambiguation table; loader-prefetch-into-Query integration |
| 29 | `ecosystem/state-management-landscape.md` | 🟢 | Decision table, not a tour (277 lines). Zustand v5 / Jotai v2 / RTK v2 surfaces verified against official docs (esp. Zustand v5 object-selector-crashes-without-`useShallow`). Spine = the placement funnel (server→Query, form→Actions, config→context, local→useState; residue = cross-cutting client state). Paid the selector-store decision-table debt (context + state-mgmt recipe both point here) via the subscription-granularity mechanics + the decision table. Bonus: RTK-Query-vs-TanStack honesty (same category, don't run two caches), persistence/hydration caveat. Walkthrough added in depth pass (routing one screen's state end-to-end). Depth pass 2 (Huy coverage review, →279): added the derived-state axis across all three (informs the Jotai-for-derived-heavy cell) + the concrete Zustand-vs-Jotai call (atomic model wins for many-independent-pieces) + an async-is-usually-server-state guard. Deliberately did NOT add Redux-thunk/async-atom tours (would fight decision-not-tour + the spine) |
| 30 | `forms/forms-at-scale.md` | 🟢 | RHF v7 + Zod v4 (280 lines). Both surfaces verified against official docs (Zod v4: `z.email()` not `.email()`, `error` not `message`, `.issues` not `.errors`, `.refine`+`path` for cross-field; RHF: uncontrolled `register`, `Controller`, `useFieldArray` keyed by `field.id`, `zodResolver`). Paid the Actions-vs-RHF boundary debt (submission-complexity vs field-complexity table) + forms-controlled's Zod/RHF threshold + field-arrays/cross-field. Walkthrough: account form (field array + cross-field refine + shared schema client&server, server-only error via `setError`). Included from the start this time; self-audit added the success path (`reset`) |
| 31 | `ecosystem/testing.md` | 🟢 | Vitest 3 + RTL v16 + user-event v14 + MSW v2 (281 lines). All four surfaces verified against official docs (esp. MSW v2 `http`/`HttpResponse.json`/`setupServer` — v1's `rest`/`res(ctx.json)` fully gone; user-event v14 `setup()`-then-`await`). Thesis = test behavior not implementation. Paid: the toolchain-stance debt + custom-hooks' `renderHook`-full-story, query-by-role-first (the priority ladder + 3 query axes), compiler-transparent-to-tests. Walkthrough: async UserList tested end-to-end (loading→list→error-override→retry) with all four tools. Self-audit added the what-to-test/integration-first strategic beat + when-NOT (RTL vs E2E). Gates the testing recipe track |
| 32 | `ecosystem/styling-approaches.md` | 🟢 | Decision table, not a tour (233 lines). Tailwind v4 (CSS-first `@theme`/`@import`/`@tailwindcss/vite`) + the CSS-in-JS retreat verified against official docs (styled-components maintenance mode Mar 2025; RSC-not-SSR incompatibility; zero-runtime = vanilla-extract/Panda/StyleX). Axis = build-time vs runtime. Walkthrough: one Button three ways (Modules/Tailwind/vanilla-extract). Honest retreat section (not dead, still fits client-only/RN/runtime-theming). Self-audit added responsive + dark-mode-across-approaches. No hard debts owed; decision-table framing honored. Depth pass 2 (Huy coverage review, →237): added variant-management-at-scale (cva/tailwind-variants — the real Tailwind maintainability factor) + the organizational/token-enforcement decision axis + container-queries as a wash |
| 33 | `server/nextjs-and-rsc-in-practice.md` | ⚪ | Builds on 25–26; gates the ssr-and-rsc recipe track |
| 34 | `ecosystem/performance-profiling.md` | ⚪ | DevTools Profiler workflow; hub for the performance recipe track |
| 35 | `ecosystem/tanstack-router.md` | ⚪ | Flexible order (33–36) |
| 36 | `ecosystem/accessibility-in-react.md` | ⚪ | Flexible order (33–36) |

---

## Phase 2 — bridge layer (❌ deferred)

Additive bridge articles/annotations for Angular developers. Explicitly out of scope until Phase 1 matures. Zero rewrites of Phase 1 content; Phase 1 articles and recipes contain no Angular references anywhere.

---

## Locked editorial conventions

### Article structure (concept articles) — v2, July 2026 depth revision

1. Frontmatter (`article_id`, `concept_folder`, `wave`, `related`, `react_baseline`, `status`)
2. Lead-with-this callout (one-paragraph hook)
3. What it is
4. How it works under the hood (real internals, traced — fiber shapes, compile output, comparison semantics)
5. Basic usage (complete and runnable, imports included)
6. Walkthrough — build something small end-to-end (progressive steps, full files)
7. Real-world patterns
8. API/type reference table (when applicable)
9. Common mistakes (7–10, with code)
10. How this evolved (when applicable — anchored to the class → hooks → concurrent → 19 Actions → Compiler arc)
11. Exercises (2–3, with hints)
12. Summary
13. See also
14. References (official docs)
15. Demo source

Coverage bar, not line quota: every template section earning its place; typical range 380–650 lines, with length as a smell test only. Old-vs-new dual examples appear only where React itself has legacy forms: `forwardRef` vs ref-as-prop, class error boundaries, pre-Actions form handling, manual memoization vs Compiler.

### Recipe structure

1. Frontmatter (`recipe_id`, `primary_concept`, `difficulty`, `react_baseline`)
2. "What you'll build" callout (scenario summary)
3. The scenario (concrete failure mode, with numbers)
4. Walkthrough (multi-stage if complex)
5. Variations
6. Trade-offs and common pitfalls (incl. "when NOT to use this"; 10–15 concrete pitfalls)
7. See also (≥3 items)
8. References
9. Demo source

### Code conventions

- Function components only; classes appear once (error-boundaries article), immediately wrapped by `react-error-boundary`
- TypeScript strict throughout; no `React.FC` — plain function with a typed props interface; discriminated unions for variant props
- Effects contract: every subscription-like effect returns a cleanup; every fetch inside an effect uses `AbortController` and ignores aborts; effects synchronize with external systems — never derive state
- Derive during render, never mirror props into state (`useEffect`-sets-state shown only as the anti-pattern)
- Keys are stable IDs (`key={item.id}`), never array index
- `ref` as a prop (19+) over `forwardRef`; legacy form shown with marker
- Compiler-first memoization: pure components, Compiler assumed ON for app code; `memo`/`useMemo`/`useCallback` taught as mechanism, reserved for library code and non-compiled boundaries — no cargo-cult memoization in examples
- State placement rule: local UI state → `useState`/`useReducer`; server state → TanStack Query (never in a client store); cross-cutting client state → Zustand; static config → context
- Mutations: form actions + `useActionState`/`useOptimistic` where forms are involved; TanStack Query mutations elsewhere
- Testing: Vitest + RTL + user-event + MSW; query by role first, `data-testid` fallback only
- Named exports; co-located test files (`Component.test.tsx`)

### Legacy code preservation

Show old vs new patterns side by side with the marker comment:

```tsx
// legacy: written for React <19 — modernized in the upgrade pass
```

### Cross-referencing

- "See also" section at each recipe's end lists ≥3 related items
- Inline links use `[link text](../path/to/file.md#section-anchor)`
- Composition IS content — recipes that compose reference each other in the walkthrough, not just in "see also"

---

## Open questions / TODOs

- [ ] **Roadmap approval** — checklist at the bottom of `roadmap.md`; gates everything
- [ ] `LICENSE` — MIT assumed (matching angular-concepts); file not yet created at repo root — confirm choice
- [ ] `getting-started.md` / `typescript-for-react.md` — not in the 36-article roadmap; decide whether to add as pre-Wave-1 onboarding docs or fold into `thinking-in-react`
- [ ] Demo hosting — StackBlitz vs CodeSandbox vs local `demos/` folder for recipe demo sources
- [ ] Model allocation — Opus flags marked in the tables above (8 articles); confirm the rest as Sonnet-tier
- [ ] `recipes/index.md` longer-form index — revisit once ≥10 recipes exist

---

## Session log

- **Session 1 (July 2026):** roadmap approved; README/progress scaffolded; **Wave 1 complete** (articles 1–10, ~4,300 lines); **locked recipe trio complete** (search-race-condition, typing-lag-rerender-storm, double-submit-and-optimistic-like, ~875 lines); template v2 depth revision + coverage-bar reframe locked mid-session; session closed with `handoff-session-2.md` (ownership map + forward-reference debt registry). Next session opens Wave 2 with `rendering/how-react-renders.md`.
- **Session 2 (July 2026):** **Wave 2 complete** — `rendering/how-react-renders.md` (480 lines, Opus tier), `effects/useref-and-the-dom.md` (536 lines), `state/context.md` (397 lines), `state/usereducer-and-state-structure.md` (433 lines), `effects/custom-hooks.md` (385 lines), `rendering/memoization-and-the-compiler.md` (356 lines, Opus tier), `rendering/error-boundaries.md` (364 lines), `rendering/portals-and-the-event-system.md` (356 lines), `effects/escape-hatches-audit.md` (390 lines) delivered. All 19 articles (articles 11–19) drafted. **State-management recipe track opened** — `recipes/state-management/context-rerenders-the-whole-tree.md` (218 lines) delivered, closing the recipe-interleave debt. Session closed with `handoff-session-3.md` (full ownership map, articles 1–19 + recipe 4; Wave 3 debt registry). Next session opens Wave 3 with `concurrent/concurrent-rendering.md`.
- **Session 3 (July 2026):** **Wave 3 complete** — all 7 articles (20–26, ~2,360 lines; 4 Opus-tier): `concurrent/concurrent-rendering.md` (430 lines, Opus), `concurrent/suspense.md` (~340 lines), `concurrent/use-and-promises.md`, `concurrent/actions.md`, `rendering/react-compiler-deep-dive.md` (Opus, Vite 8 `@rolldown/plugin-babel` + `eslint-plugin-react-hooks` v6 wiring verified against official docs before writing), `server/ssr-and-hydration.md` (Opus), `server/server-components.md` (Opus). Walkthrough confirmed as the single biggest quality lever (self-audit caught the absent SSR entry files, an abstract RSC payload, an unshown `.catch()`, a missing await-vs-use line — all filled). README refreshed to current state. Session closed with `handoff-session-4.md` (ownership map for all 26 articles + 4 recipes; Wave 4 forward-reference debt registry). Next session opens Wave 4.
- **Session 4 (July 2026) — in progress:** **Overdue recipe cleared first.** `recipes/data-fetching/strictmode-double-mount.md` (~205 lines) delivered — three-sessions-overdue dedup recipe, writable since article 9, needs nothing from Wave 4. Owns the *dedup* half of the data-fetching story (sibling to `search-race-condition`'s *ordering* half): reads the dev double-mount as a StrictMode fire drill, rejects the disable-StrictMode anti-fix, applies effects-and-synchronization's abort+ignore to reads, classifies the write path (idempotency vs. relocation, `sendBeacon`-in-loader shown), and hands off to TanStack Query's key-dedup (deferred deep-dive to article 27). Self-audit filled the Stage-3 relocation hand-wave with a concrete loader snippet. **Wave 4 opened** — `ecosystem/data-fetching-tanstack-query.md` (428 lines) delivered as article 27, the heaviest-debt article in the project. v5 API surface verified against official TanStack Query docs before writing (the `loading→pending`/`isLoading→isPending`/`cacheTime→gcTime`/`useErrorBoundary→throwOnError`/`keepPreviousData→placeholderData` rename table, the two-axis `status`×`fetchStatus` machine, the `onMutate`/`onError`/`onSettled` optimistic contract, stable `useSuspenseQuery`/`useSuspenseQueries`/`usePrefetchQuery`, `QueryErrorResetBoundary`). Traced the real cache internals (`QueryClient → QueryCache → Query` + observers, hash-keyed dedup, staleness lifecycle, structural sharing) as the "under the hood." End-to-end walkthrough: raw-fetch → `useQuery` → dedup proof → mutation+invalidate → optimistic+rollback. Paid all seven Wave-4 debts owed to this article (see row 27 note). Self-audit added the three principal-reviewer gaps: dependent queries (`enabled`), a DevTools pointer, and the mandatory when-NOT beat. Next up: article 28 `ecosystem/routing-react-router.md`, OR interleave a now-unblocked data-fetching library-stage recipe (waterfall / spinner-never-resolves) since 27 landed. — `ecosystem/routing-react-router.md` (395 lines) delivered as article 28. React Router v7 **data mode**; v7-specific APIs verified against official docs before writing (the `defer()` wrapper is removed → return an object with bare promise keys; `<Await>`/`use()` unwrap; granular route `lazy` 7.5+; package consolidated to `react-router`; the two-error-channel model). Traced the navigation lifecycle (match → parallel loaders → wait-for-critical → commit + `useNavigation` state + auto-revalidation). End-to-end walkthrough: loader with params + deferred reviews → `<Suspense>`/`<Await>` → `<Form>`/action + pending + redirect → route `lazy`. Paid all four owed debts (errorElement/loader-error + the errorElement-vs-React-boundary distinction, `<Await errorElement>`, loaders/actions/deferred, route-lazy-vs-React.lazy). Self-audit added nested-routes/`<Outlet>` (was referenced but unshown), a when-NOT beat, and `useSearchParams`. Included a bonus RR-`<Form>`-vs-React-19-`<form action>` disambiguation table (prevents a real conflation footgun) and the honest loader-prefetch-into-Query integration. **Two Wave-4 articles done (27–28); 8 remain (29–36).** Data-fetching library-stage recipes and the `routing/` recipe track are now both unblocked. Next up: article 29 `ecosystem/state-management-landscape` (decision table, not a tour), OR interleave a recipe — roughly one per 2–3 articles says a recipe is about due. — **Recipe interleaved.** `recipes/data-fetching/request-waterfall.md` (206 lines) delivered as recipe #6, unblocked by 27 + 28. Nested fetch-on-render + Suspense-serializes-independent-fetches scenario (2.8s throttled → ~900ms parallel; escaped QA via localhost latency + skeleton masking). Fix taxonomy: throttle-to-see + name-the-two-causes → `useSuspenseQueries` for accidental waterfalls → real-vs-accidental test + `enabled` dependent-query + API round-trip collapse for real dependencies → route-loader `Promise.all` prefetch (with `prefetchQuery`-vs-`ensureQueryData` + block-vs-stream honesty) → create-high/read-low → verify-the-loop (before/after table). 13 pitfalls incl. warm-cache-hides-cold-start, unbounded-parallelism, and the when-NOT. Cashes article 27's waterfall-prevention, article 28's loader flatten + code-split-`lazy`, and suspense's serial-suspend — all from the bug side. Post-delivery depth pass (Huy flagged 183→now 206): Stage 2 was prose-only, filled with the `enabled` pattern + real-vs-accidental test + correct-nesting note; Stage 3 gained the prefetch/ensure/stream tradeoff; pitfalls 10→13. **Session 4 tally: 3 recipe/article deliveries (recipe #5, articles 27–28) + this recipe #6 = the overdue clear + 2 Wave-4 articles + 1 interleaved recipe.** Data-fetching track now has 3 landed recipes (search-race / strictmode / waterfall); `spinner-never-resolves` and the `auth/` refresh-storm remain open. Next up: article 29 `state-management-landscape` (decision table, not a tour — the full state-placement table across Query/Zustand/Jotai/Redux/context). **~4 deliveries into this chat; will offer the session-5 handoff around ~10.** — `ecosystem/state-management-landscape.md` (277 lines) delivered as article 29. Framed as decision-not-tour: the spine is the placement funnel (server→Query, form-shaped→Actions, config→context, local→useState; the residue = cross-cutting client state, which is all the libraries contest). Zustand v5 / Jotai v2 / RTK v2 surfaces verified against official docs before writing — the load-bearing v5 gotcha is that an object/array selector without `useShallow` now *crashes* ("Maximum update depth exceeded"), not just extra renders. Paid the selector-store decision-table debt (context + the context-rerenders recipe both point here) through the subscription-granularity mechanics (context=all-consumers / Zustand=selector-slice / Jotai=per-atom, all on `useSyncExternalStore`) + the decision table. Honest RTK-Query-vs-TanStack beat (same server-state category; never two caches) and a persistence/hydration caveat. Depth pass (caught the same thin-article smell Huy flagged on the waterfall): the non-optional **walkthrough was missing** — added "routing one screen's state" end-to-end (megacontext → Query + context + Zustand-selectors + local, with the re-render payoff), plus the persist/`atomWithStorage` pattern. Deliberately NOT padded into a per-library feature tour (handoff-forbidden). **Session 4 tally: recipe #5, articles 27–28, recipe #6, article 29 = 5 deliveries. Wave 4 at 3/10 articles (27–29); recipes at 6 total.** Next up: article 30 `forms/forms-at-scale` (RHF + Zod; where form actions end and RHF begins) — verify RHF + Zod v3/v4 surfaces first. — `forms/forms-at-scale.md` (280 lines) delivered as article 30. RHF v7 + Zod v4 surfaces verified against official docs before writing — Zod v4 is now the default `zod` import (since July 2025): top-level `z.email()` (not `.email()`), unified `error` param (not `message`), `ZodError.issues` (not `.errors`), `ctx.path` gone from `superRefine` but `.refine(fn, { path })` still targets cross-field; RHF v7 uncontrolled `register` / `Controller` / `useFieldArray` keyed by `field.id` / `zodResolver` (Standard Schema). Central debt paid: the **Actions-vs-RHF boundary** = submission-complexity vs field-complexity (decision table), resolving both actions' "where RHF begins" and forms-controlled's Zod/RHF threshold. Under-the-hood: RHF's uncontrolled-by-default + proxy-`formState` = no re-render per keystroke (the controlled-model perf fix). Walkthrough: account form with a field array, cross-field password refine (with `path`), and ONE shared Zod schema validating client (`zodResolver`) and server (`.safeParse`), server-only email-taken error mapped back via `setError`. **Walkthrough included from the start this time** (no repeat of the article-29 miss); self-audit added the success path (`reset`). Fixed a frontmatter fence bug pre-delivery. **Session 4 tally: 6 deliveries (recipe #5, articles 27–30, recipe #6). Wave 4 at 4/10 articles (27–30); recipes at 6.** Next up: article 31 `ecosystem/testing` (Vitest + RTL + user-event + MSW; gates the testing recipe track) — verify Vitest/RTL/MSW surfaces first. Handoff deferred at Huy's request; ~6 deliveries into this chat. — `ecosystem/testing.md` (281 lines) delivered as article 31. Vitest 3 + RTL v16 + user-event v14 + MSW v2 surfaces verified against official docs before writing — the big version-sensitive one is MSW v2 (`http`/`HttpResponse.json`/`setupServer` from `msw/node`, `listen`/`resetHandlers`/`close` lifecycle, `server.use` overrides; v1's `rest`/`res(ctx.json)` is entirely gone), plus user-event v14's `setup()`-then-`await` and RTL's role-first query priority. Thesis = test behavior not implementation. Paid: the toolchain-stance debt + completed custom-hooks' `renderHook` "full story", query-by-role-first (priority ladder + the getBy/queryBy/findBy three-axis model), and the compiler-transparent-to-behavior-tests note. Walkthrough (included from the start): async `UserList` tested end-to-end across all four tools — loading (`getBy`) → list (`findBy`) → error via `server.use` override → retry interaction. Real-world: `renderHook` full story, the `QueryClientProvider`+`retry:false` harness that gates the data-fetching test recipes, findBy-over-waitFor, tests-as-a11y-check. Self-audit added the what-to-test/integration-first strategic beat + when-NOT (RTL for component behavior, E2E/Playwright for full journeys). Frontmatter fence correct this time. **Session 4 tally: 7 deliveries (recipe #5, articles 27–31, recipe #6). Wave 4 at 5/10 articles (27–31) — the fixed-order block 27–31 is DONE; 32 `styling-approaches` next (decision table), then 33–36 flexible. Recipes at 6; the testing/ recipe track is now unblocked.** Next up: article 32 `ecosystem/styling-approaches` (decision table, not a tour — CSS Modules / Tailwind / the CSS-in-JS retreat). ~7 deliveries into this chat; handoff still deferred per Huy. — `ecosystem/styling-approaches.md` (233 lines) delivered as article 32, the second explicit decision-table-not-a-tour. Version-sensitive surfaces verified: Tailwind v4 CSS-first config (`@import "tailwindcss"`, `@theme` in CSS, `@tailwindcss/vite`, Oxide, auto content detection — a real break from v3's `@tailwind` trio + `tailwind.config.js`) and the CSS-in-JS retreat (styled-components maintenance mode Mar 2025, maintainer's own "don't adopt for new projects"; the RSC-not-SSR incompatibility mechanism — runtime `<style>`+`useContext` needs a browser lifecycle Server Components lack; zero-runtime successors vanilla-extract/Panda/StyleX). Defining axis = build-time vs runtime. Walkthrough: one Button styled three ways (CSS Modules / Tailwind / vanilla-extract) so the trade-offs are concrete. Honest retreat section with the nuance (not deprecated; still fits client-only/React Native/genuine runtime theming; don't panic-migrate). No hard forward-debts owed; decision-table framing honored (no per-library feature tour). Self-audit added responsive + dark-mode-across-approaches (decision-relevant, and reinforces that runtime CSS-in-JS's niche is narrow). **Session 4 tally: 8 deliveries (recipe #5, articles 27–32, recipe #6). Wave 4 at 6/10 articles (27–32) — all fixed-order (27–32) DONE; only flexible 33–36 remain. Recipes at 6.** Next up: flexible block 33–36 — likely 34 `performance-profiling` (heaviest remaining debt, gates the performance recipe track) or 33 `nextjs-and-rsc-in-practice`. ~8 deliveries into this chat; handoff still deferred per Huy but approaching the ~10 marker.

- **Session 4 coverage review (Huy-initiated):** Huy flagged that recent Wave-4 concept articles were trending short (29–32 at 233–281 vs Wave-2's 356–536). Honest finding: 27/28/30/31 defensible at length; 29 and 32 (the two decision-table-mandated articles) were the leanest and got a genuine coverage re-audit. **Filled real decision-relevant gaps** — 29: derived-state-across-three + Zustand-vs-Jotai concrete call + async-guard; 32: variant-management-at-scale (cva) + organizational/token-enforcement axis + container-queries. **Deliberately did NOT pad** — several candidate gaps (Redux-thunk tour, async-atom section, store-testing) were rejected as decision-not-tour violations, spine violations (async⇒server state⇒Query), or owned by other articles (testing→31). Line delta small (277→279, 233→237) because these articles are prose-paragraph-dense (one paragraph = one raw line), so line count tracks code density, not prose coverage. **Process correction for 33–36: raise the first-pass self-audit bar so it stops being rescue work** — audit each section against "what would a principal reviewer expect here" *before* delivery, not after. Two thin first-drafts this session (waterfall recipe, article-29 missing-walkthrough) shared a root cause: a weight-bearing section hollow on first pass.

- **Frontmatter correction (Huy-flagged via screenshots):** all six Wave-4 articles (27–32) had numeric `article_id` (`article_id: 27` …) — WRONG. The established convention (Wave 1–3) is `article_id: <filename-slug>` (e.g., `article_id: thinking-in-react`). I'd conflated the reading-order number (which lives in progress.md's # column and the roadmap) into the id field. **Fixed:** article_id now the slug in all six (`data-fetching-tanstack-query`, `routing-react-router`, `state-management-landscape`, `forms-at-scale`, `testing`, `styling-approaches`). Recipes were already correct (`recipe_id` = slug). **Open item flagged to Huy:** the `status` block internal shape (`stage: drafted` / `reviewed: false`) was invented in turn 1 without a reference file (uploads empty; handoff gave field order but not the status block's internal keys) — awaiting Huy's canonical shape to align all files (articles + recipes). Root-cause lesson: verify field *values* against an existing file's frontmatter, not just the field *names* from the handoff spec.
- **Session 4 closed.** `handoff-session-5.md` created (full ownership map incl. Wave-4 articles 27–32 + recipes 5–6; forward-debt registry for 33–36; the version facts verified this session; and — prominently — the corrections & standing directives: `article_id`=slug, copy-the-`status`-shape-from-a-real-file, **surface-every-inferred/invented-decision-for-review**, the raised first-pass audit bar, and the line-count-is-a-weak-proxy note; plus the open review items: anchors, section deviations, difficulty vocab, recipe status shape). **Final session-4 state: Waves 1–3 complete (26 articles); Wave 4 at 6/10 (fixed-order block 27–32 done, 33–36 remain); 6 recipes drafted.** Next session opens the Wave-4 tail — recommended `ecosystem/performance-profiling` (34, heaviest remaining debt) first — or clears the carried-over review items, then rolls. Recipe backlog large and mostly unblocked; single most valuable next recipe = `state-management/zustand-goes-stale`.