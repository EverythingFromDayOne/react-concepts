---
article_id: nextjs-and-rsc-in-practice
concept_folder: server
wave: 4
related:
  - server/server-components
  - server/ssr-and-hydration
  - concurrent/actions
  - ecosystem/data-fetching-tanstack-query
  - concurrent/suspense
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Next.js and RSC in practice

> Server Components are a *model* — the boundary, the serialization, what dies crossing it, all owned by [the RSC article](../server/server-components.md). This is where that model becomes a shipping application: a router, a build that emits a static shell plus an RSC payload, a caching layer you opt into, and server-side mutations that revalidate it. The one fact that reorients everyone coming from Next.js 15: in Next.js 16 caching is **opt-in**. Everything is dynamic by default, and the build will *error* if you read request-time data outside a `<Suspense>` boundary without caching it. That error is the framework doing its job.

## What it is

The roadmap's stance is that RSC ships to production through Next.js, and this article is where we cash that in. The App Router is Next.js's RSC-native router: server components render on the server, `"use client"` marks the islands that ship JS to the browser, and the build produces two artifacts per route — static HTML for the first paint and a serialized RSC payload for client navigations.

What the framework adds on top of the raw model is four things you would otherwise hand-build: a **file-system router** with layouts and streaming; a **build that prerenders a static shell** and streams the dynamic remainder (Partial Prerendering); an **explicit caching layer** (`use cache`, `cacheLife`, `cacheTag`) with tag-based invalidation; and **Server Actions** — server functions you call straight from a form to mutate data and revalidate the cache in one round trip. It also owns the toolchain: the React Compiler is built in (you do *not* wire it yourself the way the [Vite SPA does](../rendering/react-compiler-deep-dive.md#the-vite-8-wiring-and-the-gotcha)), and Turbopack is the default bundler.

Baseline for this article: **Next.js 16** (which requires **React 19.2+**), with **Cache Components** enabled. Every API here is the stable, non-`unstable_` form.

## How it works under the hood

### Dynamic by default, static by opt-in

With Cache Components on (`cacheComponents: true`), Next.js tries to **prerender everything** at build time into a static shell. A component only makes it into that shell if everything it renders can complete without request-time data. The moment something reaches for per-request input — `cookies()`, `headers()`, `searchParams`, an uncached `fetch` — that subtree can't be prerendered, and Next.js forces you to say what should happen. You have exactly two answers:

1. **Cache it** — mark the work with `use cache`, and it becomes part of the shell (revalidatable, but prerendered).
2. **Stream it** — wrap it in `<Suspense>`, and it renders at request time, streaming in after the shell paints.

If you do neither, you get a build/dev error: *"Uncached data was accessed outside of `<Suspense>`."* This is not a bug to suppress — it's the framework refusing to silently make your whole page dynamic. Every route resolves into three tiers:

- **Static** — prerendered, no request data. The header, nav, layout chrome.
- **Cached-dynamic** — data behind `use cache`, included in the shell but revalidatable by time (`cacheLife`) or tag (`cacheTag`). The product catalog everyone sees.
- **Runtime-dynamic** — request-specific, wrapped in `<Suspense>`, streamed. The logged-in user's recommendations.

This is **Partial Prerendering (PPR)**: one page, static shell served instantly from the edge, dynamic holes filled by streaming. It's the default behavior of Cache Components — the old `experimental.ppr` flag and `experimental_ppr` route segment config were *removed* in 16 in favor of this model. Under the hood a short-lived cache (the `seconds` profile, `revalidate: 0`, or `expire` under five minutes) is automatically demoted to a dynamic hole rather than being baked into the shell, which is how you mix freshness levels on one page.

### The RSC payload and the toolchain

The serialized output the server streams is the RSC payload — the same format [the RSC article](../server/server-components.md#the-rsc-payload-and-serialization) traces; here it's just the thing the App Router build emits and the client uses to reconcile on navigation. Two toolchain facts follow from the framework owning the build. First, the **React Compiler is built in** — enable `reactCompiler: true` and Next.js runs it as part of its own pipeline (historically a Babel transform, now with a native Rust port in Turbopack). You never touch `@rolldown/plugin-babel` here; that wiring is a Vite-SPA concern the [compiler deep dive](../rendering/react-compiler-deep-dive.md#the-vite-8-wiring-and-the-gotcha) owns, and the "double compile" it warns about is exactly this — the framework compiles for you, so adding your own Babel pass would run the compiler twice. Second, `params`, `searchParams`, `cookies()`, and `headers()` are **async** in 16 (synchronous access was fully removed) — you `await params`, always.

## Basic usage

A minimal App Router route. The page is an async server component that fetches on the server directly — no `useEffect`, no client fetch, no loading state to manage:

```tsx
// app/products/[id]/page.tsx
interface PageProps {
  params: Promise<{ id: string }>; // params is async in Next.js 16
}

export default async function ProductPage({ params }: PageProps) {
  const { id } = await params;
  const product = await getProduct(id);

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </main>
  );
}
```

Turn on Cache Components, and (optionally) the Compiler:

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheComponents: true,
  reactCompiler: true,
};

export default nextConfig;
```

Make the catalog read cacheable — `use cache` at the top of the function, a lifetime, and a tag to invalidate on later:

```ts
// lib/products.ts
import { cacheLife, cacheTag } from "next/cache";
import type { Product } from "./types";

export async function getProduct(id: string): Promise<Product> {
  "use cache";
  cacheLife("hours");       // time-based revalidation
  cacheTag(`product-${id}`); // on-demand invalidation target

  const res = await fetch(`https://api.example.com/products/${id}`);
  return res.json();
}
```

Interactivity lives in a client island, marked with the directive at the top of the file:

```tsx
// app/products/[id]/AddToCartButton.tsx
"use client";

import { useState } from "react";

interface AddToCartButtonProps {
  productId: string;
}

export function AddToCartButton({ productId }: AddToCartButtonProps) {
  const [added, setAdded] = useState(false);
  return (
    <button onClick={() => setAdded(true)}>
      {added ? "Added ✓" : "Add to cart"}
    </button>
  );
}
```

## Walkthrough — a production product page, end to end

We'll build one route that exercises all three tiers, a Server Action mutation with tag revalidation, and the bridge that hands server-fetched data to a client TanStack Query cache. The App Router files are shown in full; app-specific infrastructure (`@/lib/db`, `@/lib/reviews`, and the `Product`/`Review` types) is elided as a stub you'd supply.

### Step 1 — The three tiers on one page

The page prerenders its chrome, includes the cached catalog data in the shell, and streams the personalized block:

```tsx
// app/products/[id]/page.tsx
import { Suspense } from "react";
import { getProduct } from "@/lib/products";
import { Recommendations } from "./Recommendations";
import { ReviewForm } from "./ReviewForm";
import { AddToCartButton } from "./AddToCartButton";

interface PageProps {
  params: Promise<{ id: string }>;
}

export default async function ProductPage({ params }: PageProps) {
  const { id } = await params;
  const product = await getProduct(id); // cached-dynamic (use cache), in the shell

  return (
    <main>
      {/* Static + cached-dynamic — all in the prerendered shell */}
      <header>
        <h1>{product.name}</h1>
        <p>{product.description}</p>
        <p>${product.price.toFixed(2)}</p>
        <AddToCartButton productId={product.id} />
      </header>

      {/* Runtime-dynamic — reads the user's session, streams in after the shell */}
      <Suspense fallback={<p>Loading recommendations…</p>}>
        <Recommendations productId={product.id} />
      </Suspense>

      <ReviewForm productId={product.id} />
    </main>
  );
}
```

`Recommendations` is the runtime-dynamic island. It reads a cookie, so it *cannot* be prerendered or shared-cached — which is precisely why it sits inside `<Suspense>`:

```tsx
// app/products/[id]/Recommendations.tsx
import { cookies } from "next/headers";

interface RecommendationsProps {
  productId: string;
}

export async function Recommendations({ productId }: RecommendationsProps) {
  const session = (await cookies()).get("session")?.value; // request-time — async in 16
  const items = await fetchRecommendations(productId, session);

  return (
    <section>
      <h2>Recommended for you</h2>
      <ul>
        {items.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </section>
  );
}
```

> If you tried to read `cookies()` inside a shared `use cache` scope, Next.js would throw — request-specific input can't live in a shared cache. Pass it as an argument, or use the experimental `use cache: private` for genuinely per-user caching. Here we simply keep it out of the cache and stream it.

### Step 2 — A Server Action that mutates and revalidates

Reviews are submitted with a Server Action — a `"use server"` function called straight from a form. It mutates, then expires exactly the tags that changed:

```ts
// app/products/[id]/actions.ts
"use server";

import { updateTag } from "next/cache";
import { z } from "zod";
import { db } from "@/lib/db";

const ReviewSchema = z.object({
  productId: z.string(),
  rating: z.coerce.number().min(1).max(5),
  body: z.string().min(10, { error: "Review must be at least 10 characters." }),
});

export interface ReviewState {
  ok: boolean;
  error?: string;
}

export async function submitReview(
  _prev: ReviewState,
  formData: FormData,
): Promise<ReviewState> {
  const parsed = ReviewSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { ok: false, error: parsed.error.issues[0].message };
  }

  await db.reviews.insert(parsed.data);
  // Read-your-own-writes: the next read of this product's reviews is fresh
  // immediately. updateTag is Server-Action-only and clears the client cache now.
  updateTag(`product-${parsed.data.productId}`);
  return { ok: true };
}
```

The form is a client island so it can show pending and error state, wiring the action through React 19's `useActionState` (the mutation-state mechanism is owned by [the Actions article](../concurrent/actions.md#useactionstate--the-reducer-productized) — here we only connect it to a *server* action):

```tsx
// app/products/[id]/ReviewForm.tsx
"use client";

import { useActionState } from "react";
import { submitReview, type ReviewState } from "./actions";

const initial: ReviewState = { ok: false };

interface ReviewFormProps {
  productId: string;
}

export function ReviewForm({ productId }: ReviewFormProps) {
  const [state, action, pending] = useActionState(submitReview, initial);

  return (
    <form action={action}>
      <input type="hidden" name="productId" value={productId} />
      <label>
        Rating
        <input name="rating" type="number" min={1} max={5} required />
      </label>
      <textarea name="body" aria-label="Your review" required />
      {state.error && <p role="alert">{state.error}</p>}
      {state.ok && <p>Thanks — your review is live.</p>}
      <button disabled={pending}>{pending ? "Submitting…" : "Submit review"}</button>
    </form>
  );
}
```

Submit → `submitReview` runs on the server → `updateTag` expires `product-<id>` → the next render of anything tagged with it fetches fresh data, no full-page reload. The static shell never re-downloads; only the invalidated data does.

### Step 3 — The RSC → TanStack Query bridge

Some interactivity genuinely needs a client cache: a "load more reviews" list that paginates, refetches, and optimistically updates on the client. That's [TanStack Query](../ecosystem/data-fetching-tanstack-query.md)'s job — but `useQuery` runs inside `useEffect`, which does not run during a prerender, so a naive client query would show a spinner on first paint even though the server already has the data. The bridge is to **prefetch on the server and hydrate the client cache**:

```tsx
// app/products/[id]/reviews-section.tsx  (server component)
import {
  QueryClient,
  HydrationBoundary,
  dehydrate,
} from "@tanstack/react-query";
import { getReviews } from "@/lib/reviews";
import { ReviewList } from "./ReviewList";

interface ReviewsSectionProps {
  productId: string;
}

export async function ReviewsSection({ productId }: ReviewsSectionProps) {
  const queryClient = new QueryClient();
  // Server-side prefetch — the data is fetched during the server render…
  await queryClient.prefetchQuery({
    queryKey: ["reviews", productId],
    queryFn: () => getReviews(productId),
  });

  // …and handed to the client cache, so useQuery hydrates instead of spinning.
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ReviewList productId={productId} />
    </HydrationBoundary>
  );
}
```

```tsx
// app/products/[id]/ReviewList.tsx  (client island)
"use client";

import { useQuery } from "@tanstack/react-query";
import { getReviews } from "@/lib/reviews";

interface ReviewListProps {
  productId: string;
}

export function ReviewList({ productId }: ReviewListProps) {
  // No initial spinner: the cache is already warm from the server prefetch.
  const { data: reviews } = useQuery({
    queryKey: ["reviews", productId],
    queryFn: () => getReviews(productId),
  });

  return (
    <ul>
      {reviews?.map((r) => (
        <li key={r.id}>{r.body}</li>
      ))}
    </ul>
  );
}
```

Server state is fetched on the server and lives in the Query cache on the client — never copied into a client store like Zustand. That's the [state-placement spine](../ecosystem/state-management-landscape.md#the-placement-funnel) holding across the server boundary: the server is just another producer of server state.

## Real-world patterns

**The caching decision, per read.** For each piece of data, ask in order: does it depend on request-time input (session, geo, search params)? → keep it dynamic, wrap in `<Suspense>`. If not, is staleness acceptable for some window? → `use cache` + `cacheLife`. Does it change on a known event (a publish, an edit)? → add `cacheTag` and invalidate on that event. This three-question funnel replaces the old "is this page static or dynamic?" whole-route decision — in PPR the answer is per-subtree.

**Choosing the invalidation API.** Three tools, and the difference is real:

- `updateTag(tag)` — Server Actions only. Expires immediately; the next read *blocks* for fresh data. Use it for **read-your-own-writes** — the user made a change and must see it now.
- `revalidateTag(tag, "max")` — Server Actions *or* Route Handlers (the `app/**/route.ts` endpoint files). Stale-while-revalidate: serves stale, refreshes in the background, and only on next visit. Use it when a slight delay is fine (a CMS publish, a catalog refresh). Note the single-argument `revalidateTag(tag)` is **deprecated** — always pass the profile.
- `revalidatePath("/products")` — path-based and blunt; flushes everything under a path. Prefer tags; reach for this only when you truly want a whole segment gone.

**Server Actions are the mutation half of React 19 Actions.** The client `<form action>` / `useActionState` / `useOptimistic` machinery is [owned by the Actions article](../concurrent/actions.md); Next.js supplies the *server* function it calls and the revalidation that follows. The clean division: validate + mutate + `updateTag` on the server; pending + error + optimistic UI on the client. Reuse the same Zod schema on both sides exactly as [forms-at-scale](../forms/forms-at-scale.md#step-1--one-schema-both-sides) prescribes.

**Framework-mode routing lands here.** The [React Router article](../ecosystem/routing-react-router.md) deliberately covered *data mode* and fenced framework mode out of Phase 1; the App Router is that framework-mode router, RSC-native. If you're choosing between them: React Router (data mode) for a Vite SPA where you own the server story; Next.js App Router when you want RSC, streaming, and caching as framework primitives rather than things you assemble.

**Route-level conventions do the wiring for you.** The manual `<Suspense>` in the walkthrough is explicit on purpose, but in a real route the App Router special files usually do it: a `loading.tsx` beside a `page.tsx` wraps that segment in a Suspense boundary automatically (a route-level fallback), and an `error.tsx` is a route-level error boundary — the [class-boundary mechanics are owned by the error-boundaries article](../rendering/error-boundaries.md); here they're just a file convention. Use `loading.tsx` for the whole-segment fallback and inline `<Suspense>` for the finer-grained streaming islands, exactly as shown.

**Migration is a perf cliff if you're not ready.** Upgrading a Next.js 15 app to 16 flips caching to opt-in, so a page that was implicitly static becomes fully dynamic overnight and TTFB regresses. Add `use cache` incrementally — landing, blog, docs, and pricing pages first (pure static wins), then catalog reads with `cacheLife`, then tag the mutation paths. Test in `next build && next start`, never `next dev`, because caching behavior differs in dev.

## API / tool reference

| API | Where | Purpose |
| --- | --- | --- |
| `"use cache"` | Top of a file, async function, or component | Marks the scope cacheable. Requires `cacheComponents: true`. |
| `cacheLife(profile \| { stale, revalidate, expire })` | Inside a `use cache` scope | Time-based lifetime. Built-ins: `seconds`/`minutes`/`hours`/`days`. Short-lived → dynamic hole. |
| `cacheTag(tag, …)` | Inside a `use cache` scope | Tags the entry for on-demand invalidation. Multiple/idempotent. |
| `updateTag(tag)` | Server Actions only | Immediate expiry, blocking next read. Read-your-own-writes. |
| `revalidateTag(tag, profile)` | Server Actions or Route Handlers | Stale-while-revalidate. Use `"max"`; single-arg form deprecated. |
| `revalidatePath(path)` | Server Actions or Route Handlers | Path-based invalidation (blunt). |
| `"use server"` | Top of a file or function | Marks a Server Action (server-only function callable from the client). |
| `"use client"` | Top of a file | Marks a client island (ships JS, can use hooks/state). |
| `await params` / `await searchParams` | Server components | Both are Promises in Next.js 16. |
| `cacheComponents: true` / `reactCompiler: true` | `next.config.ts` | Enable PPR + `use cache`; enable the built-in React Compiler. |

## Common mistakes

**1. Expecting Next.js 15's implicit caching.** In 16 everything is dynamic by default. A page that "was fast" pre-upgrade now renders at request time until you add `use cache`. This is the single biggest 15→16 surprise — plan the cache pass as part of the upgrade, not after.

**2. `use cache` with no `cacheComponents`.** The directive is inert without the flag. If pages still render dynamically after you added `"use cache"`, check `next.config.ts` first.

```ts
// ❌ Silently does nothing without the flag.
export async function getData() {
  "use cache";
  return db.query("…");
}
// ✅ next.config.ts → cacheComponents: true
```

**3. Reading `cookies()`/`headers()` inside a shared `use cache` scope.** Request-specific input in a shared cache is a build error. Pass the value in as an argument, or use `use cache: private` for real per-user caching — never widen the cache to swallow the session.

**4. "Fixing" the *Uncached data accessed outside `<Suspense>`* error by caching request data.** The error is telling you a subtree is dynamic. The fix is usually a `<Suspense>` boundary so it streams — not forcing a `use cache` over per-request data, which would serve one user's data to another.

**5. The deprecated single-argument `revalidateTag(tag)`.** It still limps along with TS errors suppressed, but the two-argument `revalidateTag(tag, "max")` is the contract now.

**6. `updateTag` in a Route Handler.** It throws — `updateTag` is Server-Action-only. In a Route Handler (a webhook, say) use `revalidateTag(tag, "max")`, or `{ expire: 0 }` when an external system needs immediate expiry.

**7. Forgetting to `await params` / `searchParams`.** They're Promises in 16; synchronous access was removed. Destructuring them directly is a type error and a runtime one.

**8. Fetching server data in a client `useEffect`.** If a server component can `await` the data directly, do that — it's simpler, has no waterfall, and ships no fetch code to the client. Reach for a client query only for genuinely interactive client caches, and then hydrate it from a server prefetch rather than spinning on mount.

**9. Copying server data into a client store.** RSC doesn't change the [state-placement spine](../ecosystem/state-management-landscape.md#the-placement-funnel): server state belongs in the Query cache (hydrated across the boundary), not mirrored into Zustand. The server is one more producer of server state, not an excuse to stash it in a client store.

**10. Testing caching in `next dev`.** Dev intentionally behaves differently. Verify PPR, shell contents, and revalidation with `next build && next start`.

## How this evolved

The Pages Router expressed server rendering as per-page data functions — `getServerSideProps` (per request), `getStaticProps` (build time), and ISR (`revalidate` seconds) — a coarse, whole-page choice. The App Router (13) moved to RSC with **implicit** `fetch` caching: requests were memoized and cached by default, which was powerful but famously confusing — nobody could tell what was cached or why. Next.js 15 began walking that back, flipping `fetch` and Route Handlers to uncached-by-default. Next.js 16 finishes the reversal with **Cache Components**: caching is fully explicit via `use cache`, PPR is the default rendering model (the experimental flags removed), Turbopack is the default bundler, and the React Compiler is built in. The arc is one sentence — the framework moved from *"cached by default, opt out"* to *"dynamic by default, opt in"* — and every API in this article is a consequence of that decision.

## Exercises

**1. Classify a page into the three tiers.** Take a dashboard with a nav, a shared metrics panel, and a per-user activity feed. Decide which is static, which is `use cache` + `cacheLife`, and which streams inside `<Suspense>`. *Hint:* anything touching `cookies()`/`headers()` cannot be shared-cached — it either streams or uses `use cache: private`.

**2. Wire read-your-own-writes.** Add a Server Action that edits a cached record and make the edit visible on the very next read. *Hint:* tag the read with `cacheTag`, and reach for `updateTag` (not `revalidateTag`) in the action — the difference is whether the user waits for fresh data or sees stale first.

**3. Kill a first-paint spinner with the hydration bridge.** Start from a client `useQuery` list that flashes a spinner on load, and move the fetch to a server prefetch. *Hint:* `prefetchQuery` into a `QueryClient` on the server, wrap the client island in `<HydrationBoundary state={dehydrate(queryClient)}>`, and confirm the spinner is gone because the cache is warm.

## Summary

Next.js is where the RSC model becomes a production app. In Next.js 16 the defining shift is that caching is opt-in: the framework prerenders a static shell, and any request-time data must either be cached (`use cache` + `cacheLife`/`cacheTag`) or streamed (`<Suspense>`) — that's Partial Prerendering, and the "uncached data outside Suspense" error is the framework enforcing it. Mutations run as Server Actions that revalidate by tag — `updateTag` for read-your-own-writes, `revalidateTag(tag, "max")` for stale-while-revalidate. Fetch server data on the server and, when a client cache is genuinely needed, hydrate TanStack Query from a server prefetch rather than copying it into a client store. The framework owns the toolchain — the React Compiler and Turbopack are built in — so you don't wire them yourself. Reach for this when you want RSC, streaming, and caching as primitives; reach for a Vite SPA + React Router when you want React-the-library.

## See also

- [Server Components](../server/server-components.md) — the RSC model this article puts into production (boundary, serialization, payload).
- [SSR and hydration](../server/ssr-and-hydration.md#streaming-and-selective-hydration) — the streaming fundamentals PPR builds on.
- [Actions](../concurrent/actions.md#useactionstate--the-reducer-productized) — the client mutation machinery Server Actions plug into.
- [Data fetching with TanStack Query](../ecosystem/data-fetching-tanstack-query.md#how-this-evolved) — the client cache the RSC→Query bridge hydrates.
- [Suspense](../concurrent/suspense.md) — the boundary that turns a dynamic subtree into a streamed one.
- [Recipe: hydration mismatch on dates/random values](../recipes/ssr-and-rsc/hydration-mismatch.md) *(planned)* — opens the ssr-and-rsc recipe track this article gates.
- [Recipe: `"use client"` sprawl](../recipes/ssr-and-rsc/use-client-sprawl.md) *(planned)* — keeping the client bundle small.
- [Recipe: server action returns stale data](../recipes/ssr-and-rsc/server-action-stale-data.md) *(planned)* — the revalidation-tag failure mode.

## References

- Next.js — [Next.js 16 release](https://nextjs.org/blog/next-16)
- Next.js — [`use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- Next.js — [Caching and Revalidating](https://nextjs.org/docs/app/getting-started/caching-and-revalidating)
- Next.js — [`cacheLife`](https://nextjs.org/docs/app/api-reference/functions/cacheLife) · [`cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cacheTag) · [`updateTag`](https://nextjs.org/docs/app/api-reference/functions/updateTag) · [`revalidateTag`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- TanStack Query — [Advanced SSR / hydration](https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr)

## Demo source

*Demo pending — see the roadmap's demo-hosting decision (StackBlitz vs. CodeSandbox vs. local `demos/`).*