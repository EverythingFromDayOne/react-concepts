---
recipe_id: hydration-mismatch
track: ssr-and-rsc
primary_concept: server/ssr-and-hydration
difficulty: intermediate
react_baseline: "19.2"
related:
  - server/ssr-and-hydration
  - server/nextjs-and-rsc-in-practice
  - effects/escape-hatches-audit
  - concurrent/suspense
status:
  drafted: true
  reviewed: false
---

# The page flashes and the console floods with hydration errors

> **What you'll build:** an SSR page whose first client render is byte-identical to the server HTML — no `Hydration failed` errors, no flash where a date or theme snaps to a different value — by keeping environment-divergent data (time, locale, timezone, random, `localStorage`) out of the render that must match, and upgrading to live values only after mount.

## The scenario

A server-rendered product page shows three things: "Ships by {formatted delivery date}" in the user's locale, a "posted 3 minutes ago" relative timestamp, and a theme read from `localStorage` so there's no flash of the wrong theme. It's clean on localhost.

In production, users get a console **flood of hydration errors** and a visible **flash**: the timestamp and theme snap to a different value the instant the page becomes interactive. On React 19 the mismatched subtree is thrown away and re-rendered on the client, so the server-rendered content visibly flickers — you paid for SSR and got a client render plus a jolt.

Every value here diverges between the two environments:

- **Dates/relative time** — the server renders "3 minutes ago" at render time; the client hydrates a few seconds (or a CDN-cache-hit later, minutes) later and computes "4 minutes ago." Different text.
- **Locale/timezone** — the server (a datacenter on UTC, its own locale) formats the delivery date one way; the client (the user's timezone and browser locale) formats it another.
- **`localStorage` theme** — the server can't read `localStorage`, so it renders the default theme; the client reads the stored theme and renders the other one.

Why it escaped QA:

- On localhost, **server and client are the same machine** — same timezone, same locale, and the render-to-hydration gap is milliseconds — so the values match and nothing mismatches.
- The dev hydration overlay may fire, but "it looks fine" so it's dismissed.
- The bug only surfaces where the server's timezone/locale differ from a real user's, and where the render-to-hydration gap is real (CDN, network) — production conditions QA on one laptop never reproduces.

## Walkthrough

### Stage 1 — Name it: hydration is a contract, and you broke it with environment-divergent data

Hydration reuses the server's DOM and only attaches event handlers; it does **not** re-render to reconcile on first paint. So the first client render must be **identical** to the server HTML — [the agreement contract](../../server/ssr-and-hydration.md#the-agreement-contract-and-why-mismatches-are-expensive). The mismatch isn't a bug in the *value*; it's that the value is computed from data that differs between server and client — [the anti-pattern that guarantees a mismatch](../../server/ssr-and-hydration.md#the-anti-pattern-that-guarantees-a-mismatch). The rule: **the first render must be a pure function of data available identically on both sides.** Time, `Math.random()`, `typeof window`, the browser's locale, and `localStorage` are all *not* that.

### Stage 2 — Categorize the divergence, and reject the blanket silencer

Sort every offending value into one of two buckets:

- **Can be made server-deterministic** — dates, locale/timezone formatting, IDs. Fix by feeding the render identical data on both sides (below). Most mismatches are here.
- **Genuinely client-only** — a live relative clock, a value that lives only in `localStorage`. Fix by rendering the same fallback on both passes and upgrading after mount.

Do **not** reach for `suppressHydrationWarning` as a general fix. It silences the *warning* without fixing the *divergence* — the client still swaps to a different value; you've only hidden the flash's paper trail. It has exactly one legitimate use (a single element you *intend* to differ, like a timestamp) and it's one level deep.

### Stage 3 — Make the deterministic values match

- **Dates:** compute the absolute value on the server and pass it as data (an ISO string), and format it with a **server-decided locale/timezone** so both sides produce the same string:

```tsx
// locale/timezone are chosen on the SERVER (Accept-Language / a cookie / the user profile)
// and passed as props, so the client formats identically instead of using the browser's.
function DeliveryDate({ isoDate, locale, timeZone }: { isoDate: string; locale: string; timeZone: string }) {
  return (
    <time dateTime={isoDate}>
      {new Intl.DateTimeFormat(locale, { timeZone, dateStyle: "medium" }).format(new Date(isoDate))}
    </time>
  );
}
```

- **Theme:** read it from a **cookie**, not `localStorage` — a cookie is sent with the request, so the server renders the correct theme with zero flash and zero mismatch (in the App Router, `(await cookies()).get("theme")` — [cookies are request data](../../server/nextjs-and-rsc-in-practice.md#real-world-patterns)). `localStorage` is invisible to the server; a cookie is not.
- **IDs:** [`useId`](../../ecosystem/accessibility-in-react.md#useid-and-why-not-a-counter), which is SSR-stable by construction — never `Math.random()` or a module counter.

### Stage 4 — Gate the genuinely client-only values, and verify

For a value that *cannot* be server-deterministic — the live "3 minutes ago" — [render the same on both passes, then update in an effect](../../server/ssr-and-hydration.md#pattern-1--render-the-same-on-both-passes-then-update-in-an-effect): the first render (server *and* first client render) shows the stable absolute date; after hydration the client upgrades to the relative string.

```tsx
function PostedAt({ isoDate }: { isoDate: string }) {
  const [relative, setRelative] = useState<string | null>(null); // identical on server + first client render
  useEffect(() => {
    const tick = () => setRelative(formatRelative(isoDate));
    tick();
    const id = setInterval(tick, 60_000);
    return () => clearInterval(id);
  }, [isoDate]);
  return <span>{relative ?? new Date(isoDate).toISOString().slice(0, 10)}</span>;
}
```

For values backed by an external store (a media query, `localStorage` you must read on the client), [`useSyncExternalStore` with `getServerSnapshot`](../../server/ssr-and-hydration.md#pattern-2--usesyncexternalstore-with-getserversnapshot) makes the server value explicit instead of accidental — the cleaner primitive than a `mounted` flag.

**Verify the loop.** Run the server with `TZ=UTC` and a non-US `LANG`, then load the page in a US-timezone, US-locale browser: no `Hydration failed` in the console, no flash — the delivery date, theme, and timestamp all render correctly on first paint and stay stable, and the relative time upgrades smoothly a beat after hydration. The mismatch is gone because the first render no longer depends on which environment produced it.

## Variations

1. **Theme, three ways.** Cookie (server renders the right theme, zero flash — the clean answer) vs. a blocking inline `<script>` that sets `data-theme` before paint vs. `suppressHydrationWarning` on the `<html>` element. Prefer the cookie.
2. **`suppressHydrationWarning` — the one legitimate use.** A single element whose content you *accept* will differ (a rendered-at timestamp), one level deep. Not a tree-wide silencer.
3. **`useSyncExternalStore` + `getServerSnapshot`.** The correct primitive for external-store values (media queries, storage) — the [escape-hatch](../../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely) that names the server snapshot explicitly.
4. **Server-decided locale/timezone as data.** The general pattern behind every i18n mismatch: decide on the server from the request, pass as props, format identically on both sides.
5. **React 19 recovery + Suspense.** How 19 logs the diff and client-re-renders the mismatched subtree, and how a [Suspense boundary localizes](../../concurrent/suspense.md#real-world-patterns) the recovery so one bad subtree doesn't reset more than it must — see also [the SSR error pipeline](../../server/ssr-and-hydration.md#the-ssr-error-pipeline).

## Trade-offs and common pitfalls

1. **`Date.now()` / `new Date()` in render** — server time ≠ client time. Pass a server-computed absolute value; compute relative time after mount.
2. **Server locale/timezone ≠ client** — formatted dates and numbers differ. Decide on the server, pass as data.
3. **`Math.random()` / `crypto.randomUUID()` in render** — different per environment. `useId`, or generate server-side and pass as data.
4. **Reading `localStorage` / `window` during render** — the server renders the default and the client differs. Cookie (server-readable) or a mount gate.
5. **`typeof window !== "undefined"` branch in render** — the server renders one branch, the client the other. Render the same, adjust in an effect.
6. **`suppressHydrationWarning` as a general fix** — hides a real divergence; the client still swaps the value.
7. **A `mounted` gate on everything** — throws away SSR for content that could have been server-deterministic. Gate only the truly client-only parts.
8. **Invalid HTML nesting** (`<div>` in `<p>`, `<tr>` without `<tbody>`) — the browser corrects the DOM before hydration, so React's tree can't match. Fix the markup.
9. **Testing only on localhost** — server and client share timezone/locale; the mismatch never appears. Test with a divergent `TZ`/`LANG`.
10. **Ignoring the dev hydration overlay** — it's reporting a real production bug, not dev noise.
11. **Non-deterministic data order** — an unsorted API array, `Set`/`Map` iteration, or object-key order rendered differently between passes. Sort/stabilize before render.
12. **Third-party or extension DOM mutation** — less controllable, but don't render *your own* content that itself differs between environments.

### When NOT to eliminate the difference

If the value is **intended** to differ per environment — a live local clock that must show the user's time immediately, where you accept that the server renders a placeholder — then a mount gate or a narrowly-scoped `suppressHydrationWarning` on that one element is the correct *design*, not a bug. And if you're **not doing SSR at all** — a pure client-rendered SPA — there is no hydration contract, so `Date.now()` in render is fine (one environment, nothing to match). The test: *is this render output required to be identical on server and client?* Only under SSR/hydration. A CSR app has no contract to break, and a fully-static value needs no special handling.

## See also

- [`ssr-and-hydration`](../../server/ssr-and-hydration.md#the-agreement-contract-and-why-mismatches-are-expensive) — the agreement contract, the guaranteed-mismatch anti-pattern, and the two render-the-same patterns this recipe applies.
- [`nextjs-and-rsc-in-practice`](../../server/nextjs-and-rsc-in-practice.md#real-world-patterns) — reading request data (cookies) on the server, the clean fix for theme.
- [`escape-hatches-audit`](../../effects/escape-hatches-audit.md#usesyncexternalstore--subscribing-to-external-state-safely) — `useSyncExternalStore` with a server snapshot for external-store values.
- [`suspense`](../../concurrent/suspense.md#real-world-patterns) — how a boundary localizes React 19's hydration-mismatch recovery.
- `ssr-and-rsc/use-client-sprawl` — the sibling RSC boundary bug *(planned)*.
- `ssr-and-rsc/server-action-stale-data` — the sibling revalidation bug *(planned)*.

## References

- React — the `Hydration failed` error and its common causes.
- MDN — `Intl.DateTimeFormat` (locale/timezone-explicit formatting).
- MDN — `Set-Cookie` (server-readable client preferences).

## Demo source

- `demos/ssr-and-rsc/hydration-mismatch/` — the date/theme/timestamp page mismatching under a divergent server `TZ`/`LANG`, then the server-data + cookie + mount-gate fix. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*