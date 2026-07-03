# react-concepts

A modern, opinionated React learning resource — original English-language content built alongside the [nxhhuy.tech](https://nxhhuy.tech) roadmap.

**Targets React 19.2** with modern idioms throughout: function components + hooks only, Actions (`useActionState` / `useOptimistic` / form actions), `use()`, ref-as-prop, React Compiler (auto-memoization by default), strict TypeScript, Vite 8, TanStack Query for server state, React Router v7 (data mode), Vitest + Testing Library.

## Where to start

- **New to React** → `foundations/thinking-in-react.md`, then follow the reading order in [`roadmap.md`](roadmap.md)
- **Existing React dev** → pick a concept article (`effects-and-synchronization`, `memoization-and-the-compiler`, `actions`) or jump straight to a recipe that matches a real-world problem you're hitting
- **Looking for a specific bug fix** → `recipes/` is organized by problem domain; the recipes lead with the symptom

## Structure

> Greenfield project — the tree below is the target layout. See [`progress.md`](progress.md) for what has actually landed.

```
docs/
├── roadmap.md                 versions, priorities, conventions — the planning source of truth
├── progress.md                maintainer-facing status tracking
│
├── foundations/               thinking in React, JSX & rendering, components & props,
│                              conditional rendering & events, composition, rules of React
├── state/                     useState & snapshots, useReducer & state structure, context
├── effects/                   effects & synchronization, refs & the DOM, custom hooks,
│                              escape hatches (useSyncExternalStore, useLayoutEffect, flushSync)
├── rendering/                 lists & keys, how React renders, memoization & the Compiler,
│                              error boundaries, portals, React Compiler deep dive
├── forms/                     controlled & uncontrolled, forms at scale (RHF + Zod)
├── concurrent/                transitions & deferred values, Suspense, use(), Actions
├── server/                    SSR & hydration, Server Components, Next.js in practice
├── ecosystem/                 TanStack Query, React Router, state-management landscape,
│                              styling, testing, performance profiling, TanStack Router, a11y
│
└── recipes/                   problem-solving — concrete bugs, concrete fixes
    ├── data-fetching/         race conditions, deduplication, caching, abort — the foundational track
    ├── performance/           re-render storms, bundle size, LCP, virtualization
    ├── forms-and-ux/          double-submit, optimistic UI, validation timing, debounced search
    ├── state-management/      stale server data in stores, prop drilling, context re-renders
    ├── auth/                  token refresh, protected routes, session expiry
    ├── routing/               deep links, scroll restoration, route-level code splitting
    ├── testing/               flaky async tests, act() warnings, MSW patterns
    ├── real-time/             WebSocket/SSE lifecycle, StrictMode double-connect, reconnection
    ├── ssr-and-rsc/           hydration mismatches, server/client boundary, server actions
    └── pwa-offline/           stale cached shells, offline mutation queues
```

## Recipes index — quick lookup by symptom

The index below grows as recipes land. First three in the pipeline:

| Problem | Recipe |
| --- | --- |
| "User types fast — stale search results overwrite the fresh ones" | `data-fetching/search-race-condition` *(planned)* |
| "Typing in one input lags — every keystroke re-renders a 300-row table" | `performance/typing-lag-rerender-storm` *(planned)* |
| "User clicks Buy, nothing happens for 800ms, clicks again — two orders" | `forms-and-ux/double-submit-and-optimistic-like` *(planned)* |

## Phases

| Phase | Status | Description |
| --- | --- | --- |
| Roadmap | 🟢 Drafted | Versions, meta-framework stance, priorities, locked conventions |
| Phase 1a | ⚪ Queued | Foundation concept articles (Waves 1–2, 19 articles) |
| Phase 1b | ⚪ Queued | Recipes — starts once Wave 1 is complete, then runs in parallel |
| Phase 1c | ⚪ Queued | Advanced + ecosystem articles (Waves 3–4, 17 articles) |
| Phase 2 | ⚪ Deferred | Bridge layer for developers arriving from other frameworks — additive only |

See [`progress.md`](progress.md) for detailed status.

## License

MIT — see [`LICENSE`](LICENSE).