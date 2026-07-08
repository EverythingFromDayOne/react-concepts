---
recipe_id: shared-react-duplicated
track: micro-frontends
primary_concept: architecture/module-federation
difficulty: advanced
react_baseline: "19.2"
related:
  - architecture/module-federation
  - architecture/micro-frontends
  - rendering/error-boundaries
  - foundations/rules-of-react
status:
  drafted: true
  reviewed: false
---

# The remote crashes with "Invalid hook call" — but only inside the shell

> **What you'll build:** a Module Federation setup where the host and every remote share exactly one React, so a remote that works standalone doesn't explode with `Invalid hook call` the moment it mounts in the shell — and stays fixed through routine version bumps, because React is a *strict singleton* rather than a coincidence of aligned versions.

## The scenario

A shell loads a `checkout` remote. Checkout is flawless on its own dev server (`localhost:5001`) — it renders, hooks work, its tests are green — and it has been running composed inside the shell for months. Then the checkout team does a routine `react@19.2.0 → 19.2.1` patch bump and deploys. Now every user who opens `/checkout` in the shell gets a blank panel and a console full of:

> Invalid hook call. Hooks can only be called inside the body of a function component.

The message helpfully lists three possible causes — mismatched React versions, breaking the rules of hooks, or more than one copy of React — and the team burns an afternoon on the first two. It is the third.

Checkout's federation config shares `react`/`react-dom` but **without `singleton: true`**. While both sides sat on 19.2.0 the runtime happened to load one copy and everything worked. After the bump, checkout wants 19.2.1 and the host still offers 19.2.0; with no singleton constraint the runtime lets each side keep the version it's satisfied with — **two React instances on one page.** React's hooks dispatch through a single module-level object; the remote's `useState` reaches into the remote's React, whose dispatcher isn't the one the host installed while rendering. It's null, so the call is "invalid."

Why it escaped QA:

- The remote works **standalone** — one React, one dispatcher — so its unit tests, Storybook, and dev server all pass.
- The bug is **composition-only** — it needs the remote mounted *inside* the host, which the remote's own QA never does.
- It was **latent** — the shared-but-not-singleton config had been wrong for months, masked by matching versions. A patch bump, not a code change, tripped it. "We didn't change anything" is technically true and totally misleading.
- It hits every user on the checkout route after the deploy, while the remote's own pipeline stays green the entire time.

## Walkthrough

### Stage 1 — Name it: two Reacts, one dispatcher

The tell is in the symptom: it works standalone and breaks composed, so *composition* introduced a second React. Confirm it directly — the same `react` import, logged from both sides, is a different object when there are two:

```tsx
// drop into a host component AND the remote widget, then compare the console
import * as ReactNS from "react";
console.log("react:", ReactNS.version, ReactNS); // two different object refs = two Reacts
```

You can also read the Module Federation shared graph (the MF manifest / devtools list the resolved shared versions) — more than one `react` entry is the same finding.

The mechanism is why this is fatal rather than cosmetic. Hooks have no instance identity; a bare `useState` finds its state through the *currently-installed dispatcher*, a module-level singleton inside React. Two React modules mean two dispatchers: the host renders with its dispatcher installed, and the remote's `useState` calls into the remote's React, whose dispatcher is null mid-render. This is **not** a [rules-of-hooks](../../foundations/rules-of-react.md#how-it-works-under-the-hood) violation — the hook code is correct; the runtime is doubled. (The dedup machinery that's supposed to prevent this is traced in [article 38's two substrates](../../architecture/module-federation.md#how-it-works-under-the-hood).)

### Stage 2 — Reject the fixes the error suggests

- **"Align React versions exactly everywhere."** Works by accident — one satisfying copy — and is brittle: the next patch bump re-breaks it, and you'd be pinning every remote's React in lockstep forever.
- **"Check the rules of hooks."** A dead end; the hook code is fine.
- **`resolve.dedupe: ['react']` / a bundler alias.** Dedupes within *one* build. Federation composes *separate* builds, so this can't see the remote's copy. Wrong layer.

The durable lever is the `shared` singleton contract, not version lockstep.

### Stage 3 — Make React a strict singleton on both sides

```ts
// on BOTH the host and every remote's federation() config
shared: {
  react: { singleton: true, requiredVersion: "^19.2.0" },
  "react-dom": { singleton: true, requiredVersion: "^19.2.0" },
}
```

`singleton: true` means exactly one instance on the page even when versions differ slightly: the runtime picks the highest version that satisfies everyone, routes both sides to it, and warns rather than forking on a mismatch. The 19.2.1 bump no longer splits React — the singleton constraint forces one copy. (Full `shared` semantics, including `strictVersion` and `eager`, are in [article 38](../../architecture/module-federation.md#the-shared-contract-in-depth).)

### Stage 4 — Share every runtime-stateful library, and guard the build

React isn't the only thing that breaks when doubled. Any library with module-level state or React context fails the same way: **react-dom**, the router, `react-error-boundary`, a context-providing design-system library, a shared store. Sharing `react` but not `react-dom` (or not the router) is the most common *partial* fix — it relocates the crash instead of removing it. Put the whole runtime kernel in `shared` as singletons.

Then decide the mismatch policy and guard against regression:

- **`strictVersion`** — reject an incompatible remote and render a fallback (fail loud) versus warn-and-proceed (fail soft). Choose deliberately: a rejected remote is a *different* outage, so pair strict rejection with an [error boundary + fallback](../../architecture/module-federation.md#basic-usage) so the shell degrades rather than blanks.
- **Guard in CI.** The shared graph in the MF manifest reports the resolved shared versions; fail the build if it shows more than one React. The latent config that caused this was invisible for months — a build assertion makes the next regression loud.

**Verify the loop.** Re-run the composed **production** build (`build && preview`, not just dev), bump checkout's React patch again, and open `/checkout`: the shared graph shows one React, the widget mounts, no `Invalid hook call`. The config is now correct *and* bump-proof.

## Variations

1. **It's not just React.** The router, `react-error-boundary`, a design-system provider, or a shared store each need `singleton: true` for the same reason — module-level state or context that must be one instance.
2. **Transitive React from a shared component library.** A remote's dependency bundled its own React (it didn't honor React as a peer dependency), so a second copy sneaks in through the lib. Fix: the library declares React a peer, and you share React across the federation.
3. **`eager: true` for the host baseline.** The host always needs React on first paint, so eager-loading its shared React can be right — but `eager` on a remote, or misused, double-loads and breaks the async boundary. Default `false`; reach for it deliberately.
4. **Dev-vs-prod divergence.** Vite dev is bundleless and can dedup differently than the built output. Always test the built-and-previewed *composition*, not just dev, or a prod-only double-load slips through.
5. **Native-ESM / import-map substrate.** On a native-ESM build, "two Reacts" means two import-map entries or a React that isn't mapped; the fix is one specifier → one URL ([substrate B](../../architecture/module-federation.md#how-it-works-under-the-hood)).

## Trade-offs and common pitfalls

1. **React not shared as a singleton** — the core bug. Two instances, two dispatchers, `Invalid hook call`.
2. **Chasing the error's first two suggestions** (version mismatch / rules of hooks) instead of the third (multiple copies). The message ranks the least-likely cause first.
3. **Sharing `react` but not `react-dom` / router / context libs** — a partial fix that just relocates the crash.
4. **Omitting `singleton`** — even with `shared`, a version drift forks React on the next bump. This is the latent trap that ships fine and breaks later.
5. **`strictVersion: true` with no fallback** — a mismatch hard-rejects the remote, which is a *different* outage. Pair strict rejection with an error boundary.
6. **Aligning versions by hand as "the fix"** — brittle; re-breaks on the next patch. Singleton is the durable fix.
7. **`resolve.dedupe` / alias as the fix** — single-build only; blind to federated remotes' copies.
8. **A shared component library bundling its own React** (peer-dependency violation) — a transitive second copy that no `shared: ['react']` on your side will catch until the lib is fixed.
9. **Testing only the standalone remote** — the composition-only bug escapes entirely.
10. **Testing only dev** — the production build double-loads (or, rarely, the reverse). Test the built composition.
11. **`eager: true` misused** — double-loads the shared dep or breaks async registration.
12. **No build-time guard** — a future dependency bump silently re-introduces a second React. Assert exactly one in CI.

### When NOT to force a shared singleton

If you're **not federating** — one build, a monorepo, a single Vite app — this machinery doesn't apply. A "two Reacts" crash there comes from a duplicated or mislinked `node_modules` (often an `npm link`ed local library) and is fixed by `resolve.dedupe` / deduping the install, **not** by Module Federation `shared`. And if a remote is **isolated by design** — an iframe, or a web component with its own Shadow DOM and its own React that never shares hooks or context with the host — then two Reacts is intentional and correct. The crash only happens when two Reacts are *expected to share one dispatcher*. The test: *are the host and remote meant to share React's runtime — hooks and context?* If yes, one strict singleton. If they're deliberately isolated, leave them separate; it isn't this bug.

## See also

- [`module-federation`](../../architecture/module-federation.md#the-shared-contract-in-depth) — the full `shared` singleton contract and the two substrates that deduplicate.
- [`micro-frontends`](../../architecture/micro-frontends.md#shared-dependency-governance-and-version-skew) — the version-governance policy this recipe enforces.
- [`error-boundaries`](../../rendering/error-boundaries.md#placement-the-granularity-ladder) — degrading a rejected or failed remote instead of blanking the shell.
- `micro-frontends/remote-failure-blanks-shell` — the sibling isolation bug *(planned)*.
- `micro-frontends/version-skew-breaks-host` — the sibling governance bug *(planned)*.

## References

- Module Federation — the `shared` / `singleton` model, `requiredVersion`, `strictVersion`.
- React — the "Invalid hook call" guidance (its three causes; the third is a duplicate React).
- `@module-federation/vite` — inspecting the resolved shared graph.

## Demo source

- `demos/micro-frontends/shared-react-duplicated/` — the version-bump repro: shared-without-singleton crashing composed, then the strict-singleton fix and the CI guard. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*