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
| 33 | `server/nextjs-and-rsc-in-practice.md` | 🟢 | Next.js 16 + React 19.2 (440 lines). App Router caching model verified against official Next.js docs before writing (the big shift: caching is opt-in / dynamic-by-default; Cache Components = PPR + `use cache`; `cacheLife`/`cacheTag` stable; `updateTag` Server-Action-only read-your-own-writes vs `revalidateTag(tag,'max')` stale-while-revalidate vs `revalidatePath`; async `params`; framework-owned React Compiler; Turbopack default; `middleware`→`proxy`). Paid all owed debts: production RSC (App Router/caching/revalidation), production SSR + PPR (static-shell-plus-stream three tiers + the "uncached data outside Suspense" enforcement), the Next.js double-compile note (framework owns the compiler wiring — cited 24), Server Actions in the App Router (cited 23 for the client half), the framework-mode routing note (cited 28). Walkthrough = a production product page end-to-end (three tiers on one page → Server Action + `updateTag` → RSC→TanStack-Query hydration bridge — cited 27). State-placement spine held across the server boundary (server state → Query cache, never a client store). Self-audit added: `@/lib/*` infra-elision note, a Route-Handler gloss, and the route-level `loading.tsx`/`error.tsx` conventions. Gates the ssr-and-rsc recipe track |
| 34 | `ecosystem/performance-profiling.md` | 🟢 | Heaviest remaining debt (439 lines). React 19.2 Performance Tracks + React Scan + INP/`web-vitals` + `<Profiler>` API all verified against official/current docs before writing. Two measurement layers (React: DevTools Profiler / `<Profiler>` / React Scan / 19.2 tracks — browser: CWV via `web-vitals`). Thesis = measure before optimizing; count-vs-cost distinction. Paid all owed debts: ecosystem-depth Profiler workflow (memoization's verification was the preview), measuring-whether-concurrent-features-helped (Scheduler track Blocking-vs-Transition lane check) + measuring-whether-the-Compiler-helped (✨ badge + `actualDuration`-vs-`baseDuration`), the React Scan triage layer, and the INP workflow (typing-lag recipe's INP tie-in). Walkthrough = the measurement loop end-to-end on a 500-row filtered table (triage → locate/cost → confirm impact via Scheduler track + INP 342ms→180ms → one fix `useDeferredValue` → verify the lane moved → confirm Compiler's ✨ half). Deliberately did NOT re-teach the typing-lag colocation fix (cited recipe as owner) or the memo/Compiler mechanism (cited 16/24) — this article owns *measurement*. Self-audit added the missing `Product` type to the walkthrough. Hub for the performance recipe track |
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
- [ ] **Cross-link anchors (carried from session 4, still open):** guessed heading→slug anchors in articles 27–34 + recipes 5–6 need verification against the real target files (e.g. `#render-and-commit`, `#bailouts`, `#usedeferredvalue`, `#bailout`, `#status-unions`, `#usesyncexternalstore`, etc.). Convention sanctioned; specific slugs unverified
- [ ] **Recipe cross-refs in article frontmatter:** `related` block confirmed (from a real file) to use concept `<folder>/<slug>` entries only; recipe cross-refs kept in "See also" as relative links. Confirm whether recipes should ever appear in the `related` block and, if so, the prefix convention

---

## Session log

- **Session 1 (July 2026):** roadmap approved; README/progress scaffolded; **Wave 1 complete** (articles 1–10, ~4,300 lines); **locked recipe trio complete** (search-race-condition, typing-lag-rerender-storm, double-submit-and-optimistic-like, ~875 lines); template v2 depth revision + coverage-bar reframe locked mid-session; session closed with `handoff-session-2.md` (ownership map + forward-reference debt registry). Next session opens Wave 2 with `rendering/how-react-renders.md`.
- **Session 2 (July 2026):** **Wave 2 complete** — `rendering/how-react-renders.md` (480 lines, Opus tier), `effects/useref-and-the-dom.md` (536 lines), `state/context.md` (397 lines), `state/usereducer-and-state-structure.md` (433 lines), `effects/custom-hooks.md` (385 lines), `rendering/memoization-and-the-compiler.md` (356 lines, Opus tier), `rendering/error-boundaries.md` (364 lines), `rendering/portals-and-the-event-system.md` (356 lines), `effects/escape-hatches-audit.md` (390 lines) delivered. All 19 articles (articles 11–19) drafted. **State-management recipe track opened** — `recipes/state-management/context-rerenders-the-whole-tree.md` (218 lines) delivered, closing the recipe-interleave debt. Session closed with `handoff-session-3.md` (full ownership map, articles 1–19 + recipe 4; Wave 3 debt registry). Next session opens Wave 3 with `concurrent/concurrent-rendering.md`.
- **Session 3 (July 2026):** **Wave 3 complete** — all 7 articles (20–26, ~2,360 lines; 4 Opus-tier): `concurrent/concurrent-rendering.md` (430 lines, Opus), `concurrent/suspense.md` (~340 lines), `concurrent/use-and-promises.md`, `concurrent/actions.md`, `rendering/react-compiler-deep-dive.md` (Opus, Vite 8 `@rolldown/plugin-babel` + `eslint-plugin-react-hooks` v6 wiring verified against official docs before writing), `server/ssr-and-hydration.md` (Opus), `server/server-components.md` (Opus). Walkthrough confirmed as the single biggest quality lever (self-audit caught the absent SSR entry files, an abstract RSC payload, an unshown `.catch()`, a missing await-vs-use line — all filled). README refreshed to current state. Session closed with `handoff-session-4.md` (ownership map for all 26 articles + 4 recipes; Wave 4 forward-reference debt registry). Next session opens Wave 4.
- **Session 4 (July 2026) — in progress:** **Overdue recipe cleared first.** `recipes/data-fetching/strictmode-double-mount.md` (~205 lines) delivered — three-sessions-overdue dedup recipe, writable since article 9, needs nothing from Wave 4. Owns the *dedup* half of the data-fetching story (sibling to `search-race-condition`'s *ordering* half): reads the dev double-mount as a StrictMode fire drill, rejects the disable-StrictMode anti-fix, applies effects-and-synchronization's abort+ignore to reads, classifies the write path (idempotency vs. relocation, `sendBeacon`-in-loader shown), and hands off to TanStack Query's key-dedup (deferred deep-dive to article 27). Self-audit filled the Stage-3 relocation hand-wave with a concrete loader snippet. **Wave 4 opened** — `ecosystem/data-fetching-tanstack-query.md` (428 lines) delivered as article 27, the heaviest-debt article in the project. v5 API surface verified against official TanStack Query docs before writing (the `loading→pending`/`isLoading→isPending`/`cacheTime→gcTime`/`useErrorBoundary→throwOnError`/`keepPreviousData→placeholderData` rename table, the two-axis `status`×`fetchStatus` machine, the `onMutate`/`onError`/`onSettled` optimistic contract, stable `useSuspenseQuery`/`useSuspenseQueries`/`usePrefetchQuery`, `QueryErrorResetBoundary`). Traced the real cache internals (`QueryClient → QueryCache → Query` + observers, hash-keyed dedup, staleness lifecycle, structural sharing) as the "under the hood." End-to-end walkthrough: raw-fetch → `useQuery` → dedup proof → mutation+invalidate → optimistic+rollback. Paid all seven Wave-4 debts owed to this article (see row 27 note). Self-audit added the three principal-reviewer gaps: dependent queries (`enabled`), a DevTools pointer, and the mandatory when-NOT beat. — `ecosystem/routing-react-router.md` (395 lines) delivered as article 28. React Router v7 **data mode**; v7-specific APIs verified against official docs before writing (the `defer()` wrapper is removed → return an object with bare promise keys; `<Await>`/`use()` unwrap; granular route `lazy` 7.5+; package consolidated to `react-router`; the two-error-channel model). Traced the navigation lifecycle (match → parallel loaders → wait-for-critical → commit + `useNavigation` state + auto-revalidation). End-to-end walkthrough: loader with params + deferred reviews → `<Suspense>`/`<Await>` → `<Form>`/action + pending + redirect → route `lazy`. Paid all four owed debts (errorElement/loader-error + the errorElement-vs-React-boundary distinction, `<Await errorElement>`, loaders/actions/deferred, route-lazy-vs-React.lazy). Self-audit added nested-routes/`<Outlet>` (was referenced but unshown), a when-NOT beat, and `useSearchParams`. Included a bonus RR-`<Form>`-vs-React-19-`<form action>` disambiguation table (prevents a real conflation footgun) and the honest loader-prefetch-into-Query integration. **Two Wave-4 articles done (27–28); 8 remain (29–36).** — **Recipe interleaved.** `recipes/data-fetching/request-waterfall.md` (206 lines) delivered as recipe #6, unblocked by 27 + 28. Nested fetch-on-render + Suspense-serializes-independent-fetches scenario (2.8s throttled → ~900ms parallel; escaped QA via localhost latency + skeleton masking). Fix taxonomy: throttle-to-see + name-the-two-causes → `useSuspenseQueries` for accidental waterfalls → real-vs-accidental test + `enabled` dependent-query + API round-trip collapse for real dependencies → route-loader `Promise.all` prefetch → create-high/read-low → verify-the-loop. Post-delivery depth pass (Huy flagged 183→206): filled Stage 2 (`enabled` + real-vs-accidental test), Stage 3 (prefetch/ensure/stream tradeoff), pitfalls 10→13. — `ecosystem/state-management-landscape.md` (277→279 lines) delivered as article 29 (decision-not-tour; Zustand v5 / Jotai v2 / RTK v2 verified; placement-funnel spine; walkthrough added in depth pass; Huy coverage review added derived-state axis + Zustand-vs-Jotai call + async-guard). — `forms/forms-at-scale.md` (280 lines) delivered as article 30 (RHF v7 + Zod v4 verified; Actions-vs-RHF boundary; shared-schema client+server walkthrough). — `ecosystem/testing.md` (281 lines) delivered as article 31 (Vitest 3 + RTL v16 + user-event v14 + MSW v2 verified; test-behavior-not-implementation; async UserList walkthrough; gates the testing recipe track). — `ecosystem/styling-approaches.md` (233→237 lines) delivered as article 32 (decision-not-tour; Tailwind v4 + CSS-in-JS retreat verified; build-time-vs-runtime axis; one-Button-three-ways walkthrough; Huy coverage review added variant-management + org/token axis + container-queries). **Session 4 tally: 8 deliveries (recipe #5, articles 27–32, recipe #6). Wave 4 at 6/10 (fixed-order block 27–32 DONE; 33–36 remain). Recipes at 6.**
- **Session 4 coverage review (Huy-initiated):** flagged Wave-4 articles trending short. Filled real decision-relevant gaps in 29 & 32; deliberately did NOT pad (rejected Redux-thunk tour, async-atom section, store-testing as decision-not-tour / spine / other-owner violations). **Process correction for 33–36: raise the first-pass self-audit bar so it stops being rescue work** — audit each section against "what would a principal reviewer expect here" *before* delivery.
- **Frontmatter correction (Huy-flagged):** all six Wave-4 articles (27–32) had numeric `article_id` — WRONG; fixed to filename slugs. Open item: canonical `status` block shape awaited.
- **Session 4 closed.** `handoff-session-5.md` created. Final state: Waves 1–3 complete (26 articles); Wave 4 at 6/10; 6 recipes drafted.
- **Session 5 (July 2026) — in progress:** Opened the Wave-4 tail. **Canonical `status` shape locked from a real Wave-1 file** (`effects-and-synchronization.md` frontmatter, via screenshot): `status:` → `drafted: <bool>` / `reviewed: <bool>` — NOT the session-4 invented `stage:` form. Applied to the new article; existing files' status blocks fixed by Huy directly. Uploads were empty this session (no on-disk reference/roadmap/progress files — only in-context text), so `progress.md` was regenerated from the in-context copy with just the row-34 flip + this log entry; reconcile if other manual edits exist. — `ecosystem/performance-profiling.md` (439 lines) delivered as article 34 — the heaviest remaining forward-debt and most-pointed-to Wave-4 article. All four version-sensitive surfaces verified against official/current docs before writing: **React 19.2 Performance Tracks** (Scheduler track's Blocking/Transition/Suspense/Idle subtracks + Components flamegraph; dev-auto vs profiling-build behavior; cascading-update detection), **React Scan** (visual necessary-vs-unnecessary triage; `scan()` / CLI / script-tag; dev-only; `dangerouslyForceRunInProduction` never), **Core Web Vitals** (INP good ≤200ms / poor >500ms, replaced FID Mar 2024, = input-delay+processing+presentation; LCP 2.5s/4s; CLS 0.1/0.25) + the **`web-vitals`** API (`onINP`/`onLCP`/`onCLS`, attribution build, `sendBeacon`, SPA hard-navigation caveat, LoAF attribution), and the **`<Profiler>`** `onRender` signature (6-arg; the 7th `interactions` arg removed in React 18). Thesis = measure at two layers (React: what/why/how-much rendered; browser: what the user felt), never optimize before measuring; the count-vs-cost distinction is the spine. Paid ALL owed debts: the ecosystem-depth Profiler workflow (memoization's verification was the preview), measuring-whether-concurrent-features-helped (Scheduler track Blocking-vs-Transition lane check) + measuring-whether-the-Compiler-helped (✨ badge + `actualDuration`-vs-`baseDuration`), the React Scan triage layer, and the INP workflow (typing-lag recipe's INP tie-in). Walkthrough (highest-leverage section, ~190 lines) = the measurement loop end-to-end on a 500-row filtered table: React Scan triage → Profiler locate+cost (ranked chart + why-rendered) → 4×-throttle Scheduler track + `web-vitals` INP baseline (342ms poor) → one fix (`useDeferredValue` to move the table render off the Blocking lane — concurrent-rendering owns the mechanism, this article measures it) → verify the lane moved + INP 180ms good → confirm the Compiler's ✨ half → verify-the-loop table. Deliberately did NOT re-teach the typing-lag recipe's colocation/composition fix (cited it as owner) or the memo/Compiler mechanism (cited 16/24) — this article owns *measurement*, not the fixes. **Raised first-pass self-audit caught one real gap before delivery:** the walkthrough claimed "full files" but omitted the `Product` type — patched. **Inferred/unverified decisions surfaced for review:** (a) cross-link anchors `#render-and-commit`, `#bailouts`, `#usedeferredvalue` are guessed slugs (same carried-over anchor review item — no real target files on disk to verify against); (b) recipe cross-refs kept in "See also" as relative links rather than the frontmatter `related` block, since the real-file `related` shape only showed concept `<folder>/<slug>` entries — did not invent a recipe-prefix convention. **Wave 4 at 7/10 (27–32 + 34); 33 + 35–36 remain. Recipes at 6.** — `server/nextjs-and-rsc-in-practice.md` (440 lines) delivered as article 33 — the second-heaviest forward-debt; gates the ssr-and-rsc recipe track. Next.js 16 (requires React 19.2) App Router caching model verified against official Next.js docs before writing: the load-bearing shift is that **caching is opt-in / dynamic-by-default** — Cache Components = **PPR + `use cache`** (the experimental `experimental.ppr` flag + `experimental_ppr` route config removed), the framework prerenders a static shell and errors ("Uncached data accessed outside `<Suspense>`") if request-time data is neither cached nor streamed; `cacheLife`/`cacheTag` now stable (no `unstable_` prefix); invalidation trio `updateTag` (Server-Action-only, immediate read-your-own-writes) vs `revalidateTag(tag,'max')` (stale-while-revalidate, also Route Handlers; single-arg form deprecated) vs `revalidatePath` (blunt); `params`/`searchParams`/`cookies()`/`headers()` async-only in 16; React Compiler built-in (`reactCompiler:true`, Babel→Rust); Turbopack default; `middleware.ts`→`proxy.ts`. Thesis = Next.js is where the RSC *model* (owned by 26) becomes a production app; the three-tier mental model (static / cached-dynamic via `use cache` / runtime-dynamic via `<Suspense>`) is the spine. Paid ALL owed debts: production RSC (App Router/caching/revalidation), production SSR + partial-prerendering (static-shell-plus-stream), the Next.js double-compile note (framework owns the compiler wiring — you don't wire `@rolldown/plugin-babel` yourself; cited 24), Server Actions in the App Router (server half here; cited 23 for `useActionState`/client half), the framework-mode routing note (cited 28 — App Router is the framework-mode router RR's data-mode article fenced out). Walkthrough (highest-leverage, ~215 lines) = a production product page end-to-end: three tiers on one page (cached `getProduct` + streamed cookie-reading `Recommendations`) → Server Action `submitReview` + `updateTag` read-your-own-writes wired through `<form action>`/`useActionState` → the **RSC→TanStack-Query hydration bridge** (`prefetchQuery` on the server → `<HydrationBoundary state={dehydrate(qc)}>` → warm `useQuery` on the client, no first-paint spinner; cited 27). State-placement spine held across the server boundary (server state → Query cache, never mirrored into a client store). Deliberately did NOT re-teach the RSC model (26), SSR fundamentals (25), the Compiler mechanism (24), the Actions client machinery (23), or the Query cache (27) — cited each owner; this article owns *production wiring*. **Raised first-pass self-audit caught three real gaps before delivery:** `@/lib/*` infra referenced without stating it's an elided stub (fixed with a one-line note so "full files" is honest), "Route Handler" leaned on without a gloss (fixed), and the route-level `loading.tsx`/`error.tsx` conventions absent next to the manual `<Suspense>` (added, connecting the primitive to how real apps wire streaming/errors; cited 17 for the error-boundary mechanics). **Inferred/unverified decisions surfaced for review (carry-over anchor item):** cross-link anchors `#the-payload`, `#vite-wiring`, `#useactionstate`, `#hydration`, `#the-placement-funnel`, `#shared-schema`, `#streaming` are guessed slugs — no real target files on disk to verify against. **Wave 4 at 8/10 (27–34 done); only 35–36 remain. Recipes at 6.** Next up: recipe interleave is due (2 articles since the last recipe) — recommended `state-management/zustand-goes-stale` (the single-most-valuable recipe; server-state-in-a-client-store, article 29's #1 mistake points at it), OR finish the article tail with 35 `tanstack-router` / 36 `accessibility-in-react`.