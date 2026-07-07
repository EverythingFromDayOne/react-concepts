---
article_id: styling-approaches
concept_folder: ecosystem
wave: 4
related:
  - server/server-components
  - ecosystem/state-management-landscape
  - rendering/react-compiler-deep-dive
  - ecosystem/accessibility-in-react
  - server/ssr-and-hydration
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this:** Styling isn't a beauty contest between libraries; it's a decision on three axes — runtime vs build-time, how styles get scoped, and how you prefer to author them. The axis that reorganized everything is the first one: the industry has moved styling work *out of runtime and into the build*, and React 19's Server Components accelerated it by making runtime CSS-in-JS genuinely awkward. For a new React app the default is now build-time — CSS Modules, Tailwind, or zero-runtime CSS-in-JS — and runtime CSS-in-JS has retreated to the cases that still need it. This is a decision table, not a tour.

## What it is

Every approach here solves the same two problems: **global CSS collides** (two `.button` rules fight), and **styles drift from the components they belong to**. They differ on *when* the work happens and *how* you write it. Four families:

- **CSS Modules** — plain CSS in `.module.css` files; the build tool scopes each class to its file. Zero runtime, zero new syntax.
- **Utility-first (Tailwind)** — compose pre-defined utility classes in markup; the build generates only the ones you used. Zero runtime.
- **Zero-runtime CSS-in-JS** (vanilla-extract, Panda, StyleX) — author styles in TypeScript; extracted to static CSS at build. CSS-in-JS ergonomics, no runtime cost.
- **Runtime CSS-in-JS** (styled-components, Emotion) — styles generated in the browser at render, via a managed `<style>` tag and context-based theming.

The defining split is **build-time vs runtime**. Everything except that last family produces static CSS at build; the last generates it as the app runs. That single distinction drives performance, bundle shape, and — decisively now — React Server Component compatibility. Choose by three questions: does this app use RSC/SSR? how much *runtime* theming do you truly need? and how does the team prefer to author — plain CSS, utilities, or typed styles?

## How it works under the hood

**Scoping, four ways.** CSS Modules hands the build tool `styles.button`, which it rewrites to a hashed, collision-proof class like `Button_button__x7f2`; the output is ordinary CSS. Tailwind's Oxide engine statically scans your source for class names and emits *only the utilities you used* into one static stylesheet — the classes are global but atomic (`.p-4` means one thing everywhere), so nothing collides. Zero-runtime CSS-in-JS runs your `.css.ts` at build, producing static CSS plus a generated class reference the component imports. All three ship a plain stylesheet; no styling code runs in the browser.

**Why runtime CSS-in-JS is now awkward — the mechanism.** styled-components generates styles *during render* by managing a `<style>` tag, and reads its theme through `useContext`. Both are runtime, lifecycle-bound behaviors — and [`server-components`](../server/server-components.md) never touch the browser or run a React lifecycle. So a Server Component can't host a styled-component: every component that renders one must become a Client Component (`"use client"`), which in a styling-heavy app is nearly all of them, collapsing the RSC benefit. It compounded: styled-components never adopted React 18's `useInsertionEffect` (the hook built precisely for CSS-in-JS), so it injects styles in a way that can thrash layout. Build-time approaches sidestep all of this because there's no styling code left to run.

**The honest correction:** this is an *RSC* problem, not an SSR problem — people conflate them. Runtime CSS-in-JS works fine under plain SSR ([`ssr-and-hydration`](../server/ssr-and-hydration.md)) and in client-only SPAs. It's specifically the Server Component model it fights.

## Minimal shapes

**CSS Modules** — plain CSS, scoped by filename:

```css
/* Button.module.css */
.button { padding: 0.5rem 1rem; border-radius: 0.5rem; }
.primary { background: var(--color-brand); color: white; }
```

```tsx
import styles from "./Button.module.css";
<button className={`${styles.button} ${styles.primary}`}>Save</button>;
```

**Tailwind v4** — utilities in markup; note the v4 setup is CSS-first (no `tailwind.config.js`):

```ts
// vite.config.ts
import tailwindcss from "@tailwindcss/vite";
export default defineConfig({ plugins: [react(), tailwindcss()] });
```

```css
/* app.css — v4: one import, config in CSS, NOT the v3 @tailwind trio */
@import "tailwindcss";
@theme {
  --color-brand: #4f46e5; /* generates the bg-brand / text-brand utilities + the CSS var */
}
```

```tsx
<button className="px-4 py-2 rounded-lg bg-brand text-white">Save</button>;
```

**Zero-runtime CSS-in-JS (vanilla-extract)** — typed styles in `.css.ts`, extracted at build:

```ts
// Button.css.ts
import { style } from "@vanilla-extract/css";
export const button = style({ padding: "0.5rem 1rem", borderRadius: "0.5rem" });
```

```tsx
import { button } from "./Button.css";
<button className={button}>Save</button>;
```

Three authoring styles — plain CSS, utilities, typed TS — all producing static CSS.

## Walkthrough — one Button, three ways

The clearest way to feel the trade-offs is to style the *same* component — a Button with `primary`/`secondary` variants — across the three build-time approaches.

### CSS Modules

```css
/* Button.module.css */
.button { padding: 0.5rem 1rem; border-radius: 0.5rem; border: none; font: inherit; }
.primary { background: var(--color-brand); color: white; }
.secondary { background: transparent; border: 1px solid var(--color-brand); color: var(--color-brand); }
```

```tsx
// Button.tsx
import styles from "./Button.module.css";

export function Button({ variant = "primary", ...props }: ButtonProps) {
  return <button className={`${styles.button} ${styles[variant]}`} {...props} />;
}
```

Familiar CSS, scoped for free, but the styles live in a separate file from the logic.

### Tailwind

```tsx
// Button.tsx — variants via a lookup; cva/clsx are the common helpers at scale
const variants = {
  primary: "bg-brand text-white",
  secondary: "border border-brand text-brand",
};

export function Button({ variant = "primary", ...props }: ButtonProps) {
  return <button className={`px-4 py-2 rounded-lg ${variants[variant]}`} {...props} />;
}
```

Everything colocated in one file, no CSS file at all — but the markup carries the styling, which gets verbose, and dynamic class strings need care (mistake 5).

### vanilla-extract

```ts
// Button.css.ts
import { style, styleVariants } from "@vanilla-extract/css";
const base = style({ padding: "0.5rem 1rem", borderRadius: "0.5rem" });
export const button = styleVariants({
  primary: [base, { background: "var(--color-brand)", color: "white" }],
  secondary: [base, { border: "1px solid var(--color-brand)", color: "var(--color-brand)" }],
});
```

```tsx
// Button.tsx
import { button } from "./Button.css";
export function Button({ variant = "primary", ...props }: ButtonProps) {
  return <button className={button[variant]} {...props} />;
}
```

Typed variants, autocomplete, zero runtime — at the cost of a `.css.ts` indirection and a build step. Same component, three ergonomics; all three ship static CSS and all three work in an RSC app.

## The CSS-in-JS retreat

The roadmap for React styling ran *toward* runtime CSS-in-JS through ~2016–2020 — styled-components and Emotion made co-located, prop-driven, themeable styles feel great — and then ran *away* from it. Two forces:

- **Runtime cost.** Generating and injecting styles as components render is main-thread work, most noticeable on mid-range mobile during hydration.
- **RSC incompatibility.** As covered above, the runtime model needs a browser lifecycle Server Components don't have.

The signal event: in **March 2025** styled-components entered maintenance mode, its maintainer stating plainly that he wouldn't recommend it for new projects. That's not "dead" — and the nuance matters for a fair account. styled-components is **not deprecated**, still works in client-only and SSR apps, and even gained RSC integration docs; millions of components run it in production. What changed is the *default*: new projects, especially RSC-heavy ones, now reach for build-time styling first, and the ecosystem grew zero-runtime CSS-in-JS (vanilla-extract, Panda CSS, Meta's StyleX) to keep the authoring ergonomics without the runtime.

**When runtime CSS-in-JS still fits:** a client-only SPA with no RSC, React Native, or an app that genuinely needs *runtime* theming (user-authored themes computed in the browser). Outside those, prefer build-time. And **don't panic-migrate** a working styled-components codebase — migrate incrementally, and only with a concrete reason.

## The decision table

Reach for the simplest row that fits; server state of styling is "static CSS," so build-time is the default.

| Approach | Model | Reach for it when | RSC | Avoid when |
| --- | --- | --- | --- | --- |
| **CSS Modules** | scoped plain CSS, build-time | you want plain CSS, zero deps (built into Vite), a small/simple app | ✅ | you want design-token enforcement or utility velocity |
| **Tailwind** | utility-first, build-time | fast iteration, a consistent design system, one styling vocabulary across a team | ✅ | the team rejects utility-in-markup, or styles are highly dynamic/computed |
| **Zero-runtime CSS-in-JS** (vanilla-extract / Panda / StyleX) | typed styles in TS, extracted at build | you want CSS-in-JS ergonomics + type-safe tokens, RSC-compatible, at scale | ✅ | you want the least tooling, or the team prefers plain CSS |
| **Runtime CSS-in-JS** (styled-components / Emotion) | styles generated in-browser | client-only/React Native, or genuine runtime theming; an existing codebase | ⚠️ needs client boundaries | starting a new RSC/Next App Router project |

The one-line version: **new app → CSS Modules for simplicity, Tailwind for velocity, vanilla-extract/Panda if you want typed CSS-in-JS; runtime CSS-in-JS only for the cases that need runtime.**

## Real-world patterns

**Design tokens are the shared substrate.** Underneath every approach are CSS custom properties. Tailwind v4's `@theme` *generates* them; vanilla-extract has `createTheme`; CSS Modules just reference `var(--color-brand)`. Define your tokens once as custom properties and every approach can consume them — which is also how you mix approaches without duplicating the palette (mistake 7).

**Combining approaches deliberately.** Tailwind for the 95% plus a CSS Module for the one gnarly component that fights utilities is a fine, common combination. A component library authored in vanilla-extract, consumed by a Tailwind app, works because both emit plain CSS and share token custom properties. What you don't want is *two* systems solving the same problem in the same layer.

**Variants don't scale by hand.** The walkthrough's `variants` lookup object is fine for one Button; at scale, variant management *is* Tailwind's maintainability question, and the standard answer is **cva** (class-variance-authority) or **tailwind-variants** — they turn a matrix of variant × size × state into a typed, declarative config instead of hand-concatenated class strings. CSS Modules and vanilla-extract have the equivalent built in (composed classes, `styleVariants`). This is a real factor in the decision: adopt Tailwind without a variant helper past a handful of components and the markup degrades into unmaintainable class-string soup — so budget for cva the moment a component has more than one or two axes.

### Styling is orthogonal to the Compiler and to a11y

The [React Compiler](../rendering/react-compiler-deep-dive.md) optimizes re-renders and has nothing to do with your styling choice. And utility classes or scoped modules don't make markup *accessible* — roles, labels, and focus order are a separate concern ([`accessibility-in-react`](./accessibility-in-react.md), *planned*). A pretty button with no accessible name is still broken.

**Responsive and dark mode, across approaches.** Responsive design is a wash on the decision — Tailwind expresses it with variant prefixes (`md:flex`), CSS Modules and vanilla-extract use plain media queries; same capability, different ergonomics, not a reason to pick one. Container queries (respond to a parent's size, not the viewport) are now in Tailwind v4 core and in plain CSS, so they're a wash too. Dark mode is where tokens pay off again: swap CSS custom properties at a single boundary (a `.dark` class or `[data-theme]` attribute) and *every* approach re-themes instantly, with no re-render and no runtime library. That's also why genuine *runtime* theming — the last real reason to keep runtime CSS-in-JS — is rarer than teams assume: most "theming" is just custom properties changing at a boundary, which every build-time approach already supports.

**The decision is often organizational, not technical.** All four families can build a good UI; at team scale the deciding factor is usually *token enforcement and consistency*, not syntax. A design team shipping a component library against a fixed token set is better served by an approach that enforces the constraint at compile time — vanilla-extract/Panda's typed tokens, or Tailwind's deliberately constrained scale — than by free-form CSS Modules where any developer can type any pixel value. Conversely, a small team that prizes plain CSS and minimal tooling is better served by Modules. Match the approach to how much the organization needs guardrails, not to which syntax is trending.

## Common mistakes

1. **Runtime CSS-in-JS for a new RSC/App Router project.** Every component that renders a styled-component becomes a Client Component, and in a styling-heavy app that's nearly all of them — you've opted most of your tree out of RSC. Choose build-time for RSC.
2. **Tailwind v3 idioms in v4.** The `@tailwind base/components/utilities` trio, a `tailwind.config.js`, and a `content: [...]` array are all v3. v4 is CSS-first: `@import "tailwindcss"`, config in `@theme`, automatic content detection, and the `@tailwindcss/vite` plugin.
3. **Panic-migrating a working styled-components app.** It's maintenance mode, not deprecation; a client-only or SSR app can keep it. Migrate incrementally, with a concrete reason (RSC adoption, a measured perf problem), not because a blog said so.
4. **Unscoped global CSS.** Hand-writing global `.button` classes and hoping they don't collide is the exact problem all four families solve. If you're not scoping, you're accruing collisions.
5. **Dynamic Tailwind class names.** `` `bg-${color}-500` `` won't generate — Tailwind's scanner is a static text scan and can't see the interpolation. Use full class names in a lookup, or `@source inline(...)` for computed ones.
6. **Fighting the tool.** A wall of `@apply` in Tailwind is CSS Modules wearing a Tailwind costume; arbitrary values (`p-[13px]`) everywhere defeat the design system. If you're doing either constantly, you may want a different approach.
7. **Duplicating tokens per approach.** Palettes copied into Tailwind's `@theme` *and* a vanilla-extract theme *and* a CSS file drift apart. One source — CSS custom properties — feeds them all.
8. **Declaring CSS-in-JS "dead."** It retreated from the *default* for RSC reasons; it still fits client-only apps, React Native, and true runtime theming. "Not the default" isn't "unusable."
9. **A styling library for a tiny app.** CSS Modules ship with Vite and need zero dependencies. A three-page app doesn't need a framework to be styled.
10. **Confusing styling with semantics.** No styling approach makes markup accessible. A `<div>` with perfect utilities is still not a button; the role and accessible name are orthogonal to how it's painted.

## How this evolved

The whole history is styling work migrating from runtime toward build-time. Global CSS gave way to **BEM** (naming discipline to fake scoping), then **CSS Modules** (real build-time scoping). Around 2016 **runtime CSS-in-JS** (styled-components, Emotion) traded a runtime cost for co-location, dynamic props, and ergonomic theming — and dominated for years. Then the reckoning: performance-consciousness plus, decisively, React Server Components made the runtime model a liability, since [`server-components`](../server/server-components.md) have no browser lifecycle to generate styles in. The response was two-pronged: **utility-first** (Tailwind, now v4 with a Rust engine and CSS-first config) and **zero-runtime CSS-in-JS** (vanilla-extract, Panda, StyleX) — both producing static CSS at build. React 19's RSC and Compiler-first direction cemented build-time as the default, and pushed the beloved runtime libraries into maintenance and the specific niches that still need runtime.

## Exercises

1. **One component, three ways.** Style a Card with a `highlighted` variant in CSS Modules, Tailwind, and vanilla-extract. Compare the authoring feel and check that each emits static CSS (no styling JS at runtime). *Hint: inspect the built output — build-time approaches leave a `.css` file, not a `<style>` injected at render.*
2. **Justify against RSC.** For a Next.js App Router app that's mostly Server Components, pick a styling approach and defend it on the RSC axis specifically. *Hint: which approaches need a `"use client"` boundary to work?*
3. **Migrate one component.** Convert a small styled-component to CSS Modules. Name what you lose (in-browser dynamic theming) and gain (RSC compatibility, zero runtime, plain CSS). *Hint: prop-driven styles become variant classes.*

## Summary

- Styling is a decision on three axes: runtime vs build-time, scoping model, authoring ergonomics. The build-time-vs-runtime axis now dominates.
- New app default is build-time: CSS Modules (simplicity, zero deps), Tailwind (velocity, design system), or zero-runtime CSS-in-JS (typed tokens, RSC-safe). Runtime CSS-in-JS only for client-only/React Native/true runtime theming.
- Runtime CSS-in-JS retreated because it needs a browser lifecycle RSC doesn't have (not an SSR problem — an RSC one). styled-components is in maintenance mode, not deprecated.
- Tailwind v4 is CSS-first: `@import "tailwindcss"`, `@theme` in CSS, `@tailwindcss/vite`, automatic content detection — no `tailwind.config.js`.
- Design tokens as CSS custom properties are the shared substrate; define once, consume from any approach, and don't duplicate.
- Styling is orthogonal to the Compiler and to accessibility — a well-styled inaccessible component is still broken.

## See also

- [`server-components`](../server/server-components.md) — the RSC model whose no-browser-lifecycle drove the CSS-in-JS retreat
- [`ssr-and-hydration`](../server/ssr-and-hydration.md) — why this is an RSC problem, not an SSR one
- [`state-management-landscape`](./state-management-landscape.md) — the sibling decision-table article; same "pick by the axis that matters" method
- [`react-compiler-deep-dive`](../rendering/react-compiler-deep-dive.md) — styling is orthogonal to the Compiler
- [`accessibility-in-react`](./accessibility-in-react.md) — semantics and focus are orthogonal to how a component is painted *(planned)*

## References

- Tailwind CSS — [v4 announcement](https://tailwindcss.com/blog/tailwindcss-v4) and [v3→v4 upgrade guide](https://tailwindcss.com/docs/upgrade-guide) (CSS-first `@theme`, `@import`, `@tailwindcss/vite`)
- Vite — [CSS Modules](https://vite.dev/guide/features.html#css-modules)
- vanilla-extract — [Styling](https://vanilla-extract.style/documentation/styling/) · Panda CSS — [overview](https://panda-css.com/) · StyleX — [docs](https://stylexjs.com/)
- styled-components — [maintenance status](https://github.com/styled-components/styled-components) and its [RSC guidance](https://styled-components.com/docs/advanced#react-server-components)

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).