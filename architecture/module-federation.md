---
article_id: module-federation
concept_folder: architecture
wave: 5
related:
  - architecture/micro-frontends
  - rendering/error-boundaries
  - ecosystem/state-management-landscape
  - rendering/react-compiler-deep-dive
  - server/ssr-and-hydration
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Module Federation in practice

> **Lead with this:** this article assumes you've already made the decision — [`micro-frontends`](./micro-frontends.md#the-decision) is where you decide *whether* to federate; this is *how*. Module Federation lets a host load code from an independently-deployed remote at runtime, and — the part that actually matters — share a single copy of React across both. We'll build a host and a remote on Vite 8, then trace what the runtime is really doing, because the two substrates underneath (a bundler runtime with a *share scope* vs. native ES modules with an *import map*) explain every config option and every failure mode.

## What it is

Module Federation is **runtime code sharing between separately built, separately deployed applications.** A *host* (the shell) declares *remotes* it will load at runtime; each *remote* declares what it `exposes`; both declare what they `shared`, so common dependencies load once and deduplicate to a singleton instead of shipping N copies.

The version that matters in 2026 is **Module Federation 2.0** — no longer a webpack plugin but a **framework-agnostic runtime** (`@module-federation/runtime`) plus a **manifest** (`mf-manifest.json`) describing each remote's exposes and shared surface. It runs on Rspack and webpack natively, and on Vite/Rolldown through `@module-federation/vite`. The mental model is unchanged from the webpack era; the substrate is not.

The whole game is the **shared singleton**. Two independently-built React apps, each with its own `import React from "react"`, must end up using the *same* React instance at runtime — because React's hooks rely on a single module-level dispatcher. Two Reacts on one page is the canonical federation bug: `Invalid hook call`. Everything Module Federation does is machinery to guarantee one React.

## How it works under the hood

There are two substrates that deliver "load a remote and share one React," and they dedupe in fundamentally different ways. Understanding both is the difference between configuring federation and debugging it.

### Substrate A — a bundler runtime with a share scope

The classic model (webpack 5, and what `@module-federation/runtime` re-implements portably). Every federated app carries a small **runtime** and participates in a global **share scope** — a registry mapping a package name to every registered version of it.

The lifecycle:

1. **Build.** The federation plugin emits `remoteEntry.js` (an entry that exposes the container's `get`/`init` functions) and, in MF 2.0, an `mf-manifest.json` describing exposes + shared + versions. Shared dependencies are split into their own chunks.
2. **Host init.** On startup the host's runtime registers its own shared packages into the share scope: `"react" → { "19.2.0": <the host's React module> }`.
3. **Route activation.** When the host loads `checkout/Widget`, the runtime fetches the remote's manifest, calls the remote's `init(shareScope)` so the remote can see what's already registered, then `get('./Widget')` to retrieve the exposed module factory.
4. **Dedup via `loadShare`.** When the remote's code does `import React from "react"`, the runtime intercepts it, looks in the share scope, finds the host's already-registered `react@19.2.0` that satisfies the remote's `requiredVersion`, and hands back **that** module — not a fresh copy. One React, by interception.

The sharp edge this creates: **shared modules must be registered before anything consumes them.** If the host synchronously imports React during the same task that's still registering the share scope, you get the historical *"Shared module is not available for eager consumption"* error. The fix is an **async boundary**: the app's real entry sits behind a dynamic `import()`, so registration completes before consumption. The Vite plugin injects this init for you (into `index.html`, or into the entry for SSR hosts without an `index.html`), so you rarely write the split by hand — but the ordering constraint is why it exists.

### Substrate B — native ES modules with an import map

Where the ecosystem is heading (native ESM federation; Rolldown-native Module Federation is landing). No userland runtime re-implements module loading — the browser does it.

1. **Build.** Each app emits normal ESM chunks plus a static manifest. Shared packages are pre-built as standalone ESM.
2. **Startup.** An init step fetches the manifests, negotiates one version per shared package, and installs a single **import map** — `"react" → https://host/assets/react-<hash>.mjs` — via `<script type="importmap">`.
3. **Everything resolves through the map.** When *either* the host or the remote runs `import React from "react"`, the browser's native loader resolves the bare specifier `"react"` through the import map to **one URL**. The ES module spec keeps exactly one module record per URL — so both sides get the same instance *automatically*. Dedup isn't intercepted; it falls out of module identity.

This substrate has its own ordering constraint for the same reason, differently caused: **an import map must be installed before the first module import resolves through it.** So the init that fetches manifests and injects the map must complete before any framework import runs — the same async boundary, a different mechanism.

### Why the substrate matters to you

- **Config options map to a substrate.** `shared: { react: { singleton: true, strictVersion: true, requiredVersion: "19.2.0" } }` is share-scope negotiation in substrate A and version selection for the import map in substrate B. Same declaration, different enforcement.
- **`Invalid hook call` is always a dedup failure.** Either the share scope didn't match (A) or two import-map entries / a non-shared React (B). The fix is always "make React a strict singleton," never "wrap it in something."
- **On Vite 8 / Rolldown today**, `@module-federation/vite` gives you substrate A's runtime; Rolldown-native federation is moving toward substrate B. The manifest and the `shared` declaration are the stable surface across both — write to those.

## Basic usage

A remote exposing one component, and a host consuming it. Vite 8, `@module-federation/vite`.

```ts
// checkout/vite.config.ts (the remote)
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { federation } from "@module-federation/vite";

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: "checkout",
      filename: "remoteEntry.js",
      manifest: true, // emit mf-manifest.json
      exposes: { "./Widget": "./src/Widget.tsx" },
      shared: ["react", "react-dom"],
    }),
  ],
  // the remote must know its own public origin so its chunk URLs are absolute
  server: { origin: "http://localhost:5001" },
});
```

```ts
// shell/vite.config.ts (the host)
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { federation } from "@module-federation/vite";

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: "shell",
      remotes: {
        checkout: {
          type: "module",
          name: "checkout",
          entry: "http://localhost:5001/mf-manifest.json",
        },
      },
      shared: ["react", "react-dom"],
    }),
  ],
});
```

```tsx
// shell/src/App.tsx — consume the remote, lazily, isolated
import { lazy, Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";

const Checkout = lazy(() => import("checkout/Widget"));

export function App() {
  return (
    <ErrorBoundary fallback={<p>Checkout is unavailable right now.</p>}>
      <Suspense fallback={<p>Loading checkout…</p>}>
        <Checkout />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Three things are load-bearing already: `shared: ["react", "react-dom"]` on **both** sides (one React), the `import("checkout/Widget")` behind `lazy` (an async boundary, and it only fetches the remote when rendered), and the `ErrorBoundary` (a remote *will* fail — [more below](#step-4--failure-isolation-and-resilient-loading)).

## Walkthrough — a shell that loads a checkout remote, typed and isolated

We'll build both apps end to end: the remote exposes a typed widget, the host consumes it with types, and the host survives the remote failing.

### Step 1 — The remote exposes a component and its types

```tsx
// checkout/src/Widget.tsx
import { useState } from "react";

export interface CheckoutWidgetProps {
  cartId: string;
  onComplete: (orderId: string) => void;
}

export default function CheckoutWidget({ cartId, onComplete }: CheckoutWidgetProps) {
  const [pending, setPending] = useState(false);
  // ... checkout UI; owned and deployed by the checkout team
  return <button disabled={pending} onClick={() => setPending(true)}>Pay</button>;
}
```

Turn on **type sharing** so the host gets `CheckoutWidgetProps` for free — the TS-safety story that makes federation tolerable at scale:

```ts
// checkout/vite.config.ts (add to the federation() call)
federation({
  name: "checkout",
  filename: "remoteEntry.js",
  manifest: true,
  exposes: { "./Widget": "./src/Widget.tsx" },
  shared: ["react", "react-dom"],
  dts: { generateTypes: true }, // emits @mf-types the host can consume
});
```

`vite build` now emits the ESM chunks, `mf-manifest.json`, and a `@mf-types` bundle describing the exposed surface.

### Step 2 — The host consumes it with real types

Point the host's `dts` at the remote so `import("checkout/Widget")` is typed, and add a tsconfig path so the editor resolves the generated declarations:

```ts
// shell/vite.config.ts (federation() call)
federation({
  name: "shell",
  remotes: {
    checkout: { type: "module", name: "checkout", entry: "http://localhost:5001/mf-manifest.json" },
  },
  shared: ["react", "react-dom"],
  dts: { consumeTypes: true },
});
```

```jsonc
// shell/tsconfig.json
{ "compilerOptions": { "paths": { "checkout/*": ["./@mf-types/checkout/*"] } } }
```

Now the consuming code is fully typed — no hand-written `declare module` shims:

```tsx
// shell/src/routes/Checkout.tsx
import { lazy, Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";

// typed: CheckoutWidgetProps flows from the remote's @mf-types
const CheckoutWidget = lazy(() => import("checkout/Widget"));

export function CheckoutRoute({ cartId }: { cartId: string }) {
  return (
    <ErrorBoundary fallback={<CheckoutUnavailable />}>
      <Suspense fallback={<CheckoutSkeleton />}>
        <CheckoutWidget cartId={cartId} onComplete={(id) => navigate(`/orders/${id}`)} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### Step 3 — Prove the singleton

Run both (`checkout` on 5001, `shell` on 5000). In the shell, log `React` identity from a host component and from inside the remote widget — same object. Flip the remote's config to `shared: []` and reload: you'll get `Invalid hook call`, because the remote now bundled and ran its *own* React and its hooks hit a different dispatcher than the host's. That error is the singleton contract failing, and the fix is never a wrapper — it's restoring `shared` with `singleton: true`. This is exactly [the version-skew discipline from the decision article](./micro-frontends.md#shared-dependency-governance-and-version-skew), made concrete.

### Step 4 — Failure isolation and resilient loading

The remote is a separate deployment over the network. It *will* 404, time out, or ship a runtime error. The `ErrorBoundary` already degrades a broken remote to a fallback instead of blanking the shell — the same mechanism as a [failed lazy chunk](../rendering/error-boundaries.md#the-stale-chunk-fallback), because a remote entry that won't load *is* a failed chunk. Place the boundary [per remote, at the mount](../rendering/error-boundaries.md#placement-the-granularity-ladder), so one dead remote never takes a sibling down.

For dynamic remotes and real resilience, drop to the runtime API instead of the static `import()`:

```tsx
import { loadRemote, registerRemotes } from "@module-federation/runtime";

// URLs discovered at runtime (e.g. from an environment manifest), not baked in
registerRemotes([
  { name: "checkout", entry: "https://checkout.acme.com/mf-manifest.json" },
]);

async function loadCheckout() {
  // wrap with your own timeout / retry / fallback-URL / circuit-breaker policy
  const mod = await loadRemote<{ default: React.ComponentType<CheckoutWidgetProps> }>(
    "checkout/Widget",
  );
  return mod!.default;
}
```

`registerRemotes` lets you resolve remote URLs from a runtime manifest, so one host build ships to every environment and swaps remotes by swapping the manifest — no rebuild. Wrap `loadRemote` in the resilience policy the decision article calls for (timeout, retry-with-jitter, fallback URL, circuit breaker); the runtime doesn't add those for you.

### Step 5 — Wire the Vite plugin correctly

Two gotchas that cost an afternoon each:

- **Do not set `manualChunks`** (`build.rollupOptions.output.manualChunks` / `rolldownOptions`). The federation plugin owns the chunk graph so the runtime init and `loadShare` stay isolated; forcing custom chunking breaks the bootstrap order. The plugin ignores it, but don't fight it.
- **A remote must be *built* to be consumed.** Vite dev is bundleless, so the host's dev server can consume a remote, but the remote itself needs `vite build` (use `--watch` for HMR-like iteration). Only the host side has a first-class dev experience — the same asymmetry as the Vite compiler wiring in [the compiler deep dive](../rendering/react-compiler-deep-dive.md#the-vite-8-wiring-and-the-gotcha): the plugin order and build model are load-bearing, and most tutorials show a stale form.

## Real-world patterns

### The `shared` contract, in depth

`shared` is where federation is won or lost. The defaults you almost always want for the framework:

```ts
shared: {
  react: { singleton: true, strictVersion: true, requiredVersion: "19.2.0" },
  "react-dom": { singleton: true, strictVersion: true, requiredVersion: "19.2.0" },
}
```

- **`singleton: true`** — exactly one instance on the page. Mandatory for React, React-DOM, the router, and any library with module-level state. Without it you get multiple copies and, for React, `Invalid hook call`.
- **`strictVersion: true`** — refuse (loudly) rather than silently run a mismatched version. Pair it with a real policy: the host rejects an incompatible remote and renders a fallback, instead of limping along with a subtly-wrong React.
- **`requiredVersion`** — pin the shared kernel's versions explicitly; `"auto"` reads `package.json`. This is the concrete form of the shared-kernel contract from [the decision article](./micro-frontends.md#the-decision).
- **`eager: true`** — bundles the shared dep into the initial chunk instead of loading it async. Occasionally needed for the host's own baseline, but it defeats the async boundary if misused — a common cause of eager-consumption errors. Default to `false`.

### Dynamic remotes and per-environment manifests

Static `remotes` in the config bake URLs into the build. `registerRemotes` at runtime reads them from a manifest fetched at startup, so staging and production run the *same* host build pointed at different manifests. This is how you get environment independence and canary-by-swapping-a-URL without rebuilding the shell.

### Cross-remote state stays a contract, not a store

The runtime lets a remote reach anything the host exposes — which makes it tempting to `shared` a global store across remotes. Don't. As [the decision article's anti-pattern](./micro-frontends.md#cross-mfe-state-is-the-anti-pattern) spells out, a cross-remote mutable store re-couples independently-deployed apps and re-creates every problem the [placement funnel](../ecosystem/state-management-landscape.md#the-placement-funnel) prevents — across a boundary you can't refactor atomically. Communicate with events or a small read-mostly context from the shared kernel; keep each remote's server state in its own cache.

### Server-side Module Federation

Client-side federation assumes a browser; SSR needs the init to complete before `hydrateRoot`, which is why the Vite plugin can inject init into the entry rather than `index.html` for SSR hosts (Nitro, TanStack Start). Beyond that, stitching server-rendered remotes is genuinely harder, and the 2026 answer for teams that need both SSR and composition is to move the boundary to the server — [Server Components across origins and a meta-framework's streaming](../server/ssr-and-hydration.md#real-world-patterns) — rather than forcing client MF through the server. If SSR is a hard requirement, revisit [the decision](./micro-frontends.md#the-decision) before committing to client-side federation.

## API / type reference

| Surface | Key options | Notes |
| --- | --- | --- |
| `federation()` (remote) | `name`, `filename`, `manifest`, `exposes`, `shared`, `dts.generateTypes` | `exposes` maps a public key (`"./Widget"`) to a source path |
| `federation()` (host) | `name`, `remotes`, `shared`, `dts.consumeTypes` | `remotes` entries can be a URL or `{ type, name, entry }` pointing at `mf-manifest.json` |
| `shared` sub-options | `singleton`, `strictVersion`, `requiredVersion`, `eager` | `singleton + strictVersion` is the standard framework combo |
| Runtime API | `init`, `registerRemotes`, `loadRemote`, `loadShare` | For dynamic remotes and hand-rolled resilience |
| Build artifacts | `remoteEntry.js`, `mf-manifest.json`, `@mf-types` | Manifest + types are the inspectable, stable surface |

## Common mistakes

1. **Not sharing React as a singleton.** The `Invalid hook call` classic — two Reacts, two dispatchers. `shared` React and React-DOM as `singleton: true` on both sides.
2. **Setting `manualChunks`.** Breaks the federation runtime's chunk graph and bootstrap order. Let the plugin own chunking.
3. **Importing a remote statically at the top level.** A synchronous `import checkout from "checkout/Widget"` defeats the async boundary and risks eager-consumption failures. Load remotes behind `lazy`/dynamic `import()` or the runtime API.
4. **`eager: true` on shared deps by reflex.** It bundles the dep into the initial chunk and can break the async-registration ordering. Default to `false`; use it only for a deliberate host baseline.
5. **Wrong remote `origin`/`publicPath`.** The remote's chunk URLs must be absolute against its real deployed origin, or the host fetches assets from *its own* origin and 404s. Set `server.origin` (dev) and the deploy base (prod).
6. **No error boundary around the remote.** One remote's failure blanks the whole shell. Wrap each remote mount in an `ErrorBoundary` with a real fallback.
7. **Version skew without a policy.** Letting remotes drift React majors and hoping the singleton negotiates. Pin the shared kernel, `strictVersion: true`, reject incompatible remotes at load.
8. **Expecting the host dev server to serve an unbuilt remote.** Remotes must be built (`vite build`, `--watch` for iteration); only the host gets a bundleless dev server.
9. **Baking remote URLs into the build for every environment.** Use `registerRemotes` + a per-environment manifest so one build serves all environments.
10. **Sharing a mutable store across remotes.** The coupling that undoes the whole architecture. Contracts and events, not a shared store.

## How this evolved

- **Webpack 5 Module Federation (2020).** The original: a webpack-runtime container, `remoteEntry.js` as an executable IIFE, share scope for dedup, and the eager-consumption error that made the bootstrap split a rite of passage. Powerful, but coupled to webpack and heavy per app.
- **Module Federation 2.0.** The runtime was extracted into a **framework-agnostic package** (`@module-federation/runtime`) and paired with a **manifest** (`mf-manifest.json`), first-class **type sharing** (`@mf-types`), and adapters for Rspack, webpack, and Vite. Federation became a runtime concern with real governance tooling rather than a bundler trick.
- **Native ESM federation / import maps.** Dedup rebuilt on browser standards — one specifier, one URL, one module record — so no userland runtime is needed. Rolldown-native Module Federation moves Vite toward this substrate.
- **Server-side Module Federation (2025).** Federation extended to the server so remotes can participate in SSR and RSC-across-origins, shifting composition out of the client bundle.

The `shared` declaration and the manifest are the through-line; write to those and the substrate can change underneath you.

## Exercises

1. **Expose and consume, typed.** Add a second exposed component to the checkout remote (`"./MiniCart"`), enable `dts` on both sides, and consume it in the host with full types and no `declare module` shim. Confirm the props error if you pass the wrong shape. (Hint: `exposes` key + `tsconfig` path.)
2. **Break it, then isolate it.** Point the host at a remote URL that 404s. Show the shell blanking without a boundary, then wrap the mount so it degrades to a fallback, then add a timeout+fallback around `loadRemote`. (Hint: the boundary is the [stale-chunk fallback](../rendering/error-boundaries.md#the-stale-chunk-fallback) pattern.)
3. **Find the second React.** Set `shared: []` on the remote, reproduce `Invalid hook call`, then use the runtime/share-scope (or the import map, on a native-ESM build) to explain *which* React each side resolved and why the fix is `singleton: true`, not a wrapper. (Hint: log `React` identity on both sides.)

## Summary

Module Federation shares code between independently-deployed apps at runtime, and its entire job is guaranteeing one shared React across the boundary. On Vite 8 you wire it with `@module-federation/vite`: a remote `exposes` components and emits an `mf-manifest.json` (plus `@mf-types` for type sharing), a host declares `remotes` and consumes them behind `lazy`/dynamic `import()` or the runtime API, and both declare `shared` with `singleton: true` so React deduplicates. Underneath, one of two substrates does the dedup — a bundler runtime negotiating a share scope, or native ES modules resolving one URL through an import map — and both impose the same async-boundary ordering for opposite reasons. Wrap every remote in an error boundary, govern the shared kernel's versions with `strictVersion`, keep cross-remote state in contracts not stores, and don't fight the plugin's chunk graph. When you need to decide whether any of this is worth it, that's [`micro-frontends`](./micro-frontends.md#the-decision).

## See also

- [`micro-frontends`](./micro-frontends.md#the-decision) — whether to federate at all, the composition-style menu, and the shared-kernel/governance/isolation discipline this article implements.
- [`error-boundaries`](../rendering/error-boundaries.md#the-stale-chunk-fallback) — degrading a failed remote (a failed chunk) to a fallback instead of a blank shell.
- [`state-management-landscape`](../ecosystem/state-management-landscape.md#the-placement-funnel) — why cross-remote state is a contract, not a shared store.
- [`react-compiler-deep-dive`](../rendering/react-compiler-deep-dive.md#the-vite-8-wiring-and-the-gotcha) — the Vite 8 plugin-wiring model this federation plugin sits alongside.
- [`ssr-and-hydration`](../server/ssr-and-hydration.md#real-world-patterns) — the server boundary and why SSR pushes composition server-side.

## References

- Module Federation documentation — the runtime, the `shared`/singleton model, and `mf-manifest.json`.
- `@module-federation/vite` — the Vite/Rolldown plugin, config surface, and the `manualChunks` caveat. Verify the exact version and Rolldown-native status against the npm registry before pinning.
- Import Maps — MDN (the native-ESM substrate).
- `react-error-boundary` — the boundary component used around remote mounts.

## Demo source

- `demos/architecture/module-federation/` — the shell + checkout remote from the walkthrough: typed exposes, the singleton proof, and the failing-remote degradation. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*