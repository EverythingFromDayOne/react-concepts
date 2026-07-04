---
recipe_id: double-submit-and-optimistic-like
track: forms-and-ux
primary_concept: actions
difficulty: foundational
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Double Submit and the Optimistic Like

> **What you'll build:** the two canonical mutation UX bugs, killed with React 19's primitives. Part A: a checkout button that charged 340 duplicate orders a month becomes structurally unable to double-fire — `<form action>` + `useActionState` pending state on the client, an idempotency key as the *actual* guarantee on the server. Part B: a Like button that felt broken at 400ms becomes instant with `useOptimistic` — including the automatic rollback nobody believes until they see it, and the base-commit step everybody forgets. Between them sits this recipe's real artifact: the **mutation classification table** — which writes get blocked, which get optimistic, which get neither — completing the race-policy table that [`../data-fetching/search-race-condition.md`](../data-fetching/search-race-condition.md) opened.

## The scenario

**Part A — the double charge.** Checkout POST: p50 **800ms**, p95 **2.4s**. The "Place order" button gives zero feedback while the request flies; median user re-click arrives at **~1.2s** — squarely inside the window. Result: **0.7% of orders duplicated — ~340/month**, each one a refund, a support ticket, and an inventory correction. The race-policy classification is unambiguous: this is a *write*, so the search recipe's latest-wins tools are forbidden (aborting a POST abandons the confirmation, not the charge — the server may have committed). Writes that must happen at most once take **first-wins**: block newcomers while one is pending.

**Part B — the sluggish Like.** Comment likes: POST p50 **350ms**. The correct-but-honest version — click, wait, flip on response — reads as *broken* at that latency; users click again mid-flight, and the hand-rolled "optimistic" version someone then writes has a drift bug: two rapid toggles produce two racing `setState` reconciliations, and the count ends one off. Bug reports: *"liked a comment, count went 12 → 13 → 12 → 13 → 14."* Same family as the search race — interleaved responses writing shared state — wearing mutation clothes.

Two bugs, two policies, one recipe: because choosing *between* them is the skill.

## Walkthrough

### Stage 0 — Part A's patient

```tsx
// ❌ the checkout that double-charges
<button
  onClick={async () => {
    const order = await placeOrder(cart);
    navigate(`/orders/${order.id}`);
  }}
>
  Place order
</button>
```

The full charge sheet: no pending state (the 800ms–2.4s window is wide open); no disable; errors vanish into an unhandled rejection; and it isn't even a form — no Enter-key submission, no `FormData`, none of the platform semantics [`../../forms/forms-controlled-and-uncontrolled.md`](../../forms/forms-controlled-and-uncontrolled.md) established as the substrate.

### Stage 1 — the fix by hand, so the primitive isn't magic

Pre-19, you build first-wins from parts you already own — a status union ([`../../foundations/thinking-in-react.md`](../../foundations/thinking-in-react.md)) and a guard:

```tsx
type OrderFlow =
  | { status: 'idle' }
  | { status: 'pending' }
  | { status: 'error'; message: string };

const [flow, setFlow] = useState<OrderFlow>({ status: 'idle' });

async function handlePlaceOrder() {
  if (flow.status === 'pending') return;          // first-wins: newcomers bounce
  setFlow({ status: 'pending' });
  try {
    const order = await placeOrder(cart, { idempotencyKey });
    navigate(`/orders/${order.id}`);
  } catch (err) {
    setFlow({ status: 'error', message: messageOf(err) });
  }
}
```

And the line that matters more than any React code — **the client guard is UX; the server is correctness.** A crashed tab, a retried request, a second device: none of them respect `disabled`. The invariant lives in an idempotency key the server dedupes on:

```tsx
// One key per checkout *intent*: stable across retries of this attempt,
// rotated only after success (lazy init: rules-of-react, exercise 3b).
const [idempotencyKey, setIdempotencyKey] = useState(() => crypto.randomUUID());
// on success: setIdempotencyKey(crypto.randomUUID())  — the next order is a new intent
```

Same key on a retry → server returns the original order instead of charging twice. This works and ships today; its tax is the plumbing — per-form union, per-trigger guards (miss one call site and the hole reopens), pending state hand-carried into every submit button. That tax is exactly what React 19 standardized away.

### Stage 2 — the platform version: `<form action>` + `useActionState`

```tsx
type OrderState =
  | { status: 'idle' }
  | { status: 'success'; orderId: string }
  | { status: 'error'; message: string };

export function CheckoutForm({ cart }: { cart: Cart }) {
  const [idempotencyKey, setIdempotencyKey] = useState(() => crypto.randomUUID());

  const [state, formAction, isPending] = useActionState(
    async (_prev: OrderState, formData: FormData): Promise<OrderState> => {
      try {
        const order = await placeOrder({
          cartId: cart.id,
          note: (formData.get('note') as string) ?? '',
          idempotencyKey,
        });
        setIdempotencyKey(crypto.randomUUID());          // next intent, next key
        return { status: 'success', orderId: order.id };
      } catch (err) {
        return { status: 'error', message: messageOf(err) };
      }
    },
    { status: 'idle' },
  );

  if (state.status === 'success') return <OrderConfirmation orderId={state.orderId} />;

  return (
    <form action={formAction}>
      <label>Delivery note <input name="note" /></label>
      {state.status === 'error' && <p role="alert">{state.message}</p>}
      <SubmitButton />
    </form>
  );
}
```

What the primitive folded in: the pending flag (`isPending`), the returned-state channel (errors are *data*, rendered like any state), Enter-key submission and `FormData` for free because it's a real form, and — since actions run inside React's transition machinery — successive submissions of the same action are queued sequentially rather than interleaved (the mechanics belong to [`../../concurrent/actions.md`](../../concurrent/actions.md); this recipe uses the surface). The guard itself becomes declarative, and it lives in the *button* — via the third primitive:

```tsx
// SubmitButton.tsx — design-system-grade: pending-aware with zero props
import { useFormStatus } from 'react-dom';

export function SubmitButton() {
  const { pending } = useFormStatus();      // reads the NEAREST enclosing <form>'s status
  return (
    <button disabled={pending} aria-busy={pending}>
      {pending ? 'Placing order…' : 'Place order'}
    </button>
  );
}
```

`useFormStatus` is the `useContext`-of-forms: any child of the form reads its submission state with no drilling — which is how a shared `Button` gains pending-awareness once, everywhere. Guard placement note: because the block lives in the action pathway and the button, every trigger converges on it — Enter key, click, `form.requestSubmit()` — closing the "we guarded `onClick` but not the other door" hole from stage 1. One forms-article callback: after a *successful* action, React resets the form's uncontrolled fields automatically — the `note` clears itself; on error, re-seed anything you want preserved from the returned state.

Part A's numbers close: pending window guarded client-side, idempotency guarding the universe — duplicate orders **340/month → 0** in the month after ship (the three residual duplicates were double *taps* on a payment provider's own page — someone else's recipe).

### Stage 3 — Part B's patient, and the drift trace

The honest-but-sluggish Like, then the hand-rolled optimistic that drifts:

```tsx
// ❌ hand-rolled optimistic — two racing writers, one shared truth
const [liked, setLiked] = useState(comment.liked);
const [count, setCount] = useState(comment.likeCount);

async function toggle() {
  setLiked(!liked);                                //  optimistic flip (snapshot!)
  setCount((c) => c + (liked ? -1 : 1));
  try {
    const server = await setLike(comment.id, !liked);
    setLiked(server.liked);                        //  "reconcile"
    setCount(server.count);
  } catch {
    setLiked(liked);                               //  "rollback" (that snapshot again)
    setCount((c) => c + (liked ? 1 : -1));
  }
}
```

Trace the double-tap (like at t=0, unlike at t=120ms, server p50 350ms): request A (like) and B (unlike) are both in flight; B resolves first and reconciles to `{liked:false, 12}`; then A lands late and reconciles to `{liked:true, 13}` — the UI now shows *liked* when the user's last intent was *unliked*, and the server (which processed like→unlike) disagrees with the screen. Every arm of this function reads a `liked` snapshot from a different render ([`../../state/state-and-usestate.md`](../../state/state-and-usestate.md)); with two writers and no notion of "base truth vs. pending overlay," interleavings multiply faster than you can patch them.

### Stage 4 — `useOptimistic`: base + overlay, rollback for free

The primitive's model is the fix: there is one **base** (the confirmed truth, owned above) and an **overlay** of optimistic updates that exists only while an action is in flight. When the action settles, the overlay evaporates and the UI shows the base — whatever it now is:

```tsx
interface LikeSnapshot { liked: boolean; count: number }

export function LikeButton({
  comment,
  onCommit,                                  // parent commits server truth to the base
}: {
  comment: LikeSnapshot & { id: string };
  onCommit: (next: LikeSnapshot) => void;
}) {
  const [isPending, startTransition] = useTransition();
  const [optimistic, applyOptimistic] = useOptimistic(
    { liked: comment.liked, count: comment.count },            // base: from props
    (state: LikeSnapshot, nextLiked: boolean): LikeSnapshot => ({
      liked: nextLiked,
      count: state.count + (nextLiked ? 1 : -1),
    }),
  );

  function toggle() {
    const nextLiked = !optimistic.liked;                        // last intent
    startTransition(async () => {
      applyOptimistic(nextLiked);                               // overlay: instant
      try {
        const server = await setLike(comment.id, nextLiked);   // absolute payload — see pitfalls
        onCommit(server);                                       // commit truth to the base
      } catch {
        toast("Couldn't update like");                          // base unchanged →
      }                                                         // overlay evaporates = rollback
    });
  }

  return (
    <button onClick={toggle} aria-pressed={optimistic.liked} data-pending={isPending}>
      ♥ {optimistic.count}
    </button>
  );
}
```

The two mechanics that make the drift structurally impossible:

**Rollback is the absence of a commit.** On failure, nothing wrote the base; when the transition settles, the overlay is discarded and the UI *snaps back to truth automatically* — no rollback arm, no inverse math to get wrong. The flip side is the #1 confusion in the wild: **on success you must commit the base** (`onCommit` → parent state, or a TanStack cache write). Skip it and the UI "rolls back" after *succeeding* — the overlay evaporated and the base was still old. If your optimistic UI flashes correct-then-stale, you're not watching a failure; you're watching an uncommitted success.

**One base, one overlay, one queue.** Rapid toggles apply overlay updates against a single source; the async work runs in transitions that queue sequentially, so intents apply in order and the final state is the last intent — the interleaving from stage 3 has no second writer to race. Belt-and-suspenders on the wire: send the *absolute* target (`liked: true`), never a relative toggle — an idempotent payload makes even network-level reordering harmless, the same absolutism the idempotency key bought Part A.

### Stage 5 — the classification table

The artifact this recipe exists to leave behind. For every mutation in the app, one row:

| Mutation shape | Policy | Primitive | Why |
| --- | --- | --- | --- |
| Money / at-most-once (checkout, send, book) | **First-wins block** + server idempotency | `<form action>` + `useActionState` / `useFormStatus` | Duplicate cost is catastrophic; latency is tolerable with honest pending UI |
| Cheap reversible toggle (like, star, follow, mute) | **Optimistic, last-intent-wins** | `useOptimistic` in a transition + base commit | Failure is rare, rollback is one glyph; latency here reads as breakage |
| Create-into-list (post a comment) | **Optimistic insert** with temp identity | `useOptimistic` (array reducer) — variation below | Perceived speed, with a visible "sending" affordance |
| Destructive with regret window (delete item) | **Optimistic + undo** | Optimistic remove + undo toast — variation below | Rollback becomes a *feature* the user drives |
| Irreversible destructive (delete account, refund) | **Neither** — confirm, then first-wins | Explicit confirm + blocked action | No optimism where there's no undo; friction is the UX |
| Ordered dependent writes (reorder steps) | **Sequential queue** | Queued actions | Order *is* the data — real-time / state-management tracks (queued) |

The three-question test for the optimistic column: is success the overwhelmingly likely outcome? is rollback semantically and visually cheap? does the user's mental model say "done" at click-time? Three yeses or it's a pending-state mutation. **Money never gets optimism** — a rollback is not a refund.

## Variations

### Optimistic list insert — posting a comment

`useOptimistic` over an array, temp identity minted at creation:

```tsx
const [optimisticComments, addOptimistic] = useOptimistic(
  comments,
  (list: Comment[], draft: Comment) => [...list, draft],
);
// in the action: addOptimistic({ ...draft, id: `temp-${crypto.randomUUID()}`, sending: true });
```

Render `sending` rows dimmed with a retry affordance on failure. The key discipline from [`../../rendering/rendering-lists-and-keys.md`](../../rendering/rendering-lists-and-keys.md) applies with a twist: the temp id is the row's key *until the server truth commits into the base* — at which point the optimistic row evaporates and the real row (server id) mounts. Accept that identity swap (a remount) or, if the row holds state worth preserving, key on a client-generated id the server *echoes back* — the id-minted-at-creation rule doing double duty.

### The TanStack mutation mapping

In a cache-integrated app the same pattern wears library clothes: `useMutation` with `onMutate` (cancel outgoing queries for the key, snapshot the cache, write the optimistic value), `onError` (restore the snapshot), `onSettled` (invalidate). It's this recipe's base/overlay model with the *cache as the base* — reach for it when the liked state must be coherent across every component reading the query, which is exactly the dedup boundary the search recipe drew ([`../../ecosystem/data-fetching-tanstack-query.md`](../../ecosystem/data-fetching-tanstack-query.md), Wave 4).

### Optimistic delete with undo

Remove the row instantly; float a 5-second toast with **Undo**; only when the toast expires does the DELETE fire. The genuinely deferred write means undo is free (nothing to un-delete server-side) at the cost of a consistency window (another tab still sees the item). The immediate-delete-plus-restore variant inverts the trade. Either way the timer is event-owned — it started from a click, so it lives with the handler's closure, not in an effect watching a `pendingDelete` flag ([`../../effects/effects-and-synchronization.md`](../../effects/effects-and-synchronization.md), branch 4).

## Trade-offs and common pitfalls

### When NOT to use this

- **Never optimistic on money or the irreversible.** The rollback animation does not un-charge the card or un-send the email. Those rows of the table are pending-state rows, permanently.
- **Never first-wins on high-frequency toggles.** A Like that `disabled`s itself for 350ms per tap feels broken in the other direction — the policy-inversion twin of search-with-first-wins from the race recipe. Match the policy to the interaction, not to whichever primitive you learned first.
- **Never client-only guards for correctness.** Disable, pending flags, and queues are UX. The at-most-once invariant is the server's idempotency contract; without it you've built a very polite race condition.

### Pitfalls

1. **Guarding one door.** The `onClick` guard misses Enter-key submission and `requestSubmit()` callers. Guards live where submission converges — the action and the form-status-aware button — not per-trigger.
2. **The uncommitted success.** Optimistic value "reverts" after the request *succeeds* — because nothing wrote the base. `useOptimistic` is overlay-only by design; the success path must commit truth (parent state / cache) or the evaporating overlay exposes stale base every time.
3. **State updates after `await` fall out of the transition.** Inside `startTransition(async () => { … await …; setX() })`, the post-await `setX` is no longer transition-scoped — wrap continuation updates in a fresh `startTransition`, or route them through the action-state channel. Silent, and it degrades pending semantics rather than crashing.
4. **Two-writer optimism.** The stage-3 pattern — optimistic `setState` plus reconcile `setState` — drifts under any interleaving. If you can't use the primitive, at minimum: single reducer, absolute server payloads, request tagging (the search recipe's diagnostics apply to writes too).
5. **Relative payloads.** Sending `toggle` instead of `liked: true` means reordered requests invert the result even with a perfect client. Idempotent, absolute mutation payloads are the wire-level half of every fix in this recipe.
6. **Idempotency key lifecycle wrong in either direction.** New key per *click* deduplicates nothing; one key *forever* blocks the user's genuine second order. The key's lifetime is the intent's lifetime: stable across retries, rotated on success.
7. **Abort-on-unmount for mutations.** Copying the read recipe's cleanup onto a POST abandons the confirmation of a charge that may have landed. Writes complete; what you cancel is the *UI's interest*, by letting the settled handler no-op if the surface is gone.
8. **`useFormStatus` outside the form.** It reads the nearest *enclosing* form; rendered as a sibling it reports `pending: false` forever — the guard silently vanishes. It's a child-of-`<form>` hook, structurally.
9. **The auto-reset surprise.** React 19 resets uncontrolled fields when the action completes — delightful on success, data loss on a validation round-trip unless the action returns the entered values and the form re-seeds `defaultValue`s from state ([`../../forms/forms-controlled-and-uncontrolled.md`](../../forms/forms-controlled-and-uncontrolled.md)).
10. **Debounce as a mutation guard.** A 300ms click-debounce still double-fires for slow double-clickers and taxes every single-clicker. Debounce throttles *reads* (the search recipe); mutations get guards and idempotency.
11. **Silent rollbacks.** Optimism that reverts without a toast/`aria-live` announcement gaslights the user ("I *did* like that") and hides your failure rate from you. Every rollback is surfaced and counted — the metric is the health of the whole pattern.

## See also

- [`../data-fetching/search-race-condition.md`](../data-fetching/search-race-condition.md) — the race-policy table this recipe completes; the write-cancellation rule it inherits
- [`../../forms/forms-controlled-and-uncontrolled.md`](../../forms/forms-controlled-and-uncontrolled.md) — the `FormData`/action substrate, auto-reset behavior, uncontrolled-first discipline
- [`../../concurrent/actions.md`](../../concurrent/actions.md) *(Wave 3)* — the full Actions machinery: transitions, queueing, `useActionState` internals
- [`../../foundations/thinking-in-react.md`](../../foundations/thinking-in-react.md) — status unions, the hand-rolled half of stage 1
- [`../../state/state-and-usestate.md`](../../state/state-and-usestate.md) — snapshot semantics behind the stage-3 drift
- [`../../ecosystem/data-fetching-tanstack-query.md`](../../ecosystem/data-fetching-tanstack-query.md) *(Wave 4)* — the cache-as-base variant

## References

- [`<form>` — the `action` prop (react.dev)](https://react.dev/reference/react-dom/components/form)
- [`useActionState` (react.dev)](https://react.dev/reference/react/useActionState)
- [`useFormStatus` (react.dev)](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [`useOptimistic` (react.dev)](https://react.dev/reference/react/useOptimistic)
- [`useTransition` (react.dev)](https://react.dev/reference/react/useTransition)
- [TanStack Query — Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../../progress.md) TODOs.