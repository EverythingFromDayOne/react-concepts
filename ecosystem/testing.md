---
article_id: testing
concept_folder: ecosystem
wave: 4
related:
  - effects/custom-hooks
  - ecosystem/data-fetching-tanstack-query
  - forms/forms-at-scale
  - rendering/react-compiler-deep-dive
  - ecosystem/accessibility-in-react
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this:** Test what the user does, not how the component does it. A test that asserts on internal state or class names breaks the moment you refactor — even when nothing the user sees changed. A test that clicks a button by its accessible name and checks what appears on screen survives the refactor and catches the regression. The whole stack — Vitest to run, React Testing Library to query, user-event to interact, MSW to mock the network — exists to make behavior-testing the path of least resistance. This article is the full toolchain that [`custom-hooks`](../effects/custom-hooks.md) previewed with `renderHook` and deferred.

## What it is

Four tools, four jobs, one philosophy:

- **Vitest** — the test runner. Vite-native, so it reuses your app's exact transform/TS config (no separate Babel or ts-jest), and its API is Jest-compatible (`describe`/`it`/`expect`, plus `vi` where Jest had `jest`).
- **React Testing Library (RTL)** — renders a component into a jsdom DOM and gives you queries that find elements *the way users do*: by role, label, and text.
- **user-event** — simulates real interactions by dispatching the full sequence of events a browser fires, not a single synthetic one.
- **MSW (Mock Service Worker)** — mocks the network at the *boundary*, intercepting requests on the wire so you never patch `fetch` or a module.

The philosophy underneath all four, in RTL's words paraphrased: the more a test resembles how the software is actually used, the more confidence it gives you. That's the whole thesis — **test behavior, not implementation** — and every tool choice below serves it. Tests coupled to internals are liabilities; tests coupled to observable behavior are assets.

## How it works under the hood

### The query priority ladder

RTL's queries are ranked, and the ranking *is* the guidance — reach for the highest that fits:

1. `getByRole` (with `{ name }`) — queries the **accessibility tree**, so it tests what a screen-reader user and a sighted user both perceive. The default.
2. `getByLabelText` — form fields by their label (how users find inputs).
3. `getByPlaceholderText` / `getByText` — visible text.
4. `getByDisplayValue`, `getByAltText`, `getByTitle` — narrower cases.
5. `getByTestId` — the escape hatch, coupled to a `data-testid` you added for tests. Last resort.

Querying by role isn't just a style preference: if `getByRole('button', { name: /save/i })` can't find your button, it's usually because the button *isn't accessible* — no role, no accessible name. The test failing is the accessibility bug surfacing. `logRoles(container)` prints what's queryable when you're stuck.

**Three query axes, three jobs.** For each query type there are three variants, and mixing them up is the most common flake source (mistake 5):

| Variant | Returns | Use for |
| --- | --- | --- |
| `getBy…` | element, **throws** if missing | asserting something *is present now* |
| `queryBy…` | element or **null** | asserting something is *absent* |
| `findBy…` | **Promise**, retries until timeout | asserting something *appears* after async work |

**user-event vs fireEvent.** `fireEvent.click` dispatches one synthetic `click`. `user.click` dispatches the real sequence — pointer events, focus, mousedown/up, then click — so it catches bugs (a handler that depends on focus, a disabled button) that `fireEvent` misses. v14 requires `userEvent.setup()` first, and every interaction is `await`ed.

**MSW intercepts at the boundary.** Traditional mocking swaps the thing that makes the request (`vi.mock('axios')`, monkey-patching `fetch`), coupling the test to an implementation detail. MSW sits *below* the request client: your component makes a real `fetch('/api/users')`, the request leaves your code exactly as in production, and MSW catches it on the wire and returns a mock. The same handlers work whether you use `fetch`, axios, or TanStack Query — you're mocking the network contract, not the call site.

**The Compiler is transparent.** React Compiler transforms at build time; a Vitest run typically executes un-compiled source unless you wire the plugin into the test config. Because the Compiler is behavior-preserving ([`react-compiler-deep-dive`](../rendering/react-compiler-deep-dive.md)), your *behavior* tests pass identically either way — which is the point: if a test breaks depending on whether the Compiler ran, it was testing implementation, not behavior.

## Basic usage

Config (in `vite.config.ts`) plus a setup file, then a test:

```ts
// vite.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom", // a DOM in Node so components can render
    globals: true, // describe/it/expect without imports; RTL auto-cleanup
    setupFiles: "./src/test-setup.ts",
  },
});
```

```ts
// src/test-setup.ts
import "@testing-library/jest-dom/vitest"; // toBeInTheDocument, toBeVisible, …
```

```tsx
// Counter.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Counter } from "./Counter";

test("increments when the button is clicked", async () => {
  const user = userEvent.setup(); // v14: set up, then interact
  render(<Counter />);

  await user.click(screen.getByRole("button", { name: /increment/i }));

  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

No `data-testid`, no reaching into state — the test clicks the button a user would click and asserts the text a user would read.

## Walkthrough — testing an async component with the full stack

We'll test a `UserList` that fetches from `/api/users`: assert the loading state, the loaded list, an error path (via a runtime override), and a retry interaction — Vitest + RTL + user-event + MSW working together.

### Step 1 — MSW handlers and the server lifecycle

```ts
// src/mocks/handlers.ts
import { http, HttpResponse } from "msw"; // v2: http + HttpResponse, NOT v1's rest + res(ctx.json)

export const handlers = [
  http.get("/api/users", () =>
    HttpResponse.json([
      { id: "1", name: "Ada Lovelace" },
      { id: "2", name: "Alan Turing" },
    ]),
  ),
];
```

```ts
// src/mocks/server.ts
import { setupServer } from "msw/node"; // handlers from 'msw', server from 'msw/node'
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

```ts
// add to src/test-setup.ts — the mandatory lifecycle
import { server } from "./mocks/server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers()); // undo per-test overrides so they don't leak (mistake 7)
afterAll(() => server.close());
```

### Step 2 — The test: loading → list

```tsx
// UserList.test.tsx
import { render, screen } from "@testing-library/react";
import { UserList } from "./UserList";

test("shows a spinner, then the users", async () => {
  render(<UserList />);

  expect(screen.getByRole("status")).toBeInTheDocument(); // loading now → getBy

  // The list appears after the fetch resolves → findBy retries until it's there
  expect(await screen.findByRole("listitem", { name: /ada lovelace/i })).toBeInTheDocument();
  expect(screen.getByRole("listitem", { name: /alan turing/i })).toBeInTheDocument();
  expect(screen.queryByRole("status")).not.toBeInTheDocument(); // spinner gone → queryBy
});
```

Three variants, each doing its job: `getBy` for the spinner that's present immediately, `findBy` for the list that appears after the async fetch, `queryBy` for the spinner that must be *absent* after loading.

### Step 3 — The error path, via a runtime override

Instead of a separate mock, override the handler for this one test:

```tsx
import { http, HttpResponse } from "msw";
import { server } from "./mocks/server";
import userEvent from "@testing-library/user-event";

test("shows an error and retries", async () => {
  server.use(http.get("/api/users", () => new HttpResponse(null, { status: 500 })));
  const user = userEvent.setup();
  render(<UserList />);

  expect(await screen.findByRole("alert")).toHaveTextContent(/couldn't load/i);

  // Restore success and let the retry button re-fetch
  server.use(http.get("/api/users", () => HttpResponse.json([{ id: "1", name: "Ada Lovelace" }])));
  await user.click(screen.getByRole("button", { name: /retry/i }));

  expect(await screen.findByRole("listitem", { name: /ada lovelace/i })).toBeInTheDocument();
});
```

`server.use` layers a one-off handler; `resetHandlers()` in `afterEach` peels it back before the next test. The test drives the same path a user would hit a flaky server on — error, then retry — with zero changes to `UserList`.

## Real-world patterns

**Testing a custom hook — the full story.** [`custom-hooks`](../effects/custom-hooks.md) showed `renderHook` as a preview; here's the rest. `renderHook` mounts a hook in a throwaway component and exposes its return via `result.current`:

```tsx
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

test("increments", () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => result.current.increment()); // state updates outside an event → wrap in act
  expect(result.current.count).toBe(1);
});
```

For a hook that needs context (a store, a Query client), pass a `wrapper`. But prefer testing hooks *through a component that uses them* when you can — that tests behavior; `renderHook` is for genuinely reusable library hooks with no natural component.

**Testing components that use TanStack Query.** Wrap in a `QueryClientProvider` with a fresh client per test and **retry disabled**, and let MSW supply the network:

```tsx
function renderWithClient(ui: React.ReactElement) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}
```

`retry: false` is not optional (mistake 8): the default retries a failed query several times, so an error-path test that forgets it waits through the backoff and times out. This helper gates the whole data-fetching test recipe track — see [`data-fetching-tanstack-query`](./data-fetching-tanstack-query.md).

**Prefer `findBy` over `waitFor`.** `findBy*` is `waitFor` + a query in one, with better errors. Reach for bare `waitFor` only to assert on something that isn't a query (a mock was called, a non-DOM side effect settled) — and never put a `getBy` that throws inside a `waitFor` when a `findBy` says the same thing more clearly.

**Tests as an accessibility check.** Because `getByRole` reads the accessibility tree, a role query that can't find your element is often the component telling you it's not accessible — the test is a free a11y smoke test. `logRoles(container)` shows what's exposed. Deep a11y testing (focus traps, live regions) is [`accessibility-in-react`](./accessibility-in-react.md)'s subject *(planned)*.

**Fake timers + user-event.** If you use `vi.useFakeTimers()`, pass user-event the matching advance config (`userEvent.setup({ advanceTimers: vi.advanceTimersByTime })`) or interactions hang — a classic flaky-test cause the `testing/` recipe track covers.

**What to test — and what not to.** The highest-confidence-per-line tests are *integration-style component tests*: render a component with its real children, mock only the network with MSW, and assert behavior — the "Testing Trophy" bias toward integration over isolated units. Spend tests on what breaks in ways users notice: behavior, edge cases, error and empty states, the interactions that carry logic. Skip what earns nothing — a component that just renders a prop (types already guarantee that), library internals, and snapshot tests of large trees (they fail on every change and nobody reads the diff). And know when *not* to write a test here at all: a throwaway spike, pure static config, or a full multi-page user journey — that last belongs in an end-to-end tool (Playwright/Cypress) driving a real browser, not jsdom. RTL tests component behavior; E2E tests the assembled app. Reaching for the wrong level is how suites get slow and brittle without getting more trustworthy.

## API / type reference

| Symbol | Kind | Notes |
| --- | --- | --- |
| `render(ui)` / `screen` | RTL | render into jsdom; `screen` queries the whole document |
| `getBy` / `queryBy` / `findBy` | query axes | present-now (throws) / absent (null) / appears (async) |
| `…ByRole` / `…ByLabelText` / `…ByText` / `…ByTestId` | query types | priority order; role first, testid last |
| `userEvent.setup()` → `user.click/type/…` | interaction | v14: setup first, all interactions `await`ed |
| `renderHook(fn)` → `{ result }` | RTL | `result.current`; wrap state changes in `act` |
| `http.get/post(...)` / `HttpResponse.json(...)` | MSW v2 | handlers from `msw`; `HttpResponse.error()` for network failure |
| `setupServer(...handlers)` | MSW v2 | from `msw/node`; `listen`/`resetHandlers`/`close` lifecycle |
| `server.use(handler)` | MSW v2 | per-test override; `{ once: true }` third arg for one-shot |
| `vi.fn` / `vi.mock` / `vi.spyOn` / `vi.importActual` | Vitest | Jest's `jest.*`, renamed |

## Common mistakes

1. **Testing implementation details.** Asserting on state, class names, or a component's internal methods couples the test to *how* it works, so a behavior-preserving refactor breaks it. Assert on what the user perceives.
2. **`getByTestId` first.** A `data-testid` couples the test to markup you'll change. `getByRole` is the resilient, accessible default; drop to testid only when there's genuinely no role/label/text.
3. **`fireEvent` instead of user-event.** `fireEvent` fires one synthetic event; `user.click` fires the real browser sequence (focus, pointer, click), catching bugs `fireEvent` sails past.
4. **Not awaiting interactions or `findBy`.** Every user-event call and every `findBy` is async. A missing `await` produces `act` warnings and flakiness; enable `no-floating-promises` and TypeScript catches them.
5. **Wrong query axis.** `getBy` for absence *throws* and crashes the test; `queryBy` for presence returns null and your assertion silently never fires. Present → `getBy`, absent → `queryBy`, appears → `findBy`.
6. **Mocking `fetch`/axios/modules instead of MSW.** Patching the request client couples the test to that client and to your fetch code's shape. MSW mocks the network contract at the boundary, surviving a switch from fetch to TanStack Query.
7. **Forgetting `server.resetHandlers()`.** A `server.use` override from one test leaks into the next, causing spooky order-dependent failures. `listen` / `resetHandlers` (afterEach) / `close` is the mandatory lifecycle.
8. **Not disabling Query retry in tests.** A failing query retries with backoff by default, so an error test times out instead of failing fast. Set `retry: false` on the test `QueryClient`.
9. **MSW v1 idioms.** `rest.get()` and `res(ctx.json())` are gone in v2. Use `http.get()` and `HttpResponse.json()`; if a tutorial shows the old form, it's v1-era.
10. **Testing the library, not your code.** Don't assert that TanStack Query caches or that Zod validates — those have their own tests. Test *your* behavior: that your component shows the cached data, that your form blocks an invalid submit.

## How this evolved

The shift that defines modern React testing is from *internals* to *behavior*. Enzyme (the early standard) did shallow rendering and let you assert on component instances, state, and props — tests that mirrored the implementation and shattered on every refactor. React Testing Library inverted the premise: render to a real DOM, query by role and text, assert on output — so tests describe the contract with the user, not the code. The runner moved from Jest to **Vitest** for Vite-native projects (same transform pipeline, faster, ESM-first). Interaction moved from `fireEvent` to **user-event** for realistic event sequences. Network mocking moved from `nock`/`fetch-mock` (patch the client) to **MSW** (intercept the boundary), whose v2 rewrite aligned handlers with the standard Fetch API (`http` + `HttpResponse`). And React 19's Compiler is deliberately transparent to all of it — behavior tests don't know or care whether it ran. The common thread across fifteen years: the closer a test sits to how the software is used, the longer it stays true.

## Exercises

1. **Role-ify a testid test.** Take a test using `getByTestId` and rewrite it with `getByRole`/`getByLabelText`. If you can't, the component has an accessibility gap — fix the semantics, and the test gets simpler. *Hint: `logRoles(container)` shows what's queryable.*
2. **Async success and failure.** Test a component that fetches a list: assert loading, then the list (`findBy`), then use `server.use` to force a 500 and assert the error UI. *Hint: `getBy` for the spinner, `findBy` for the list, `queryBy` to prove the spinner left.*
3. **A custom hook.** Test `useToggle` (or any small hook) with `renderHook`, wrapping state changes in `act`. Then argue whether it'd be better tested through a component. *Hint: `result.current` holds the return; `renderHook` is for genuinely reusable hooks.*

## Summary

- Test behavior, not implementation. Tests coupled to internals break on refactors; tests coupled to what the user does and sees survive them and catch real regressions.
- The stack: Vitest (Vite-native runner), RTL (accessible queries), user-event (real interactions, `setup()` then `await`), MSW (network-boundary mocking, v2 `http`/`HttpResponse`).
- Query by role first; `getByTestId` is the last resort. A failing role query is often an accessibility bug surfacing.
- Three query axes: `getBy` (present, throws), `queryBy` (absent, null), `findBy` (appears, async). Matching them to intent kills most flakiness.
- MSW mocks the contract at the boundary, so handlers survive a change of request client; `listen`/`resetHandlers`/`close` is the mandatory lifecycle, and Query tests need `retry: false`.
- `renderHook` tests reusable hooks, but prefer testing through a component; the Compiler is transparent to behavior tests.

## See also

- [`custom-hooks`](../effects/custom-hooks.md) — the `renderHook` preview this article completes
- [`data-fetching-tanstack-query`](./data-fetching-tanstack-query.md) — testing queries: the `QueryClientProvider` + `retry: false` harness
- [`forms-at-scale`](../forms/forms-at-scale.md) — testing forms by label and role, and validation behavior
- [`accessibility-in-react`](./accessibility-in-react.md) — the a11y dividend of role queries, in depth *(planned)*
- [`react-compiler-deep-dive`](../rendering/react-compiler-deep-dive.md) — why the Compiler is transparent to behavior tests

## References

- Vitest — [Getting Started](https://vitest.dev/guide/) and [config](https://vitest.dev/config/)
- React Testing Library — [About queries](https://testing-library.com/docs/queries/about/) (the priority order) and [`renderHook`](https://testing-library.com/docs/react-testing-library/api/#renderhook)
- user-event — [Setup](https://testing-library.com/docs/user-event/setup/) (v14 `setup()` pattern)
- MSW — [Quick start](https://mswjs.io/docs/quick-start/) and [`setupServer`](https://mswjs.io/docs/api/setup-server/)

## Demo source

> _Pending_ — demo hosting choice tracked in `progress.md` (StackBlitz vs CodeSandbox vs local `demos/`).