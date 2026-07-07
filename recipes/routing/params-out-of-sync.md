---
recipe_id: params-out-of-sync
track: routing
primary_concept: ecosystem/routing-react-router
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/routing-react-router
  - state/usereducer-and-state-structure
  - ecosystem/tanstack-router
status:
  drafted: true
  reviewed: false
---

# URL params drift out of sync with component state

> **What you'll build:** you'll take a product listing whose filters vanish when the user hits Back, trace it to a `useState` mirror of the URL that never re-syncs, and fix it by making the URL the single source of truth — derived during render, not copied into state. This is [derive-don't-mirror](../../state/usereducer-and-state-structure.md#derive-vs-mirror--the-deep-dive) applied to the address bar, and it's the routing-track opener.

## The scenario

A product listing has a category filter and a page number. Both belong in the URL — they should survive a refresh, be shareable, and respond to Back. The team put them in the URL *and* mirrored them into `useState`, seeding the state from the URL on mount:

```tsx
// ProductList.tsx — mirrors the URL into state, and drifts
import { useState } from "react";
import { useSearchParams } from "react-router";

export function ProductList() {
  const [params, setSearchParams] = useSearchParams();

  // Seed local state from the URL — but only once, on mount.
  const [category, setCategory] = useState(params.get("category") ?? "all");
  const [page] = useState(Number(params.get("page") ?? "1"));

  function selectCategory(next: string) {
    setCategory(next);                    // update local state
    setSearchParams({ category: next });  // and the URL (which wipes `page`!)
  }

  return <Grid category={category} page={page} onSelect={selectCategory} />;
}
```

It demos fine: land on `?category=electronics&page=3` and the initializer reads it, so the grid is correct. Then the drift:

- **Back button.** A user filters to Electronics, paginates to page 3, opens a product, and hits Back. The URL correctly restores `?category=electronics&page=3` — but the component didn't remount, so the `useState` initializers never re-ran. The grid shows the *stale* state. The URL and the UI now disagree. Users report *"the Back button forgets my filters."*
- **A shared link mid-session.** A "Clear filters" button calls `setSearchParams({})`; the URL clears but the mirrored `category`/`page` state doesn't, so the grid keeps showing the old filter.
- **Lost params.** `setSearchParams({ category: next })` replaces the whole search string, silently dropping `page`.

Two sources of truth for one piece of state, drifting apart.

**Why it escaped QA:** the `useState` initializer reads the URL *on mount*, so every test that navigates *to* a URL and asserts on load passes — that's the happy path. The bug only appears when the URL changes **without a remount**: Back/Forward, a `<Link>` that changes only the query string, or a programmatic `navigate`. QA scripts that drive the app forward and assert on load never exercise it.

## Walkthrough

### Stage 1 — Name it

The URL is already state — durable, shared, history-aware state that the browser and router own. Copying it into `useState` creates a *second* copy with none of those properties, and now you have to keep two things in sync by hand. You can't, reliably: the URL changes through paths (Back, Forward, external links, `navigate`) that don't re-run your initializer. The mirror is derived state that should never have been stored.

### Stage 2 — The naive fix syncs with an effect

The reflex is to bridge the two with an effect:

```tsx
// ❌ mirror the URL into state on every change
useEffect(() => {
  setCategory(params.get("category") ?? "all");
  setPage(Number(params.get("page") ?? "1"));
}, [params]);
```

This is the textbook [derive-don't-mirror](../../state/usereducer-and-state-structure.md#derive-vs-mirror--the-deep-dive) anti-pattern, and [you might not need an effect](https://react.dev/learn/you-might-not-need-an-effect) names it directly. It adds a render (URL changes → commit → effect → setState → second commit), it lags for one frame, and if you also sync state→URL you get a loop. You're spending an effect to recompute a value you could just read.

### Stage 3 — Read the URL; don't store it

Delete the `useState`. Derive `category` and `page` during render straight from the URL, and write through `setSearchParams` — merging so you don't clobber sibling params:

```tsx
// ProductList.tsx — the URL is the single source of truth
import { useSearchParams } from "react-router";

export function ProductList() {
  const [params, setSearchParams] = useSearchParams();

  // Derived during render — always in step with the URL, on Back/Forward and all.
  const category = params.get("category") ?? "all";
  const page = Number(params.get("page") ?? "1");

  function selectCategory(next: string) {
    setSearchParams(
      (prev) => {
        prev.set("category", next);
        prev.set("page", "1"); // resetting page here is intentional, not a wipe
        return prev;
      },
      { replace: true }, // a filter tweak shouldn't spam the history stack
    );
  }

  return <Grid category={category} page={page} onSelect={selectCategory} />;
}
```

Now Back, Forward, a pasted link, and "Clear filters" all Just Work, because there's nothing to keep in sync — the render reads whatever the URL currently says. The functional `setSearchParams(prev => …)` preserves the params you're not touching, and `{ replace: true }` keeps each filter keystroke out of the history stack. This is the whole fix: the mirror is gone. ([React Router's](../../ecosystem/routing-react-router.md#api--type-reference) `useSearchParams` owns this API.)

### Stage 4 — Parse and validate the params

`params.get("page")` is a string or `null`; `Number("abc")` is `NaN`. Beyond trivial cases, validate on read so a mangled URL degrades gracefully instead of rendering `NaN`:

```tsx
const rawPage = Number(params.get("page"));
const page = Number.isInteger(rawPage) && rawPage > 0 ? rawPage : 1; // fallback
```

Hand-parsing every param gets old fast, which is exactly the friction [TanStack Router](../../ecosystem/tanstack-router.md#search-param-validation) removes: a `validateSearch` schema gives you typed, validated search params with fallbacks (`z.number().catch(1)`) and structural sharing — it *designs this whole failure class out*, because there's no untyped `params.get` to mishandle and no place to mirror. If URL state is central to your app, that's a strong reason to reach for it.

**Verify against the incident.** Reproduce it: filter to Electronics, go to page 3, click a product, hit Back. Before: the grid resets while the URL says page 3. After: the grid reads page 3 from the URL and matches it — no mirror to fall behind.

## Variations

- **A search box you're typing in.** Keep the *input's* immediate value in local `useState` (a controlled input has to be responsive), but derive the *committed* search that drives fetching from the URL, and debounce the write to `setSearchParams`. The rule holds: ephemeral keystrokes are local; the committed, shareable query is the URL.
- **Preserve vs. reset siblings.** Changing `category` should reset `page` to 1 (a deliberate write) but preserve unrelated params (sort, view mode). The functional updater lets you be precise about which you touch.
- **Push vs. replace.** Use `{ replace: true }` for filter tweaks so Back doesn't step through every filter state; use a normal push for navigations the user should be able to reverse.
- **TanStack Router.** `validateSearch` + `Route.useSearch()` makes the URL a typed store with `<Link search={(prev) => …}>` writes — the same "URL is the source of truth" model, with the parsing and drift designed away.

## Trade-offs and common pitfalls

1. **Mirroring URL params into `useState`** creates a second source of truth that drifts on Back/Forward/link changes. Derive from the URL during render.
2. **`useEffect` that syncs URL → state** is the derive-don't-mirror anti-pattern: an extra render, a frame of lag, and a loop risk if you sync both ways. Delete it and read the URL.
3. **`useState(() => params.get(...))` initializers** run once on mount, so any later URL change that doesn't remount is ignored. This is the exact shape of the bug.
4. **`setSearchParams({ page })` replacing the whole query** silently drops sibling params. Use the functional updater and set only what changed.
5. **Pushing history on every filter tweak** turns the Back button into a per-keystroke maze. `replace: true` for tweaks; push for real navigations.
6. **Putting ephemeral UI state in the URL** (an open accordion, a hover) produces ugly, shareable URLs and history spam. The [state-placement funnel](../../ecosystem/state-management-landscape.md#the-placement-funnel) decides what belongs there.
7. **Treating string params as typed** (`page` is `"3"`, not `3`) causes off-by-one and `NaN` bugs. Parse and validate on read.
8. **Not handling malformed params** (`?page=abc`) renders `NaN` or crashes. Provide a fallback (`.catch()`/default).
9. **A controlled input driven directly off the URL** with no local buffer lags on every keystroke (each one navigates and re-renders). Buffer the input locally; derive the committed value from the URL.
10. **A two-way sync effect** (state→URL *and* URL→state) fights itself into a render loop. Pick one source of truth — the URL — and there's nothing to sync.

### When NOT to put it in the URL

State belongs in the URL when it should **survive a refresh**, be **shareable**, or respond to **Back/Forward** — filters, pagination, tabs, sort, a selected item. State that has none of those needs stays local: a tooltip's open flag, a hover state, an in-progress unsaved form draft, a transient animation state. Forcing genuinely ephemeral UI state into the URL is the opposite mistake — it clutters the address bar and spams history. `useState` is right for the ephemeral; the URL is right for the durable and shareable. The drift bug comes from putting durable state in *both*.

## See also

- [Routing with React Router](../../ecosystem/routing-react-router.md#api--type-reference) — the `useSearchParams` read/write model this recipe leans on.
- [useReducer and state structure](../../state/usereducer-and-state-structure.md#derive-vs-mirror--the-deep-dive) — the derive-don't-mirror rule, here applied to the URL.
- [TanStack Router](../../ecosystem/tanstack-router.md#search-param-validation) — typed search params that design this failure out.
- [Recipe: back button loses scroll position](./back-button-scroll.md) *(planned)* — the sibling routing recipe; scroll restoration when the URL is the source of truth.
- [Recipe: lazy route flashes blank](./lazy-route-flashes-blank.md) *(planned)* — the route-code-splitting routing recipe.

## References

- React Router — [`useSearchParams`](https://reactrouter.com/api/hooks/useSearchParams)
- React docs — [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- TanStack Router — [Search Params](https://tanstack.com/router/latest/docs/guide/search-params)

## Demo source

*Demo pending — see the roadmap's demo-hosting decision (StackBlitz vs. CodeSandbox vs. local `demos/`).*