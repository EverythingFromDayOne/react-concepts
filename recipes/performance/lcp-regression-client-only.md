---
recipe_id: lcp-regression-client-only
track: performance
primary_concept: ecosystem/performance-profiling
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/performance-profiling
  - server/ssr-and-hydration
  - concurrent/suspense
  - server/nextjs-and-rsc-in-practice
  - recipes/performance/route-splitting-bundle
status:
  drafted: true
  reviewed: false
---

# The hero paints five seconds late because the page is client-rendered

> **What you'll build:** a content page whose largest above-the-fold element is in the initial server response and paints before the JavaScript even runs — turning a 5-second client-rendered LCP into a sub-2.5s server-rendered one — by getting the LCP content into the HTML (SSR/SSG/streaming) and prioritizing its resource, instead of shipping an empty `<div id="root">` and rendering everything on the client.

## The scenario

A marketing/product landing page built as a Vite SPA — client-only rendered. The hero (a large image, a headline, a CTA) is the LCP element. In the field, on a mid-tier phone on 4G, **LCP is ~5 seconds**: the browser gets an empty shell, then downloads ~400KB of JavaScript, parses and executes it, React mounts, a client-side fetch for the hero content resolves, and *only then* does the hero paint. Google's Core Web Vitals flag the URL; the drop in LCP correlates with a measurable dip in organic traffic and conversion.

The initial HTML is literally this:

```html
<div id="root"></div>
<script src="/assets/index-abc123.js"></script>
```

There is nothing for the browser to paint until the whole chain — download JS → parse → execute → mount → fetch → render — completes, because the LCP element doesn't exist in the DOM until the very end of it.

Why it escaped QA:

- In **dev** (warm cache, localhost, fast CPU) LCP is ~200ms — the JS and data are instant.
- **Lab Lighthouse on desktop** passes; the regression lives in the field, on real mobile and slow networks.
- The team watches lab scores, not **field data** (CrUX / RUM), so the 5-second mobile LCP is invisible until Search Console reports it.

## Walkthrough

### Stage 1 — Name it: with client-only rendering, nothing can paint until JS runs

LCP is the moment the largest above-the-fold element paints. With client-only rendering the initial HTML is empty, so the LCP element can't paint until JavaScript downloads, parses, executes, mounts React, and (often) resolves a data fetch — an entire serial chain before the content exists ([the CSR blank-then-paint pattern is visible in the Performance panel](../../ecosystem/performance-profiling.md#real-world-patterns)). To improve LCP you have to get the LCP element into the **initial server response** (or at minimum make its resource discoverable and high-priority early). And measure in the **field** — [LCP's threshold is 2.5s / 4.0s on real devices](../../ecosystem/performance-profiling.md#walkthrough--the-measurement-loop-end-to-end), not on your laptop.

### Stage 2 — Reject the fixes that don't put content in the HTML

- **Code-split the JavaScript.** Smaller JS helps TTI ([the route-splitting recipe](./route-splitting-bundle.md)), but the LCP element still doesn't exist until JS runs — you've shortened one link of the chain, not removed the chain.
- **Show a skeleton in the shell.** It paints *something* early, but a skeleton isn't the LCP content — and worse, a big gray placeholder can *become* the LCP element, so you've optimized the metric for painting a gray box.
- **Preload the JS bundle.** Faster JS, still no early content.

The real fix is to render the actual above-the-fold content on the server so it's in the HTML the browser receives.

### Stage 3 — Server-render the above-the-fold content

Get the LCP element into the initial response. By architecture:

- **Static generation (SSG)** for a marketing page — the hero HTML is baked at build time and served instantly. The cheapest, best LCP win for static content.
- **Server rendering (SSR/RSC)** for dynamic content — [render the hero on the server](../../server/nextjs-and-rsc-in-practice.md#step-1--the-three-tiers-on-one-page) and hydrate interactivity after, so the browser paints the hero before any JS executes.

```tsx
// a server component (or SSG page): the hero is in the initial HTML, not a client fetch
export default async function Landing() {
  const hero = await getHero(); // fetched on the server → in the response
  return (
    <section>
      <img src={hero.image} alt={hero.alt} width={1200} height={600} fetchPriority="high" />
      <h1>{hero.headline}</h1>
      <CTAButton /> {/* a small client leaf for interactivity */}
    </section>
  );
}
```

The key move is that the hero no longer waits on a *client* fetch — the data is fetched on the server and the element ships in the HTML. For above-the-fold content that genuinely needs slow data, [stream it](../../server/ssr-and-hydration.md#streaming-and-selective-hydration): server-render the hero immediately and put the slow, below-the-fold parts behind [Suspense boundaries](../../concurrent/suspense.md#real-world-patterns) so they stream in without blocking the LCP paint.

```tsx
<Hero /> {/* above the fold — in the shell, paints first */}
<Suspense fallback={<BelowFoldSkeleton />}>
  <SlowRecommendations /> {/* streams after; never blocks LCP */}
</Suspense>
```

### Stage 4 — Prioritize the LCP resource, and verify

Even server-rendered, the LCP *image* has to be discovered and downloaded early:

- **Never `loading="lazy"` on the above-the-fold hero image** — the classic LCP killer; it defers the very resource LCP measures. Load it eagerly and mark it `fetchPriority="high"`, or `<link rel="preload" as="image" fetchpriority="high">` it in the head.
- **Size it** (`width`/`height` or `aspect-ratio`) so it doesn't cause a layout shift as it loads.
- **Serve a modern format and a responsive `srcset`** so the download itself is small.
- **Remove render-blocking resources** from the critical path in the head.

**Verify the loop.** Under field-realistic conditions (4G throttle + mid-tier CPU): view-source shows the hero markup in the initial HTML, the Performance panel shows the hero painting *before* the main bundle finishes executing, and `web-vitals` `onLCP` (and later CrUX/RUM) reports LCP under 2.5s — not just a green lab score. The regression is closed because the LCP element is in the response, discovered early, and no longer gated on the client.

## Variations

1. **SSG for static marketing pages** — the LCP element in build-time HTML; the cheapest win when content isn't per-request.
2. **Streaming SSR + Suspense** for data-dependent above-the-fold — shell and hero now, slow parts streamed, so the LCP paint isn't blocked by the slow query.
3. **Prioritize the LCP image** — preload / `fetchPriority="high"` / no lazy-loading above the fold / responsive `srcset` / modern format.
4. **RSC server-fetch** — fetch the LCP data on the server so it's in the HTML instead of a client waterfall ([the three-tier model](../../server/nextjs-and-rsc-in-practice.md#step-1--the-three-tiers-on-one-page)).
5. **Keep the client boundary small** — server-render the content and hydrate only the interactive leaves ([`use-client-sprawl`](../ssr-and-rsc/use-client-sprawl.md)), so LCP content isn't gated on hydrating the whole page and INP stays good too.

## Trade-offs and common pitfalls

1. **Client-only rendering a content page** — empty HTML, nothing paints until JS runs. SSR/SSG the above-the-fold.
2. **LCP element behind a client data fetch** — an HTML→JS→fetch→paint waterfall. Server-fetch it.
3. **`loading="lazy"` on the hero image** — the classic LCP killer; load it eagerly and prioritize it.
4. **A skeleton becoming the LCP element** — you optimized for painting a gray box. Server-render real content.
5. **Code-splitting as the LCP fix** — helps TTI, but the LCP element still doesn't exist until JS runs.
6. **Not preloading / prioritizing the LCP image** — it's discovered late in the request chain.
7. **Render-blocking CSS/JS in the head** — delays first paint.
8. **No dimensions on the LCP image** — CLS as it loads (a different metric, but it ships alongside).
9. **Measuring LCP in the lab on desktop** — the regression is field- and mobile-only. Use CrUX/RUM.
10. **SSR but a giant hydration that blocks interaction** — good LCP, bad INP. Server-render *and* keep the client boundary small.
11. **A huge unoptimized LCP image** — slow to download even when prioritized. Modern format + `srcset`.
12. **Not measuring after** — confirm the LCP element is in the HTML and paints early under field-realistic throttling.

### When NOT to migrate for LCP

If the page is **behind auth and not first-impression-sensitive** — an internal dashboard, an app users log into daily — LCP matters far less than TTI and INP, and a full SSR migration for LCP alone may not pay off; a fast shell plus prioritized real content after auth is usually fine (though you'd still never lazy-load the LCP image). And if you're **already on SSR/RSC**, this is a *tuning* problem (prioritize the resource, don't lazy-load it), not a migration. The test: *is the largest above-the-fold content gated behind JS execution or a client fetch, on a page where first-impression speed matters (public / marketing / SEO)?* If yes, server-render it. For an internal post-auth app, INP and TTI are the metrics to chase instead.

## See also

- [`performance-profiling`](../../ecosystem/performance-profiling.md#real-world-patterns) — LCP thresholds, measuring in the field (`web-vitals`, CrUX/RUM), and the CSR blank-then-paint trace.
- [`ssr-and-hydration`](../../server/ssr-and-hydration.md#streaming-and-selective-hydration) — SSR and streaming that put the LCP content in the initial response.
- [`suspense`](../../concurrent/suspense.md#real-world-patterns) — streaming boundaries so slow below-the-fold content doesn't block the LCP paint.
- [`nextjs-and-rsc-in-practice`](../../server/nextjs-and-rsc-in-practice.md#step-1--the-three-tiers-on-one-page) — server-fetching the LCP data and the static/streamed tier model.
- [`route-splitting-bundle`](./route-splitting-bundle.md) — the bundle-size sibling; smaller JS helps the chain but doesn't put content in the HTML.

## References

- web.dev — Largest Contentful Paint, optimizing LCP, and `fetchpriority`.
- MDN — `<link rel="preload">`, `fetchPriority`, and responsive images (`srcset`).
- `web-vitals` — `onLCP` and field measurement.

## Demo source

- `demos/performance/lcp-regression-client-only/` — the CSR landing page with a 5s field LCP, then SSG/streamed hero + a prioritized, eagerly-loaded LCP image, measured with `web-vitals` under 4G throttling. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*