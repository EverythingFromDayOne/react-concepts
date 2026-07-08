---
recipe_id: remote-failure-blanks-shell
track: micro-frontends
primary_concept: architecture/module-federation
difficulty: advanced
react_baseline: "19.2"
related:
  - architecture/module-federation
  - architecture/micro-frontends
  - rendering/error-boundaries
  - concurrent/suspense
  - recipes/micro-frontends/shared-react-duplicated
status:
  drafted: true
  reviewed: false
---

# One remote 404s and the whole shell goes white

> **What you'll build:** a shell where a remote that fails to load — a 404, a timeout, a bad deploy — degrades to a placeholder in *its own slot* instead of blanking the entire app, with a resilient loader (timeout, retry, circuit breaker) so a transient blip never becomes an outage and a struggling remote never gets hammered.

## The scenario

The shell composes four remotes: `nav`, `catalog`, `checkout`, and `recommendations`. Each is loaded at runtime with `React.lazy(() => import("checkout/Widget"))`. One afternoon the checkout team's CDN has a three-minute blip and `checkout/mf-manifest.json` starts returning 404.

For those three minutes, **every user loading the app gets a white screen** — not "checkout is unavailable, the rest works," but the entire product dead. A three-minute blip in *one* remote became a total outage, and the remote that did it could just as easily have been `recommendations` — the least important thing on the page.

Why: the failed dynamic `import("checkout/Widget")` rejects; `React.lazy` turns that rejection into a throw; the throw propagates to the nearest [error boundary](../../rendering/error-boundaries.md#placement-the-granularity-ladder) — and if there's no boundary, or the only boundary is at the app root, React unmounts the whole tree and renders the root fallback (or nothing). The blast radius of a partial failure is the entire shell.

Why it escaped QA:

- **Remotes are always up in dev and staging** — localhost, or deployed together — so the failure path never runs. "What if the remote is down?" is a test nobody writes because in dev it's never down.
- The happy path (remote loads) is all anyone exercises.
- It's an **availability** bug that only appears during a real remote incident in production — exactly when you can least afford a partial failure to go total.
- The blast radius is **inverted**: the least-critical remote can blank the most-critical flow. A `recommendations` deploy gone wrong takes down checkout.

## Walkthrough

### Stage 1 — Name it: a remote is a network dependency that will fail

A remote isn't a component; it's a *runtime network fetch that renders a component*. Network fetches 404, time out, and serve bad deploys — so a remote *will* fail, and the only question is blast radius. Left alone, a failed remote import throws, React [propagates the throw to the nearest boundary](../../concurrent/suspense.md#suspending-is-a-throw-catching-is-a-boundary), and with no boundary (or a root one) the failure is total. The goal is **graceful degradation**: a dead remote becomes a placeholder in its slot, and everything else keeps working. That isolation is something you build, not something federation gives you — exactly [what article 37 warned](../../architecture/micro-frontends.md#failure-isolation-is-not-free--you-build-it).

### Stage 2 — Isolate each remote behind its own boundary

Wrap **every** remote mount — not the app root — in its own error boundary plus Suspense. A failed remote is a [failed lazy chunk](../../rendering/error-boundaries.md#the-stale-chunk-fallback); the boundary catches it and renders a fallback [scoped to that one slot](../../rendering/error-boundaries.md#placement-the-granularity-ladder).

```tsx
import { Suspense, type ReactNode } from "react";
import { ErrorBoundary } from "react-error-boundary";

export function Remote({ name, children }: { name: string; children: ReactNode }) {
  return (
    <ErrorBoundary
      fallbackRender={({ resetErrorBoundary }) => (
        <RemoteUnavailable name={name} onRetry={resetErrorBoundary} />
      )}
    >
      <Suspense fallback={<RemoteSkeleton name={name} />}>{children}</Suspense>
    </ErrorBoundary>
  );
}
```

Now `checkout` 404ing shows a "Checkout is temporarily unavailable" card in the checkout slot; `nav`, `catalog`, and `recommendations` render normally. The `resetErrorBoundary` from `react-error-boundary` gives the fallback a manual retry — the [reset design](../../rendering/error-boundaries.md#reset-design) that lets a user re-attempt once the remote recovers, instead of being stuck until a full reload. Keep the fallback [dumb and self-contained](../../rendering/error-boundaries.md#fallbacks-stay-dumb) — it renders when the surrounding code is broken, so it can't depend on it.

### Stage 3 — Make the loader resilient: timeout and retry

The boundary catches a *hard* failure. But a hanging remote (slow CDN, not a 404) would spin Suspense forever, and a transient 503 that a retry would fix hard-fails. `React.lazy` can't express timeout or retry, so wrap the [runtime loader](../../architecture/module-federation.md#step-4--failure-isolation-and-resilient-loading) instead:

```ts
import { loadRemote } from "@module-federation/runtime";

function withTimeout<T>(p: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    p,
    new Promise<T>((_, reject) => setTimeout(() => reject(new Error("remote timeout")), ms)),
  ]);
}

export async function loadRemoteResilient<T>(
  id: string,
  { timeout = 5000, retries = 2 } = {},
): Promise<T> {
  let lastError: unknown;
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await withTimeout(loadRemote<T>(id) as Promise<T>, timeout);
    } catch (error) {
      lastError = error;
      // exponential backoff + jitter so N clients don't retry in lockstep
      await new Promise((r) => setTimeout(r, 200 * 2 ** attempt + Math.random() * 100));
    }
  }
  throw lastError; // exhausted — let the boundary show the fallback
}
```

A timeout turns a hang into a fast, catchable failure; a jittered retry rides out a blip without a thundering herd. When retries are exhausted it throws, and Stage 2's boundary takes over.

### Stage 4 — Circuit breaker, criticality, and verify

A *flapping* remote (fails intermittently) shouldn't be retried on every render and navigation — that hammers a struggling remote and janks the UI. A per-remote circuit breaker skips loading entirely once a remote is clearly down, for a cooldown:

```ts
const breakers = new Map<string, { failures: number; openedAt: number }>();
const THRESHOLD = 3;
const COOLDOWN = 30_000;

export function circuitOpen(id: string): boolean {
  const b = breakers.get(id);
  return !!b && b.failures >= THRESHOLD && Date.now() - b.openedAt < COOLDOWN;
}
export function recordFailure(id: string): void {
  const b = breakers.get(id) ?? { failures: 0, openedAt: 0 };
  breakers.set(id, { failures: b.failures + 1, openedAt: Date.now() });
}
export function recordSuccess(id: string): void {
  breakers.delete(id); // recovered — close the circuit
}
```

Check `circuitOpen(id)` at the top of `loadRemoteResilient` and throw immediately when open, so a known-down remote shows its fallback instantly instead of re-timing-out on every navigation. Then record failure/success around the load.

Not every remote degrades the same way — that's a **product decision per remote**, not a default:

- `recommendations` failing → hide it silently; the user never needed to know.
- `checkout` failing → a prominent, honest fallback with a retry — silently hiding the thing the user came to do is worse than saying it's down.
- `nav`/`auth` failing → possibly a genuine full-page error, because the shell is unusable without them.

**Verify the loop.** 404 the checkout manifest and load the app: checkout shows its fallback, everything else renders — no white screen. Throttle the remote to a hang: it times out to the fallback instead of spinning forever. Fail it three times: the breaker opens and the fallback appears instantly on the next navigation. Restore the remote and click retry (or wait out the cooldown): it recovers. The three-minute blip is now a three-minute placeholder in one slot.

## Variations

1. **Criticality tiers.** Encode per-remote whether failure is *optional* (hide), *degraded* (fallback + retry), or *critical* (full-page error). The boundary's fallback is the tier.
2. **Fallback to a last-known-good version.** With immutable, versioned remote URLs, a bad deploy can fall back to the previous `mf-manifest.json` instead of a placeholder — the remote stays up on the old version.
3. **Gate in the route loader.** In [React Router data mode](../../ecosystem/routing-react-router.md#real-world-patterns), load the remote in the route loader with the resilient wrapper so a dead *critical* remote redirects to an error route before the layout even mounts.
4. **SSR / streaming.** During server render, a failing remote's boundary streams its fallback with the rest of the shell (the [error half of the Suspense triad](../../concurrent/suspense.md#real-world-patterns)) rather than failing the whole response.
5. **Observability.** Report every remote-load failure (which remote, which version, the error) and the circuit-breaker state. A bad deploy should page you, not wait for user complaints — and you need to know *which* remote and version failed.

## Trade-offs and common pitfalls

1. **No error boundary** — a remote failure blanks the shell. The core bug.
2. **One root boundary instead of per-remote** — still all-or-nothing: any remote down renders the whole-app fallback. Boundary *per remote mount*.
3. **No timeout** — a hanging remote (slow, not 404) spins Suspense forever; the route never renders. `Promise.race` a timeout.
4. **No retry** — a transient 503 hard-fails when one retry would have succeeded.
5. **Retrying on every render/navigation** — no breaker means a struggling remote gets hammered and the UI janks. Open the circuit after N failures.
6. **Treating every remote as degradable** — silently hiding checkout is worse than a clear "it's down." Critical remotes need loud fallbacks.
7. **Treating every remote as critical** — a `recommendations` blip should not full-page-error the app.
8. **A fallback of a different size than the content** — a CLS jump when it swaps in. Reserve the slot's space.
9. **No manual retry in the fallback** — the user is stuck until a full reload even after the remote recovers. Wire `resetErrorBoundary`.
10. **A boundary that can't reset** — once tripped it stays tripped for the session; use `resetKeys` / `resetErrorBoundary` so recovery is possible.
11. **No observability** — a bad remote deploy is invisible until users complain, and you can't tell which remote or version failed.
12. **Assuming remotes are always up** — the QA trap; never testing the failure path is why this ships.

### When NOT to build the isolation machinery

If you have exactly **one remote and it *is* the product** — its failure genuinely means the app is useless — then a full-page error is the correct behavior; isolating it into a slot buys nothing because there's no other UI to keep alive. And **build-time-composed remotes** (bundled into the host via a monorepo/package, not loaded at runtime) can't fail *at load* — they're already in the host bundle, so they fail like any component: a plain error boundary, no loader resilience. The test: *can this remote fail to load independently at runtime, and is there other UI worth keeping alive when it does?* Both yes → isolate and add resilience. Otherwise a plain boundary — or nothing — is enough.

## See also

- [`module-federation`](../../architecture/module-federation.md#step-4--failure-isolation-and-resilient-loading) — the runtime loader (`loadRemote`) and the isolation step this recipe hardens.
- [`micro-frontends`](../../architecture/micro-frontends.md#failure-isolation-is-not-free--you-build-it) — why isolation is something you build, and the per-remote fallback-ownership contract.
- [`error-boundaries`](../../rendering/error-boundaries.md#the-stale-chunk-fallback) — a failed remote is a failed chunk; granularity, reset design, and dumb fallbacks.
- [`shared-react-duplicated`](./shared-react-duplicated.md) — the sibling federation crash (the singleton dedup bug).
- `micro-frontends/version-skew-breaks-host` — the sibling governance bug *(planned)*.

## References

- Module Federation — the runtime API (`loadRemote`, `registerRemotes`) and manifest loading.
- `react-error-boundary` — `fallbackRender`, `resetErrorBoundary`, `resetKeys`.
- Release It! (Nygard) — circuit breaker and timeout patterns (the resilience vocabulary).

## Demo source

- `demos/micro-frontends/remote-failure-blanks-shell/` — the four-remote shell with one remote 404ing: the white-screen before, per-slot degradation + timeout/retry/breaker after. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*