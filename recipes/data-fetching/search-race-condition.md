---
recipe_id: search-race-condition
track: data-fetching
primary_concept: effects-and-synchronization
difficulty: foundational
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Search Race Condition: Stale Results Overwrite Fresh Ones

> **What you'll build:** a search-as-you-type box that is *incapable* of showing results for a query the user is no longer looking at — built in stages: make the race visible, kill it with the ignore-flag, kill it properly with `AbortController`, debounce the synchronization (not the typing), pick the right race *policy* for the interaction, and know the exact line where the hand-rolled version should hand over to TanStack Query. This is the foundational recipe of the data-fetching track: nearly every async recipe in the library composes with the mechanics locked here.

## The scenario

A support tool has ticket search. Backend search p50 is **120ms**; p99 is **1.8s** (cold shards, mobile networks, VPNs). A user types `rea`, pauses a beat, finishes typing `react`. Two requests are now in flight, and the network does not care what order they left in:

| t | Event | What the UI shows |
| --- | --- | --- |
| 0ms | Request A (`rea`) fires — cold shard, will take 1400ms | spinner |
| 180ms | Request B (`react`) fires — warm, will take 130ms | spinner |
| 310ms | **B resolves** → `setResults(reactResults)` | ✅ correct results for `react` |
| 1400ms | **A resolves** → `setResults(reaResults)` | ❌ results for `rea`, under an input that says `react` |

The bug report, verbatim from the wild: *"Search shows the wrong tickets sometimes. If I wait, the right ones appear, then they get replaced by wrong ones. Can't reproduce reliably."* QA can't reproduce it either — on office wifi against a warm staging backend, latency is a uniform ~15ms and responses arrive in send order, so the bug requires **tail latency to reorder responses**, which is precisely the condition dev environments lack. It shipped for three weeks. Meanwhile, two side symptoms confused triage: the spinner sometimes never cleared (request A's completion also raced the *loading flag*), and in development every query fetched twice, which a teammate "fixed" by turning StrictMode off — treating the auditor as the bug ([`../../foundations/rules-of-react.md`](../../foundations/rules-of-react.md)).

Out-of-order responses are not an edge case. They are the *default* behavior of concurrent requests over a real network; the demo just never has two in flight.

## Walkthrough

### Stage 0 — the naive version, and why every symptom is in it

The code that shipped (recognize it — some version of this is in most codebases):

```tsx
// ❌ TicketSearch v0 — the patient
export function TicketSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Ticket[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    if (query === '') { setResults([]); return; }
    setIsLoading(true);
    fetch(`/api/tickets?q=${encodeURIComponent(query)}`)
      .then((r) => r.json())
      .then((data: Ticket[]) => {
        setResults(data);
        setIsLoading(false);
      });
  }, [query]);

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {isLoading ? <Spinner /> : <TicketList tickets={results} />}
    </div>
  );
}
```

Map the symptoms to the lines. **Wrong results:** every in-flight `.then(setResults)` is a pending write with no notion of "am I still the current query" — last responder wins, and the network picks the last responder. **Stuck spinner:** with A and B interleaved, B's `setIsLoading(false)` can be followed by nothing (A already set it `true` earlier) or A's late `setIsLoading(false)` can clear a spinner that a *third* request C legitimately owns — the booleans race independently of the data. **Impossible states:** `isLoading && results.length > 0` renders as… whatever the JSX happens to do — the flag pair can express states that don't exist ([`../../foundations/thinking-in-react.md`](../../foundations/thinking-in-react.md)). **StrictMode double-fetch:** the audit cycle fires setup twice and nothing cancels the first ([`../../effects/effects-and-synchronization.md`](../../effects/effects-and-synchronization.md)). **No error truth:** a 500 or a network drop leaves the spinner or the old list, silently.

One structural note before fixing anything: this *is* a legitimate effect — "keep `results` synchronized with `query`" names an external system — so it survives the decision tree. The problem isn't that an effect exists; it's that the synchronization has no cancellation half.

### Stage 1 — make the race visible: the union, and tagging responses

Before the fix, the diagnosis tooling. Replace the boolean pile with the state machine, and make every response carry *which query it answers*:

```tsx
type SearchState =
  | { status: 'idle' }
  | { status: 'loading'; query: string }
  | { status: 'success'; query: string; results: Ticket[] }
  | { status: 'error'; query: string; message: string };

const [search, setSearch] = useState<SearchState>({ status: 'idle' });
```

Two immediate payoffs. The impossible states are gone at the type level — spinner-plus-stale-error can't be expressed. And the UI can now *tell the truth about provenance*:

```tsx
{search.status === 'success' && (
  <>
    <p>Results for “{search.query}”</p>
    <TicketList tickets={search.results} />
  </>
)}
```

Run the broken version with this header and the bug becomes self-evident in screenshots: *Results for "rea"* under an input reading `react`. When a race is suspected anywhere, tagging responses with their request parameters is the five-minute diagnostic that turns "sometimes wrong" into "provably answering the old question." Keep the header even after the fix — it's honest UI and free regression detection.

### Stage 2 — the ignore flag: latest-wins in four lines

The minimal correct fix rides directly on effect cleanup semantics — each render's effect gets its own closure, and cleanup runs when the synchronization is superseded ([`../../effects/effects-and-synchronization.md`](../../effects/effects-and-synchronization.md)):

```tsx
useEffect(() => {
  if (query === '') { setSearch({ status: 'idle' }); return; }

  let ignored = false;                        // this render's private flag
  setSearch({ status: 'loading', query });

  (async () => {
    try {
      const res = await fetch(`/api/tickets?q=${encodeURIComponent(query)}`);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const results: Ticket[] = await res.json();
      if (!ignored) setSearch({ status: 'success', query, results });
    } catch (err) {
      if (!ignored) setSearch({ status: 'error', query, message: messageOf(err) });
    }
  })();

  return () => { ignored = true; };           // superseded → this response may not speak
}, [query]);
```

Mechanics: when `query` changes from `rea` to `react`, React runs the `rea` render's cleanup (flipping *that closure's* `ignored`) before the `react` render's setup. Request A still completes at t=1400ms — but its `if (!ignored)` gate is closed, and the write never happens. StrictMode's audit cycle is handled by the same gate. This is the fix to reach for when you *can't* cancel — a third-party SDK call, a promise you don't own the transport of.

Its honest limits: the request itself survives — bandwidth spent, server load taken, and if that endpoint isn't a pure read, the side effect landed. Latest-wins-by-silencing works; latest-wins-by-*cancelling* is strictly better when the transport allows it.

### Stage 3 — `AbortController`: cancel the synchronization at the network

The locked convention from [`../../effects/effects-and-synchronization.md`](../../effects/effects-and-synchronization.md), applied whole — abort in cleanup, `signal.aborted` swallowed in catch:

```tsx
useEffect(() => {
  if (query === '') { setSearch({ status: 'idle' }); return; }

  const controller = new AbortController();
  setSearch({ status: 'loading', query });

  (async () => {
    try {
      const res = await fetch(`/api/tickets?q=${encodeURIComponent(query)}`, {
        signal: controller.signal,
      });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const results: Ticket[] = await res.json();
      setSearch({ status: 'success', query, results });
    } catch (err) {
      if (controller.signal.aborted) return;   // superseded ≠ failed — say nothing
      setSearch({ status: 'error', query, message: messageOf(err) });
    }
  })();

  return () => controller.abort();
}, [query]);
```

What changed versus the flag: open the Network tab and type — superseded requests show **(canceled)**. The browser tears down the connection; the server (for streaming responses) can observe the disconnect and stop working; the promise rejects immediately with an `AbortError`, which the `signal.aborted` check classifies as "superseded, not failed." One mechanism now covers the race, the unmount zombie, *and* the StrictMode double — and note the `ignored` gate is gone entirely: an aborted fetch can't reach the success line, so there's nothing to gate. Prefer `controller.signal.aborted` over `err.name === 'AbortError'` — the signal check survives error-wrapping layers in api clients; the name check doesn't.

### Stage 4 — debounce the synchronization, never the typing

Correctness done; now cost. Six keystrokes currently means six fetches, five canceled — correct, wasteful. The fix is to let the *query settle* before synchronizing. Critically, do **not** debounce `setQuery` — the input is controlled ([`../../forms/forms-controlled-and-uncontrolled.md`](../../forms/forms-controlled-and-uncontrolled.md)), and delaying its state makes typing itself lag. Debounce a *derived* value:

```tsx
// use-debounced-value.ts — this recipe's graduate into the shared hooks folder
export function useDebouncedValue<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [value, delayMs]);

  return debounced;
}
```

A worthwhile aside: state-synced-from-state-via-effect is normally branch 1 of the decision tree (derive it!) — this is the sanctioned exception, because the derivation is *"the value, 250ms after it stops changing,"* and **time is the external system** being synchronized with. The `setTimeout`/`clearTimeout` pair is a textbook setup/cleanup symmetry; every keystroke inside the window cancels the previous timer.

Wire it with normalization, so `React`, `react`, and `react ` are one query, not three:

```tsx
const [query, setQuery] = useState('');
const normalized = query.trim().toLowerCase();
const debouncedQuery = useDebouncedValue(normalized, 250);

useEffect(() => {
  /* stage 3's effect, reading debouncedQuery */
}, [debouncedQuery]);
```

Layering matters and both layers stay: debounce collapses keystroke *bursts*; abort handles the overlap that debounce can't — whenever latency exceeds the debounce window (1400ms ≫ 250ms), two settled queries are in flight again and only cancellation saves you. Debounce is an optimization; abort is the correctness.

### Stage 5 — latest-wins is a *policy*; pick per interaction

This recipe implemented one race policy. There are four, and applying the wrong one is its own bug class:

| Policy | Interaction shape | React mechanism | Where it lives |
| --- | --- | --- | --- |
| **Latest wins** — cancel/ignore predecessors | Reads that track live input: search, filters, typeahead, previews | Abort in effect cleanup (this recipe) | Here |
| **First wins** — block newcomers while pending | Mutations a double-fire would duplicate: submit, pay, send | Disable-while-pending; `useActionState`'s pending state ([`../../concurrent/actions.md`](../../concurrent/actions.md)) | [`../forms-and-ux/double-submit-and-optimistic-like.md`](../forms-and-ux/double-submit-and-optimistic-like.md) |
| **All complete, keyed** — every response lands in its own slot | Independent parallel reads: one tile per symbol, one panel per widget | One synchronization per key; results stored *by* key (the `PriceTile` shape) | [`../../effects/effects-and-synchronization.md`](../../effects/effects-and-synchronization.md) |
| **Sequential queue** — order preserved, one at a time | Dependent mutations: reorder steps, collaborative ops | Explicit queue drained by a single worker | real-time / state-management tracks (queued) |

The classifying question: **is this a read or a write, and does the user's latest intent invalidate the previous one?** Reads tracking intent → latest wins. Writes → never silently cancel (an aborted POST is not an undone POST — the server may have committed); block or queue instead. Getting this backwards produces the two famous inversions: search with first-wins ("why is my search ignoring me while the old one loads") and checkout with latest-wins ("we charged the card and threw away the confirmation").

### Stage 6 — the handover line: TanStack Query

Count what the hand-rolled version still lacks: no cache (back-navigate re-fetches everything), no dedup (two components searching the same term fire twice), no retries, no revalidation, no request-time telemetry. Each is another 30–80 lines done honestly. This project's baseline says stop here:

```tsx
import { useQuery, keepPreviousData } from '@tanstack/react-query';

const { data, status, isFetching } = useQuery({
  queryKey: ['tickets', debouncedQuery],
  queryFn: async ({ signal }) => {                          // ← TanStack hands you the signal
    const res = await fetch(`/api/tickets?q=${encodeURIComponent(debouncedQuery)}`, { signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json() as Promise<Ticket[]>;
  },
  enabled: debouncedQuery !== '',
  placeholderData: keepPreviousData,                        // stage-4 UX, one line
});
```

Read the mapping, because it's exact: `queryKey` is stage 1's tagging *and* stage 3's identity (key changed → previous query is stale); the `signal` parameter is stage 3's controller, wired for you; `enabled` is the empty-query early return; `placeholderData: keepPreviousData` is the anti-flicker variation below. The stages weren't throwaway — they're what the library does, and having built them once is what makes its behavior debuggable instead of magical. **The handover criteria:** hand-roll when the fetch is a one-off, uncached, single-consumer synchronization (and carry the locked abort pattern); reach for the library the moment any of *cache, dedup across components, retries, or pagination* enters the requirements — which, in application code, is almost immediately ([`../../ecosystem/data-fetching-tanstack-query.md`](../../ecosystem/data-fetching-tanstack-query.md), Wave 4).

## Variations

### Keep the previous results visible (no-flicker loading)

Blanking the list for a spinner on every settled keystroke makes search feel broken even when it's fast. Carry the last success through the loading state — dimmed, honest, replaceable:

```tsx
type SearchState =
  | { status: 'idle' }
  | { status: 'loading'; query: string; stale?: Ticket[] }   // ← carries the previous answer
  | { status: 'success'; query: string; results: Ticket[] }
  | { status: 'error'; query: string; message: string };

setSearch((prev) => ({
  status: 'loading',
  query,
  stale: prev.status === 'success' ? prev.results : undefined,
}));
```

```tsx
{search.status === 'loading' && search.stale && (
  <div className="results results--stale" aria-busy="true">
    <TicketList tickets={search.stale} />
  </div>
)}
```

Pair with a spinner that only appears after ~150ms (a second debounced value) and fast responses never flash at all. This is the hand-rolled `keepPreviousData`.

### Search-on-submit instead of search-as-you-type

For expensive backends, the form goes uncontrolled — `FormData` at submit ([`../../forms/forms-controlled-and-uncontrolled.md`](../../forms/forms-controlled-and-uncontrolled.md)) — and the race *narrows but doesn't vanish*: impatient users double-submit. Same abort discipline, different trigger site: the controller lives in a ref (`useRef<AbortController | null>`), the handler aborts the previous before firing the next ([`../../effects/useref-and-the-dom.md`](../../effects/useref-and-the-dom.md) owns the ref mechanics). Latest-wins in a handler instead of an effect — the policy is the invariant, the location is circumstance.

### Pagination joins the key

Add a page selector and the race dimension doubles: query changes *and* page changes both invalidate in-flight requests, and a `page: 2` response for the old query is doubly stale. The fix generalizes stage 1: the request's identity is the whole descriptor. Derive one object of the inputs (`{ query: debouncedQuery, page }`), depend the effect on its *fields* (primitives — identity discipline from [`../../foundations/components-and-props.md`](../../foundations/components-and-props.md)), tag responses with the full descriptor, and reset `page` to 1 when `query` changes (a guarded adjust-during-render — the escalation ladder, [`../../state/state-and-usestate.md`](../../state/state-and-usestate.md)). In TanStack terms: everything that identifies the request belongs in the `queryKey`.

## Trade-offs and common pitfalls

### When NOT to use this

- **Don't cancel writes.** Aborting a POST abandons the *response*, not the operation — the server may have committed, and you've thrown away the confirmation and any error. Mutations take first-wins or a queue (stage 5), never silent latest-wins.
- **Don't abort what others share.** Once a caching/dedup layer serves one request to many consumers, an abort from one consumer kills it for all. Per-consumer cancellation across shared requests is exactly the bookkeeping that justifies the library — hand-rolled abort assumes sole ownership.
- **Don't fetch what you already have.** A few hundred rows already on the client is a `filter` during render ([`../../foundations/thinking-in-react.md`](../../foundations/thinking-in-react.md)) — zero requests is the only race-free architecture, and it's also the fastest.

### Pitfalls

1. **Gating results but not loading state.** The ignore flag wraps `setSearch('success'…)` while a separate `setIsLoading(false)` runs ungated → stuck or flickering spinners. The union makes this structural: one state, one gated write per outcome.
2. **Surfacing the abort as an error.** Users see *"Search failed: The user aborted a request"* on every keystroke. Every catch in the pipeline needs the `signal.aborted` early return — including catches inside your api-client wrapper.
3. **Debouncing the input state.** `setQuery` behind a timer = the controlled input lags the keyboard by 250ms. Debounce the derived request value; the input stays live.
4. **Debounce as the only defense.** "We debounce, so there's no race" holds only while latency < window. The scenario's 1400ms tail sails through a 250ms debounce; abort is the correctness layer, debounce the cost layer.
5. **Un-normalized queries.** `React`, `react`, and `react ` fire three fetches and three cache entries. Normalize (`trim().toLowerCase()`) *before* the debounce so the settled value is canonical.
6. **Fetching the empty query.** Clearing the box fires `/api/tickets?q=` — often the most expensive query the backend has. Early-return to `idle` before any request machinery.
7. **"Fixing" StrictMode's double fetch by disabling StrictMode.** The audit found your missing cancellation; stage 3 makes the double invisible for free. Removing the auditor ships the leak ([`../../foundations/rules-of-react.md`](../../foundations/rules-of-react.md)).
8. **An api client that eats the signal.** `api.get(url)` wrappers that don't accept/forward `{ signal }` make every downstream abort a no-op — and it fails silently (requests just… complete). Audit the client's signature first; this is the most common "we added abort and nothing changed."
9. **Two inputs, two effects, one result slot.** `query` and `page` synchronized by separate effects writing the same state interleave into cross-parameter staleness. One request descriptor, one synchronization, one tagged write.
10. **Untagged success writes.** Even post-abort, transitional renders can pair old results with a new header if the results don't carry their query. Store provenance in the state (`{ query, results }` together) — never in two variables that update separately.
11. **A later cache layer memoizing rejections.** Adding naive `Map`-based caching that stores the aborted promise turns one supersession into a permanently "failed" query. Aborts are non-results: never cache them (and the union's early-return already refuses to record them as errors — keep that property when caching arrives).
12. **Stale-carry that survives errors.** The no-flicker variation dimming *old* results next to a *new* error message reads as "these results are wrong AND current." Clear `stale` on the error transition; dimmed data is a promise that fresh data is coming, and errors break that promise.
13. **Testing the race with real timers.** Reproducing this in Vitest with live `setTimeout`s produces the same flake the office wifi did. Fake timers + manually resolved fetch promises let you *choose* the interleaving — resolve B, then A, assert A's write never landed (testing track, queued).

## See also

- [`../../effects/effects-and-synchronization.md`](../../effects/effects-and-synchronization.md) — the synchronization contract, the locked abort convention, and the "full bill" this recipe pays down
- [`../../foundations/thinking-in-react.md`](../../foundations/thinking-in-react.md) — discriminated-union state, the tool that made the race visible
- [`../../effects/custom-hooks.md`](../../effects/custom-hooks.md) — where `useDebouncedValue` graduates to, and the extraction rules it should follow
- [`../../forms/forms-controlled-and-uncontrolled.md`](../../forms/forms-controlled-and-uncontrolled.md) — the input side: controlled typing, submit-mode variation
- [`../forms-and-ux/double-submit-and-optimistic-like.md`](../forms-and-ux/double-submit-and-optimistic-like.md) *(planned)* — the first-wins policy this recipe's table defers to
- [`../performance/typing-lag-rerender-storm.md`](../performance/typing-lag-rerender-storm.md) *(planned)* — the sibling failure: when the keystrokes themselves are the bottleneck
- [`../../ecosystem/data-fetching-tanstack-query.md`](../../ecosystem/data-fetching-tanstack-query.md) *(Wave 4)* — the handover target, in full

## References

- [`AbortController` (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [`AbortSignal` (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
- [Synchronizing with Effects — fetching (react.dev)](https://react.dev/learn/synchronizing-with-effects)
- [You Might Not Need an Effect — fetching section (react.dev)](https://react.dev/learn/you-might-not-need-an-effect)
- [TanStack Query — Query Cancellation](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation)
- [TanStack Query — Placeholder Data / keepPreviousData](https://tanstack.com/query/latest/docs/framework/react/guides/placeholder-query-data)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../../progress.md) TODOs.