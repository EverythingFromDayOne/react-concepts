---
recipe_id: virtualization-long-list
track: performance
primary_concept: ecosystem/performance-profiling
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/performance-profiling
  - rendering/memoization-and-the-compiler
  - ecosystem/data-fetching-tanstack-query
  - ecosystem/accessibility-in-react
  - recipes/performance/route-splitting-bundle
status:
  drafted: true
  reviewed: false
---

# Switching to the "All transactions" tab freezes the page for six seconds

> **What you'll build:** a long list that stays responsive at any length — the tab switch is instant, scrolling is smooth, and only the ~20 visible rows are ever in the DOM — by virtualizing (rendering just the visible window plus a small overscan) instead of mounting all 10,000 rows in one synchronous commit.

## The scenario

An admin transaction table renders every row the API returns. In production a busy account has **10,000 rows**. When the user clicks the "All transactions" tab, the page **freezes for ~6 seconds** — the tab is unresponsive, a spinner elsewhere on the page keeps spinning, clicks queue up and fire late. Once it finally paints, scrolling is janky.

The cause is raw volume: the list `.map()`s over all 10,000 rows and renders them, so React mounts 10,000 row components and inserts ~50,000 DOM nodes (cells, wrappers) in a **single synchronous commit**. That commit is one long task that blocks the main thread for seconds, and the resulting DOM is heavy enough that scrolling can't hit 60fps.

Why it escaped QA:

- Dev and staging seed **~20 rows**, which render instantly. Nobody tests with 10,000.
- The freeze is **proportional to row count** — invisible at seed volume, brutal at production volume.
- On a fast desktop it's a ~1s annoyance; on a mid-tier phone it's a 6s hang — and QA is on desktop.
- The rows *do* render (eventually), so every functional test passes.

## Walkthrough

### Stage 1 — Name it: the cost is proportional to rows *mounted*, and you mounted all of them

Mounting 10,000 rows is 10,000 reconciliations plus ~50,000 DOM insertions in one commit — [a long task that freezes the main thread](../../ecosystem/performance-profiling.md#real-world-patterns), which the Performance panel will show as a single multi-second block. The work is real; React can't optimize away a mount you asked for. And the user can only *see* about 15 rows at once. So render those, not ten thousand. **Virtualization** (windowing) mounts only the visible window plus a small overscan, keeping the mounted-node count constant no matter how long the list is. [Measure first](../../ecosystem/performance-profiling.md#walkthrough--the-measurement-loop-end-to-end) — confirm the long task on tab switch and the DOM node count before you change anything.

### Stage 2 — Reject the fixes that don't touch mount cost

- **`memo` / `useMemo` on rows.** These prevent *re-renders*; the freeze is the *initial mount* of 10,000 nodes, which memoization doesn't reduce ([memo's cost model is about re-render, not mount](../../rendering/memoization-and-the-compiler.md#cost-model)).
- **`content-visibility: auto` (CSS).** Helps the browser skip *layout/paint* for off-screen content, but React still mounts all 10,000 components and the DOM nodes still exist — the reconciliation cost remains. Partial, not the fix.
- **`startTransition` / deferring.** Makes the giant render *interruptible* so the UI doesn't lock, but the 10,000-mount work still happens — you've hidden the freeze, not removed it.

The only fix that removes the cost is not mounting what's off-screen.

### Stage 3 — Virtualize with a windowing library

Render a scroll container whose inner spacer is the full list height (so the scrollbar is correct), but mount only the rows in the visible window, absolutely positioned at their computed offsets. TanStack Virtual:

```tsx
import { useRef } from "react";
import { useVirtualizer } from "@tanstack/react-virtual";

function TransactionTable({ rows }: { rows: Transaction[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40, // row height in px
    overscan: 8,            // a few extra rows above/below so scroll doesn't flash
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      {/* full-height spacer: correct scrollbar, only the window is mounted */}
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map((item) => (
          <div
            key={rows[item.index].id}
            style={{ position: "absolute", top: 0, width: "100%", height: item.size, transform: `translateY(${item.start}px)` }}
          >
            <TransactionRow row={rows[item.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

Now ~20 rows are mounted at any moment regardless of whether the list is 100 or 100,000. The tab switch is instant, and scroll recycles the window as offsets change.

### Stage 4 — Dynamic heights, the trade-offs, and verify

Fixed-height rows are the easy case (`estimateSize` is a constant). For variable heights, measure each rendered row so offsets stay correct:

```tsx
// on each row element:
<div ref={virtualizer.measureElement} data-index={item.index} /* … */ />
```

Two trade-offs to own, because virtualization has real costs:

- **Find-in-page and anchors break.** Off-screen rows aren't in the DOM, so Ctrl+F, "scroll to this row" links, and browser find won't reach them. Provide an in-app search/jump instead of relying on the browser.
- **Accessibility needs the true size.** A screen reader sees only the windowed rows, so set `aria-rowcount` / `aria-rowindex` on a virtualized grid so assistive tech knows the [real list length](../../ecosystem/accessibility-in-react.md#real-world-patterns), not the ~20 mounted.

**Verify the loop.** Load 10,000 rows and switch to the tab: the Performance panel shows no long task over ~50ms (the multi-second block is gone), INP is good, scrolling holds 60fps, and the element inspector shows ~20 row nodes in the DOM at once instead of 10,000. The six-second freeze is gone because you stopped mounting rows nobody can see.

## Variations

1. **Fixed vs dynamic row height.** A constant `estimateSize` for uniform rows; `measureElement` for variable content so the scrollbar and offsets stay accurate.
2. **Virtualized grid/table.** Virtualize rows (and columns too, for very wide tables) with the total-size spacer and absolute positioning; sticky headers live outside the scroll window.
3. **Infinite scroll + virtualization.** They compose: virtualize the rendered window *and* fetch the next page as you near the end ([`useInfiniteQuery`](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns)) — you never hold or mount the whole list.
4. **Pagination as the alternative.** Discrete pages with controls avoid the find-in-page/a11y trade-offs entirely and are simpler; the right call when the UX doesn't need one continuous scroll.
5. **Server-side filtering instead of scroll-to-find.** Often the better answer is to not ship 10,000 rows — filter/paginate on the server and virtualize the (smaller) result.

## Trade-offs and common pitfalls

1. **Rendering all N rows** — the mount cost freezes the main thread. Virtualize.
2. **Reaching for `memo` to fix a mount freeze** — memo prevents re-renders, not the initial mount of N nodes.
3. **`content-visibility: auto` as the fix** — helps paint, but all the React components and DOM nodes still mount.
4. **A wrong `estimateSize` on dynamic rows** — the scrollbar jumps and offsets drift. Use `measureElement`.
5. **Ignoring find-in-page / anchors** — off-screen rows aren't in the DOM; provide an in-app search/jump.
6. **Broken screen-reader navigation** — set `aria-rowcount`/`aria-rowindex` so AT knows the true size.
7. **Forgetting the total-size spacer** — the scrollbar is wrong and you can't scroll to the end.
8. **Keys tied to array index** — the wrong row is recycled on filter/sort. Stable IDs.
9. **Heavy per-row work** — virtualization caps the *count*, but a slow row is still slow ×20. Keep rows cheap.
10. **Measuring the wrong scroll element / nested scrollers** — offsets drift. Point the virtualizer at the real scroll parent.
11. **Virtualizing a short list** — absolute positioning, measurement, and broken Ctrl+F for no benefit. Only virtualize when N is large.
12. **Not measuring after** — assuming it's faster. Confirm the long task is gone and only the window is in the DOM.

### When NOT to virtualize

If the list is **short** — dozens, not thousands — virtualization buys nothing and costs real complexity (absolute positioning, measurement, broken find-in-page, extra a11y work); just render it. If **discrete pages fit the UX** (search results, a data grid with page controls), pagination is simpler and sidesteps the find-in-page and accessibility trade-offs. And frequently the honest fix is on the **server** — don't send 10,000 rows; filter or paginate server-side. The test: *is the list long enough that mounting it all is a measurable freeze, and does the UX genuinely need one continuous scroll?* Both yes → virtualize. Short, pageable, or better-filtered-on-the-server → don't.

## See also

- [`performance-profiling`](../../ecosystem/performance-profiling.md#real-world-patterns) — measuring the long task and DOM node count before and after, so you confirm the freeze is gone.
- [`memoization-and-the-compiler`](../../rendering/memoization-and-the-compiler.md#cost-model) — why memo (re-render cost) can't fix a mount-cost freeze.
- [`data-fetching-tanstack-query`](../../ecosystem/data-fetching-tanstack-query.md#real-world-patterns) — infinite scroll and server-side pagination that compose with, or replace, virtualization.
- [`accessibility-in-react`](../../ecosystem/accessibility-in-react.md#real-world-patterns) — keeping a virtualized list accessible (`aria-rowcount`, in-app search).
- [`route-splitting-bundle`](./route-splitting-bundle.md) — the download-cost sibling; this is the render-cost half of the performance hub.

## References

- TanStack Virtual — `useVirtualizer`, `getVirtualItems`, `getTotalSize`, `measureElement`.
- web.dev — long tasks, INP, and rendering performance.
- MDN — `content-visibility` (and why it isn't a substitute for virtualization).

## Demo source

- `demos/performance/virtualization-long-list/` — the 10,000-row table freezing on tab switch, then virtualized to a constant-size window with dynamic heights and `aria-rowcount`, measured in the Performance panel. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*