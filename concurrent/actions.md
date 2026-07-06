---
article_id: actions
concept_folder: concurrent
wave: 3
related:
  - state/usereducer-and-state-structure
  - concurrent/concurrent-rendering
  - forms/forms-controlled-and-uncontrolled
  - rendering/error-boundaries
  - recipes/forms-and-ux/double-submit-and-optimistic-like
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this.** Every form you wrote before React 19 reinvented the same four things by hand: a `isSubmitting` flag toggled around the request, a `try/catch` to catch failures into an error state, a manual optimistic update with a manual rollback in the `catch`, and a `preventDefault` + `FormData` dance to read the fields. Actions collapse all four into primitives. An **Action** is just an async function run inside a transition ([`concurrent-rendering`](./concurrent-rendering.md) built the substrate: "an Action is an async transition with a form-shaped reducer wrapped around it"). React manages the pending flag, threads the state, surfaces the errors, and reverts the optimistic overlay — so the form code becomes the *logic* and nothing else. This article is the mechanism under `useActionState`, `useFormStatus`, and `useOptimistic`; the [`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) recipe already uses these APIs against a real bug — here's what they're doing underneath.

## What Actions are

Three connected pieces:

1. **Function form actions.** `<form action={fn}>` where `fn` is a function, not a URL. On submit, React calls `fn(formData)` inside a transition, tracks its pending state, and (for uncontrolled inputs) resets the form on success.
2. **`useActionState`** — threads state through an async action, reducer-style: `(previousState, formData) => nextState`. Returns the current state, a wrapped action, and a pending flag.
3. **`useFormStatus`** and **`useOptimistic`** — read the ambient form's pending state from anywhere inside it, and layer a transition-scoped optimistic overlay with automatic rollback.

The unifying idea: a mutation has a **pending** phase, a **success** result, a **failure** result, and often an **optimistic** preview. Before Actions you wired each by hand per form. Actions make all four fall out of "run this async function as a transition."

## How it works under the hood

### Function actions on a form

`<form action={fn}>` changes what submit does. Instead of navigating to a URL, React:

1. Calls `preventDefault` for you.
2. Collects the form's fields into a `FormData` object ([`forms-controlled-and-uncontrolled`](../forms/forms-controlled-and-uncontrolled.md) owns `FormData` and the uncontrolled-first model).
3. Runs `startTransition(async () => await fn(formData))` — so the action is a transition: non-blocking, interruptible, pending-tracked, with rejections thrown to the nearest error boundary.
4. Publishes the submission state (`pending`, the `FormData`, method, action) into a **form-status context** that wraps the form's subtree.
5. On success, **resets uncontrolled fields** (you can opt out with `requestFormReset` from `react-dom`, or by using controlled inputs whose value you own).

Step 4 is the detail that explains `useFormStatus`'s odd placement rule, and step 3 is the through-line to everything else: it's the same async transition from [`concurrent-rendering`](./concurrent-rendering.md#async-transitions-forward-pointer), now triggered by a submit and handed a `FormData`.

### `useActionState` — the reducer, productized

[`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md) promised this article. A reducer is `(previousState, action) => nextState`. `useActionState` is that exact shape, with two changes: the "action" is form input, and the reducer may be **async**.

```tsx
const [state, formAction, isPending] = useActionState(
  async (previousState, formData) => {
    // reducer body: read formData, do async work, RETURN the next state
    const title = String(formData.get("title"));
    if (!title) return { ...previousState, error: "Title required" };
    const saved = await api.createPost({ title });
    return { post: saved, error: null };
  },
  { post: null, error: null }, // initialState
);
```

The internals are `useReducer` fused with an async transition and a form binding:

- React stores the state, initialized to `initialState`.
- `formAction` is a wrapper you hand to `<form action={formAction}>` (or `<button formAction>`). When the form submits, React runs the reducer as a transition, passing **the current state as `previousState`** and the submitted `FormData` as the second argument.
- The reducer's **return value becomes the next state** — which is threaded in as `previousState` on the next submit. It's the reducer accumulator, exactly as in `useReducer`, just with `await` in the middle.
- `isPending` reflects the transition: `true` while the reducer runs, `false` when it settles.

So the mental model isn't "a new form hook to memorize" — it's "`useReducer` where dispatch is a form submit and the reducer can await." Everything you know about reducers (events-not-setters, return-don't-mutate, derive-don't-mirror) carries over unchanged ([`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md)).

The signature trap worth pre-empting: the first argument is **previous state**, not the form data. `(previousState, formData)`. Reversing them is the most common `useActionState` bug.

### `useFormStatus` — reading the ambient form

```tsx
import { useFormStatus } from "react-dom"; // note: react-dom, not react

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "Saving…" : "Save"}</button>;
}
```

`useFormStatus` reads the form-status context published in step 4 above. That's why it has a placement rule that surprises people: **it reads the nearest *parent* `<form>`, and must be called from a component rendered inside that form** — not from the component that renders the form. A `<SubmitButton>` placed as a child of `<form>` sees the pending state; if you called `useFormStatus` in the same component that renders the `<form>`, you'd be *above* the provider and always read `pending: false`.

This is a deliberate composition win: the submit button doesn't need the pending flag prop-drilled into it. It reads the ambient form status, so one `<SubmitButton>` works in every form with zero wiring — the same ambient-value-from-context pattern [`context`](../state/context.md) describes, here provided by the form itself. It returns `{ pending, data, method, action }` — `data` being the in-flight `FormData`, handy for showing *what* is being submitted.

### `useOptimistic` — a transition-scoped overlay

```tsx
const [optimisticLikes, addOptimisticLike] = useOptimistic(
  likes,                             // the real, committed state
  (current, delta: number) => current + delta, // how to apply an optimistic value
);
```

`useOptimistic` keeps two things: the real `actualState` (here `likes`) and a temporary **overlay** applied only while a transition is pending. The mechanism:

1. Inside an action, you call `addOptimisticLike(1)`. React applies `updateFn(actualState, 1)` and `optimisticLikes` immediately shows the incremented value — before the server responds.
2. The overlay is bound to the surrounding transition's lifecycle. While the action is pending, the display shows the optimistic value.
3. When the action **succeeds**, the real state updates (via the action's returned state or a refetch), the transition ends, the overlay is discarded, and `optimisticLikes` now reflects the new `actualState` — seamlessly, because real and optimistic agree.
4. When the action **fails**, the action threw, so no real state change happened; the transition ends, the overlay is discarded, and `optimisticLikes` snaps back to the unchanged `actualState`. That's **automatic rollback** — you write no `catch`-and-revert.

The key constraint follows from the mechanism: `useOptimistic` only overlays *during a pending transition*. Calling `addOptimistic` outside an action does nothing lasting. And the overlay is display-only — the real state must actually update, or the optimistic value vanishes at settle-time with nothing behind it (mistake #3).

`useOptimistic` isn't form-specific, either — it binds to *any* transition. A like button that calls `startTransition` in its `onClick` gets the same overlay-and-rollback without a `<form>` anywhere; the form APIs and `useOptimistic` are independent tools that happen to compose well.

## Basic usage

### A function form action

```tsx
export function QuickNote() {
  async function saveNote(formData: FormData) {
    await api.saveNote(String(formData.get("text")));
    // uncontrolled fields reset automatically on success
  }
  return (
    <form action={saveNote}>
      <input name="text" />
      <button>Save</button>
    </form>
  );
}
```

### `useActionState` with a pending flag

```tsx
import { useActionState } from "react";

export function Subscribe() {
  const [state, formAction, isPending] = useActionState(
    async (_prev: { message: string }, formData: FormData) => {
      const email = String(formData.get("email"));
      await api.subscribe(email);
      return { message: `Subscribed ${email}` };
    },
    { message: "" },
  );

  return (
    <form action={formAction}>
      <input name="email" type="email" />
      <button disabled={isPending}>{isPending ? "…" : "Subscribe"}</button>
      {state.message && <p>{state.message}</p>}
    </form>
  );
}
```

## Walkthrough: an add-comment form, all four pieces

We'll build a comment form that validates, posts, shows an instant optimistic comment, rolls back on failure, and uses a reusable submit button — composing function actions, `useActionState`, `useFormStatus`, and `useOptimistic`. This is the mechanism under the [`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) recipe.

### Stage 1 — the reducer-shaped action with validation as state

Validation errors are *expected* outcomes, so they're **returned as state**, not thrown — the errors-as-state side of [`error-boundaries`](../rendering/error-boundaries.md#errors-as-state-vs-errors-as-thrown)'s taxonomy. Only genuinely exceptional failures throw to the boundary.

```tsx
import { useActionState } from "react";

type CommentState = { error: string | null };

function CommentForm({ postId }: { postId: string }) {
  const [state, formAction, isPending] = useActionState<CommentState, FormData>(
    async (_prev, formData) => {
      const body = String(formData.get("body")).trim();
      if (body.length === 0) return { error: "Comment can't be empty" };
      if (body.length > 500) return { error: "Comment too long" };

      await api.postComment(postId, body); // throws only on real failure → error boundary
      return { error: null };
    },
    { error: null },
  );

  return (
    <form action={formAction}>
      <textarea name="body" aria-invalid={state.error != null} />
      {state.error && <p role="alert">{state.error}</p>}
      <SubmitButton />
    </form>
  );
}
```

`isPending` is available here, but the submit button reads its own pending state from context — so we don't even thread it down.

### Stage 2 — the reusable submit button

```tsx
import { useFormStatus } from "react-dom";

// Works inside ANY form — reads the ambient form status, no props.
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "Posting…" : "Post"}</button>;
}
```

Because `<SubmitButton>` is rendered *inside* `<form>`, it reads that form's pending state. Drop it into a dozen forms; each one just works. This is the design-system payoff of `useFormStatus` living in context.

### Stage 3 — optimistic comments with rollback

Show the new comment the instant the user submits; revert automatically if the post fails.

```tsx
import { useActionState, useOptimistic } from "react";

function CommentThread({ postId, comments }: { postId: string; comments: Comment[] }) {
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (current, pending: string) => [...current, { id: "pending", body: pending, pending: true }],
  );

  const [state, formAction] = useActionState<{ error: string | null }, FormData>(
    async (_prev, formData) => {
      const body = String(formData.get("body")).trim();
      if (!body) return { error: "Comment can't be empty" };

      addOptimistic(body);        // overlay applies immediately, within this transition
      await api.postComment(postId, body); // if this throws, overlay auto-reverts
      return { error: null };     // real `comments` refreshes via the parent → overlay discarded
    },
    { error: null },
  );

  return (
    <>
      <ul>
        {optimisticComments.map((c) => (
          <li key={c.id} style={{ opacity: c.pending ? 0.5 : 1 }}>
            {c.body}
          </li>
        ))}
      </ul>
      <form action={formAction}>
        <textarea name="body" />
        {state.error && <p role="alert">{state.error}</p>}
        <SubmitButton />
      </form>
    </>
  );
}
```

The flow: submit → `addOptimistic(body)` shows the dimmed pending comment instantly → `await api.postComment` runs → on success the parent's real `comments` updates and the overlay dissolves into the real list; on failure the action throws, the overlay reverts, and the pending comment disappears. No manual rollback code — the transition lifecycle does it.

One honest caveat this walkthrough inherits from the recipe: **`isPending`/disabling the button is UX, not a correctness guarantee against double-submit.** A determined double-click or a retry can still fire twice; the real defense is server idempotency. The [`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) recipe owns that client-guard-vs-server-correctness distinction and its mutation-classification table; this article owns the APIs it's built from.

## Real-world patterns

**Reusable submit button (and pending affordances) via `useFormStatus`.** The canonical pattern from Stage 2 — one component, every form, no wiring. Extend it to spinners, disabled fieldsets, or "Saving…" text.

**Validation-as-state, exceptions-as-throw.** Return `{ error }` for expected validation and business-rule failures the user should see inline; `throw` only for genuinely exceptional failures that belong to an error boundary. This split is the errors-as-state-vs-errors-as-thrown taxonomy applied to mutations.

**Optimistic lists, toggles, and counters.** `useOptimistic` shines for append-to-list (Stage 3), like/unlike toggles, and counters where the likely outcome is success and instant feedback matters. The `updateFn` is where you encode "what the UI should look like assuming this succeeds."

**Server Actions and progressive enhancement.** A function action can be a *server* function (RSC), and `useActionState`'s third argument, `permalink`, lets the form fall back to a real navigation if JS hasn't loaded yet — the form works before hydration. That's [`server-components`](../server/server-components.md)'s territory; the client-side APIs here are identical whether the action runs on the client or the server.

**Passing extra arguments to an action.** A form action receives only `FormData`, but real mutations often need an ID or context alongside it. Two ways: close over the value (as `postId` does in the walkthrough), or bind it as a leading argument — `action={updatePost.bind(null, postId)}` — which prepends `postId` before the `FormData`. Binding is the idiomatic choice for **Server Actions**, where the bound argument is serialized and travels to the server, and it keeps the action defined outside the component:

```tsx
// updatePost(postId, formData) — postId bound, FormData supplied by the form.
<form action={updatePost.bind(null, post.id)}>
```

**Where Actions end and TanStack Query / RHF begin.** Actions own *form-shaped* mutations. Non-form mutations (a button that toggles a setting, an imperative retry) are often clearer as TanStack Query mutations; complex client-side validation and field arrays are RHF + Zod's job ([`forms-at-scale`](../forms/forms-at-scale.md) owns that boundary). Per the state-placement rule, don't force every mutation through a `<form action>` — use it where the interaction *is* a form.

## API and type reference

| API | Import | Signature | Returns |
| --- | --- | --- | --- |
| `useActionState` | `react` | `useActionState<S, P>(action: (prev: S, payload: P) => S \| Promise<S>, initialState: S, permalink?: string)` | `[state: S, dispatch: (payload: P) => void, isPending: boolean]` |
| `useFormStatus` | `react-dom` | `useFormStatus()` | `{ pending, data: FormData \| null, method, action }` |
| `useOptimistic` | `react` | `useOptimistic<S, O>(state: S, updateFn: (current: S, optimistic: O) => S)` | `[optimisticState: S, addOptimistic: (optimistic: O) => void]` |
| form action | — | `<form action={(formData: FormData) => void \| Promise<void>}>` | — |

Type notes: `useActionState` is generic in both the state `S` and the payload `P` — when the action is bound to a `<form>`, `P` is `FormData`. `useFormStatus` takes no arguments and reads context, so it's only meaningful inside a form. `useOptimistic`'s `updateFn` returns the *display* state, which must be the same type as the base state.

## Common mistakes

**1. Calling `useFormStatus` in the form-rendering component.** It reads the nearest *parent* form via context; called above the form's own provider, it always returns `pending: false`. It must live in a component rendered *inside* `<form>` (Stage 2).

**2. Reversing the `useActionState` reducer arguments.** The signature is `(previousState, formData)` — previous state first. Treating the first argument as the form data is the top `useActionState` bug; TypeScript catches it only if your state and `FormData` types differ enough.

**3. Expecting `useOptimistic` to persist.** It's a display overlay tied to the pending transition, not durable state. If the real state doesn't actually update on success (returned from the action, or refetched), the optimistic value evaporates at settle-time with nothing behind it. The overlay previews a real update; it doesn't replace one.

**4. Using `useOptimistic` outside an action.** The overlay only applies while a transition is pending. `addOptimistic` called outside an action does nothing lasting — there's no transition lifecycle to bind to.

**5. Mutating state in the action instead of returning it.** Same rule as reducers: return a new state object, don't mutate `previousState`. Mutation breaks the accumulator and the bailout checks ([`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md)).

**6. Throwing validation errors instead of returning them.** A thrown error unwinds to the error boundary and replaces the whole form with an error UI — wrong for "email is required." Return expected failures as state; throw only the exceptional (mistake-free forms shouldn't be able to hit the error boundary).

**7. Treating `isPending` as double-submit protection.** Disabling the button while pending improves UX but doesn't guarantee correctness — races and retries still slip through. Correctness is server idempotency ([`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md)).

**8. Controlled inputs and the auto-reset surprise.** React resets *uncontrolled* fields after a successful function action. With controlled inputs you own the value, so it won't clear unless you clear it — and if you expected the auto-reset, it won't happen. Pick one model per form ([`forms-controlled-and-uncontrolled`](../forms/forms-controlled-and-uncontrolled.md)).

**9. Reading `state` expecting a synchronous update.** `useActionState`'s state updates *after* the async action resolves, because it runs in a transition. There's no synchronous "just-submitted" value; gate on `isPending` for the in-flight UI.

**10. Forcing non-form mutations through form actions.** A settings toggle or an imperative retry isn't a form; a TanStack Query mutation is usually clearer. Use `<form action>` where the interaction is genuinely form-shaped (state-placement rule).

## How this evolved

- **Pre-19 (the manual trio):** `onSubmit` + `preventDefault`, a hand-managed `isSubmitting` boolean, a `try/catch` funneling failures into an error state, and — for optimistic UI — a `setState` up front with a manual revert in the `catch`. Every form reimplemented pending + error + optimistic + `FormData` parsing. This is exactly the "before" the [`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) recipe opens on.
- **18:** async transitions and `useTransition`'s pending flag ([`concurrent-rendering`](./concurrent-rendering.md)) laid the substrate — but there was no form binding and no optimistic primitive; you still wired forms by hand.
- **19:** Actions shipped as primitives — function form actions, `useActionState` (briefly named `useFormState` in `react-dom` during the RC before being renamed and moved to `react`), `useFormStatus`, `useOptimistic`, and Server Actions for RSC. The pending/error/optimistic trio became declarative.

```tsx
// legacy: React 19 RC — renamed to useActionState (from "react") before stable
// import { useFormState } from "react-dom";
```

- **19.2 (this baseline):** Actions are the default mutation story for form-shaped interactions, composing with everything else in this wave — an Action is an async transition, its pending state rides the same lanes, and its failures use the same error boundaries. The arc closes: React turned the four things every form hand-rolled into four primitives.

## Exercises

**1. Validation as state.** Build a form with `useActionState` that returns `{ error }` for empty/too-long input and only calls the API when valid. Show the error inline with no error boundary involved.
*Hint:* Return the error object from the reducer; render `state.error`. The reducer's first arg is previous state.

**2. Ambient submit button.** Write one `<SubmitButton>` using `useFormStatus` and drop it into two different forms unchanged. Confirm each shows its own form's pending state.
*Hint:* It must be a child of each `<form>`, and imported from `react-dom`.

**3. (Stretch) Optimistic with rollback.** Add `useOptimistic` so a new list item appears instantly (dimmed) and reverts automatically when you make the API call reject.
*Hint:* Call `addOptimistic` inside the action *before* the `await`; throw from the API to see the auto-revert. You write no `catch`.

## Summary

- An **Action** is an async function run inside a transition; React manages pending, errors, and optimistic reverts around it. `<form action={fn}>` triggers one on submit with a `FormData` payload and auto-resets uncontrolled fields.
- **`useActionState`** is `useReducer` productized: `(previousState, formData) => nextState`, async, transition-wrapped, returning `[state, formAction, isPending]`. Previous state is the first argument.
- **`useFormStatus`** (from `react-dom`) reads the ambient form's pending state from context — so it must be called *inside* the form, enabling a wire-free reusable submit button.
- **`useOptimistic`** overlays a transition-scoped optimistic value on real state and reverts automatically on success (dissolves into real state) or failure (snaps back) — no manual rollback.
- Return **expected** failures (validation) as state; **throw** exceptional ones to an error boundary. `isPending` is UX, not double-submit correctness — that's server idempotency.
- Use Actions where the interaction *is* a form; non-form mutations belong to TanStack Query, complex forms to RHF + Zod.

## See also

- [`usereducer-and-state-structure`](../state/usereducer-and-state-structure.md) — the reducer shape `useActionState` productizes; return-don't-mutate carries over.
- [`concurrent-rendering`](./concurrent-rendering.md) — async transitions, the substrate every Action runs on.
- [`forms-controlled-and-uncontrolled`](../forms/forms-controlled-and-uncontrolled.md) — `FormData`, the uncontrolled-first model, and the auto-reset behavior.
- [`error-boundaries`](../rendering/error-boundaries.md) — errors-as-state vs errors-as-thrown, the split validation-vs-exceptional failures follow.
- [`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) — these APIs against a real bug, plus the client-guard-vs-server-idempotency distinction.
- [`server-components`](../server/server-components.md) *(planned)* — Server Actions and progressive enhancement with `permalink`.

## References

- React docs — [`useActionState`](https://react.dev/reference/react/useActionState)
- React docs — [`useOptimistic`](https://react.dev/reference/react/useOptimistic)
- React docs — [`useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus)
- React docs — [`<form>` actions](https://react.dev/reference/react-dom/components/form)

## Demo source

_Demo source pending — StackBlitz link to be added in the demo pass._