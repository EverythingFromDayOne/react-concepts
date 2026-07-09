---
recipe_id: use-client-sprawl
track: ssr-and-rsc
primary_concept: server/server-components
difficulty: intermediate
react_baseline: "19.2"
related:
  - server/server-components
  - server/nextjs-and-rsc-in-practice
  - ecosystem/performance-profiling
  - ecosystem/data-fetching-tanstack-query
  - recipes/ssr-and-rsc/hydration-mismatch
status:
  drafted: true
  reviewed: false
---

# `"use client"` crept up the tree and shipped your whole page to the browser

> **What you'll build:** an RSC page that keeps its server components on the server — small client boundaries at the interactive leaves, heavy libraries and data fetching staying server-side — by understanding that `"use client"` marks a *boundary* (everything below it goes to the client) and pushing that boundary as far *down* as it will go, plus the slot pattern that lets a client wrapper hold server children.

## The scenario

A Next.js App Router product page. Someone needs a favorite button with an `onClick`, so they add `"use client"` — but they add it to `ProductPage` (or to the shared `components/index.ts` barrel that re-exports everything, or to the `Layout`). The button works. Ship it.

Weeks later the client bundle for that route has gone from ~90KB to ~400KB and the page is slow on mobile. The cause: `"use client"` on `ProductPage` turned **the entire subtree** — `ProductDetails`, `Reviews`, `Recommendations`, and everything *they* import — into client components. The markdown renderer that formatted the description, the date/i18n library, the syntax highlighter in the reviews — all server-appropriate, all now shipped to the browser. Data that was fetched on the server now needs a client fetch. A component that read a server-only value can't anymore. The page renders and works perfectly; it's just secretly a client-rendered app wearing RSC's clothes.

Why it escaped QA:

- **There's no functional bug.** The page renders, the button clicks, every test passes. It's an architecture and bundle regression, and nobody runs the bundle analyzer in QA.
- **`"use client"` always makes the error go away.** A hook-in-a-server-component error, a "this only works in a client component" — adding the directive to the nearest ancestor fixes it every time, so it's the path of least resistance.
- **It compounds silently.** Each `"use client"` looks harmless in its own PR; the aggregate is a route where 90% of a tree that should be server is client. `"use client"` count only ever goes up.

## Walkthrough

### Stage 1 — Name it: `"use client"` is a boundary, not a label

`"use client"` doesn't mark *one component* as client — it marks a **boundary**, and everything a client component imports, transitively, is bundled to the client ([the module-graph split is where "zero bundle" comes from, and where it's lost](../../server/server-components.md#the-module-graph-split--where-zero-bundle-comes-from)). So a directive high in the tree — on a page, a layout, or a barrel that re-exports the whole component library — drags its entire subtree client-side. Server is the default; `"use client"` is an **opt-out you push as far down as possible**, to the single leaf that actually needs browser interactivity. Every `"use client"` costs bundle size and gives up server-only powers (server data fetching, secrets, zero-bundle rendering), so it belongs at the smallest interactive leaf — never "up, to be safe."

### Stage 2 — Find the real boundary, and reject the reflexes

Three reflexes cause the sprawl:

- **`"use client"` on a barrel `index.ts`.** Catastrophic — it poisons *every* re-export, so importing one interactive component pulls the whole barrel client-side. Never on a barrel.
- **`"use client"` on a layout or page.** Makes the whole route client. The interactivity was in one button.
- **`"use client"` "to fix the hook error."** The error is telling you a *leaf* is interactive ([Server Components can't be interactive](../../server/server-components.md#why-server-components-cant-be-interactive)). Mark that leaf, not its ancestor.

Diagnose with the [bundle analyzer / the RSC boundary in devtools](../../ecosystem/performance-profiling.md#real-world-patterns): if a heavy server-only library is in your *client* bundle, a `"use client"` is sitting too high above it.

### Stage 3 — Push the boundary down to the interactive leaf

Split the interactive bit into its own small client component; keep the parent a server component that fetches and renders on the server.

```tsx
// ProductPage.tsx — a SERVER component (no directive): async, fetches on the server
export default async function ProductPage({ id }: { id: string }) {
  const product = await getProduct(id); // server-only fetch, secrets stay server-side
  return (
    <article>
      <ProductDetails product={product} /> {/* server — markdown/i18n libs stay server */}
      <FavoriteButton productId={id} />     {/* client leaf — the only boundary */}
      <Reviews productId={id} />            {/* server, can stream */}
    </article>
  );
}
```

```tsx
// FavoriteButton.tsx — the ONLY "use client" in this tree
"use client";
import { useState } from "react";

export function FavoriteButton({ productId }: { productId: string }) {
  const [favorited, setFavorited] = useState(false);
  return <button onClick={() => setFavorited((f) => !f)}>{favorited ? "★" : "☆"}</button>;
}
```

A server component importing and rendering a client leaf **is** the boundary, placed at the leaf. The heavy libraries and the data fetching stay on the server; only the button's kilobyte of interactivity ships.

### Stage 4 — Keep server components inside a client wrapper via the slot, and verify

Sometimes a client component genuinely has to *wrap* server content — a client-side `<Tabs>`, a `<Collapsible>`, a context `<Provider>`. Don't make the content client to do it: pass server components as **`children`** ([the slot pattern](../../server/server-components.md#stage-2--a-server-component-inside-a-client-component-the-slot)). The children are rendered on the server and passed in as already-rendered RSC payload; the client wrapper renders `{children}` without turning them client.

```tsx
// ClientTabs.tsx — a client wrapper
"use client";
import { useState, type ReactNode } from "react";

export function ClientTabs({ tabs }: { tabs: ReactNode[] }) {
  const [active, setActive] = useState(0);
  return (
    <>
      <nav>{tabs.map((_, i) => <button key={i} onClick={() => setActive(i)}>Tab {i + 1}</button>)}</nav>
      {tabs[active]} {/* a server component, passed in — stays server */}
    </>
  );
}
```

```tsx
// page.tsx — server: hands SERVER components to the client wrapper as props/children
<ClientTabs tabs={[<ServerHeavyPanel />, <ServerReviews />]} />
```

Put context providers high but let server children flow *through* them the same way. The boundary stays thin even when a client component must wrap server stuff.

**Verify the loop.** Run the bundle analyzer before and after: the heavy server-only libraries are gone from the client bundle (400KB back to ~90KB), the RSC devtools boundary shows most of the tree server-rendered again, a server-only API in the now-server parent no longer errors, and the `"use client"` count has dropped to the genuinely interactive leaves. The page behaves identically — you just stopped shipping it.

## Variations

1. **Client wrapper, server children (the slot).** The key technique — a provider or interactive shell wrapping server content via `children`/props so the content stays server.
2. **Barrel-file poisoning.** Why `"use client"` on `index.ts` is a footgun; keep client components out of shared barrels, or split client/server barrels.
3. **Move data fetching back to the server.** A component that went client *to fetch data* should instead be an async server component that fetches on the server and hands data to a small client leaf — no client waterfall, no exposed endpoint ([server as the source of truth](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns)).
4. **`"use client"` vs `"use server"`.** Different directives — a client boundary vs. a server-action/function boundary. Mixing them up is common; `"use server"` does not make something a client component.
5. **A third-party component with no directive.** A library component that uses hooks but ships no `"use client"` — wrap it in your own thin `"use client"` file rather than marking your page client to accommodate it.

## Trade-offs and common pitfalls

1. **`"use client"` on a page/layout/route root** — the whole route becomes client. Mark the leaf.
2. **`"use client"` on a barrel `index.ts`** — poisons every re-export. Never.
3. **`"use client"` to silence a hook error on an ancestor** — the error means a *leaf* is interactive; mark the leaf.
4. **Pushing the boundary up "to be safe"** — every "just in case" directive is bundle size plus lost server-only powers.
5. **Making content client because a wrapper is client** — use the `children` slot; a client wrapper can hold server children.
6. **Heavy server-only libraries dragged client-side** (markdown, highlighter, date/i18n) — measure the bundle; they're the tell.
7. **Fetching data client-side that could be fetched on the server** — waterfalls and exposed endpoints. Fetch in the async server parent.
8. **Assuming `"use client"` is per-component** — it's a boundary; it's transitive downward through imports.
9. **Confusing `"use client"` with `"use server"`** — different directives, different purpose.
10. **A server-only API** (`cookies()`, `fs`, a DB client) **accidentally in a client subtree** — a build/runtime error whose real fix is shrinking the boundary, not removing the API.
11. **Never measuring** — the sprawl is invisible without the bundle analyzer / RSC devtools; add a bundle budget to CI.
12. **Marking your page client to use a hook-based third-party component** — make a thin `"use client"` wrapper file for *it* instead.

### When NOT to fight the boundary

If a page is **genuinely, thoroughly interactive** — a dashboard, an editor, a canvas app where nearly everything responds to client state — then a high client boundary is honest, and contorting a mostly-interactive tree into server components plus slots for a few static bits is premature. RSC pays off when a *meaningful* part of the tree is static or server-appropriate; a fully interactive app is legitimately mostly client. And if you're **not on an RSC framework** (a Vite SPA) there is no `"use client"` and no boundary — everything is client by definition, and this recipe doesn't apply. The test: *is a meaningful part of this subtree static or server-only?* If yes, push the boundary down to protect it. If the whole thing is interactive, a high boundary is correct.

## See also

- [`server-components`](../../server/server-components.md#the-module-graph-split--where-zero-bundle-comes-from) — the module-graph split that makes `"use client"` a transitive boundary, and the slot pattern for server children in a client wrapper.
- [`nextjs-and-rsc-in-practice`](../../server/nextjs-and-rsc-in-practice.md#real-world-patterns) — where the boundary lives in a production App Router page and the three-tier model.
- [`performance-profiling`](../../ecosystem/performance-profiling.md#real-world-patterns) — measuring the client bundle so the sprawl stops being invisible.
- [`hydration-mismatch`](./hydration-mismatch.md) — the sibling SSR bug in the same track.
- `ssr-and-rsc/server-action-stale-data` — the sibling Server Action / revalidation bug *(planned)*.

## References

- React — the `"use client"` and `"use server"` directives and the server/client boundary.
- Next.js — Server and Client Components, and composition patterns (passing Server Components to Client Components).
- `@next/bundle-analyzer` — inspecting what actually ships to the client.

## Demo source

- `demos/ssr-and-rsc/use-client-sprawl/` — the product page with `"use client"` on the page (bloated bundle), then the boundary pushed to a `FavoriteButton` leaf plus a slot-based `ClientTabs`. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*