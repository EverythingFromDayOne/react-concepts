---
article_id: forms-at-scale
concept_folder: forms
wave: 4
related:
  - forms/forms-controlled-and-uncontrolled
  - concurrent/actions
  - ecosystem/routing-react-router
  - ecosystem/state-management-landscape
  - rendering/rendering-lists-and-keys
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this:** React 19's form Actions own the *submission* lifecycle — submit, pending, optimistic, error. React Hook Form owns the *field* lifecycle — per-key validation, touched/dirty state, field arrays, cross-field rules. The decision isn't "which library is better"; it's "where is the complexity — in the mutation or in the fields?" A login form is submission-shaped; reach for Actions. A 30-field onboarding form with a dynamic list of team invites is field-shaped; reach for RHF. This article draws that line and shows RHF + Zod doing the field-heavy half, with one schema validating both the client and the server.

## What it is

**React Hook Form (RHF)** is an *uncontrolled-first* form-state library: instead of holding every field in React state (the controlled model from [`forms-controlled-and-uncontrolled`](./forms-controlled-and-uncontrolled.md)), it registers inputs by ref and reads their values from the DOM, so typing in one field doesn't re-render the whole form. **Zod** is a schema validator whose schema doubles as a TypeScript type and — critically — runs on *both* the client (instant UX) and the server (actual correctness).

The two debts this article pays are the same line drawn from two directions. [`actions`](../concurrent/actions.md) noted that form Actions handle the simple case and *"where RHF begins"* is out of scope there; [`forms-controlled-and-uncontrolled`](./forms-controlled-and-uncontrolled.md) named a *"Zod/RHF threshold."* Here's the threshold made explicit:

| Signal | Stay with Actions (`<form action>` + `useActionState`) | Reach for RHF + Zod |
| --- | --- | --- |
| Field count | a handful | many (10+), or grows dynamically |
| Validation timing | on submit (server, or a quick client check) | per-field, on blur/change, with live feedback |
| Field UX | pending + a submit error is enough | touched/dirty state, per-field errors, focus management |
| Structure | flat | field arrays (add/remove rows), conditional/nested fields, wizards |
| Cross-field rules | none, or server-checked | password-match, "expedited requires phone", etc. |

The axis is **submission complexity vs field complexity**. When the mutation is the hard part, Actions win — don't pull in RHF for a two-field form (mistake 1). When the *fields* are the hard part, RHF earns its weight. And they compose: RHF's `handleSubmit` can call a server action, and the same Zod schema validates on both sides.

## How it works under the hood

Two mechanisms explain why RHF scales where controlled-everything doesn't.

**Uncontrolled by default.** `register("email")` returns `{ name, ref, onChange, onBlur }` and spreads onto a native input. The value lives in the DOM, not in React state, so a keystroke doesn't call `setState` and doesn't re-render the form — the opposite of the controlled enforcement loop ([`forms-controlled-and-uncontrolled`](./forms-controlled-and-uncontrolled.md#the-enforcement-loop)) that re-renders on every character. For a 50-field form that's the difference between smooth and janky, and it's the same uncontrolled-via-refs insight, productized.

**Proxy-based `formState`.** `formState` is a proxy: you only *subscribe* to the pieces you read. Destructure `errors` and your component re-renders when errors change but not when `isDirty` flips; read `isSubmitting` and you track only that. Nothing re-renders on keystrokes unless a field you're displaying an error for changes validity. (One consequence, flagged in the RHF docs: the `useForm` return is being memoized, so putting the whole return object in a `useEffect` dependency array can cause loops — depend on specific fields.)

**The resolver pipeline.** On the configured `mode` trigger (submit, blur, change…), RHF gathers the current values and hands them to `zodResolver(schema)`, which runs `schema.safeParse`, then maps each `ZodError` issue onto `formState.errors` by its `path`. A field array error at `invites.2.email` lands exactly at `errors.invites[2].email`. Validation logic lives entirely in the schema; RHF just routes the results to fields.

## Basic usage

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod"; // v4: the default `zod` import since July 2025

const schema = z.object({
  email: z.email("Enter a valid email"), // v4: top-level z.email(), not z.string().email()
  age: z.number().min(18, "Must be 18+"),
});
type FormValues = z.infer<typeof schema>;

export function SignupForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormValues>({ resolver: zodResolver(schema), mode: "onTouched" });

  return (
    <form onSubmit={handleSubmit((values) => console.log(values))} noValidate>
      <input {...register("email")} placeholder="Email" />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <input type="number" {...register("age", { valueAsNumber: true })} />
      {errors.age && <p role="alert">{errors.age.message}</p>}

      <button disabled={isSubmitting}>Sign up</button>
    </form>
  );
}
```

Two details that bite if skipped: `valueAsNumber` (native inputs yield *strings*, so `z.number()` fails without it — mistake 8), and `mode: "onTouched"` (validate after the first blur, then live — usually the least-annoying UX; the default `onSubmit` only validates on submit).

## Walkthrough — an account form with a field array, cross-field rules, and one shared schema

We'll build account creation: email, a password + confirmation (cross-field), and a dynamic list of team invites (field array) — then submit to a server action that re-validates with the *same* schema and surfaces a server-only error back onto a field.

### Step 1 — One schema, both sides

```tsx
// account-schema.ts
import { z } from "zod";

export const accountSchema = z
  .object({
    email: z.email("Enter a valid email"),
    password: z.string().min(8, "At least 8 characters"),
    confirmPassword: z.string(),
    invites: z.array(z.object({ email: z.email("Invalid invite email") })).max(5, "Up to 5 invites"),
  })
  .refine((data) => data.password === data.confirmPassword, {
    error: "Passwords don't match", // v4: `error`, not `message`
    path: ["confirmPassword"], // attach to the field, not the form root (mistake 6)
  });

export type AccountInput = z.infer<typeof accountSchema>;
```

The cross-field rule is a `.refine` with an explicit `path` so the error lands on `confirmPassword`, where the user can see it — a rule with no `path` attaches to the form root and vanishes from the UI.

### Step 2 — The form: register, field array, per-field errors

```tsx
import { useForm, useFieldArray, type SubmitHandler } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { accountSchema, type AccountInput } from "./account-schema";
import { createAccount } from "./actions";

export function AccountForm() {
  const {
    register,
    control,
    handleSubmit,
    setError,
    reset,
    formState: { errors, isSubmitting },
  } = useForm<AccountInput>({
    resolver: zodResolver(accountSchema),
    mode: "onTouched",
    defaultValues: { email: "", password: "", confirmPassword: "", invites: [] },
  });

  const { fields, append, remove } = useFieldArray({ control, name: "invites" });

  const onSubmit: SubmitHandler<AccountInput> = async (values) => {
    const result = await createAccount(values);
    if (result.fieldErrors) {
      for (const [name, message] of Object.entries(result.fieldErrors)) {
        setError(name as keyof AccountInput, { message }); // map server-only errors onto fields
      }
      return;
    }
    reset(); // success: clear the form (or navigate / show a confirmation)
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <input {...register("email")} placeholder="Email" />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <input type="password" {...register("password")} placeholder="Password" />
      {errors.password && <p role="alert">{errors.password.message}</p>}

      <input type="password" {...register("confirmPassword")} placeholder="Confirm password" />
      {errors.confirmPassword && <p role="alert">{errors.confirmPassword.message}</p>}

      {fields.map((field, i) => (
        <div key={field.id}> {/* field.id, NEVER the index (mistake 4) */}
          <input {...register(`invites.${i}.email`)} placeholder="Invite email" />
          {errors.invites?.[i]?.email && <p role="alert">{errors.invites[i]!.email!.message}</p>}
          <button type="button" onClick={() => remove(i)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ email: "" })}>Add invite</button>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Creating…" : "Create account"}
      </button>
    </form>
  );
}
```

### Step 3 — The server never trusts the client

The client validation is UX; the server re-parses with the *same* schema for correctness, and owns the checks only it can make (uniqueness):

```tsx
// actions.ts
import { accountSchema } from "./account-schema";

export async function createAccount(input: unknown) {
  const parsed = accountSchema.safeParse(input); // defense in depth — re-validate server-side
  if (!parsed.success) {
    return { fieldErrors: flattenIssues(parsed.error.issues) }; // v4: .issues, not .errors
  }
  if (await emailTaken(parsed.data.email)) {
    return { fieldErrors: { email: "That email is already registered" } }; // server-only rule
  }
  await insertAccount(parsed.data);
  return { fieldErrors: null as null };
}
```

The `email-taken` error is something the client *cannot* know, so it comes back from the action and `setError` drops it onto the email field. That round trip — client-validate for speed, server-validate for truth, map server-only failures back to fields — is the production shape of a serious form.

## Real-world patterns

**One schema, two validations.** The highest-leverage pattern in the walkthrough: `z.infer` gives the form's TypeScript type, `zodResolver` validates on the client, and the action calls `.safeParse` on the server. One source of truth for shape *and* rules; the client copy is a UX accelerator, never the authority.

**Field arrays.** `useFieldArray({ control, name })` returns `{ fields, append, prepend, remove, swap, move, insert, update }`. Always key the rows with `field.id` (RHF generates it) — using the array index recycles state across reorders and removals, the exact corruption [`rendering-lists-and-keys`](../rendering/rendering-lists-and-keys.md#index-keys) warns about, now inside a form.

**Controlled UI-library inputs via `Controller`.** `register` works for native inputs; a custom `<Select>` or an MUI component that doesn't forward a ref needs `Controller` (or `useController`), which bridges RHF to a controlled component:

```tsx
<Controller
  name="country"
  control={control}
  render={({ field, fieldState }) => (
    <CountrySelect value={field.value} onChange={field.onChange} invalid={fieldState.invalid} />
  )}
/>
```

**Validation-mode UX.** `mode` is the feel of the form: `onSubmit` (default; nothing until submit), `onBlur`, `onChange` (eager, can nag), `onTouched` (validate after first blur then live — usually best), `all`. The `validation-fires-on-first-keystroke` recipe *(planned)* is precisely the wrong-mode failure.

**Wizards / multi-step.** For a form split across steps, `FormProvider` + `useFormContext` share one form instance across step components without prop-drilling `control`. The form's state is *not* global state — it lives with the form and dies on unmount ([`state-management-landscape`](../ecosystem/state-management-landscape.md)); don't lift it into Zustand.

**Bundle note.** Zod v4's top-level formats (`z.email()`) are tree-shakable where the old chained `.email()` wasn't; for extreme bundle sensitivity, `zod/mini` exposes the same runtime with a fully functional, tree-shakable API.

## API / type reference

| Symbol | Shape | Notes |
| --- | --- | --- |
| `useForm(options)` | `{ resolver, defaultValues, mode }` → `{ register, handleSubmit, control, setError, reset, watch, formState }` | `mode`: `onSubmit`(default)/`onBlur`/`onChange`/`onTouched`/`all` |
| `register(name, opts?)` | `{ name, ref, onChange, onBlur }` | uncontrolled; `valueAsNumber`/`valueAsDate` for coercion |
| `Controller` / `useController` | render-prop / hook | bridge to controlled UI-lib components |
| `useFieldArray({ control, name })` | → `{ fields, append, prepend, remove, swap, move, insert, update }` | key rows with `field.id`, never index |
| `formState` | `{ errors, isSubmitting, isDirty, isValid, touchedFields, dirtyFields }` | a proxy — you subscribe only to what you read |
| `zodResolver(schema)` | resolver | `@hookform/resolvers/zod`; Zod is a Standard Schema lib |
| `z.object` / `z.email` / `.refine(fn, { error, path })` | Zod v4 | `z.email()` not `.email()`; `error` not `message`; `.issues` not `.errors` |

## Common mistakes

1. **Pulling in RHF for a two-field form.** A login or newsletter form is submission-shaped: `<form action>` + `useActionState` is lighter and needs no library. Cross the threshold deliberately, not by habit.
2. **Controlling every input by hand for a large form.** Ties each keystroke to a `setState` and re-renders the whole form — the jank RHF's uncontrolled model exists to remove. If you're writing `value`/`onChange` for 30 fields, you want `register`.
3. **Trusting client validation.** RHF + Zod on the client is UX; a malicious or buggy client can send anything. Re-validate on the server with the same schema, always. Client speed, server truth.
4. **`key={index}` in a field array.** Removing or reordering rows recycles the wrong state into the wrong row — form corruption. Use `key={field.id}`.
5. **Zod v3 idioms in v4.** `z.string().email()`, `{ message }`, and `error.errors` are deprecated/changed: use `z.email()`, `{ error }`, and `error.issues`. And `ctx.path` is gone from `superRefine` — target cross-field errors with `.refine(fn, { path })` instead.
6. **Cross-field error with no `path`.** `.refine(fn, { error })` without `path` attaches the error to the form root, where no field renders it and the user never sees it. Always give the `path` to the field that should show it.
7. **Putting the whole `useForm()` return in a `useEffect` dep array.** The return is memoized; the full object in deps can loop. Depend on the specific value (`formState.isValid`), not the object.
8. **Forgetting `valueAsNumber`/`valueAsDate`.** Native inputs return strings; `register("age", { valueAsNumber: true })` is required or `z.number()` rejects a `"25"`.
9. **Mixing `register` and controlled `value` on one input.** Pick one: `register` (uncontrolled) or `Controller` (controlled). Both on the same field fight each other.
10. **Hand-rolling validation instead of the resolver.** Manual `if (!email.includes("@"))` checks drift from your types and duplicate server rules. A shared Zod schema gives typed, single-source validation for free.

## How this evolved

The controlled model — every input in React state — was React's default and re-renders the form on every keystroke; fine for three fields, painful for thirty. RHF answered with the *uncontrolled* model (register by ref, read from the DOM) plus proxy-subscribed form state, trading a little "React-idiomatic" purity for large-form performance. Validation moved from hand-written checks to schemas (Yup, then Zod), and **Zod v4** (July 2025, now the default `zod` import) sharpened that: top-level tree-shakable formats (`z.email()`), a unified `error` param, and Standard Schema conformance so resolvers plug in uniformly. Then **React 19 Actions** absorbed the *simple* submission case — `useActionState`/`useFormStatus`/`useOptimistic` ([`actions`](../concurrent/actions.md)) — which re-drew the boundary rather than erasing it: Actions took the submission-shaped forms, RHF kept the field-shaped ones, and the shared-Zod-schema-across-client-and-server pattern became the connective tissue between them. The routing layer adds a third form entry point — React Router's `<Form>`/`action` ([`routing-react-router`](../ecosystem/routing-react-router.md)) — same `FormData`, different system; pick by whether the mutation is navigation-shaped, component-shaped (Actions), or field-shaped (RHF).

## Exercises

1. **Draw the boundary.** For each, decide Actions vs RHF and say why: (a) a login form, (b) a 25-field employee profile with a variable list of dependents, (c) a one-click "follow" button. *Hint: submission complexity vs field complexity — one of these isn't even a form.*
2. **Field array with per-row validation.** Build a form with a dynamic list of URLs, each validated with `z.url()`, add/remove buttons, and errors shown per row. *Hint: `useFieldArray`, `key={field.id}`, register as `` `links.${i}.url` ``.*
3. **Share a schema across the boundary.** Wire one Zod schema to both an RHF client form and its server action; force a server-only failure (email already taken) and surface it on the email field. *Hint: `.safeParse` on the server, return `fieldErrors`, `setError` on the client.*

## Summary

- The Actions/RHF boundary is submission-complexity vs field-complexity. Simple submit → Actions; many fields, per-field UX, field arrays, cross-field rules → RHF + Zod. Don't cross it by reflex.
- RHF scales because it's uncontrolled (register by ref, no re-render per keystroke) with proxy-subscribed `formState`; that's the controlled-model performance problem, solved.
- The resolver runs your Zod schema and maps issues to fields by path. One schema gives the form's type, client validation, and server validation.
- Validate on the client for UX and re-validate on the server for truth; map server-only errors (uniqueness) back onto fields with `setError`.
- Zod v4 idioms: `z.email()`, `{ error }`, `error.issues`, `.refine(fn, { path })` for cross-field. Field arrays key on `field.id`, never index.
- Form state is not global state — it lives with the form, not in a store.

## See also

- [`forms-controlled-and-uncontrolled`](./forms-controlled-and-uncontrolled.md) — the controlled/uncontrolled model and the Zod/RHF threshold this article crosses
- [`actions`](../concurrent/actions.md) — the submission lifecycle that owns the *simple* half of the boundary
- [`routing-react-router`](../ecosystem/routing-react-router.md) — the third form entry point (`<Form>`/`action`), same `FormData`, different system
- [`rendering-lists-and-keys`](../rendering/rendering-lists-and-keys.md) — why field arrays key on `field.id`, not index
- [`state-management-landscape`](../ecosystem/state-management-landscape.md) — why form state stays with the form and not in a store
- [`double-submit-and-optimistic-like` recipe](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) — the pending/`isSubmitting` guard and optimistic UI companion

## References

- React Hook Form — [`useForm`](https://react-hook-form.com/docs/useform), [`useFieldArray`](https://react-hook-form.com/docs/usefieldarray), [`Controller`](https://react-hook-form.com/docs/usecontroller/controller)
- React Hook Form — [Resolvers](https://github.com/react-hook-form/resolvers) (`zodResolver`, Standard Schema support)
- Zod — [Migration guide (v4)](https://zod.dev/v4/changelog) and [Customizing errors](https://zod.dev/error-customization)
- Zod — [Defining schemas](https://zod.dev/api) (`z.email`, `.refine`, cross-field `path`)

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).