---
recipe_id: mutate-async-unhandled-rejection
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - forms/forms-at-scale
  - recipes/forms-and-ux/double-submit-and-optimistic-like
status:
  drafted: true
  reviewed: false
---

# `mutateAsync` crashes the page — or nested `onSuccess` hell blocks a multi-step signup

> **What you'll build:** a clear rule — default to `mutate` for fire-and-forget writes; reach for `mutateAsync` only when you need a Promise (chained APIs, `Promise.all`, RHF `setError`) — with `try/catch` so rejections never become unhandled.

## The scenario

**Crash path.** A profile form uses `await updateProfile.mutateAsync(values)` inside RHF's `onSubmit` but the `catch` only handles 422. A 500 rejects the promise; nothing catches it → **Unhandled Promise Rejection** (and in some setups a redbox). Users see a broken submit with a console scream.

**Hell path.** Company signup needs `createCompany` → `createAdmin(company.id)` → `sendWelcomeEmail`. Devs nest `mutate({ onSuccess: () => mutate({ onSuccess: … }) })` — unreadable, error paths diverge, partial success leaves orphan companies.

**Why it escaped QA:** happy-path 200s never reject; nested callbacks "work" in the demo script; RHF examples online use `mutateAsync` without showing failure branches.

## Walkthrough

### Stage 1 — Name the return types

[`mutate` vs `mutateAsync`](../../ecosystem/data-fetching-tanstack-query.md#mutate-vs-mutateasync): `mutate` → `void`, errors via callbacks (Query handles the rejection). `mutateAsync` → `Promise`, **you** must catch.

### Stage 2 — Reject nesting `mutate` for sequencing

Callback pyramids lose shared `try/catch`, complicate loading flags, and duplicate toast logic. If you need the result of mutation 1 for mutation 2, you want `async/await`.

### Stage 3 — Pick the right API

**Fire-and-forget (default):**

```tsx
const del = useMutation({
  mutationFn: deletePost,
  onSuccess: () => {
    toast.success("Deleted");
    queryClient.invalidateQueries({ queryKey: ["posts"] });
  },
  onError: () => toast.error("Delete failed"),
});

<button type="button" onClick={() => del.mutate(postId)}>Delete</button>
```

**Sequenced / form integration:**

```tsx
const company = useMutation({ mutationFn: createCompany });
const admin = useMutation({ mutationFn: createAdmin });

async function register(data: FormValues) {
  try {
    const co = await company.mutateAsync(data.company);
    await admin.mutateAsync({ ...data.admin, companyId: co.id });
    toast.success("Registered");
  } catch (e) {
    toast.error(e instanceof Error ? e.message : "Registration failed");
  }
}
```

**RHF field errors:**

```tsx
const onSubmit = handleSubmit(async (values, helpers) => {
  try {
    await profile.mutateAsync(values);
  } catch (e) {
    if (isAxiosError(e) && e.response?.status === 422) {
      helpers.setError("email", { message: "Email already used" });
      return;
    }
    throw e; // or toast — but don't leave it unhandled
  }
});
```

### Stage 4 — Harden + verify the loop

- One argument to `mutate`/`mutateAsync` — pack `{ id, body }`.
- Don't mix: if you `mutateAsync`, don't also rely only on hook-level `onError` without catch (both can run; still catch the await).
- Partial multi-step: compensate (delete company) or use a server-side saga — client sequencing isn't a transaction.

**Verify the loop.** Force 500 on profile submit with `mutateAsync`: no unhandled rejection; user sees toast/field error. Run three-step register with step-2 failure: catch runs once; no nested callback spaghetti in the PR.

## Variations

1. **`Promise.all` uploads** — `await Promise.all(files.map((f) => upload.mutateAsync(f)))`.
2. **`Promise.allSettled`** — continue after one failure; report which files failed.
3. **mutate + per-call callbacks** — `mutate(vars, { onSuccess })` when one-off UI differs from hook defaults.
4. **Server Action forms** — prefer React 19 actions for form-shaped writes ([double-submit](../forms-and-ux/double-submit-and-optimistic-like.md)); Query mutations for non-form cache writes.
5. **Idempotent retries** — safe to `mutateAsync` again only if the API is idempotent.

## Trade-offs and common pitfalls

1. **`mutateAsync` without `try/catch`** — crash/unhandled rejection.
2. **Nested `onSuccess` chains** — unmaintainable sequencing.
3. **Assuming hook `onError` prevents unhandled rejection on await** — still catch awaits.
4. **Multiple args to `mutate`** — only one variables arg.
5. **Using `mutateAsync` "everywhere for consistency"** — loses the safe default.
6. **Ignoring partial success in chains** — orphan rows.
7. **Duplicating invalidate in await path and `onSettled`** — double fetch; pick one place.
8. **Blocking UI without pending state** — use `isPending` from the mutation(s).
9. **Catch that swallows 422 and 500 identically** — map status to UX.
10. **Testing only resolve paths** — inject rejects in Vitest.

### When NOT to use `mutateAsync`

If the UI only needs toast + invalidate, **`mutate` is correct**. Promises add footguns without leverage. Reach for `mutateAsync` when the **caller must await**.

## See also

- [`mutate` vs `mutateAsync`](../../ecosystem/data-fetching-tanstack-query.md#mutate-vs-mutateasync)
- [Forms at scale](../../forms/forms-at-scale.md) — RHF submit integration
- [Upload can't be cancelled](./upload-cant-be-cancelled.md) — mutation abort is separate

## References

- TanStack Query — [Mutations](https://tanstack.com/query/latest/docs/framework/react/guides/mutations)

## Demo source

- `demos/data-fetching/mutate-async-unhandled-rejection/` — reject paths + sequenced signup. *(Demo host TBD)*
