---
recipe_id: cross-remote-state-coupling
track: micro-frontends
primary_concept: architecture/micro-frontends
difficulty: advanced
react_baseline: "19.2"
related:
  - architecture/micro-frontends
  - ecosystem/state-management-landscape
  - ecosystem/data-fetching-tanstack-query
  - recipes/state-management/zustand-goes-stale
  - recipes/micro-frontends/version-skew-breaks-host
status:
  drafted: true
  reviewed: false
---

# A shared store couples your remotes back into a monolith

> **What you'll build:** cross-remote communication that keeps remotes independent — the acting remote fetches server truth and emits a versioned event, and every other remote refetches its own view — instead of a shared mutable store that every remote reads and writes and that re-couples independently-deployed apps into a distributed monolith.

## The scenario

The shell composes `catalog`, `checkout`, and `header`, and they need to share "the cart." The obvious move: a `shared` Zustand store (`@acme/cart-store`, shared as a federation singleton so all remotes get one instance) holding `{ items, total, addItem }`. Catalog's "Add to cart" writes it, header reads `total` for the badge, checkout reads `items`. The demo is beautiful.

Then it rots, three ways:

1. **Shape coupling.** The checkout team refactors the store — `total` becomes `subtotal`, `items` becomes `{ lines, meta }` — and deploys the new `@acme/cart-store`. Header, not rebuilt, reads `state.total` → `undefined` → the badge shows `$NaN`. A store-shape change is a *silent breaking change to every remote that reads it*, and because the store is shared, one team's refactor breaks another team's independent deploy. (It's [version skew](./version-skew-breaks-host.md), but on shared *state* read ad-hoc everywhere, which is worse.)
2. **Mount-order races.** Header mounts and reads `cart.total` before catalog has populated the store → reads the initial empty value, shows 0, then flickers. Correctness now depends on which remote lazy-loaded first.
3. **Refactor lockstep.** You can't change the cart store's internals in *one* remote without touching every consumer, so every store change is a coordinated release. The independent deployment you paid for is gone — [architecture without autonomy is cosplay](../../architecture/micro-frontends.md#the-decision).

Why it escaped QA: in integrated dev, all remotes are on the *current* store version and mount in a predictable order, so shape coupling and races never fire. The breakage needs a remote to deploy a store change independently, or a different mount order — production conditions that single-team, all-current testing never reproduces.

## Walkthrough

### Stage 1 — Name it: a shared mutable store is the monolith's coupling without its safety

A shared store across remotes gives you the frontend monolith's *tight coupling* — every remote depends on one mutable shape — with none of the monolith's *atomic refactor*: you can't change the shape and all its readers in one commit, because the readers are separately-built, separately-deployed apps. It re-creates every problem the [state-placement funnel](../../ecosystem/state-management-landscape.md#the-placement-funnel) exists to prevent, now across a boundary you can't refactor atomically — exactly [37's cardinal anti-pattern](../../architecture/micro-frontends.md#cross-mfe-state-is-the-anti-pattern).

And notice what "the cart" actually is: **server state.** It lives on the server; it's the same for every remote; it can go stale. The funnel says server state is never a client store ([the whole point of `zustand-goes-stale`](../state-management/zustand-goes-stale.md#stage-1--name-the-category-error)) — and that's *doubly* true across teams. So the reframe is two moves: cross-remote coupling goes through an explicit **contract** (an event), and the shared *data* goes to its real home (**each remote's own server-state cache**), not a shared mutable object.

### Stage 2 — Reject the "fix the store" fixes

- **"Make the store a strict singleton."** Fixes the dual-instance write-loss (the [React-dedup cousin](./shared-react-duplicated.md) — two stores, writes lost), but it makes the coupling *reliable*, not *gone*. Shape coupling and refactor-lockstep remain.
- **"Version the store package."** Helps — semver like [version-skew](./version-skew-breaks-host.md) — but you're still forcing lockstep consumers who read the shape ad-hoc. You've documented the coupling, not removed it.

A shared mutable store is the wrong tool for cross-team communication. The fix is a different model.

### Stage 3 — Communicate with a versioned event; each remote owns its own read

The cart's source of truth is the server, so each remote fetches it into [its own Query cache](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns). When something changes, the acting remote [writes the server](../../ecosystem/data-fetching-tanstack-query.md#stage-4--a-mutation-with-the-server-as-source-of-truth) and emits a small, **versioned** event; every other remote listens and refetches its own view. No shared mutable object; the only cross-remote surface is the event contract.

```ts
// @acme/cart-events — the ONLY cross-remote surface: a versioned event, not a store
const bus = new EventTarget();
export function emitCartChanged() {
  bus.dispatchEvent(new CustomEvent("cart:changed", { detail: { v: 1 } }));
}
export function onCartChanged(fn: () => void): () => void {
  const handler = () => fn();
  bus.addEventListener("cart:changed", handler);
  return () => bus.removeEventListener("cart:changed", handler);
}
```

```tsx
// header remote — owns its own cart query, refetches on the event
function CartBadge() {
  const queryClient = useQueryClient();
  const { data: cart } = useQuery({ queryKey: ["cart"], queryFn: fetchCart });
  useEffect(() => onCartChanged(() => queryClient.invalidateQueries({ queryKey: ["cart"] })), [queryClient]);
  return <Badge count={cart?.itemCount ?? 0} />;
}

// catalog remote — write the server, then signal; never reach into header's state
async function addToCart(item: CartItem) {
  await api.addToCart(item); // server is the source of truth
  emitCartChanged();          // "something changed" — listeners refetch their own view
}
```

The event carries "something changed," not the data — so checkout can restructure its *internal* cart representation freely, and header keeps working because it reads server truth plus a stable event, never checkout's internals. The mount-order race is gone too: a remote that mounts late just fetches the current cart on mount.

### Stage 4 — Place the rest with the funnel, and verify

Not everything is server state. Run each cross-remote piece through the funnel across the boundary:

- **Server state** (cart, user's orders) → each remote's own Query cache + event-driven invalidation (Stage 3).
- **Navigational / shareable state** (selected category, current entity) → [the URL](../routing/params-out-of-sync.md#stage-3--read-the-url-dont-store-it), which every remote already shares and which survives reload.
- **True cross-cutting client state** (theme, locale, the current user) → a small **read-mostly** value from the [shared kernel](../../architecture/micro-frontends.md#the-shared-kernel-and-the-integration-contract), provided by the shell and consumed as a stable contract — not a mutable store every remote writes.
- **A remote's own UI state** → stays local to that remote.

The event payload, the URL params, and the kernel contract are all **public APIs across the boundary** — version them additively, exactly like [an exposed component's props](./version-skew-breaks-host.md).

**Verify the loop.** Catalog adds an item → emits `cart:changed` → header's badge and checkout's list update via their own refetch, no shared store touched. The checkout team refactors its internal cart representation and deploys → header still works (it never read checkout's internals). A remote mounts late → it fetches the current cart on mount, no zero-then-flicker. The coupling that would have made every store change a coordinated release is gone.

## Variations

1. **Transport choices.** An `EventTarget` bus, plain custom DOM events on a shared element, or `BroadcastChannel` (which also crosses tabs). Pick one; keep the contract named and minimal.
2. **Server state is usually the whole answer.** Most "cross-remote state" is server state; each remote's Query cache + invalidate-on-event dissolves it, and you inherit staleness handling for free ([`zustand-goes-stale`](../state-management/zustand-goes-stale.md#stage-3--move-server-state-to-its-home-tanstack-query)).
3. **URL for shareable/navigational pieces** — the address bar is already shared across every remote and is history-aware; use it before inventing a channel.
4. **Read-mostly kernel context** for genuine cross-cutting client state (theme/locale/user) — one contract from the shell, consumed, never mutated by remotes.
5. **Versioned event payloads.** If an event *must* carry data, its shape is a public API — evolve it additively (`v: 2` adds fields, keeps `v: 1` readable) so a payload change doesn't silently break listeners.

## Trade-offs and common pitfalls

1. **A shared mutable store across remotes** — the core anti-pattern; the distributed monolith.
2. **Putting server state in a cross-remote store** — stale data plus the whole `zustand-goes-stale` problem, now multiplied across teams.
3. **A store-shape change** — a silent breaking change to every reader that didn't rebuild.
4. **"Strict singleton" as the fix** — reliable coupling is still coupling; shape and refactor-lockstep remain.
5. **Mount-order reads** — reading shared state a sibling hasn't written yet. Fetch on mount instead.
6. **Unversioned event payloads** — an event-shape change silently breaks listeners (the version-skew trap again).
7. **Reaching into another remote's store slice** — coupling to an implementation, not a contract. Consume the published surface only.
8. **A global `window.__cart__`** — the untyped, unversioned worst case of a shared store.
9. **Event soup** — chatty, ad-hoc events that are impossible to trace. Keep the contract small and named; emit "changed," not a firehose.
10. **Duplicating cross-cutting client state per remote** (each remote its own theme copy) — drift. One kernel contract.
11. **Two-way writable shared state between remotes** — ownership ambiguity ("who wins?"). Give it one owner; others react via events.
12. **Assuming all remotes are current** — a lagging remote reads a shape that already changed. The contract, not the shape, is what stays stable.

### When NOT to build a contract

If the "remotes" are **one team on one cadence**, or build-time-composed into a single build, a shared store is just normal app state — the placement funnel applies as usual (Zustand for cross-cutting *client* state is fine *within one app*), and the event-contract ceremony is pure overhead. The coupling only bites when *independent teams* deploy the writers and readers on *different* cadences. And if the shared thing is a tiny, stable, write-once value the shell provides (feature flags fetched once, the logged-in user), a read-mostly kernel value needs no event bus at all. The test: *do independent teams deploy the writers and readers of this state on different cadences?* If no, use the funnel and move on. If yes, a contract — never a shared mutable store.

## See also

- [`micro-frontends`](../../architecture/micro-frontends.md#cross-mfe-state-is-the-anti-pattern) — cross-remote state as the cardinal anti-pattern, and the shared-kernel/contract discipline.
- [`state-management-landscape`](../../ecosystem/state-management-landscape.md#the-placement-funnel) — the placement funnel that decides where each piece belongs, applied across the boundary.
- [`zustand-goes-stale`](../state-management/zustand-goes-stale.md#stage-1--name-the-category-error) — the same "server state isn't a client store" lesson within one app; this recipe is its cross-team edition.
- [`version-skew-breaks-host`](./version-skew-breaks-host.md) — why the event payload (like an exposed API) must be a versioned contract.
- [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns) — the per-remote server-state cache and invalidation this recipe leans on.

## References

- micro-frontends.org — team autonomy and communication patterns.
- Martin Fowler — Micro Frontends (cross-app communication via custom events).
- MDN — `EventTarget`, `CustomEvent`, `BroadcastChannel`.

## Demo source

- `demos/micro-frontends/cross-remote-state-coupling/` — the shared cart store breaking on an independent refactor, then the event + per-remote-Query rewrite that keeps the remotes independent. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*