---
recipe_id: css-leaks-across-remotes
track: micro-frontends
primary_concept: ecosystem/styling-approaches
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/styling-approaches
  - architecture/micro-frontends
  - architecture/module-federation
  - recipes/micro-frontends/shared-react-duplicated
  - recipes/micro-frontends/remote-failure-blanks-shell
status:
  drafted: true
  reviewed: false
---

# One remote's CSS restyles another — depending on load order

> **What you'll build:** a federation where each remote's styles stay inside that remote — no global `button {}` from catalog restyling checkout's buttons, no reset from one remote shifting another's layout, and no "it depends which route you visited first" — by scoping class names by construction, banning global selectors in remotes, ordering the cascade with `@layer`, and reaching for Shadow DOM when isolation must be absolute.

## The scenario

The shell composes `catalog` and `checkout`. Catalog, like a lot of apps, ships some global CSS: a reset (`* { box-sizing: border-box; margin: 0 }`), an element rule (`button { background: navy; color: white }`), and a `.card { padding: 24px; box-shadow: … }`. Checkout is pixel-perfect on its own dev server.

Composed in the shell, checkout falls apart: its **Pay button turns navy** (catalog's `button {}` selector matched it), its own `.card` elements get catalog's padding (**class-name collision**), and catalog's reset shifts checkout's spacing. The checkout team is baffled — nothing in *their* code changed, and it looks fine standalone.

The nastiest part: **it's load-order-dependent.** Whether catalog's `.card` or checkout's `.card` wins comes down to which stylesheet appears later in source order — which comes down to which remote lazy-loaded first, which comes down to what route the user visited first. So the bug is intermittent and path-dependent: fine if you land on `/checkout` directly, broken if you browse catalog first.

Why it escaped QA:

- Each remote is developed and tested **standalone** — its CSS is the only CSS on the page, so nothing collides. Green.
- The collision is **composition-only** *and* **load-order-dependent** — intermittent, and it shifts with navigation order.
- The usual culprits — resets, element selectors (`button`, `h2`), and generic class names (`.card`, `.active`, `.btn`) — look completely fine in isolation and are exactly what teams that don't share a codebase reach for.

## Walkthrough

### Stage 1 — Name it: CSS has no runtime module boundary

Remotes compose into **one document**, and CSS is global by default — every remote's rules land in one namespace and fight each other by source order and specificity. An element selector, a reset, or a generic class name in one remote is a **global mutation** that reaches every other remote on the page. There is no `import`-scoped boundary for CSS at runtime the way there is for modules; if you want isolation, you build it. So treat each remote's styles as if they must coexist with unknown other CSS on the same page — because they do. ([Article 37 flagged exactly this.](../../architecture/micro-frontends.md#styling-isolation))

### Stage 2 — Reject the fragile fixes

- **"Just use BEM / unique class prefixes per remote."** Helps with class names, but it doesn't scope element selectors (`button {}`), resets (`* {}`), or third-party CSS, and it relies on naming discipline across teams that never see each other's code. One slip and it's back.
- **"`!important` everything."** A specificity arms race that makes precedence *less* predictable and the result impossible to override downstream.
- **"Coordinate load order."** Brittle — lazy-load timing and navigation order decide it, so it breaks the moment a user takes a different path.

The durable answers make collisions impossible *by construction*, and make precedence deterministic regardless of load order.

### Stage 3 — Scope by construction, and kill global selectors

Two moves together:

**Hash class names per build.** [CSS Modules](../../ecosystem/styling-approaches.md#css-modules) (or any hash-scoped approach — a zero-runtime CSS-in-JS like vanilla-extract) turns `.card` into a per-build hash, so catalog's and checkout's collide-by-name problem vanishes:

```tsx
// checkout/src/Pay.module.css → .card {} becomes .card_c3d4e in the build
import styles from "./Pay.module.css";
export const Pay = () => <div className={styles.card}>…</div>;
// catalog's .card builds to .card_a1b2f — no collision, ever
```

**Ban global and element selectors inside remotes.** Hashing scopes *class* selectors; it does nothing for `button {}`, `* {}`, or an un-namespaced `:root`. So a remote must not ship element selectors or a global reset — scope even resets under a remote-root class, and take shared resets and [design tokens from the kernel](../../architecture/micro-frontends.md#the-shared-kernel-and-the-integration-contract), not from each remote's own `:root`. One reset and one token set, owned by the shell; remotes style only their own scoped classes.

### Stage 4 — Make precedence deterministic with `@layer`, and isolate hard with Shadow DOM

Hashing kills name collisions, but two remotes can still both legitimately style *their own* elements and have specificity/order decide edge cases. **CSS Cascade Layers** make cross-remote precedence explicit and load-order-independent — the shell declares the order once, and each remote emits into its named layer:

```css
/* shell — declared once; later layers always win, regardless of load order */
@layer reset, catalog, checkout, shell-overrides;
```

Now a rule in `shell-overrides` beats one in `checkout` beats one in `catalog` *no matter which stylesheet loaded first* — the intermittent, path-dependent behavior is gone.

When isolation must be **absolute** — a genuinely untrusted remote, or a widget that must never be touched by host CSS — mount it behind a **Shadow DOM** (a web-component seam, the strongest boundary [the styling article covers](../../ecosystem/styling-approaches.md#real-world-patterns)): CSS does not cross a shadow root in either direction. The trade-off is that shared theming must be passed in deliberately — **CSS custom properties pierce shadow boundaries** (that's how you keep one design system), while everything else is sealed out. (Related: cross-remote `z-index` wars are their own version of this — prefer the browser [top layer via native `<dialog>`](../../ecosystem/accessibility-in-react.md#step-2--the-native-dialog-your-default) over a shared z-index scale.)

**Verify the loop.** Load catalog then checkout, then checkout then catalog — checkout's Pay button and cards look identical in both orders. Catalog's reset no longer shifts checkout's spacing. Change a shared design token in the kernel and both remotes update together. The path-dependence is gone because precedence is declared, not discovered.

## Variations

1. **CSS Modules / hashed scoping** — the baseline; per-remote build hashing removes class-name collisions with zero coordination.
2. **`@layer` for deterministic precedence** — the shell owns the layer order; load order stops deciding who wins.
3. **Shadow DOM for hard isolation** — the strongest boundary; remember custom properties still pierce it, so theming survives while everything else is sealed.
4. **A shared design system through the kernel** — one `Button`, one token set, one reset from the [shared kernel](../../architecture/micro-frontends.md#the-shared-kernel-and-the-integration-contract) so remotes don't each ship conflicting base styles in the first place.
5. **Tailwind across remotes** — two remotes each shipping their own Tailwind means duplicate utilities *and* two copies of Preflight (a global reset) colliding. Share one [Tailwind build](../../ecosystem/styling-approaches.md#tailwind) through the kernel, or disable Preflight in remotes and let the shell own the reset.

## Trade-offs and common pitfalls

1. **Element selectors in a remote** (`button {}`, `h2 {}`) — a global mutation hitting every remote. Scope to classes.
2. **A reset/normalize shipped per remote** — resets collide and shift each other's layout. One reset, owned by the shell.
3. **Generic unscoped class names** (`.card`, `.active`, `.btn`) — collide by name across builds. Hash them (CSS Modules).
4. **Relying on BEM/naming discipline across teams** — one convention slip in a codebase you can't see breaks another remote.
5. **`!important` arms race** — makes precedence less predictable and un-overridable downstream.
6. **Load-order "coordination" as the fix** — lazy-load timing and nav order change who wins; use `@layer`.
7. **Each remote shipping its own `:root` tokens** — clashing variable values. Share tokens through the kernel.
8. **Each remote shipping its own Tailwind Preflight** — the global reset leaks and utilities duplicate. One Tailwind build, or Preflight only in the shell.
9. **Testing only standalone** — the collision is composition-only; test composed *and* in both navigation orders.
10. **Shadow DOM without piercing tokens** — the remote loses shared theming; custom properties *do* cross, so theme through them.
11. **Assuming Shadow DOM seals everything** — inherited properties and custom properties still cross; global `@font-face` and portals need thought.
12. **Cross-remote `z-index` / stacking collisions** — a remote's modal under another's header. Prefer the top layer (`<dialog>`/popover) over a hand-coordinated z-index scale.

### When NOT to build heavy isolation

If all remotes already share **one design system and one stylesheet build through the kernel** — one Tailwind build, one token set, one component library, one reset — there is very little to leak, and Shadow-DOM-per-remote or elaborate layering is overkill; hashed class names plus no-global-selectors is enough. And **build-time-composed remotes** (bundled into one host build) don't have this problem the same way — a single build's CSS Modules already scope everything; it's *separately-built stylesheets meeting at runtime* that create the cross-build namespace clash. The test: *are these stylesheets built separately and meeting in one document?* If yes → scope + layer (+ Shadow DOM if the remote is untrusted). If it's one build / one design system → hashed classes suffice.

## See also

- [`styling-approaches`](../../ecosystem/styling-approaches.md#real-world-patterns) — CSS Modules hashing, the build-time-vs-runtime families, and Shadow DOM encapsulation.
- [`micro-frontends`](../../architecture/micro-frontends.md#styling-isolation) — styling isolation as an architectural concern and the shared-kernel design-system contract.
- [`module-federation`](../../architecture/module-federation.md#basic-usage) — how remotes mount into the host document, which is why their CSS shares a namespace.
- [`shared-react-duplicated`](./shared-react-duplicated.md) — the shared-*dependency* version of "the same thing loaded twice."
- [`remote-failure-blanks-shell`](./remote-failure-blanks-shell.md) — the isolation sibling for runtime failures rather than styles.

## References

- MDN — Cascade Layers (`@layer`) and the cascade.
- MDN — Shadow DOM and CSS scoping (what pierces a shadow boundary).
- CSS Modules — build-time class-name hashing.

## Demo source

- `demos/micro-frontends/css-leaks-across-remotes/` — catalog + checkout colliding by load order, then hashed classes + `@layer` + a Shadow-DOM remote fixing it. *(Demo host TBD — see the demo-hosting TODO in `progress.md`.)*