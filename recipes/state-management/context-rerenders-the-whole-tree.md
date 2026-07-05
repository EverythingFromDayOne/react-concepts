---
recipe_id: context-rerenders-the-whole-tree
track: state-management
primary_concept: state/context
difficulty: intermediate
react_baseline: "19.2"
related:
  - state/context
  - foundations/component-composition
  - rendering/how-react-renders
  - rendering/memoization-and-the-compiler
  - effects/escape-hatches-audit
status:
  drafted: true
  reviewed: false
---

# Context Re-Renders the Whole Tree

> **What you'll build:** a 200-card product grid where clicking "Add to cart" currently re-renders every visible card and measurably drops frames. The fix runs in three stages — split the megacontext by change cadence, pull the context-reading part down into a leaf so the expensive card body stops subscribing at all, then know exactly where that stops being enough.

## The scenario

Starting point: a storefront grid of `<ProductCard>` — 200 SKUs in the full catalog. Cart, wishlist, and the toolbar's sort/search all live in one provider:

```tsx
// store-context.tsx — one context, four concerns
import { createContext, use } from "react";
import type { CartItem, SortOrder } from "./types";

interface StoreValue {
  cart: { items: CartItem[] };
  wishlist: { ids: Set<string> };
  sortOrder: SortOrder;
  searchQuery: string;
  addToCart: (item: CartItem) => void;
  toggleWishlist: (id: string) => void;
  setSortOrder: (order: SortOrder) => void;
  setSearchQuery: (query: string) => void;
}

const StoreContext = createContext<StoreValue | null>(null);

export function useStore(): StoreValue {
  const store = use(StoreContext);
  if (store === null) throw new Error("useStore must be used inside <StoreProvider>");
  return store;
}
```

```tsx
// ProductCard.tsx — every card subscribes to everything
function ProductCard({ product }: { product: Product }) {
  const { cart, wishlist, addToCart, toggleWishlist } = useStore();
  const inCart = cart.items.some((i) => i.id === product.id);
  const isWishlisted = wishlist.ids.has(product.id);

  return (
    <article className="product-card">
      <img src={product.image} alt={product.name} loading="lazy" />
      <h3>{product.name}</h3>
      <p>{formatPrice(product.price)}</p>
      <button onClick={() => toggleWishlist(product.id)}>{isWishlisted ? "♥" : "♡"}</button>
      <button onClick={() => addToCart({ id: product.id, qty: 1 })}>
        {inCart ? "✓ In cart" : "Add to cart"}
      </button>
    </article>
  );
}
```

The failure, with numbers: Profiler on an "Add to cart" click shows **all 200 `ProductCard`s re-rendering**, ~1.1ms apiece — 220ms of render work against a 16.7ms frame budget, dropping roughly 13 frames as a visible stutter. Typing in the search box costs the identical 220ms *per keystroke*, for the same reason: `sortOrder`, `searchQuery`, `cart`, and `wishlist` all live in one `value` object, so any one changing broadcasts to every reader — [context](../../state/context.md)'s bailout-check-3 rule, billed in full.

Why it escaped QA: staging seeds ~18 products, and the per-card cost is invisible below a few dozen cards — nobody profiles a grid that renders in under 20ms. The pre-launch perf checklist covered bundle size and LCP, not interaction cost, so a load test against production-scale seed data was never run. The regression surfaced two weeks post-launch, once the full 200-SKU catalog went live, as an INP alert on `/shop` climbing from a comfortable 90ms to 310ms — solidly in "needs improvement" territory. Profiler's "why did this render," on any card, for any of the four triggers, reads the same: *context changed.*

## Walkthrough

**Stage 1 — confirm the diagnosis before touching code.** Record a search keystroke and an add-to-cart click separately. Both show the identical shape: 200 renders, "context changed." This rules out the *other* context bug — an unstabilized provider `value` re-broadcasting on unrelated parent renders ([context](../../state/context.md)'s identity-churn fix) — which looks the same in the Profiler but needs a `useMemo` on the value, not a split. Here the value's identity is fine; the data it carries is *genuinely* changing, every time, for everyone.

**Stage 2 — split by change cadence, not by topic.** Cart, wishlist, and filters change independently and at different rates; bundling them was the mistake. (Provider bodies follow the same shape as [context](../../state/context.md)'s `SessionProvider` — state via `useState`, actions closing over the setters, `value` memoized per the identity-churn fix; omitted here since Stage 1 already ruled that bug out.)

```tsx
interface CartValue { items: CartItem[]; addToCart: (item: CartItem) => void; }
interface WishlistValue { ids: Set<string>; toggleWishlist: (id: string) => void; }
interface FiltersValue {
  sortOrder: SortOrder;
  searchQuery: string;
  setSortOrder: (order: SortOrder) => void;
  setSearchQuery: (query: string) => void;
}

const CartContext = createContext<CartValue | null>(null);
const WishlistContext = createContext<WishlistValue | null>(null);
const FiltersContext = createContext<FiltersValue | null>(null); // read only by the toolbar

function useCart(): CartValue {
  const v = use(CartContext);
  if (!v) throw new Error("useCart must be used inside <CartProvider>");
  return v;
}
function useWishlist(): WishlistValue {
  const v = use(WishlistContext);
  if (!v) throw new Error("useWishlist must be used inside <WishlistProvider>");
  return v;
}
```

```tsx
function ProductCard({ product }: { product: Product }) {
  const { items, addToCart } = useCart();
  const { ids, toggleWishlist } = useWishlist();
  const inCart = items.some((i) => i.id === product.id);
  const isWishlisted = ids.has(product.id);
  // …identical JSX
}
```

Re-profile: search and sort keystrokes now touch **zero** `ProductCard`s — the toolbar reads `FiltersContext` alone, and cards never subscribed to it. That's 220ms → 0ms per keystroke, for free. But the add-to-cart click is unchanged: every card still reads `items` from `CartContext` to check membership, so every card is still a genuine subscriber to the thing that just changed. Splitting fixed the *cross-concern* pollution; the *within-concern* broadcast — what this recipe is actually about — remains exactly as expensive. Reading less is the only lever left. Stage 3 is reading less.

**Stage 3 — stop the expensive component from subscribing at all.** `ProductCard`'s image and price formatting have nothing to do with cart membership. Pull the membership check into two leaves small enough that re-rendering them is free, and let the card body opt out entirely:

```tsx
// ProductCard.tsx — no longer reads Cart or Wishlist context
function ProductCard({ product }: { product: Product }) {
  return (
    <article className="product-card">
      <img src={product.image} alt={product.name} loading="lazy" />
      <h3>{product.name}</h3>
      <p>{formatPrice(product.price)}</p>
      <WishlistHeart productId={product.id} />
      <CartBadge productId={product.id} />
    </article>
  );
}

// The ONLY subscribers — kept deliberately tiny
function CartBadge({ productId }: { productId: string }) {
  const { items, addToCart } = useCart();
  const inCart = items.some((i) => i.id === productId);
  return (
    <button onClick={() => addToCart({ id: productId, qty: 1 })}>
      {inCart ? "✓ In cart" : "Add to cart"}
    </button>
  );
}

function WishlistHeart({ productId }: { productId: string }) {
  const { ids, toggleWishlist } = useWishlist();
  return (
    <button onClick={() => toggleWishlist(productId)}>{ids.has(productId) ? "♥" : "♡"}</button>
  );
}
```

This is the composition move from [component-composition](../../foundations/component-composition.md) — colocate the piece that changes with the state that changes it, and let everything else opt out by never touching it. `ProductCard`'s only prop is `product`, unchanged across a cart mutation — Compiler-stabilized, bailout check 1 passes, and the card body (image, price formatting, the wrapper markup) **does not re-render at all** ([memoization-and-the-compiler](../../rendering/memoization-and-the-compiler.md), [how-react-renders](../../rendering/how-react-renders.md)). Only `CartBadge` — 200 of them, each a boolean check and a button — actually respond to the broadcast.

Re-profile: add-to-cart now renders 200 `CartBadge`s at ~0.05ms each, ≈10ms total, no dropped frames. `WishlistHeart` doesn't render at all on a cart action — it subscribes to a different channel. INP on `/shop` returns to ~95ms.

The full audit trail:

| Interaction | Before (megacontext) | After split (Stage 2) | After leaf extraction (Stage 3) |
| --- | --- | --- | --- |
| Search keystroke | 220ms (200 cards) | 0ms (0 cards) | 0ms (0 cards) |
| Add to cart | 220ms (200 cards) | 220ms (200 cards) | ~10ms (200 badges) |
| Toggle wishlist | 220ms (200 cards) | 220ms (200 cards) | ~10ms (200 hearts) |
| `/shop` INP | 310ms | 310ms (cart/wishlist untouched) | ~95ms |

**What didn't change:** this is still 200 components rendering, not one. Context has no mechanism to notify only the *one* card whose membership actually flipped — every reader of a changed context re-renders, full stop. What changed is what those 200 renders cost. Variation C is the one primitive that gets you true one-card granularity, and when it's worth reaching for.

## Variations

**Small catalogs — skip this entirely.** At ~1.1ms/card, cost scales linearly with what's mounted: the staging seed's 18 products cost ~20ms — a single frame at most, imperceptible as a one-off. The visible stutter only shows up once card count climbs into the hundreds. Profile your actual production scale before deciding whether three extra files and two extra hooks are worth it.

**Server-persisted cart.** If the cart is tied to a logged-in session and lives on the server, don't mirror it in a client `Context` at all — it's server state, per the placement rule, and belongs in the query cache once [data-fetching-tanstack-query](../../ecosystem/data-fetching-tanstack-query.md) *(Wave 4, planned)* exists. A guest-cart-in-Context / logged-in-cart-in-query-cache split is common and fine; `useCart()` becomes the seam hiding which one is live.

**True one-card granularity via `useSyncExternalStore`.** Context's floor — every reader of a changed context re-renders — isn't a limitation of this recipe's approach; it's structural. The one vanilla-React primitive that clears it: a module-level store plus a per-id hook, each card's own snapshot compared independently ([escape-hatches-audit](../../effects/escape-hatches-audit.md) owns the contract):

```tsx
function useIsInCart(productId: string): boolean {
  return useSyncExternalStore(cartStore.subscribe, () => cartStore.has(productId));
}
```

Because each call site's `getSnapshot` closes over its own `productId` and returns a primitive boolean, React compares *that card's* last boolean to its new one — a card whose membership didn't flip doesn't re-render, even though every card's listener was notified. This is genuinely O(1)-relevant-cards, not O(n)-cheap-cards, and it's the exact primitive Zustand's selector hooks are built on. Reach for it when leaf-extraction's O(n) still isn't cheap enough — a much larger catalog, or leaves that aren't as trivial as a checkmark.

## Trade-offs and common pitfalls

1. **Splitting before measuring.** Three contexts and two extra hooks for a 20-card grid is ceremony solving a problem that never existed — profile first.
2. **A leaf that still does the expensive part.** If `CartBadge` received the whole `product` object and did anything beyond a membership check, the cost just moved, it didn't leave.
3. **A leaf that re-derives from the whole array, per render, for every card.** `items.some(...)` inside a 200-instance leaf is O(n) work done 200 times — fine at small n, worth a `Set` or the Variation C store once it isn't.
4. **Grouping by topic instead of change cadence.** "Cart and wishlist are both product state" is a reason to bundle them right back into one context — recreating this exact bug under a nicer name. Group by *how often, and together, things change*.
5. **Over-fragmenting.** Five contexts for five independently-trivial, rarely-changing fields is the opposite failure — ceremony with no payoff. The judgment call is clusters, not maximum separation.
6. **Confusing this with the identity-churn bug.** An unstabilized provider `value` re-broadcasting on unrelated parent renders looks identical in the Profiler but needs `useMemo` on the value, not a split — Stage 1 exists to tell them apart.
7. **Reaching for a state library before confirming Context is the actual ceiling.** Split-then-extract solves this shape cheaply; a library earns its dependency once you've measured the residual and it's still not enough.
8. **Testing only in isolation.** A Storybook story or unit test renders one `ProductCard`; this bug is invisible below list scale and will never surface there. Profile the full grid with production-scale seed data as a standing perf-regression check, not a one-time fix.
9. **Trusting "it feels snappier."** Re-profile and record the number — 220ms → 10ms is a claim you can put in a PR description; a feeling isn't.
10. **Forgetting the middle layer can silently opt back in.** If `ProductCard` picks up an unrelated rules-of-react violation elsewhere in the file, the Compiler skips it, bailout check 1 stops passing, and the fix quietly degrades with no error — the ✨ badge is the five-second check.
11. **When NOT to do any of this: check virtualization first.** If the grid is windowed and only ~12 cards are ever mounted, 200-card blast radius was never real — confirm what's actually mounted before splitting or extracting anything. Fixing render *cost* matters far less when mount *count* is already capped.

## See also

- [context](../../state/context.md) — the split-context mechanics, bailout check 3, and the placement table this recipe builds on
- [component-composition](../../foundations/component-composition.md) — the colocation move Stage 3 applies
- [how-react-renders](../../rendering/how-react-renders.md) — Profiler methodology and bailout check 1
- [memoization-and-the-compiler](../../rendering/memoization-and-the-compiler.md) — why the stabilized card body bails out once it stops reading context
- [escape-hatches-audit](../../effects/escape-hatches-audit.md) — the `useSyncExternalStore` contract behind Variation C
- [typing-lag-rerender-storm](../performance/typing-lag-rerender-storm.md) — the measure-first discipline this recipe's Stage 1 follows
- [state-management-landscape](../../ecosystem/state-management-landscape.md) *(Wave 4, planned)* — the selector-store escalation past this recipe's ceiling
- [data-fetching-tanstack-query](../../ecosystem/data-fetching-tanstack-query.md) *(Wave 4, planned)* — the server-persisted cart variation

## References

- [createContext — react.dev](https://react.dev/reference/react/createContext)
- [useSyncExternalStore — react.dev](https://react.dev/reference/react/useSyncExternalStore)
- [Interaction to Next Paint — web.dev](https://web.dev/articles/inp)
- [<Profiler> — react.dev](https://react.dev/reference/react/Profiler)

## Demo source

Demo pending — hosting decision tracked in `progress.md` TODOs.