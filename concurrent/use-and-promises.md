---
article_id: use-and-promises
concept_folder: concurrent
wave: 3
related:
  - concurrent/suspense
  - rendering/error-boundaries
  - state/context
  - foundations/rules-of-react
  - ecosystem/data-fetching-tanstack-query
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this.** Every hook you've met has one iron rule: call it at the top level, unconditionally, in the same order every render ([`rules-of-react`](../foundations/rules-of-react.md)). `use()` is the deliberate exception — you can call it inside an `if`, inside a loop, after an early return. That's not a loophole React forgot to close; it's the whole design. `use()` reads a **resource** (a promise or a context) at the moment you call it, and reading a resource doesn't require a persistent per-render slot the way `useState` does. Because it needs no slot, it needs no stable call order. This article is where that claim gets earned — and where the promise you hand `use()` has one non-negotiable requirement (it must be stable across renders) that, when violated, produces the "fallback that never resolves" bug from [`suspense`](./suspense.md).

## What `use` is

`use(resource)` reads the value of a resource during render. It accepts exactly two kinds of resource, and does something different with each:

```tsx
import { use } from "react";

const data = use(somePromise);     // Promise<T> → T (suspends while pending)
const theme = use(ThemeContext);   // Context<T> → T (like useContext, but callable conditionally)
```

- **Promise** → returns the resolved value. If pending, it **suspends** the component (the throw-a-thenable mechanism [`suspense`](./suspense.md) owns) so the nearest `<Suspense>` shows a fallback. If rejected, it **throws** the rejection to the nearest error boundary ([`error-boundaries`](../rendering/error-boundaries.md)).
- **Context** → returns the current context value, identical to `useContext(Context)` — except `use(Context)` may be called conditionally.

`use` is the API that makes a promise a **first-class render input**: instead of "fetch in an effect, store the result in state, branch on a loading flag," you read the promise directly in render and let Suspense/error boundaries handle the pending and failed states. The loading-is-a-boundary shift from [`suspense`](./suspense.md) is what makes this ergonomic; `use` is the read that triggers it.

## How it works under the hood

### Reading a promise, traced

The reason the promise must be stable is visible the moment you trace what `use` does with it. React tracks a thenable by tagging it with its settled state — internally, a `status` of `"pending" | "fulfilled" | "rejected"` plus the resolved `value` or `reason` (these are React's internal bookkeeping fields, not public API, but knowing they exist explains everything):

1. `use(promise)` is called. React looks at the promise it was handed.
2. **Untracked / pending:** React attaches a `.then()` that will record the outcome on the promise and schedule a re-render (a "ping") of the suspended boundary, then **throws the thenable** to suspend. The Suspense boundary shows its fallback.
3. **Fulfilled:** React returns the recorded value — synchronously, no suspend.
4. **Rejected:** React throws the recorded reason, which propagates to the nearest error boundary.

The pivotal detail is step 2→3 across renders. When the promise resolves, React re-renders the boundary. On that retry render, `use` is called **again** — and if it's handed the *same promise object*, that object is now tagged `fulfilled`, so `use` returns the value and the component renders for real. If it's handed a *new* promise (because you created it inline in render), that new object is untracked and pending, so `use` suspends again — and the retry creates yet another new promise, forever. That's the infinite-fallback bug. The promise's *identity* is the memory; a fresh promise has amnesia.

For context, there's no promise to track — `use(Context)` reads the current value from the nearest provider using the exact propagation [`context`](../state/context.md#how-a-value-change-finds-its-readers) already describes. Nothing new mechanically; the only difference from `useContext` is where you're allowed to call it.

### Why `use` escapes the hook-order rule

Regular hooks are stored as a linked list on the fiber, walked in call order every render ([`how-react-renders`](../rendering/how-react-renders.md#how-it-works-under-the-hood)). `useState` has to find *last render's state for this exact hook*, and the only thing identifying "this exact hook" is its position in the call sequence. Skip a hook conditionally and every subsequent index shifts — React reads the wrong slot. That's why the rule exists: it's not stylistic, it's how state is addressed.

`use` needs none of that, and the reason is sharp: **`use` persists nothing between renders.** `useState` must remember a value across renders → it needs a stable slot → it needs stable call order. `use(promise)` reads the *current settled state of a resource you handed it* → there's nothing to remember from last render → no slot → no call-order dependency. `use(Context)` reads the *current ambient value* → again nothing to persist → no slot. The exemption isn't special-casing; it falls directly out of "reading a resource is stateless."

So the rules relax exactly this far:

| | Regular hooks (`useState`, `useEffect`, …) | `use()` |
| --- | --- | --- |
| Conditionally / in an `if`? | ❌ | ✅ |
| After an early return? | ❌ | ✅ |
| In a loop? | ❌ | ✅ |
| In `try`/`catch`/`finally`? | ❌ | ❌ |
| In event handlers or effects? | ❌ (render-only) | ❌ (render-only) |
| Addressed by | call-order slot on the fiber | the resource's own identity |

Two prohibitions survive because they're about *when*, not *order*. `use` is still render-only — it's how you read data *during* rendering, so it makes no sense in an event handler or effect. And it can't sit in a `try`/`catch`: `use` works by throwing (to suspend or to surface a rejection), and your `try`/`catch` would intercept that throw and break the boundary machinery. To handle a rejection, use an error boundary, or attach `.catch()` to the promise to supply a fallback value *before* it reaches `use`.

## Basic usage

### Read a promise

```tsx
import { Suspense, use } from "react";

function Message({ messagePromise }: { messagePromise: Promise<string> }) {
  const text = use(messagePromise); // suspends until resolved
  return <p>{text}</p>;
}

export function Chat({ messagePromise }: { messagePromise: Promise<string> }) {
  return (
    <Suspense fallback={<p>Loading…</p>}>
      <Message messagePromise={messagePromise} />
    </Suspense>
  );
}
```

The promise arrives as a **prop**, created by the parent — not inside `Message`. That's the shape that keeps it stable, and it's exactly how a Server Component hands a promise to a Client Component.

### Read a context conditionally

```tsx
import { use } from "react";

function Panel({ expanded, children }: { expanded: boolean; children: ReactNode }) {
  if (!expanded) {
    return <Collapsed />; // early return — no context read on this path
  }
  // ✅ Legal: use(Context) after an early return. useContext here would break the rules.
  const theme = use(ThemeContext);
  return <div className={theme.panelClass}>{children}</div>;
}
```

This is the conditional read [`context`](../state/context.md) forward-referenced. With `useContext` you'd be forced to call it unconditionally at the top even on the `Collapsed` path; `use(Context)` lets the read live where it's actually needed.

## Walkthrough: a comment thread that reads promises

We'll build a thread view that fetches comments, reads them with `use`, handles the stable-promise requirement honestly, fetches in parallel, and reads a context conditionally. The composition with Suspense and error boundaries is owned by [`suspense`](./suspense.md); here the focus is the `use` reads and where the promises come from.

### Stage 1 — the footgun, then the fix

The tempting first draft suspends forever:

```tsx
// ❌ New promise every render → use() suspends → retry → new promise → suspends…
function Comments({ threadId }: { threadId: string }) {
  const comments = use(fetch(`/api/threads/${threadId}/comments`).then((r) => r.json()));
  return <CommentList comments={comments} />;
}
```

The fix is a stable resource — the same promise for the same input, created outside render:

```tsx
// resources.ts — a minimal promise cache keyed by identity.
const commentsCache = new Map<string, Promise<Comment[]>>();

export function getComments(threadId: string): Promise<Comment[]> {
  let promise = commentsCache.get(threadId);
  if (!promise) {
    promise = fetch(`/api/threads/${threadId}/comments`).then((r) => r.json());
    commentsCache.set(threadId, promise);
  }
  return promise; // same threadId → same promise object → stable across renders
}
```

```tsx
function Comments({ threadId }: { threadId: string }) {
  const comments = use(getComments(threadId)); // stable: use() resolves on retry
  return <CommentList comments={comments} />;
}
```

Be honest about what this cache is and isn't. It gives `use` a stable promise, which is all the *mechanics* require. It does **not** invalidate, refetch, garbage-collect, or handle a stale entry — the map grows forever and never updates. That gap is precisely the job of a real server-cache library; a hand-rolled cache is fine for understanding `use`, not for production. The production stable-promise source is [`data-fetching-tanstack-query`](../ecosystem/data-fetching-tanstack-query.md)'s `useSuspenseQuery` (or an RSC passing the promise down), which give you a stable *and* cache-managed resource.

### Stage 2 — create promises high, read them low (parallel, not waterfall)

Where you *create* the promise decides whether fetches run in parallel or serialize. Create both promises in the parent before rendering either child, and both requests are in flight before either `use` runs:

```tsx
function Thread({ threadId }: { threadId: string }) {
  // Both fetches start here, together — before either child renders and suspends.
  const commentsPromise = getComments(threadId);
  const authorPromise = getAuthor(threadId);

  return (
    <ErrorBoundary FallbackComponent={ThreadError}>
      <Suspense fallback={<ThreadSkeleton />}>
        <ThreadAuthor authorPromise={authorPromise} />
        <Comments commentsPromise={commentsPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}

function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise);
  return <CommentList comments={comments} />;
}
```

If instead each child called `getComments`/`getAuthor` itself on its own first render, the author fetch couldn't start until the comments boundary resolved and rendered the next child — a waterfall. `use` doesn't cause waterfalls, but *where you start the promise* does. Hoist the starts; read low.

### Stage 3 — a conditional read on the same tree

Suppose comments render differently in a live "presenting" mode carried in context. Only the presenting branch needs it:

```tsx
function CommentList({ comments }: { comments: Comment[] }) {
  if (comments.length === 0) return <Empty />;

  // Only read presentation config when there's something to present.
  const presentation = use(PresentationContext);
  return (
    <ul data-mode={presentation.mode}>
      {comments.map((c) => (
        <li key={c.id}>{c.body}</li> // stable key from identity
      ))}
    </ul>
  );
}
```

The `use(PresentationContext)` sits after an early return — impossible with `useContext`, routine with `use`. On the empty path the context is never read, so that render doesn't depend on it.

## Real-world patterns

**Server Component → Client Component.** The flagship use of `use(promise)`. A Server Component starts a fetch and passes the *unawaited* promise as a prop to a Client Component, which reads it with `use`. The server streams the shell immediately and the data as it resolves; the client suspends on the promise. This is the primary RSC data path — [`server-components`](../server/server-components.md) owns it. The promise is stable because it's created once on the server and serialized to the client.

The distinction that trips people up: in a Server Component you usually don't need `use` at all — Server Components can be `async`, so you just `await` the data directly. `use` earns its place where `await` isn't available: **Client Components can't be async**, so a Client Component reading a promise uses `use`; and `use` reads context and can be called conditionally, which `await` cannot. Rule of thumb: `await` on the server when you can, `use` on the client (or for conditional/context reads) when you can't.

**Graceful degradation with `.catch()`.** When a failed fetch should render a fallback *value* rather than trip an error boundary, resolve the rejection into a safe default before `use` ever sees it:

```tsx
// Rejection is converted to a value, so use() returns [] instead of throwing.
function getCommentsSafe(threadId: string): Promise<Comment[]> {
  return getComments(threadId).catch(() => [] as Comment[]);
}
// const comments = use(getCommentsSafe(id)); // never throws; empty on failure
```

Reserve this for genuinely optional data (recommendations, a sidebar) where "show nothing" beats "show an error." For data whose failure the user must know about, let it throw to the error boundary instead.

**`useSuspenseQuery` for real data.** In an SPA, you almost never hand-roll the cache from Stage 1. TanStack Query's suspense-enabled queries hand you a stable, cached, refetchable, invalidatable resource and integrate with Suspense/error boundaries — the production answer to "where does the stable promise come from" ([`data-fetching-tanstack-query`](../ecosystem/data-fetching-tanstack-query.md)).

**`use(Context)` for conditional and early-return reads.** Feature flags, optional theming, a context only one branch cares about — anywhere `useContext` forced an awkward top-level call, `use(Context)` reads where it belongs. For the common *unconditional* top-level read, `useContext` is still the clearer choice; reach for `use(Context)` when the conditionality is the point.

**Parallelizing independent data.** Generalize Stage 2: start every independent promise at the highest common ancestor, pass them down, `use` them at the leaves. All requests fly together; each leaf suspends only for its own.

## API and type reference

| API | Signature | Behavior |
| --- | --- | --- |
| `use` (promise) | `use<T>(promise: Promise<T>): T` | Suspends while pending, returns value when fulfilled, throws reason when rejected. Promise must be stable across renders. |
| `use` (context) | `use<T>(context: Context<T>): T` | Returns current context value; callable conditionally / after early return / in loops. |

Type note: `use` is overloaded on `Promise<T>` vs `Context<T>` and preserves `T` exactly. There is no type-level signal that a component reading a promise will suspend — suspension is runtime behavior (a throw), so the Suspense boundary is a placement decision you make from knowing the data flow, not something the compiler enforces (same caveat as [`suspense`](./suspense.md)'s API note).

## Common mistakes

**1. Creating the promise in render.** The defining footgun, traced in the mechanics section: a new promise each render is untracked and pending, so `use` suspends forever. The promise must come from a stable source — a cache keyed by input, a parent/RSC prop, or a Suspense-enabled library. Symptom: a fallback that never resolves.

**2. Wrapping `use()` in try/catch.** `use` throws to suspend and to surface rejections; a surrounding `try`/`catch` swallows that throw and breaks the boundary handoff. Handle failures with an error boundary, or `.catch()` on the promise to supply a fallback value before `use` sees it.

**3. Calling `use` in an event handler or effect.** It's render-only — the whole point is reading a resource *during* render. In an event handler you already have imperative control; `await` the promise there. In an effect, you're back to the fetch-then-setState model `use` is meant to replace.

**4. Expecting `use` to cache or dedupe.** On the client, `use` does not cache anything — it reads whatever promise you hand it. Two components handed two separate promises for the same data fetch twice. Dedup/caching is your cache's job or the library's, not `use`'s.

**5. Thinking conditional `use` means lazy fetching.** `use` controls the *read*, never the *start*. If you create the promise unconditionally and only `use` it in a branch, the fetch still ran. To fetch lazily, create the promise lazily (only build it when the branch is taken).

**6. No error boundary around a `use(promise)`.** A rejected promise throws during render; with no error boundary it unwinds to the root and blanks the app. Every `use(promise)` needs an error boundary somewhere above it, just as it needs a Suspense boundary ([`suspense`](./suspense.md) Stage 3).

**7. Reaching for `use(Context)` reflexively.** For an unconditional top-level context read, `useContext` is more conventional and reads more clearly. `use(Context)`'s advantage is *only* conditional/early-return/loop reads. Using it everywhere trades clarity for nothing.

**8. Starting promises low and serializing them.** Creating each child's promise inside that child's first render waterfalls the fetches. Hoist promise creation to the common parent so independent requests run in parallel (Stage 2).

**9. Passing an inline-constructed context or object to `use`.** Same identity-churn family as mistake #1 for promises — build stable resources, not fresh ones each render.

**10. Treating `use` as a Suspense replacement.** `use` is the *read*; it doesn't remove the need for a fallback and an error UI. You still design boundaries. `use` changes how you consume async data, not whether you handle its states.

## How this evolved

- **fetch-in-`useEffect`:** the pre-Suspense-data norm — fetch in an effect, stash the result in state, branch on `isLoading`/`error` flags. No render-time data reads; every async leaf reinvents the same three-state dance (the failure mode the [`search-race-condition`](../recipes/data-fetching/search-race-condition.md) recipe dissects).
- **Experimental "Suspense for data":** frameworks (Relay, later Next.js) and userland used the throw-a-thenable convention to read data in render, but there was no stable public API — you were told "don't do this by hand."
- **18:** Suspense for data became real inside frameworks and streaming SSR standardized the throw-a-thenable contract, but app code still lacked a blessed way to read a promise.
- **19:** `use()` shipped as the stable public API — read promises and context in render, with the conditional-call relaxation. The first time hand-written app code could officially treat a promise as a render input.
- **19-era RSC:** `use` became the standard bridge for Client Components to consume promises passed from Server Components, with server-side request memoization via `cache()`. The arc: reading async data moved *out* of effects and *into* render, where Suspense and error boundaries handle its states declaratively.

## Exercises

**1. Kill an infinite fallback.** Given a component that does `use(fetch(url).then(...))` in render and never resolves, move the fetch to an input-keyed cache and confirm the Suspense fallback now resolves.
*Hint:* Same input must return the same promise object. Log the promise identity across renders to prove it's stable.

**2. Conditional context.** Take a component that calls `useContext` at the top but only uses the value in one branch, and rewrite it so the read happens with `use(Context)` inside that branch, after an early return.
*Hint:* This won't type-check with `useContext` in the branch — that's the point. `use(Context)` is legal there.

**3. (Stretch) Parallelize.** Two independent `use(promise)` reads currently waterfall because each child starts its own fetch. Hoist both promise starts to the parent and pass them down; measure the wall-clock difference in the network panel.
*Hint:* The fetches must *start* before either child suspends — creation site, not read site, controls timing.

## Summary

- `use(resource)` reads a **promise** or a **context** during render. A pending promise suspends (→ Suspense); a rejected one throws (→ error boundary); a fulfilled one returns its value.
- `use` may be called **conditionally, after early returns, and in loops** — because it persists nothing between renders, so it needs no call-order slot. It still may **not** be called in `try`/`catch` or outside render.
- The promise handed to `use` **must be stable across renders**. Its identity carries the tracked settled state; a fresh promise each render is untracked and suspends forever. Get stable promises from a cache keyed by input, a parent/RSC prop, or a Suspense-enabled library — never `fetch(...)` inline in render.
- **Where you create the promise** decides parallel-vs-waterfall. Hoist promise starts to the common parent; read them low with `use`.
- `use(Context)` is the conditional/early-return form of context reading; `useContext` remains the clearer choice for unconditional top-level reads.
- `use` is the *read*, not the state handler — it still needs Suspense and error boundaries around it, and it doesn't cache for you.

## See also

- [`suspense`](./suspense.md) — the boundary that catches `use`'s pending throw; the stable-promise symptom lives on both sides.
- [`error-boundaries`](../rendering/error-boundaries.md) — catches `use`'s rejection throw; the error half of the triad.
- [`context`](../state/context.md) — `useContext` and the propagation `use(Context)` shares; the conditional read this article pays off.
- [`rules-of-react`](../foundations/rules-of-react.md) — the hook-order rule `use` is the deliberate exception to.
- [`actions`](./actions.md) *(planned, next)* — async transitions and the 19 mutation story that consume async work the other direction.
- [`data-fetching-tanstack-query`](../ecosystem/data-fetching-tanstack-query.md) *(planned)* — the production source of stable, cached, refetchable promises for `use`.

## References

- React docs — [`use`](https://react.dev/reference/react/use)
- React docs — [Suspense: data fetching with `use`](https://react.dev/reference/react/Suspense)
- React docs — [`cache`](https://react.dev/reference/react/cache) (Server Components)
- React blog — [React 19](https://react.dev/blog/2024/12/05/react-19)

## Demo source

_Demo source pending — StackBlitz link to be added in the demo pass._