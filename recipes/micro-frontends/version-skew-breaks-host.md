---
recipe_id: version-skew-breaks-host
track: micro-frontends
primary_concept: architecture/module-federation
difficulty: advanced
react_baseline: "19.2"
related:
  - architecture/micro-frontends
  - architecture/module-federation
  - ecosystem/testing
  - recipes/micro-frontends/shared-react-duplicated
  - recipes/micro-frontends/remote-failure-blanks-shell
status:
  drafted: true
  reviewed: false
---

# A remote's independent deploy silently breaks the host

> **What you'll build:** a federation where a remote can ship on its own cadence *without* breaking the host — by treating the remote's exposed surface as a versioned public API, validating compatibility at runtime (not just build time), and evolving the contract additively so old hosts keep working through a deprecation window.

## The scenario

The `checkout` remote exposes `<CheckoutWidget cartId onComplete />`, and the host renders it with those props — fully typed via the shared `@mf-types` from [article 38's type sharing](../../architecture/module-federation.md#step-2--the-host-consumes-it-with-real-types). It has worked for months.

The checkout team refactors: they rename `onComplete` to `onSuccess` and change it to receive `{ orderId }` instead of a bare string. They update their own callers, their tests pass, they bump and deploy. **The host is not rebuilt** — it still passes `onComplete={goToOrder}`.

At runtime the new widget never reads `onComplete` (it's gone), so it calls `onSuccess`, which the host doesn't provide. Depending on which way the break falls: either the host's post-purchase navigation never fires and **users pay and land nowhere** (silent), or the new widget assumes a prop the host doesn't pass and throws a `TypeError` mid-checkout. Either way, a breaking change to an exposed contract — deployed independently, with nothing checking compatibility — corrupted the host.

The insidious part: **both pipelines were green.** The host built and tested against the *previous* remote's types; the remote built and tested against its *own* updated callers; both deploys succeeded. The break exists only in the *combination* at runtime — which neither pipeline ran.

Why it escaped QA:

- The host's build baked in a `@mf-types` snapshot from the remote's **previous** version, so the host compiled and tested green against a contract that no longer exists at runtime.
- The remote's tests exercised its own new callers, green.
- No pipeline runs the **current host against the current remote** — they're separate builds, separate deploys, separate teams.
- Types gave **false confidence**: "it's typed" is true at the host's build time and irrelevant to the remote that redeployed after.
- It's combination-only *and* timing-dependent — broken only after the remote deploys and before the host rebuilds.

## Walkthrough

### Stage 1 — Name it: the exposed surface is a public API, and types are a build-time snapshot

The remote's `exposes` is a **public API across a deployment boundary**, consumed by builds you don't control and don't rebuild in lockstep. Type-sharing checks compatibility at *build time*; independent deployment means the *runtime* remote can drift from the version the host built against. Types can't catch a redeploy that happens after your build — they were frozen when your build ran.

So the reframe: treat the exposed surface like any versioned public library API, and check compatibility at **runtime**, not just in the type-checker. And accept that you can't "just keep everything in sync" — the [compatibility surface grows as N·(N−1)/2](../../architecture/micro-frontends.md#shared-dependency-governance-and-version-skew), so manual coordination stops scaling at a handful of remotes.

### Stage 2 — Reject the fixes that feel safe

- **"Share types and trust them."** The false confidence that caused this. Types are a build snapshot; the runtime remote moved.
- **"Rebuild the host whenever a remote changes."** This works, and it destroys the entire point — you've re-coupled the release train into a coordinated deploy, which is the monolith you left. If you're doing this, [reconsider whether you need federation at all](../../architecture/micro-frontends.md#the-decision).
- **"Pin every remote to an exact version."** Safe, but you lose "ship the remote without touching the host." It's the right call for a *critical* remote, not the general answer.

The durable lever is a **versioned contract validated at runtime**, plus **additive evolution** so breaks are rare.

### Stage 3 — Version the exposed contract and gate it at load

The remote publishes a semantic version for its exposed surface; the host declares the range it supports and refuses to mount an incompatible major, degrading loudly instead of running corrupted.

```ts
// checkout remote — expose the contract version (exposes: { "./version": "./src/version.ts" })
export const CONTRACT_VERSION = "2.0.0";
```

```ts
// host — check compatibility before mounting
import { loadRemote } from "@module-federation/runtime";
import { satisfies } from "semver";

const SUPPORTED = "^2.0.0"; // the range the host was built against

export async function loadCheckout() {
  const { CONTRACT_VERSION } = await loadRemote<{ CONTRACT_VERSION: string }>("checkout/version");
  if (!satisfies(CONTRACT_VERSION, SUPPORTED)) {
    // a major mismatch: refuse, let the boundary show a fallback — never run on a broken contract
    throw new Error(`checkout ${CONTRACT_VERSION} incompatible with host's ${SUPPORTED}`);
  }
  return (await loadRemote<{ default: React.ComponentType<CheckoutWidgetProps> }>("checkout/Widget"))!.default;
}
```

A rejected remote is a *controlled* degradation — pair it with the [per-remote fallback from the isolation recipe](./remote-failure-blanks-shell.md) so the checkout slot shows "temporarily unavailable" instead of corrupting the flow. Loud beats silent: a fallback the user sees is far better than a purchase that pays and navigates nowhere.

### Stage 4 — Evolve the contract additively, and test the combination

Gating is a safety net; the way you actually keep velocity is to **not break the contract in the first place** — evolve it the way you'd evolve a public library: additive within a major, deprecate-then-remove across one, with a window where both work.

```tsx
interface CheckoutWidgetProps {
  cartId: string;
  onSuccess?: (result: { orderId: string }) => void;
  /** @deprecated use onSuccess; removed in the 3.0 contract */
  onComplete?: (orderId: string) => void;
}

function CheckoutWidget({ cartId, onSuccess, onComplete }: CheckoutWidgetProps) {
  const finish = (orderId: string) => {
    onSuccess?.({ orderId });
    onComplete?.(orderId); // keep hosts built against 2.x working through the window
  };
  // ...
}
```

Old hosts keep calling `onComplete`; new hosts adopt `onSuccess`; the remote drops `onComplete` in the next *major* after telemetry shows no host still uses it. That's a minor bump, not a break — the host that never rebuilt keeps working.

Then close the gap the two green pipelines left with a **[contract test in CI](../../ecosystem/testing.md#real-world-patterns)**: the remote asserts its exposed surface still matches the published schema (nothing removed without a major), and the host asserts it only consumes the published surface. That's cheaper than spinning up both apps and it catches the break *before* deploy.

**Verify the loop.** Ship a breaking major *without* a window: the host's runtime check catches the version mismatch and renders the fallback — no silent pay-and-nowhere. Ship the same change *additively* (with `onSuccess` added, `onComplete` deprecated): the un-rebuilt host keeps working. Try to remove `onComplete` while a host still uses it: the contract test fails in CI, before it can ship.

## Variations

1. **Criticality-tiered pinning.** Pin `checkout` to an exact remote version (coordinate its upgrades); float `recommendations` on a range. The [criticality tiers from the isolation recipe](./remote-failure-blanks-shell.md) applied to versioning.
2. **Consumer-driven contracts.** The host publishes what it consumes; the remote's CI fails if a change breaks a known consumer (Pact-style). Autonomy with a guardrail.
3. **Runtime feature detection.** For a soft migration, the host uses `onSuccess` if the loaded module has it, else falls back to `onComplete` — degrade by capability rather than by version number.
4. **Schema on the wire, not just types.** Validate the *data* crossing the boundary (props, events) against a runtime schema (Zod) at the seam, so a shape change is caught as data, not just as stale types.
5. **Deprecation telemetry.** Track which host versions still call the deprecated surface so you know when the window can close — otherwise it never does.

## Trade-offs and common pitfalls

1. **Treating the exposed surface as private/mutable.** It's a public API across a deploy boundary; change it like one (semver, deprecation).
2. **Trusting shared types as a runtime guarantee.** They're a build-time snapshot; the runtime remote drifts after your build.
3. **Breaking the contract with no deprecation window.** Old hosts break the instant the remote deploys.
4. **Rebuilding the host on every remote change as "the fix."** You've re-coupled the release train — that's the monolith, minus its safety.
5. **No runtime compatibility check.** A breaking deploy silently corrupts (stale props) instead of degrading loudly.
6. **Silent contract breaks.** A renamed callback that just never fires is worse than a crash — the user pays and nothing happens. Prefer loud failures.
7. **No semver on the exposed surface.** "Is this compatible?" is unanswerable; you can't gate what you don't version.
8. **No contract/integration test.** Both pipelines green, the combination broken — the gap that ships this bug.
9. **Pinning everything to exact versions.** You lose independent deployment (back to lockstep). Pin selectively by criticality.
10. **Floating a critical remote on a wide range.** A minor can still surprise the flow that matters most; tighten critical remotes.
11. **No deprecation telemetry.** You never learn when it's safe to remove, so the window never closes and the alias lives forever.
12. **Assuming "both teams tested" means "the combination works."** The combination is exactly what neither team tested.

### When NOT to build contract governance

If your remotes are **pinned and deployed together** in a coordinated release, there's no skew to govern — but then you've given up independent deployment, and it's worth asking whether you needed federation at all ([architecture without autonomy is cosplay](../../architecture/micro-frontends.md#the-decision)). And if a remote has exactly **one consumer, owned by the same team on the same cadence**, the contract ceremony is overhead — a shared package (build-time composition) is simpler and the compiler enforces the contract for free. The governance pays off specifically when *independent teams* deploy *shared* remotes on *different* cadences. The test: *do independent teams deploy this remote on a different cadence than the host?* If no, you don't have a skew problem — simplify.

## See also

- [`micro-frontends`](../../architecture/micro-frontends.md#shared-dependency-governance-and-version-skew) — the version-skew tax, the shared-kernel contract, and the integration contract this recipe operationalizes.
- [`module-federation`](../../architecture/module-federation.md#step-2--the-host-consumes-it-with-real-types) — the `@mf-types` sharing whose build-time nature is the root of this bug.
- [`testing`](../../ecosystem/testing.md#real-world-patterns) — the contract/integration test that catches the combination before deploy.
- [`shared-react-duplicated`](./shared-react-duplicated.md) — the shared-*dependency* skew crash (React singleton), a sibling of this exposed-*API* skew.
- [`remote-failure-blanks-shell`](./remote-failure-blanks-shell.md) — the fallback a rejected incompatible remote degrades into.

## References

- Module Federation — manifest metadata, the `exposes` surface, and runtime `loadRemote`.
- Semantic Versioning — the public-API discipline the exposed surface must follow.
- Pact / consumer-driven contract testing — validating a boundary without deploying both sides.

## Demo source

- `demos/micro-frontends/version-skew-breaks-host/` — the breaking-vs-additive deploy: a major bump crashing an un-rebuilt host, the runtime gate degrading it, and the additive/deprecation path keeping it alive. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*