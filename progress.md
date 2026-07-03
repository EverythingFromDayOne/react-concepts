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

## Phase 1a — foundation articles (⚪ queued)

Waves gate delivery: finish a wave before starting the next. Approval checkpoint after the first 3 articles, then rolling delivery.

### Wave 1 — Foundations

| # | Article | Status | Notes |
| --- | --- | --- | --- |
| 1 | `foundations/thinking-in-react.md` | ⚪ | **First deliverable.** Mental-model anchor — everything references it. Opus tier |
| 2 | `foundations/jsx-and-rendering.md` | ⚪ | |
| 3 | `foundations/components-and-props.md` | ⚪ | |
| 4 | `state/state-and-usestate.md` | ⚪ | Second deliverable — snapshots, batching. Opus tier |
| 5 | `rendering/rendering-lists-and-keys.md` | ⚪ | Locks the stable-key convention |
| 6 | `foundations/conditional-rendering-and-events.md` | ⚪ | |
| 7 | `forms/forms-controlled-and-uncontrolled.md` | ⚪ | |
| 8 | `foundations/component-composition.md` | ⚪ | Composition-beats-memoization groundwork for the performance track |
| 9 | `effects/effects-and-synchronization.md` | ⚪ | Third deliverable — cleanup, StrictMode double-invoke, AbortController pattern locked here. Opus tier |
| 10 | `foundations/rules-of-react.md` | ⚪ | Framed as the contract the Compiler enforces |

### Wave 2 — Intermediate

| # | Article | Status | Notes |
| --- | --- | --- | --- |
| 11 | `rendering/how-react-renders.md` | ⚪ | Fiber, render/commit — the "under the hood" anchor. Opus tier |
| 12 | `effects/useref-and-the-dom.md` | ⚪ | ref-as-prop baseline; legacy `forwardRef` with marker |
| 13 | `state/context.md` | ⚪ | What it's for / not for; split-context pattern |
| 14 | `state/usereducer-and-state-structure.md` | ⚪ | Derive-don't-mirror lives here |
| 15 | `effects/custom-hooks.md` | ⚪ | |
| 16 | `rendering/memoization-and-the-compiler.md` | ⚪ | Compiler-first stance. Opus tier |
| 17 | `rendering/error-boundaries.md` | ⚪ | The one class use-case; `react-error-boundary` in practice |
| 18 | `rendering/portals-and-the-event-system.md` | ⚪ | |
| 19 | `effects/escape-hatches-audit.md` | ⚪ | `useSyncExternalStore`, `useLayoutEffect`, `flushSync` |

---

## Phase 1b — recipes (⚪ gated on Wave 1)

Starts once Wave 1 articles are complete, then runs in parallel with Waves 2–4. Track order per `roadmap.md` §4.

### First 3 recipes (locked)

| # | Recipe | Status | Scenario |
| --- | --- | --- | --- |
| 1 | `recipes/data-fetching/search-race-condition.md` | ⚪ | Stale "rea" response lands after "react" response and overwrites it. Fix arc: ignore-flag → AbortController → TanStack Query. The foundational recipe — heaviest expected in-degree |
| 2 | `recipes/performance/typing-lag-rerender-storm.md` | ⚪ | Controlled input in a 300-row dashboard; 180ms per keystroke. Fix arc: colocation → composition → Compiler verification. Deliberately not "sprinkle useMemo" |
| 3 | `recipes/forms-and-ux/double-submit-and-optimistic-like.md` | ⚪ | Two orders from a double-click; Like count flickers back on failure. Fix arc: form actions + `useActionState` → `useOptimistic` with rollback |

### Queued recipe candidates by track

| Track | Candidates |
| --- | --- |
| `data-fetching/` | Duplicate requests on StrictMode double-mount; request waterfall from nested components; spinner never resolves after unmount |
| `performance/` | 900KB initial bundle (route splitting); 6s freeze on tab switch (virtualization); LCP regression from client-only rendering |
| `forms-and-ux/` | Validation fires on first keystroke; debounced search with loading states; multi-step wizard state survival |
| `state-management/` | Server data cached in Zustand goes stale; context re-renders the whole tree; prop drilling 6 levels deep |
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
| 20 | `concurrent/concurrent-rendering.md` | ⚪ | Transitions, `useDeferredValue`, tearing. Opus tier |
| 21 | `concurrent/suspense.md` | ⚪ | |
| 22 | `concurrent/use-and-promises.md` | ⚪ | |
| 23 | `concurrent/actions.md` | ⚪ | The 19 mutation story; pairs with the double-submit recipe |
| 24 | `rendering/react-compiler-deep-dive.md` | ⚪ | Includes the Vite 8 / `@rolldown/plugin-babel` wiring gotcha. Opus tier |
| 25 | `server/ssr-and-hydration.md` | ⚪ | Framework-agnostic fundamentals. Opus tier |
| 26 | `server/server-components.md` | ⚪ | The RSC model. Opus tier |

### Wave 4 — Ecosystem

| # | Article | Status | Notes |
| --- | --- | --- | --- |
| 27 | `ecosystem/data-fetching-tanstack-query.md` | ⚪ | Gates the data-fetching recipe track's library stage |
| 28 | `ecosystem/routing-react-router.md` | ⚪ | Data mode: loaders, actions, deferred data |
| 29 | `ecosystem/state-management-landscape.md` | ⚪ | Decision table, not a tour |
| 30 | `forms/forms-at-scale.md` | ⚪ | RHF + Zod; where form actions end and RHF begins |
| 31 | `ecosystem/testing.md` | ⚪ | Vitest + RTL + user-event + MSW; gates the testing recipe track |
| 32 | `ecosystem/styling-approaches.md` | ⚪ | Decision table |
| 33 | `server/nextjs-and-rsc-in-practice.md` | ⚪ | Builds on 25–26; gates the ssr-and-rsc recipe track |
| 34 | `ecosystem/performance-profiling.md` | ⚪ | DevTools Profiler workflow; hub for the performance recipe track |
| 35 | `ecosystem/tanstack-router.md` | ⚪ | Flexible order (33–36) |
| 36 | `ecosystem/accessibility-in-react.md` | ⚪ | Flexible order (33–36) |

---

## Phase 2 — bridge layer (❌ deferred)

Additive bridge articles/annotations for Angular developers. Explicitly out of scope until Phase 1 matures. Zero rewrites of Phase 1 content; Phase 1 articles and recipes contain no Angular references anywhere.

---

## Locked editorial conventions

### Article structure (concept articles)

1. Frontmatter (`article_id`, `related`, `react_baseline: "19.2"`)
2. Lead-with-this callout (one-paragraph hook)
3. What it is
4. How it works under the hood (mechanism explanation)
5. Basic usage
6. Real-world patterns
7. Common mistakes
8. How this evolved (when applicable — anchored to the class → hooks → concurrent → 19 Actions → Compiler arc)
9. See also

Old-vs-new dual examples appear only where React itself has legacy forms: `forwardRef` vs ref-as-prop, class error boundaries, pre-Actions form handling, manual memoization vs Compiler.

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