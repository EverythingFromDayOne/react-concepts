---
article_id: routing-react-router
concept_folder: ecosystem
wave: 4
related:
  - concurrent/suspense
  - rendering/error-boundaries
  - ecosystem/data-fetching-tanstack-query
  - forms/forms-controlled-and-uncontrolled
  - concurrent/actions
  - effects/effects-and-synchronization
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this:** In data mode, the router stops being a `URL → component` switch and becomes a `URL → (data + mutation + error + pending)` layer. A route's data is fetched *before* its component renders, mutations go through the route instead of hand-rolled `useState`, and loader failures land in a dedicated error channel that has nothing to do with React's render-time error boundaries. Getting the two channels straight is most of the battle.

## What it is

React Router v7 ships three "modes." **Declarative mode** is the classic `<Routes>/<Route>` JSX you fetch inside with effects. **Framework mode** is the full Remix-style server story (file routes, typegen, SSR). This article is **data mode** — the middle tier: you build a route *object* tree, hand it to `createBrowserRouter`, and each route can declare a `loader` (data before render), an `action` (mutation from a `<Form>`), an `errorElement` (its failure UI), and `lazy` (its code split).

> **v7 package note:** the canonical import is now `react-router` (v7 consolidated the old `react-router` / `react-router-dom` split). `react-router-dom` still re-exports for back-compat, so v6 tutorials showing it aren't wrong — but new code imports from `react-router`.

Data mode's whole reason to exist is killing the **navigation waterfall**. In the effect-fetching model, navigating to `/products/42` renders the component, *then* the component mounts, *then* an effect fires the fetch — document → JS → component → data, in series. A loader moves the fetch to route-match time, so the data request starts the instant the navigation starts, in parallel with rendering. It's the routing-level version of the same lesson [`effects-and-synchronization`](../effects/effects-and-synchronization.md) taught at the component level.

One honest boundary up front, because [`data-fetching-tanstack-query`](./data-fetching-tanstack-query.md) just landed: **a loader is a fetch-*timing* mechanism, not a cache.** It runs on every navigation, with no dedup, no `staleTime`, no background refetch. Server state still wants Query's cache. The two compose — the loader *kicks off* a Query prefetch and the component reads from the cache — and that integration is a real-world pattern below. Loaders replace *fetch-in-effect*, not *TanStack Query*.

> **When NOT to reach for data mode.** If your app is a couple of screens that fetch through Query in components, plain **declarative mode** (`<Routes>`/`<Route>`) plus `useSuspenseQuery` is lighter — Query already gives you loading, error, and cache, so loaders would be redundant plumbing. Data mode earns its keep when you want fetch-on-navigate timing, route-scoped pending and error UI, and automatic revalidation after mutations. Reach for it for that leverage, not reflexively.

## How it works under the hood

Trace one navigation and the data flow stops being mysterious.

**The navigation lifecycle.** When you click a `<Link>` (or call `navigate`):

1. **Match.** The router matches the URL against the route tree, producing an ordered list of matched routes (parent → child).
2. **Load.** It calls the `loader` of *every* matched route **in parallel** — a nested `/products/42/reviews` fires the products loader, the detail loader, and the reviews loader at once, not in sequence.
3. **Wait for critical data.** It awaits every value you `await`ed inside your loaders. Bare (un-awaited) promises you return are *not* waited on — they stream (deferred data, below).
4. **Commit.** Once critical data resolves, the router transitions and renders the matched components, each reading its slice via `useLoaderData`.

While that's happening, `useNavigation().state` moves `idle → loading → idle` (or `→ submitting → loading → idle` for an action). That single state field is your pending UI — no `isLoading` flag to thread.

**Revalidation.** After an `action` runs (a `<Form>` submit), the router automatically **re-calls the loaders** for the current routes and re-renders with fresh data. You don't manually refetch after a mutation; revalidation is the default. (This is the one place a loader's non-cache nature is a feature — it always re-reads authoritative state.)

**Two error channels — this is the crux.** Errors have *two* separate homes, and conflating them is the article's headline mistake:

| Where the error happens | Who catches it | How you read it |
| --- | --- | --- |
| A `loader`, an `action`, or a route component's render | the nearest route's `errorElement` | `useRouteError()` |
| A render inside a component, an event handler you wrapped | a React error boundary ([`error-boundaries`](../rendering/error-boundaries.md)) | `react-error-boundary`'s render prop |

A loader that throws does **not** trip a React `<ErrorBoundary>`; it trips the route's `errorElement`. And a React error boundary does **not** catch a loader rejection. They're parallel systems. `errorElement` bubbles like a React boundary does — an unhandled route error walks up to the nearest ancestor `errorElement` — but it's the router's mechanism, keyed to the route tree, not the component tree.

## Basic usage

Two pieces: a router at the root, a loader on a route.

```tsx
// router.tsx
import { createBrowserRouter, RouterProvider } from "react-router";
import { ProductsPage, productsLoader } from "./ProductsPage";
import { RootError } from "./RootError";

export const router = createBrowserRouter([
  {
    path: "/products",
    element: <ProductsPage />,
    loader: productsLoader, // runs before ProductsPage renders
    errorElement: <RootError />, // catches productsLoader throws + render errors
  },
]);

// main.tsx
createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>,
);
```

```tsx
// ProductsPage.tsx
import { useLoaderData } from "react-router";

export async function productsLoader(): Promise<Product[]> {
  const res = await fetch("/api/products");
  if (!res.ok) throw new Response("Failed to load products", { status: res.status });
  return res.json();
}

export function ProductsPage() {
  const products = useLoaderData() as Product[];
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name}</li> // data is already here — no loading state to render
      ))}
    </ul>
  );
}
```

There is no `useState`, no `useEffect`, no loading flag in the component — the data is present on first render because the router waited for it. Throwing a `Response` (not a plain string) is deliberate: it carries a status code the `errorElement` can branch on.

## Nested routes, layouts, and `<Outlet>`

Routes nest, and that nesting is what lets loaders run in parallel across a matched chain. A layout route renders shared chrome once and drops its children into an `<Outlet>`:

```tsx
createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />, // nav + <Outlet/>, renders for every child
    errorElement: <RootError />,
    children: [
      { index: true, element: <Home />, loader: homeLoader }, // matches "/" exactly
      { path: "products", element: <ProductsPage />, loader: productsLoader },
      { path: "products/:id", lazy: () => import("./productDetail") },
    ],
  },
]);

function RootLayout() {
  return (
    <div>
      <nav>
        <NavLink to="/">Home</NavLink>
        <NavLink to="/products">Products</NavLink>
      </nav>
      <main>
        <Outlet /> {/* the matched child route renders here */}
      </main>
    </div>
  );
}
```

Navigating to `/products/42` matches `RootLayout → productDetail`, so the router calls *both* loaders at once (step 2 of the lifecycle) and renders the detail inside the layout's `<Outlet>`. `index: true` is the child that matches the parent's path exactly; `<NavLink>` is `<Link>` with active-state styling. A child's `errorElement` catches its own failures before they reach the parent's.



We'll build `/products/:id`: load the product before render, stream the slow reviews, and add a review through a route action.

### Step 1 — Loader with params, deferred non-critical data

The product is critical (await it); reviews are slow and non-critical (return the bare promise — **no `defer()` wrapper**, that's gone in v7):

```tsx
// productDetail.tsx
import { useLoaderData, Await, Form, useNavigation, redirect } from "react-router";
import { Suspense } from "react";

export async function loader({ params }: { params: { id: string } }) {
  const res = await fetch(`/api/products/${params.id}`);
  if (!res.ok) throw new Response("Not found", { status: res.status });
  const product = await res.json(); // critical: awaited

  // Non-critical: return the promise itself. The router will NOT wait on it.
  const reviews = fetch(`/api/products/${params.id}/reviews`).then((r) => r.json());

  return { product, reviews }; // must be an object with keys — a bare promise won't defer
}
```

### Step 2 — Component reads critical data now, streams the rest

```tsx
export function ProductDetail() {
  const { product, reviews } = useLoaderData() as {
    product: Product;
    reviews: Promise<Review[]>;
  };

  return (
    <article>
      <h1>{product.name}</h1>
      <p>{product.description}</p>

      <Suspense fallback={<ReviewsSkeleton />}>
        <Await resolve={reviews} errorElement={<p>Couldn't load reviews.</p>}>
          {(loaded) => <ReviewList reviews={loaded} />}
        </Await>
      </Suspense>

      <AddReviewForm />
    </article>
  );
}
```

The product renders immediately; the reviews section shows a skeleton until its promise settles, and a per-section error if it rejects — the whole page never blocks on the slow request. This is the [`suspense`](../concurrent/suspense.md) machinery (`<Suspense>` + a boundary) applied at the route layer; `<Await>` is the router's unwrap, and under React 19 you can swap it for [`use()`](../concurrent/use-and-promises.md) in a child component if you prefer.

### Step 3 — The action and the `<Form>`

A mutation is an `action`. The `<Form>` submits to it, the router runs it, then **auto-revalidates the loader** so the new review appears — no manual refetch:

```tsx
export async function action({ request, params }: { request: Request; params: { id: string } }) {
  const formData = await request.formData();
  const body = formData.get("body");
  const res = await fetch(`/api/products/${params.id}/reviews`, {
    method: "POST",
    body: JSON.stringify({ body }),
    headers: { "Content-Type": "application/json" },
  });
  if (!res.ok) return { error: "Could not post review" }; // returned → useActionData
  return redirect(`/products/${params.id}`); // or just return null to stay + revalidate
}

function AddReviewForm() {
  const navigation = useNavigation();
  const submitting = navigation.state === "submitting";
  return (
    <Form method="post">
      <textarea name="body" required />
      <button type="submit" disabled={submitting}>
        {submitting ? "Posting…" : "Post review"}
      </button>
    </Form>
  );
}
```

`navigation.state === "submitting"` disables the button while the action is in flight — the router-native double-submit guard, the same concern the [`double-submit` recipe](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) handles at the component level. Note this `<Form>`/`action` is React Router's system, *not* React 19's `<form action={fn}>` — see the disambiguation below.

### Step 4 — Wire it up, lazily

Register the route with `lazy` so its code (component *and* loader *and* action) ships in its own chunk, loaded only when someone navigates there:

```tsx
{
  path: "products/:id",
  lazy: () => import("./productDetail"), // module exports Component, loader, action, ErrorBoundary
}
```

The module exports `Component`, `loader`, `action`, `ErrorBoundary` as named exports; the router wires them up. The win over wrapping the component in `React.lazy` is real: React Router loads the route chunk **in parallel with running the loader**, so data-fetching isn't gated on the JS download. `React.lazy` around a route component would serialize them — chunk first, *then* the effect can fetch (mistake 9).

## Real-world patterns

**Global pending indicator.** One `useNavigation` at the root drives an app-wide progress bar:

```tsx
function RootLayout() {
  const navigation = useNavigation();
  return (
    <>
      {navigation.state !== "idle" && <TopProgressBar />}
      <Outlet />
    </>
  );
}
```

**`useFetcher` for mutations without navigation.** A "favorite" toggle shouldn't change the URL or trip a full navigation. `useFetcher` calls a loader/action out-of-band and gives you its own `state` and `data`:

```tsx
function FavoriteButton({ id }: { id: string }) {
  const fetcher = useFetcher();
  const busy = fetcher.state !== "idle";
  return (
    <fetcher.Form method="post" action={`/products/${id}/favorite`}>
      <button disabled={busy}>{busy ? "…" : "♥"}</button>
    </fetcher.Form>
  );
}
```

Multiple fetchers run concurrently and independently — the pattern for lists where each row mutates on its own.

**`redirect` as an auth guard.** A loader can `throw redirect("/login")` before a protected route ever renders, which is cleaner than rendering-then-redirecting in an effect (and avoids the flash-of-protected-content the `auth/` recipe track targets):

```tsx
export async function loader() {
  const user = await getUser();
  if (!user) throw redirect("/login");
  return user;
}
```

**Loader + TanStack Query — the honest integration.** Loaders fetch on time; Query caches. Combine them: the loader *prefetches into* the Query cache (so the request starts at navigation time and the waterfall is flat), and the component reads via `useSuspenseQuery` (so it gets the cache, background refetch, and invalidation):

```tsx
export const loader = (queryClient: QueryClient) =>
  async ({ params }: { params: { id: string } }) => {
    // Kick off the fetch at navigation time; don't block on it here.
    queryClient.prefetchQuery(productDetailOptions(params.id));
    return null;
  };

function ProductDetail() {
  const { id } = useParams();
  const { data } = useSuspenseQuery(productDetailOptions(id!)); // reads the warm cache
  return <ProductCard product={data} />;
}
```

This is the production answer to "loaders *or* Query?" — it's *both*, and it's what [`tanstack-router`](./tanstack-router.md) *(planned)* makes even tighter with typed route context.

**Distinguishing error types.** `isRouteErrorResponse` narrows a thrown `Response` so you can branch 404 vs 500:

```tsx
function RouteError() {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    if (error.status === 404) return <NotFound />;
    return <p>Server error ({error.status}).</p>;
  }
  return <p>Something went wrong.</p>;
}
```

## React Router `<Form>` vs React 19 `<form action>` — don't conflate them

Both take `FormData`; they are different systems, and mixing them up is a real footgun:

| | React Router `<Form>` / route `action` | React 19 `<form action={fn}>` / [`actions`](../concurrent/actions.md) |
| --- | --- | --- |
| Scope | Route-level; tied to a URL | Component-level; no router needed |
| Triggers | The route's `action`, then loader revalidation | A client (or server) function, inside a transition |
| Pending | `useNavigation().state` | `useFormStatus()` / `useActionState` |
| Use when | The mutation belongs to a route and should refresh route data | A local form mutation with no navigation |

You can use React 19 actions *inside* a React Router app for non-navigational forms; use the router's `<Form>` when the submit is a navigation-shaped mutation that should revalidate loaders.

## API / type reference

| Symbol | Kind | Notes |
| --- | --- | --- |
| `createBrowserRouter(routes)` | fn | Data-mode entry; pair with `<RouterProvider router>` |
| route object | `{ path, element/Component, loader, action, errorElement/ErrorBoundary, children, index, lazy, HydrateFallback }` | `Component`/`ErrorBoundary` are the component-typed forms (used by `lazy`) |
| `loader({ params, request })` | fn | Runs before render; `return` data, `throw new Response/redirect`, or return an object with promise keys to defer |
| `action({ params, request })` | fn | Runs on `<Form>` submit; `return` data (→ `useActionData`) or `redirect` |
| `useLoaderData` / `useRouteLoaderData(id)` | hook | Route data; the second reads an ancestor route by id |
| `useActionData` / `useNavigation` | hook | Action result; `state: 'idle'\|'loading'\|'submitting'` |
| `useRouteError` / `isRouteErrorResponse` | hook / guard | Read + narrow a route error |
| `useFetcher` | hook | Loader/action calls without navigation; own `state`/`data`/`Form` |
| `useParams` / `useSearchParams` | hook | URL path params / read-write query string (URL-as-state) |
| `<Form>` / `<Await>` / `<Outlet>` / `<Link>` / `<NavLink>` | components | Route form / defer-unwrap / nested-route slot / navigation / navigation with active state |
| `redirect(to)` | fn | Return or throw from loaders/actions |
| `defer` | **removed in v7** | Return the object with bare promise keys instead |

## Common mistakes

1. **Fetching in `useEffect` on the route component.** Re-introduces the navigation waterfall the loader exists to remove. If a route needs data to render, that's a loader.
2. **Returning a single bare promise to defer.** A loader deferral must be an *object with keys* (`return { reviews }`) — a lone `return somePromise` is awaited, not streamed.
3. **Expecting the wrong error channel.** A React `<ErrorBoundary>` will not catch a loader/action throw (`errorElement` + `useRouteError` does), and `errorElement` won't catch an event-handler error inside a component. Two systems; pick the right one.
4. **Treating loaders as a cache.** They refetch on every navigation with no dedup or staleness. For shared, cached, background-refreshed data, prefetch into TanStack Query and read from the cache.
5. **Plain `<form>` instead of `<Form>`.** A native form does a full-page POST and reload; the router's `<Form>` runs the action client-side and revalidates. (Native `<form>` is right for a *non*-JS fallback, not for a data-mode mutation.)
6. **No `useNavigation` pending UI.** During a slow loader the old screen just sits there frozen — users think it broke. Drive at least a top bar off `navigation.state`.
7. **Throwing a plain object/string.** `throw new Response(msg, { status })` (or an `Error`) gives `useRouteError` something typed; `isRouteErrorResponse` then branches on status. A bare `throw "nope"` loses all of that.
8. **Awaiting deferred data in the loader.** `await`ing the reviews promise defeats deferral — the route blocks on it again. Await only critical data; return the rest unresolved and wrap it in `<Suspense>` + `<Await>`.
9. **`React.lazy` for route components.** It serializes chunk-download-then-fetch. Route-level `lazy` loads the chunk *in parallel* with the loader. Use the router's `lazy`, not React's, for routes.
10. **Building the router inside a component.** `createBrowserRouter` in a render body makes a new router (and loses all navigation state) every render. Create it once at module scope.

## How this evolved

React Router began as declarative JSX — `<Routes>`, `<Route>`, and fetch-inside-with-effects. v6.4 imported the Remix data model: `loader`/`action`/`errorElement` and the `defer`/`<Await>` deferral APIs, turning the router into a data layer. v7 (the baseline here) is the Remix *merger* — the framework and the router became one project. Concretely for data mode: the package consolidated to `react-router`; the `defer()` wrapper and the old `json()` helper were **removed** in favor of returning plain objects and bare promises (loaders return native values, the router handles serialization); route `lazy` gained a granular per-property form (7.5+) so a loader needn't wait on the whole route module; and typegen brought typed loader/action data — though that lives mostly in framework mode. Framework mode and full RSC-style server rendering are [`nextjs-and-rsc-in-practice`](../server/nextjs-and-rsc-in-practice.md)'s territory *(planned)*; data mode is the SPA baseline this project standardizes on.

## Exercises

1. **Kill a waterfall.** Take a route that fetches in `useEffect` and move it to a loader. Compare the Network timeline: the request should now start at navigation, not after the component mounts. *Hint: the loader runs at route-match time; the component just reads `useLoaderData`.*
2. **Stream the slow part.** Split a route's data into critical (product) and non-critical (reviews); await the first, defer the second with `<Suspense>` + `<Await>` and a skeleton. *Hint: return `{ product, reviews }` where `reviews` is an un-awaited promise.*
3. **Two-status error UI.** Write an `errorElement` that shows a distinct 404 page vs a generic 500. *Hint: `throw new Response(_, { status })` in the loader, then `isRouteErrorResponse(error)` in the boundary.*

## Summary

- Data mode makes the router a `URL → (data + mutation + error + pending)` layer: `loader` before render, `action` for mutations, `errorElement` for failures, `lazy` for the code split.
- Loaders move fetching to navigation time, flattening the fetch-in-effect waterfall — but a loader is *timing, not a cache*; server state still wants TanStack Query, and the two compose via prefetch-into-cache.
- The navigation lifecycle is match → load (parallel) → wait for critical → commit, with `useNavigation().state` as the pending signal and automatic loader revalidation after actions.
- Route errors (loader/action/render) go to `errorElement` + `useRouteError`; React render errors go to a React error boundary. Two channels, not one.
- Deferred data returns bare promises (no `defer()` in v7), unwrapped with `<Suspense>` + `<Await>` (or `use()`).
- React Router's `<Form>`/`action` is not React 19's `<form action>` — same `FormData`, different systems; choose by whether the mutation is navigation-shaped.

## See also

- [`suspense`](../concurrent/suspense.md) — the `<Suspense>` + boundary machinery `<Await>` plugs into, and route-level code-splitting
- [`error-boundaries`](../rendering/error-boundaries.md) — the *other* error channel; how it differs from `errorElement`
- [`data-fetching-tanstack-query`](./data-fetching-tanstack-query.md) — why loaders aren't a cache, and the prefetch-into-cache integration
- [`actions`](../concurrent/actions.md) — React 19's form actions, disambiguated from the router's `<Form>`
- [`forms-controlled-and-uncontrolled`](../forms/forms-controlled-and-uncontrolled.md) — the uncontrolled-via-`FormData` model both action systems build on
- [`tanstack-router`](./tanstack-router.md) — the type-safe alternative and when it wins *(planned)*

## References

- React Router — [Data mode / route object](https://reactrouter.com/start/data/route-object) and [`loader`](https://reactrouter.com/start/data/data-loading)
- React Router — [Streaming with Suspense](https://reactrouter.com/how-to/suspense) (the v7 deferred-data model)
- React Router — [Lazy loading blog post](https://remix.run/blog/faster-lazy-loading) (granular route `lazy`)
- React Router — [Error handling](https://reactrouter.com/how-to/error-boundary) and `useRouteError`

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).