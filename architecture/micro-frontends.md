---
article_id: micro-frontends
concept_folder: architecture
wave: 5
related:
  - architecture/module-federation
  - server/nextjs-and-rsc-in-practice
  - ecosystem/state-management-landscape
  - rendering/error-boundaries
  - ecosystem/styling-approaches
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Micro-frontends: the decision

> **Lead with this:** a micro-frontend is an *organizational* boundary, not a technical upgrade. It buys one thing — independent deployment for independent teams — and charges for it in bundle duplication, version skew, and operational surface. The dominant industry lesson of the last few years is that **most teams that reach for micro-frontends do so too early and regret it.** So this article is about the decision, not the tooling: what problem federation actually solves, the cheaper things that solve the same problem, and the honest threshold below which you should not split. The *how* — Module Federation on Vite — lives in [`module-federation`](./module-federation.md).

## What it is

A micro-frontend architecture composes **independently deployable frontend applications, each owned by a different team, into a single product the user experiences as one app.** The defining word is *deployable*: the checkout team ships a new checkout without rebuilding or redeploying catalog, account, or the shell that hosts them.

What it is **not**, and every one of these confusions is a reason teams adopt it wrongly:

- **Not a performance optimization.** A naive micro-frontend ships *more* JavaScript than a well-built monolith, not less — each remote can duplicate framework and library code, and composition adds network round-trips. Federation can approach monolith performance with disciplined shared-dependency config, but it starts from behind.
- **Not a technical upgrade over a monolith.** A monolith is not primitive; it is *focused*. One build, one deploy, one place to debug, props and context for state — those are advantages, not embarrassments.
- **Not the same as microservices.** Microservices decompose backend *services*; micro-frontends decompose frontend *modules*. They share a principle (independently deployable units) but solve different problems on different layers. Having one does not oblige you to the other.
- **Not a code-organization tool.** If your goal is "cleaner boundaries inside one codebase," you want module boundaries (a monorepo, feature-sliced structure), not runtime remotes. Splitting the *deploy* is a much bigger hammer than splitting the *code*.

The one problem a micro-frontend genuinely solves: **many teams stepping on each other in one repo and one release train.** Team A breaks the build Friday at 4pm; Team B, whose work is perfect, cannot ship. Micro-frontends let Team B deploy while Team A panics in isolation. That is an *organizational* win — team autonomy and independent release cadence — and it is the only win you should buy this architecture for.

## How it works under the hood

There is no single "micro-frontend technology." There are **composition styles**, distinguished by *where* and *when* the separate apps get stitched together. The whole decision starts with choosing the right axis, so trace what each one actually does.

### Build-time composition (npm package / monorepo)

Each "remote" is published as a package; the host lists it as a dependency and bundles it in. Composition happens in the host's build.

```ts
import { CheckoutWidget } from "@acme/checkout"; // resolved at build time
```

You get **shared code and clean boundaries**. You do **not** get independent deployment — a checkout change means republishing the package and rebuilding the host. This is the honest answer far more often than teams admit: it delivers the modularity most people actually want, without any runtime machinery. **If you don't need independent *deploys*, stop here.**

### Run-time composition in the browser (Module Federation / import maps)

The host loads remote code *at runtime*. The host build knows only a logical name; the actual URL is resolved from a manifest when the route activates.

```ts
// the host never bundled checkout — it fetches it when /checkout is hit
const { CheckoutWidget } = await loadRemote("checkout/Widget");
```

This is the style people usually mean by "micro-frontends," and it's what [`module-federation`](./module-federation.md) implements. It buys **independent deployment plus a shared runtime**: the checkout team ships to their own URL, the host picks it up with no rebuild, and both sides share one copy of React through a negotiated singleton. Two runtime substrates deliver this — a bundler runtime that re-implements module loading and deduplicates through a *share scope*, and native ES modules that deduplicate through *import-map identity* (one specifier, one URL, one module record). The trend is toward the native-ESM substrate; article 38 traces both. What you're buying: deploy independence. What you're paying: a runtime composition layer and all its failure modes.

### Server-side and edge composition

The server stitches fragments from different origins into one response before it reaches the browser — classic server-side includes, edge stitching at a CDN worker, or, in the React world, **Server Components rendered across origins** and streamed into one HTML response. Composition happens on the server, so the client ships less orchestration JavaScript and the result is SSR-friendly by construction. But the boundaries now follow [Server Component rules](../server/server-components.md#real-world-patterns) — what serializes across the boundary, what stays server-only — and in practice this means a meta-framework ([Next.js](../server/nextjs-and-rsc-in-practice.md#real-world-patterns)), not a hand-rolled shell.

### Routing-layer composition (Multi-Zones / reverse proxy)

Separate apps live behind path prefixes — `/shop` is one deployed app, `/account` another — glued by a reverse proxy or a framework's zones feature. Navigating *between* zones is a full document navigation (a hard nav), not a client transition. You get **total isolation and independent deploys with almost no shared-runtime complexity** — no share scope, no version negotiation, no singleton config — at the cost of a hard navigation on the seams. For many "separate teams, separate sections" problems this is the cheapest thing that works, and it's routinely overlooked in favor of heavier runtime federation.

### Web components and iframes

Custom elements wrap a remote's UI in a framework-agnostic DOM element; iframes give the hardest isolation the platform offers. Both compose *through the DOM* rather than through the module system. Iframes solve isolation and solve almost nothing else well (navigation, sizing, shared auth, running host scripts all get awkward); web components are a reasonable interop layer when remotes genuinely use different frameworks.

**The takeaway of this section:** these styles are not ranked. They sit on different axes — build-time buys reuse without deploy independence; browser-runtime buys deploy independence with a shared runtime; server/edge buys deploy independence with SSR and less client JS; routing-layer buys isolation and independence at the price of hard navigations. The decision is *which axis your constraint lives on* — which is the next section.

## Minimal shapes

The three configurations you'd actually compare, stripped to their essence, so the differences are visible at a glance.

**Build-time (a shared package):**
```ts
// host/package.json → "dependencies": { "@acme/checkout": "^2.0.0" }
import { Checkout } from "@acme/checkout";
```

**Browser-runtime (Module Federation, host side — full version in article 38):**
```ts
federation({
  name: "host",
  remotes: { checkout: "https://checkout.acme.com/mf-manifest.json" },
  shared: ["react", "react-dom"], // one React across host + remotes
});
```

**Routing-layer (a reverse-proxy rewrite, no shared runtime at all):**
```nginx
location /checkout/ { proxy_pass https://checkout.acme.com/; }
location /          { proxy_pass https://shell.acme.com/; }
```

The first bundles the remote in. The second loads it at runtime and shares React. The third never shares anything and hard-navigates between apps. Everything downstream — how you handle a remote failing, version skew, shared state — is a consequence of which of these you picked.

## The decision

Run the constraint through this order. Stop at the first honest "yes."

1. **Is the real problem team autonomy — multiple teams blocked by one release train?** If *no*, you do not want micro-frontends. Slow builds, tangled code, and "we want cleaner boundaries" are not autonomy problems; they are solved more cheaply below.
2. **Is it slow builds or tangled code?** Fix the monolith first: a **modular monorepo with enforced boundaries** (feature-sliced structure, package boundaries, CI that fails on cross-boundary imports) usually recovers most of the speed and all of the clarity, and it makes a *later* split far safer. This is the highest-ROI move for the majority of teams that think they need micro-frontends.
3. **Do separate teams own separate, cleanly-bounded *sections* of the app, and is a hard navigation between sections acceptable?** Reach for **routing-layer composition (zones / reverse proxy)** — independent deploys, near-zero shared-runtime complexity.
4. **Do those teams need to compose at runtime *within* a shared shell — shared chrome, client-side transitions across boundaries, lazy remotes — and can you fund a shared-dependency governance policy?** Now, and only now, **runtime Module Federation** earns its complexity.
5. **Is server rendering and minimal client JS a hard requirement?** Prefer **server/edge composition** (a meta-framework's zones or Server-Component composition) over client-side federation.

The decision table:

| Your constraint | Reach for | Not |
| --- | --- | --- |
| Cleaner code, faster builds, no team-autonomy problem | Modular monorepo + enforced boundaries | Any runtime federation |
| Separate teams own separate sections; hard nav is fine | Routing-layer (zones / reverse proxy) | Module Federation (overkill) |
| Runtime composition in a shared shell; you can fund governance | Module Federation 2.0 | A monorepo (can't deploy independently) |
| SSR / minimal client JS is non-negotiable | Server / edge composition (meta-framework) | Client-side Module Federation |
| Truly foreign frameworks must coexist | Web components at the seam | Sharing a framework singleton |
| Third-party or hostile content must be isolated | iframe | Anything that shares a global scope |

The org-scale rule of thumb the industry converged on: **micro-frontends start paying off around four or more frontend teams / tens of engineers.** Below that, the operational tax outweighs the autonomy benefit, and a well-structured monolith wins on nearly every axis. "We have long build times" is a build problem; "we have five teams and a contended release train" is a micro-frontend problem. Only the second one is worth this architecture.

## Real-world patterns

### The shared kernel and the integration contract

Independent does not mean disconnected. Every serious micro-frontend defines a **shared kernel** — the small set of things all remotes *must* agree on: the framework version (one React), the router, the design system and design tokens, auth/session, i18n, and telemetry. Everything else is private to a remote. Get the kernel wrong and the UI fragments: two button styles, two date formats, two auth prompts.

Before adding a remote, write a one-page **integration contract**: which framework major version, which shared design tokens, who owns the error boundary around the remote, and who rolls back when the remote fails at 3am. If a team can't agree on those four lines, they are not ready for micro-frontends — they are ready for a monorepo and stricter CI.

### Cross-MFE state is the anti-pattern

The single most common way micro-frontends go wrong architecturally is a **shared client store spanning remotes** — a global Redux/Zustand instance that checkout, catalog, and account all read and write. It couples the remotes at runtime, which destroys the independence you paid for, and it re-creates every problem the [state-placement funnel](../ecosystem/state-management-landscape.md#the-placement-funnel) exists to prevent, now across a deployment boundary you can't refactor atomically.

The placement funnel still decides, and it decides *harder* across the boundary:

- **Server state** stays server state — each remote owns its own data fetching and cache; shared server data is fetched by whoever needs it, not mirrored into a cross-remote store. (Server state was never client state; the boundary doesn't change that.)
- **Config / identity / tokens** live in the shared kernel, exposed as a stable, read-mostly contract — not a mutable store every remote writes.
- **Cross-remote communication** is an explicit, versioned contract: a typed event bus, custom DOM events, or URL state — a published surface, not a shared mutable object. If checkout needs to tell the shell "cart updated," it emits an event; it does not reach into a global store.

The test is the same one from the funnel, asked across teams: *would a shared mutable store here create a coupling that a second team can break with a deploy?* If yes — and for server state and cross-remote state it always is — it belongs in a contract, not a store.

### Shared-dependency governance and version skew

The hardest ongoing problem in runtime federation is **version skew**. With N remotes you are running N independently-deployed apps that each pin their own dependency versions, and the compatibility surface grows roughly as N·(N−1)/2. Checkout wants React 19.2; catalog is still on 19.0; the shell has to negotiate a singleton both can tolerate — or refuse to load one.

Discipline, not hope, is what keeps this stable:

- **Declare a shared kernel with a published version policy** (React, router, design tokens, i18n) and pin or range-limit it.
- **Validate at runtime** — the shell rejects a remote whose shared-version contract it can't satisfy, and renders a fallback rather than crashing.
- **Enforce in CI**, not in code review.
- Treat each remote's exposed surface as a **public API with semantic versioning**; breaking it is a major bump, coordinated, not a quiet Tuesday deploy.

Federation without this is how "independent deploys" turns into "any team can break production for everyone."

### Failure isolation is not free — you build it

When the shell loads a remote at runtime, the remote *will* eventually 404, time out, or ship a runtime error. Without isolation, one remote's failure blanks the whole shell. Isolation is something you construct:

- Wrap each remote mount in an **[error boundary](../rendering/error-boundaries.md#placement-the-granularity-ladder)** with a real fallback, so a dead remote degrades to a placeholder instead of a white screen — the same discipline as a [failed lazy chunk](../rendering/error-boundaries.md#the-stale-chunk-fallback), because a remote entry that won't load *is* a failed chunk.
- Add **manifest resilience** at the loader: timeouts, retries with jitter, a fallback URL, and a circuit breaker so a flapping remote doesn't take the page's main thread with it.
- Decide, per remote, **who owns the fallback UX** — that's part of the integration contract.

### Styling isolation

Remotes deployed by different teams will collide on class names and global CSS unless you scope them. Prefer approaches that isolate by construction — [CSS Modules or the shared design system enforced through the kernel](../ecosystem/styling-approaches.md#real-world-patterns), or Shadow DOM at a web-component seam for hard encapsulation. A global stylesheet shared informally across remotes is a fragmentation bug waiting to happen.

### The SSR / RSC intersection

Client-side Module Federation and server rendering fight each other: the federation runtime assumes a browser, and stitching remote client bundles into a streamed server response is awkward. The 2026 answer for React teams that need both is to move composition to the server — [Server-Component composition and a meta-framework's zones](../server/nextjs-and-rsc-in-practice.md#real-world-patterns) — where the server owns the boundary and streams one response. If SSR is non-negotiable, that pushes you off client-side federation and toward server/edge composition; weigh it in the decision, not after.

## Common mistakes

1. **Adopting micro-frontends for a build-time problem.** Slow builds are fixed by a monorepo with caching and boundaries, not by splitting the deploy. You'll spend the federation tax and still have a contended design-system release.
2. **"Architecture without autonomy is cosplay."** Shipping three remotes that still share one database migration queue, one on-call rotation, and one coordinated release buys you the complexity of micro-frontends with none of the independence. If the teams aren't actually autonomous, you've distributed the monolith, not escaped it.
3. **A shared client store across remotes.** The cardinal coupling. It re-monolithizes state across a boundary you can no longer refactor atomically. Use contracts and events; keep the funnel's discipline across teams.
4. **No shared kernel → UI fragmentation.** Without an enforced framework/router/design-token/auth contract, the product becomes a patchwork of two button styles and three date formats. Users feel the seams.
5. **No version policy → skew crashes.** Letting each remote pick its own React major and hoping the singleton negotiates is how a catalog deploy takes down checkout. Publish and enforce the shared-kernel versions.
6. **Treating it as a performance win.** It usually ships more JS and adds round-trips. It can be *made* competitive with disciplined shared config, but if performance is the goal, a monolith with route-level code splitting is the shorter path.
7. **No failure isolation.** Assuming remotes always load. One `remoteEntry` timeout blanks the shell. Error boundaries + a resilient loader (timeout/retry/circuit-breaker/fallback) are mandatory, not polish.
8. **Splitting by technical layer instead of business domain.** A "components remote," a "utils remote," a "state remote" re-creates the layered monolith with network hops between the layers. Split by *domain team* (checkout, catalog, account) or don't split.
9. **Forgetting the rollback story.** Independent deploy is only a benefit if it comes with independent *rollback*. If reverting checkout requires a coordinated full-product rollback, you never got the autonomy.
10. **Reaching for iframes to "keep it simple."** Iframes trade every integration concern (shared auth, sizing, navigation, host scripting) for isolation you probably didn't need. Use them for genuinely untrusted content, not for your own teams.

## How this evolved

The concept is old (Fowler's taxonomy of browser/build/web-component/server composition predates any specific tool). The *tooling* arc, React-relevant:

- **Frontend monolith → build-time modules.** Shared code via packages and monorepos; no runtime independence.
- **Webpack 5 Module Federation (2020).** The first practical runtime composition — a bundler runtime with a *share scope* that deduplicated dependencies across independently-built apps. Powerful, but coupled to webpack's runtime and heavy per app.
- **Module Federation 2.0 + the runtime era.** A framework-agnostic runtime, manifest-first loading (`mf-manifest.json`), first-class support in Rspack and, via plugins, Vite/Rolldown, plus type-sharing across remotes. Composition became a runtime concern with governance tooling rather than a build-plugin trick.
- **Native ESM federation.** Federation rebuilt on browser standards — ES modules plus import maps — so dedup falls out of module-record identity rather than a userland runtime, and remotes can even cross frameworks. Where the ecosystem is heading; article 38 traces the substrate difference.
- **Server-side Module Federation / RSC-across-origins (2025).** Composition moved to the server: stitch Server Components from different origins into one streamed response, shifting the micro-frontend benefit into the rendering layer and out of the client bundle.

The mental model — shell, remotes, exposes, shared, independent deploy — has not changed since 2020. Only the substrate under it has.

## Exercises

1. **Adjudicate three orgs.** (a) A 6-person startup with a 90-second build and "messy" code. (b) A 40-engineer company where the payments team and the dashboard team keep blocking each other's releases and want client-side transitions between their sections. (c) A large content site where five teams own five URL sections and full-page navigations between them are fine. Pick an approach for each and name the *constraint* that decided it. (Hint: only one of these is a runtime-Module-Federation problem; one is a monorepo; one is routing-layer.)
2. **Find the cosplay.** A team has three remotes but a shared Zustand store, one shared CI pipeline that must go green for any remote to deploy, and one on-call rotation. Which properties of "micro-frontend" do they actually have, and which are illusions? What's the smallest change that would give them real autonomy? (Hint: start with the store and the pipeline, not the bundler.)
3. **Draft the contract.** For a `checkout` remote loaded into a shell, write the one-page integration contract's four lines (framework version, shared tokens, error-boundary owner, rollback owner) and one cross-remote event checkout would emit. (Hint: the event is a published surface, not a store write.)

## Summary

A micro-frontend is an organizational tool that buys independent deployment for independent teams and charges bundle duplication, version skew, and operational surface for it. It is not a performance win, not a code-cleanliness tool, and not a technical upgrade over a monolith. The decision runs in order: if the real problem isn't team autonomy, don't split; if it's builds or code, fix the monolith with a modular monorepo; if separate teams own separate sections and hard navs are fine, use routing-layer composition; only a genuine need for runtime composition in a shared shell, backed by a funded governance policy, justifies Module Federation; and an SSR requirement pushes you to server/edge composition. Keep a shared kernel, keep cross-remote state in contracts rather than stores, and build failure isolation on purpose. Below roughly four teams, a well-structured monolith wins. When you've earned the runtime version, [`module-federation`](./module-federation.md) is the how.

## See also

- [`module-federation`](./module-federation.md) — the practice: Module Federation 2.0 on Vite 8, host/remote, shared singletons, the manifest, type-sharing, and the runtime-vs-native-ESM substrate traced in depth.
- [`state-management-landscape`](../ecosystem/state-management-landscape.md#the-placement-funnel) — the placement funnel that decides cross-remote state, now across a deployment boundary.
- [`error-boundaries`](../rendering/error-boundaries.md#placement-the-granularity-ladder) — isolating a failing remote so it degrades instead of blanking the shell.
- [`nextjs-and-rsc-in-practice`](../server/nextjs-and-rsc-in-practice.md#real-world-patterns) — server/edge composition and where the RSC boundary rules take over from client federation.
- [`styling-approaches`](../ecosystem/styling-approaches.md#real-world-patterns) — keeping remotes from colliding on CSS.

## References

- Martin Fowler — Micro Frontends (the composition-style taxonomy).
- Module Federation documentation — Introduction and the `shared`/singleton model.
- micro-frontends.org — organizational framing (verticals, team ownership).
- Feature-Sliced Design — the modular-monolith alternative and how it makes a later split safer.

## Demo source

- No standalone demo for the decision article — the working host/remote demo lives with [`module-federation`](./module-federation.md). *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*