---
article_id: forms-controlled-and-uncontrolled
concept_folder: forms
wave: 1
related:
  - state-and-usestate
  - conditional-rendering-and-events
  - components-and-props
  - actions
  - forms-at-scale
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

# Forms: Controlled and Uncontrolled

> **Lead with this:** Every form field in React answers to one of two masters. **Controlled**: React state is the value — the DOM is just displaying it, and React will forcibly overwrite anything that disagrees. **Uncontrolled**: the DOM owns the value, and React reads it when it cares (usually at submit, via `FormData`). Neither is "the React way" — the folklore that everything must be controlled produced a decade of forms paying per-keystroke re-renders for values nobody reads until submit, and React 19's form actions swung the pendulum firmly back toward the platform. The professional answer is per-*field*, driven by one question: **does anything need this value before submit?** Live validation, character counts, dependent fields → controlled. Otherwise → uncontrolled, and let the browser do its job. This article builds both models down to the enforcement mechanics, then walks a real form through the migration that requirements always force.

## What it is

**A controlled field** takes its value from React state and reports edits back through `onChange`:

```tsx
const [email, setEmail] = useState('');
<input type="email" value={email} onChange={(e) => setEmail(e.target.value)} />
```

The state is the single source of truth ([`../foundations/thinking-in-react.md`](../foundations/thinking-in-react.md)); the input renders it like any other projection. That buys you everything per-keystroke: validate as they type, filter a list, count characters, enable Submit only when valid, transform input. The price: a render per keystroke (cheap in the Compiler era, but not free), and the obligation to wire `onChange` correctly — a controlled input without a working change handler is a *frozen* input, because React enforces.

**An uncontrolled field** lets the DOM keep the value; React seeds it at most once and reads it on demand:

```tsx
<input name="email" type="email" defaultValue={user.email} />
```

No state, no re-renders while typing, and the browser's native machinery — autofill, IME composition, undo — runs untouched. The modern read mechanism is not refs but **`FormData`** at submit time, keyed by `name` attributes:

```tsx
function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const fd = new FormData(e.currentTarget);   // currentTarget: typed, and it's the form
  const email = fd.get('email') as string;
}
```

The `name` attribute is the uncontrolled contract — no `name`, no entry in the `FormData`. This platform-first shape is also exactly what React 19's form actions consume (`<form action={fn}>` hands your function the `FormData` directly — [`../concurrent/actions.md`](../concurrent/actions.md)), which is why uncontrolled stopped being the second-class citizen: it's the substrate the new mutation story is built on.

**And one field has no choice:** `<input type="file">` is always uncontrolled — browsers forbid programmatically setting a file value for security, so there is nothing for React to enforce. Read `e.currentTarget.files` or `fd.getAll('attachments')`.

## How it works under the hood

### The controlled loop, traced

Type `a` into a controlled input whose state is `''`:

1. The browser puts `a` in the DOM field and fires the native `input` event — **the DOM updates first**; React doesn't intercept typing.
2. Delegation routes it to your `onChange` ([`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md)); `e.target.value` is `'a'`.
3. `setEmail('a')` schedules a render ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)).
4. The render returns `value="a"`; at commit, React compares the prop to the live DOM value — equal, nothing to do.

Now break step 3 — omit `onChange`, or set state to something else — and the loop's teeth show: the render says `value=""`, the DOM says `a`, and **React writes the DOM back to `""`**. That's what "controlled" mechanically means: React continuously *asserts* the value after every input event. The frozen-input bug, the letter-that-vanishes bug, and "my transform works but typing feels wrong" are all this assertion, observed from different angles.

The caret is the assertion's collateral. When the committed value is exactly what the user just produced, browsers keep the cursor where it was. When it *isn't* — you upper-cased it, stripped a character, or the state update arrived asynchronously (after an `await`, from a debounced source) — the browser treats it as a programmatic write and throws the caret to the end. This is why naive `onChange={(e) => set(e.target.value.toUpperCase())}` feels broken when editing the middle of a word, and why input masking is either a library problem (they do caret math) or a format-on-blur problem — not a transform-on-change problem.

### What makes a field "controlled" is the prop's presence

React decides per render: `value` (or `checked`) present and not `null`/`undefined` → controlled; absent or `undefined` → uncontrolled. Which makes this innocent edit-form line a mode *switch*:

```tsx
<input value={user?.name} … />   // user still loading → undefined → UNCONTROLLED
```

First render: `undefined`, uncontrolled, user can type. Data arrives: `value` defined, React flips it to controlled, overwrites whatever was typed, and logs the famous *"A component is changing an uncontrolled input to be controlled."* The warning is not pedantry — keystrokes were destroyed. Fix at the boundary: `value={user?.name ?? ''}`, or don't render the form until data exists, or (often best) seed an uncontrolled `defaultValue` and remount per subject with `key` — the walkthrough's step 4.

### The React-specific element normalizations

React reshaped a few form elements so *value* is always expressed the same way, wherever HTML put it:

- **`<textarea value={…}>`** — not children. HTML puts the text between the tags; React moves it to the prop so it behaves like every input.
- **`<select value={…}>`** — selection is declared on the `select`, never as `selected` on an `<option>`. Multi-selects take `value={string[]}`.
- **Checkboxes and radios are `checked`/`defaultChecked`**, and their `value` prop is something else entirely: the string submitted in `FormData` when checked. Wiring a checkbox's state to `value` is a category error the console won't always catch.
- **Radio groups** are one logical field expressed as N inputs sharing a `name`; controlled means each option computes `checked={state === option}`.

## Basic usage

Both models, side by side, doing the same job:

```tsx
// Controlled — the value participates in render
function PromoField() {
  const [code, setCode] = useState('');
  const valid = /^[A-Z]{4}-\d{4}$/.test(code);

  return (
    <label>
      Promo code
      <input
        value={code}
        onChange={(e) => setCode(e.target.value)}
        aria-invalid={code.length > 0 && !valid ? true : undefined}
      />
      {code.length > 0 && !valid && <span role="alert">Format: ABCD-1234</span>}
    </label>
  );
}
```

```tsx
// Uncontrolled — the value exists only in the DOM until submit
function NotesForm({ onSubmit }: { onSubmit: (notes: string) => void }) {
  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const fd = new FormData(e.currentTarget);
        onSubmit((fd.get('notes') as string).trim());
        e.currentTarget.reset();          // resetting uncontrolled fields is the DOM's job
      }}
    >
      <textarea name="notes" defaultValue="" rows={4} />
      <button>Save note</button>
    </form>
  );
}
```

Read the difference in what each *can* do: the promo field renders judgments about the in-progress value — it must be controlled. The notes form doesn't care until submit — controlling it would buy re-renders and nothing else. And note the design-system dividend: a well-built wrapper like the `TextField` from [`../foundations/components-and-props.md`](../foundations/components-and-props.md) supports **both** for free, because it spreads native props through — callers pass `value`+`onChange` or `name`+`defaultValue`, and the component doesn't care.

## Walkthrough — an event-registration form, migrated field by field

The scenario: registration for a meetup — name, email, ticket type, dietary checkboxes, notes. Version 1 ships fast and uncontrolled. Then product does what product does: three per-keystroke requirements arrive. The walkthrough is the migration — because that's the job, and because it makes the decision framework concrete.

### Step 1 — uncontrolled first: the whole form is `name` attributes

```tsx
// registration.ts
export interface Registration {
  name: string;
  email: string;
  ticket: 'standard' | 'workshop';
  diets: string[];
  notes: string;
}
```

```tsx
// RegistrationForm.tsx — v1
export function RegistrationForm({ onSubmit }: { onSubmit: (r: Registration) => void }) {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const fd = new FormData(e.currentTarget);
    onSubmit({
      name: (fd.get('name') as string).trim(),
      email: (fd.get('email') as string).trim(),
      ticket: fd.get('ticket') as Registration['ticket'],
      diets: fd.getAll('diets') as string[],       // getAll — the multi-value read
      notes: (fd.get('notes') as string).trim(),
    });
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>Name <input name="name" required /></label>
      <label>Email <input name="email" type="email" required /></label>

      <label>
        Ticket
        <select name="ticket" defaultValue="standard">
          <option value="standard">Standard</option>
          <option value="workshop">Workshop day</option>
        </select>
      </label>

      <fieldset>
        <legend>Dietary needs</legend>
        {(['vegetarian', 'vegan', 'gluten-free'] as const).map((d) => (
          <label key={d}>
            <input type="checkbox" name="diets" value={d} /> {d}
          </label>
        ))}
      </fieldset>

      <label>Notes <textarea name="notes" rows={3} /></label>
      <button>Register</button>
    </form>
  );
}
```

Zero `useState`. Typing costs no renders; autofill and `required`/`type="email"` native validation work untouched; the checkbox group is three inputs sharing a `name`, read in one `getAll`. This is a complete, correct, accessible form in the platform's own idiom — and the baseline to migrate *from*, deliberately.

### Step 2 — submit-time validation: errors are the only state

Native `required` isn't enough — you need cross-field rules and your own messages. Validation results are the first thing that genuinely belongs in React state, because they're *rendered*:

```tsx
type FieldErrors = Partial<Record<keyof Registration, string>>;

function validate(r: Registration): FieldErrors {
  const errors: FieldErrors = {};
  if (r.name.length < 2) errors.name = 'Tell us your name';
  if (!/^\S+@\S+\.\S+$/.test(r.email)) errors.email = 'That email looks off';
  if (r.ticket === 'workshop' && r.diets.length === 0)
    errors.diets = 'Workshop day includes lunch — pick at least one option (or "none")';
  return errors;
}
```

```tsx
const [errors, setErrors] = useState<FieldErrors>({});

function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const registration = parse(new FormData(e.currentTarget));   // step 1's extraction, extracted
  const nextErrors = validate(registration);
  setErrors(nextErrors);
  if (Object.keys(nextErrors).length > 0) return;
  onSubmit(registration);
  e.currentTarget.reset();
}
```

Each field renders its error conditionally, wired with `aria-describedby`/`aria-invalid` exactly as [`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md) did. Notice the architecture: **values live in the DOM, judgments live in React** — a clean split that carries surprisingly far. (This `parse`/`validate` pair is also hand-rolled Zod; the honest scaling note comes in the patterns section.)

### Step 3 — the per-keystroke requirements arrive: migrate exactly those fields

Product asks for: a live character counter on notes (200 max), promo-code format feedback as you type, and a workshop-only "laptop rental?" follow-up that appears when the ticket changes. Each one needs a value *before submit* — so each becomes controlled. **Only** those:

```tsx
const [notes, setNotes] = useState('');
const [promo, setPromo] = useState('');
const [ticket, setTicket] = useState<Registration['ticket']>('standard');

const promoValid = promo === '' || /^[A-Z]{4}-\d{4}$/.test(promo);
const notesLeft = 200 - notes.length;
```

```tsx
<label>
  Ticket
  <select name="ticket" value={ticket} onChange={(e) => setTicket(e.target.value as Registration['ticket'])}>
    <option value="standard">Standard</option>
    <option value="workshop">Workshop day</option>
  </select>
</label>
{ticket === 'workshop' && (               // conditional field — driven by controlled state
  <label><input type="checkbox" name="laptop" /> Rent a laptop (+$10)</label>
)}

<label>
  Promo code
  <input name="promo" value={promo} onChange={(e) => setPromo(e.target.value)}
         aria-invalid={!promoValid ? true : undefined} />
</label>
{!promoValid && <span role="alert">Format: ABCD-1234</span>}

<label>
  Notes ({notesLeft} left)
  <textarea name="notes" rows={3} maxLength={200}
            value={notes} onChange={(e) => setNotes(e.target.value)} />
</label>
```

Name and email stay uncontrolled — nothing reads them mid-flight. This is the destination pattern: **a hybrid form, controlled islands in an uncontrolled sea**, each field justifying its own model. Note the controlled fields *kept their `name` attributes* — controlled inputs still participate in `FormData`, so the submit path didn't change at all. And the laptop checkbox exposes the follow-on decision you now own: unmounting it wipes its DOM value ([`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md)'s unmount column) — here that's *correct* (no ticket, no rental), but decide it on purpose.

### Step 4 — edit mode: prefill without the mode-switch bug

Registration editing arrives: the form opens with an existing `Registration`. The naive controlled prefill is the mode-switch bug from under-the-hood (`value={draft?.name}` while loading). For the uncontrolled majority of this form, the idiomatic prefill is **`defaultValue` + identity**:

```tsx
<RegistrationForm key={draft.id} initial={draft} onSubmit={save} />
// inside: <input name="name" defaultValue={initial.name} />
//         <select name="ticket" defaultValue={initial.ticket}> …
```

`defaultValue` seeds once per mount; the `key` remount ([`../state/state-and-usestate.md`](../state/state-and-usestate.md)) guarantees "once per *subject*" — switch drafts and every field, controlled and uncontrolled alike, resets cleanly with zero synchronization code. The controlled islands seed their state from `initial` in their `useState(initial.notes)` initializers — same `key`, same guarantee, and the `initial*` naming contract says exactly what it does.

## Real-world patterns

### The decision, as a table

| The field needs… | Model | Why |
| --- | --- | --- |
| Nothing until submit | Uncontrolled | Platform does the work; zero renders while typing |
| Live validation / format feedback | Controlled | The judgment is a render-time projection of the value |
| To drive other UI (dependent fields, char counts, filtering, enable-Submit) | Controlled | Other components render from it — it's state by definition |
| Transformation while typing (masking, casing) | Controlled — via a library, or rethink as format-on-blur | Caret math is real; naive transforms jump the cursor |
| File selection | Uncontrolled | The platform forbids the alternative |
| Prefill for editing | Either — `defaultValue` + `key` (uncontrolled) or seeded state + `key` (controlled) | Identity, not effects, handles subject changes |

The meta-rule: **controlled is per-field and earned, never per-form and assumed.** A form where every field is controlled "for consistency" is paying rent on twelve apartments to live in two.

### Validation timing is three tiers, not one

Change, blur, submit — each has a job: `onChange` for cheap format *hints* on fields you already control; `onBlur` for field-level commitment ("email looks invalid" belongs after they leave the field, not on keystroke two — the touched pattern below); submit for cross-field truth and the gate. The classic irritation — errors screaming before the user finishes typing — is tier confusion:

```tsx
const [touched, setTouched] = useState<ReadonlySet<string>>(new Set());
const touch = (f: string) => setTouched((prev) => new Set(prev).add(f));

<input name="email" onBlur={() => touch('email')} … />
{touched.has('email') && errors.email && <span role="alert">{errors.email}</span>}
```

Derive the error always; *show* it gated on touched (and always after a submit attempt). Judgments during render, exposure on commitment — both halves of earlier articles, pointed at forms.

### Typed extraction: the `parse` boundary

`FormData` is stringly-typed by nature; contain the casts in one function that returns a discriminated result instead of scattering `as string` through handlers:

```tsx
type ParseResult =
  | { ok: true; value: Registration }
  | { ok: false; errors: FieldErrors };
```

One boundary where DOM-strings become domain-types, unit-testable without rendering anything. And the honest scaling note: this pair — extraction schema + validation with per-field errors — is precisely what Zod + React Hook Form industrialize. The threshold in this project: past roughly five fields, cross-field rules, or arrays-of-fields, stop hand-rolling and go to [`forms-at-scale.md`](forms-at-scale.md). Below it, the platform + a `parse` function is *less* code and fewer dependencies, and now you know exactly what the library is doing for you.

### Number inputs: string state, parse at the edges

A controlled `type="number"` holding `useState<number>` can't represent what users actually type: empty field, a lone minus, `1e` mid-scientific-notation. The DOM value is a string with numeric *aspirations*; model it that way:

```tsx
const [qty, setQty] = useState('1');                      // string state
<input type="number" inputMode="numeric" value={qty}
       onChange={(e) => setQty(e.target.value)} />
const parsedQty = qty === '' ? null : Number(qty);        // parse where you consume
```

`e.currentTarget.valueAsNumber` is the typed read when you want the DOM's own parse (`NaN` for empty) — either way, the number is derived, the string is the state, and "field is empty" stops being unrepresentable.

### Resetting: three tools, matched to ownership

`form.reset()` resets what the DOM owns — uncontrolled fields snap back to their `defaultValue`s, controlled fields **ignore it** (React reasserts state at next render). Setting state resets what React owns. `key` resets *everything*, including nested component state, by remounting. Hybrid forms therefore reset with either "state-reset + `form.reset()`" or — usually cleaner — one `key` bump. Mixed-ownership forms that reset with only one tool exhibit the half-reset bug: some fields clear, the controlled islands don't.

## Per-element quick reference

| Element | Controlled props | Uncontrolled props | React-specific notes |
| --- | --- | --- | --- |
| `input` (text-ish) | `value` + `onChange` | `defaultValue` | `value={undefined}` means *uncontrolled* — the mode-switch trap |
| `input type="checkbox"` | `checked` + `onChange` | `defaultChecked` | `value` is the **submitted string**, not the state |
| `input type="radio"` | `checked={state === opt}` per option | `defaultChecked` | One field, N inputs, one shared `name` |
| `select` | `value` on the `select` | `defaultValue` | Never `selected` on `<option>`; `multiple` takes `string[]` |
| `textarea` | `value` | `defaultValue` | Value is a prop, never children (React normalization) |
| `input type="number"` | `value` (**string** state) | `defaultValue` | `valueAsNumber` for typed reads; empty string is a real state |
| `input type="file"` | — (impossible) | always | Read `e.currentTarget.files` / `fd.getAll(name)` |
| `FormData` | — | `new FormData(e.currentTarget)` | `.get(name)`, `.getAll(name)` for groups/multi; entries keyed by `name` attrs |

## Common mistakes

**`value` without `onChange`.** React warns, the input freezes — the enforcement loop with nobody feeding it. If read-only display is genuinely the intent, say so: `readOnly` (still focusable/submittable) or `disabled` (neither).

**The `undefined` mode switch.** `value={data?.field}` across a loading boundary flips uncontrolled→controlled and destroys keystrokes. `?? ''`, gate the render, or go `defaultValue` + `key`.

**`defaultValue` and `value` together.** React uses `value`, warns, and the `defaultValue` is dead code confusing the next reader. One master per field.

**Checkbox state on `value`.** `value` is what `FormData` submits when checked; the boolean lives on `checked`. The bug reads as "my checkbox ignores state."

**`selected` on `<option>`.** React's select model puts selection on the `select`; the option-level attribute draws a warning and does nothing.

**Half resets.** `form.reset()` on a hybrid form clears the uncontrolled fields only; setting state clears the controlled ones only. Reset by ownership, or remount with `key`.

**`useState<number>` behind a number input.** Empty-field and mid-edit states become unrepresentable; `Number('')` is `0` and `parseInt('')` is `NaN`, both wrong. String state, parse at the consumer.

**Transform-on-change.** Uppercasing/stripping in `onChange` fights the caret — edits mid-word jump to the end. Format on blur, or use a masking library that does caret arithmetic.

**Missing `name`s / missing `getAll`.** Uncontrolled fields without `name` simply don't exist to `FormData`; checkbox groups read with `.get` return only the first hit. The `name` attribute *is* the uncontrolled schema — treat it with schema-level care.

**Controlling everything reflexively.** The inverse cargo cult: twelve controlled fields, zero mid-flight readers, a render per keystroke times twelve, and a `useState` block longer than the form. Every `value=` should be able to answer "who reads this before submit?"

## How this evolved

| Era | Change | What it means now |
| --- | --- | --- |
| Class era | Controlled-everything orthodoxy; uncontrolled meant refs + `this` ceremony | The "always controlled" folklore dates from when the docs led with it |
| React 16.8 (2019) | Hooks make per-field controlled state cheap to write | Cheap to write ≠ free to run; the reflex outlived its context |
| React 17 (2020) | `onChange` unified on input semantics | Per-keystroke is the contract controlled inputs are built on ([`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md)) |
| React 19 (2024) | `<form action={fn}>` receives `FormData`; forms auto-reset after successful actions; `useActionState`/`useFormStatus` | Uncontrolled + `FormData` becomes the *substrate of the mutation story* — the platform-first form is now the forward-compatible one ([`../concurrent/actions.md`](../concurrent/actions.md)) |
| Compiler era (2025+) | Controlled re-renders get cheaper automatically | Choose models on data-flow grounds, not performance folklore — the question is still "who reads it before submit?" |

```tsx
// legacy: class-era controlled field — modernized in the upgrade pass
handleChange(e) { this.setState({ email: e.target.value }); }
<input value={this.state.email} onChange={this.handleChange.bind(this)} />
```

## Exercises

### 1. Diagnose the edit form

This ships and QA reports two bugs: typing before the profile loads gets erased, and a console warning about controlled inputs. Explain both from the enforcement mechanics, then fix it two structurally different ways.

```tsx
function ProfileName({ profile }: { profile: Profile | undefined }) {
  const [name, setName] = useState(profile?.name);
  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}
```

*Hint: `useState(profile?.name)` seeds `undefined` (uncontrolled) and never re-seeds ([`../state/state-and-usestate.md`](../state/state-and-usestate.md) — initializers run once); when you type, state becomes defined → mode flips. Fix A: don't render until `profile` exists, seed `useState(profile.name)`, and remount per subject with `key={profile.id}`. Fix B: go uncontrolled — `defaultValue={profile.name}` + `key`. Both fixes are the same insight: subject identity, not synchronization.*

### 2. The interests fieldset

Add an uncontrolled "interests" checkbox group (five options, shared `name="interests"`) to the registration form, extract it with `getAll`, and enforce "pick 1–3" at submit with a `fieldset`-level error message properly associated via `aria-describedby` on the `fieldset`. No new `useState` beyond `errors`.

*Hint: `fd.getAll('interests') as string[]`, validate `length >= 1 && length <= 3`, render the error inside the `<fieldset>` after the `<legend>`. The exercise's point is negative space: notice how much a multi-value field costs in the uncontrolled model (nothing) versus the controlled version you didn't have to write (a `Set` in state, a toggle handler, N `checked` computations).*

### 3. Convert the minimum

Requirements: "show 'passwords don't match' live, once both fields have been visited." Given an uncontrolled signup form, convert the *minimum* surface to controlled, add the touched gating, and write one sentence justifying every field you did **not** convert.

*Hint: two fields become controlled (both passwords — the judgment reads both), `touched` from the patterns section gates exposure, everything else keeps its `name` and its silence. If your answer converted email "while you were in there," reread the meta-rule.*

## Summary

You learned both masters and their mechanics. Controlled means React asserts the value after every input event — the loop that powers live validation and dependent UI, and the same loop behind frozen inputs, vanishing keystrokes, caret jumps, and the `undefined` mode-switch. Uncontrolled means the DOM owns the value, `name` attributes are the schema, and `FormData` (`get`/`getAll`) is the read path — cheaper, platform-native, and since React 19's form actions, the forward-facing substrate rather than the legacy one. The element normalizations (`textarea value`, `select value`, `checked` vs `value`, file's forced freedom) keep both models uniform. The working method is the walkthrough's: start uncontrolled, keep judgments (errors, touched) as the only React state, migrate individual fields to controlled exactly when something must read them before submit, prefill and reset through identity (`defaultValue`/seeded state + `key`), and contain the stringly-typed boundary in one `parse`. Past five-ish fields with cross-field rules, that hand-rolled boundary is your cue for `forms-at-scale` — and now you'll know precisely what the library is replacing.

## See also

- [`../foundations/conditional-rendering-and-events.md`](../foundations/conditional-rendering-and-events.md) — `onChange`/`onBlur` semantics, submit + `preventDefault`, the blur-ordering trap that haunts save-on-blur forms
- [`../state/state-and-usestate.md`](../state/state-and-usestate.md) — initializers run once, `key` resets, updater discipline behind form state
- [`../foundations/components-and-props.md`](../foundations/components-and-props.md) — the `TextField` that serves both models by spreading native props
- [`../concurrent/actions.md`](../concurrent/actions.md) — `<form action>`, `useActionState`, auto-reset: the 19 mutation story this article feeds
- [`forms-at-scale.md`](forms-at-scale.md) — RHF + Zod: where the hand-rolled `parse`/`validate`/`touched` trio graduates
- [`../effects/useref-and-the-dom.md`](../effects/useref-and-the-dom.md) — the ref-based uncontrolled reads this article deliberately didn't need

## References

- [`<input>` (react.dev)](https://react.dev/reference/react-dom/components/input)
- [`<select>` (react.dev)](https://react.dev/reference/react-dom/components/select)
- [`<textarea>` (react.dev)](https://react.dev/reference/react-dom/components/textarea)
- [`<form>` — including actions (react.dev)](https://react.dev/reference/react-dom/components/form)
- [Reacting to Input with State (react.dev)](https://react.dev/learn/reacting-to-input-with-state)
- [Sharing State Between Components (react.dev)](https://react.dev/learn/sharing-state-between-components)
- [FormData (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/FormData)

## Demo source

- Pending — demo hosting decision tracked in [`progress.md`](../progress.md) TODOs.