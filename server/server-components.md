---
article_id: server-components
concept_folder: server
wave: 3
related:
  - server/ssr-and-hydration
  - concurrent/use-and-promises
  - concurrent/actions
  - foundations/component-composition
  - server/nextjs-and-rsc-in-practice
react_baseline: "19.2"
status:
  drafted: true
  reviewed: false
---

> **Lead with this.** You added a 300 KB syntax-highlighting library to render code blocks in a blog post. It works — and it's now in every user's bundle, parsed and evaluated on their phone, even though nothing about highlighting is interactive. Or: you're fetching data in a parent, threading it through three layers of props and an effect, when the component that actually needs it could just read the database. React Server Components answer both. An RSC runs **only on the server** — its code never ships to the browser, it can `await` your database directly, and it hands the client nothing but the finished markup and the small islands that actually need to be interactive. The mental shift is the whole thing: **not every component needs to run in the browser.** This article is the model — where components run, what crosses the boundary between server and client, and what dies at that boundary. Running it in production is a framework's job ([`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md)); this is the model that framework implements.

## What Server Components are

In an RSC-capable environment there are two kinds of components, distinguished by *where they run*:

- **Server Components** (the default) render **only on the server** — at build or request time. Their code never ships to the client. They can be `async`, `await` data, and touch server-only resources (database, filesystem, secrets). They **cannot** use state, effects, event handlers, or browser APIs, because there is no client instance of them — they never hydrate.
- **Client Components** (marked with the `"use client"` directive) are ordinary React components. Their code ships to the browser, they hydrate, and they own all interactivity: state, effects, handlers, browser APIs.

### RSC is not SSR — hold this distinction

The single most important thing to get straight, because the words sound alike and the mechanisms compose: **Server Components are not server-side rendering.**

[`ssr-and-hydration`](./ssr-and-hydration.md) is about *your components producing HTML on the server and then hydrating on the client* — the component code runs in both places and ships to the browser. RSC is about *which components run in the browser at all*. A Server Component runs **once**, on the server, produces a serialized description of its output, and its code **never reaches the client and never hydrates**. There's no second render, because there's no client instance.

And they compose rather than compete: in a production RSC app, the Client Components in your tree *still* get SSR'd to HTML for fast first paint and *still* hydrate — RSC just means the Server Components around them contributed zero client JavaScript. So a single request runs three things: the RSC render (Server Components → a serialized tree), SSR (Client Components → HTML), and client hydration (Client Components become interactive). RSC is the "what runs where" layer; SSR is the "make HTML" layer; they stack.

This project's baseline is a Vite SPA (roadmap §2), where RSC isn't available — it needs a framework/bundler that implements the server/client split. So this article teaches the *model*; [`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md) is where it becomes a running app.

## How it works under the hood

### The three layers of one request

An RSC request produces output in stages:

1. **RSC render.** Server Components run on the server. Their output isn't HTML — it's a serialized description of the tree (the **RSC payload**, historically "Flight"): a stream where already-rendered server output is baked in, and each Client Component appears as a **reference to a client module plus its serialized props**.
2. **SSR.** The Client Components referenced in that payload are server-rendered to HTML so the browser has something to paint immediately (the SSR layer from [`ssr-and-hydration`](./ssr-and-hydration.md)).
3. **Client.** The browser loads the referenced client-module chunks and **hydrates** only those Client Components. Server Component output arrives as finished markup with no accompanying JS.

### The RSC payload and serialization

The payload is a serialized React tree, and serialization is where the boundary's rules come from. Values flowing from a Server Component into a Client Component's props must **cross the network**, so they must be serializable.

Illustratively — not the literal wire format, which is a framework-internal streaming protocol — the payload for the `Post` example below looks conceptually like "rendered server output, with a hole where each client component goes":

```
// Conceptual sketch of what the server sends — server output is baked in;
// LikeButton is a *reference to a client module* plus its serialized props.
["$", "article", null, {
  children: [
    ["$", "h1", null, { children: "Hello RSC" }],          // server-rendered, inline
    ["$", "div", null, { dangerouslySetInnerHTML: {/*…*/} }], // markdown output, inline
    ["$", "@client/LikeButton.js#LikeButton", null, {         // ← a reference, not code
      postId: "42", initialLikes: 7                            // ← serialized props (data only)
    }]
  ]
}]
```

The server component output is already rendered and inlined; the client component is a *pointer to a chunk to download* carrying only its data props. That shape is the whole boundary in miniature. What crosses:

- Primitives, plain objects and arrays, and many built-ins (Date, Map, Set, typed arrays).
- **JSX / React elements** — including the *rendered output of other Server Components*. This is what lets a Server Component be handed to a Client Component as a child.
- **Promises** — serialized and streamed, which is what lets a Client Component read a server-started promise with `use()` ([`use-and-promises`](../concurrent/use-and-promises.md)).
- **Server Action references** — a `"use server"` function crosses as an *ID*, not its body (below).

What **dies at the boundary**:

- **Arbitrary functions** — a click handler, a callback, a render prop. They aren't data; they can't be serialized and shipped. The one exception is a Server Action, which crosses as a reference, not code.
- **Class instances, closures, symbols** — anything whose behavior or identity can't be reduced to plain data.

The rule that makes this memorable: **props crossing server→client must be data, not behavior.** Behavior lives inside Client Components (defined there, `"use client"`) or comes back to the server as a Server Action. If you find yourself wanting to pass a function down across the boundary, that's the signal to either move the boundary or make it a Server Action.

### The module graph split — where "zero bundle" comes from

The `"use client"` directive isn't just a runtime flag; it's a **bundler instruction**. It marks a boundary in the module graph: that module and everything it imports (that isn't itself a server module) is client code and enters the client bundle. Everything *above* the boundary — pure Server Components and their imports — is server-only and **never enters the client bundle at all**.

That's the mechanism behind the lead example. The 300 KB highlighter, imported only by a Server Component, sits above every `"use client"` boundary, so it's compiled into the server render and contributes **zero bytes** to the client. The client downloads the highlighted HTML, not the highlighter. Client boundaries also become automatic code-split points — each is a chunk loaded on demand.

### Why Server Components can't be interactive

It falls straight out of "runs once on the server, never hydrates." State needs a persistent instance across renders; there's no client instance to hold it. Effects run after commit in the browser; there's no browser commit. Event handlers need the client event system attached during hydration; Server Components don't hydrate. So the constraint isn't an arbitrary rule — a Server Component has no client-side life in which any of those could happen. Interactivity is definitionally a Client Component's job.

### Server Actions, mechanically

A function marked `"use server"` is a **server function callable from the client**. When one is passed to a Client Component or a form, React serializes a *reference* — an ID the runtime knows how to invoke — not the function's code. Calling it from the client performs an RPC back to the server, where the actual function runs with server access. [`actions`](../concurrent/actions.md) owns the client-facing API (`useActionState`, form `action`, `useOptimistic`); what RSC adds is that the action can *be* a server function, so the mutation runs on the server with direct backend access and only a reference ever crosses to the client.

Two consequences worth internalizing: `"use server"` marks **actions**, not "this is a Server Component" (Server Components are the unmarked default — the directives are opposites, mistake #9); and a Server Action is a **public endpoint** — anyone can invoke the reference, so authorize and validate *inside* it (mistake #7).

## Basic usage

A Server Component fetches and renders; a Client Component adds interactivity; the boundary is the directive.

```tsx
// Post.tsx — a Server Component (no directive). Runs on the server only.
import { renderMarkdown } from "heavy-markdown-lib"; // stays server-side: 0 client bytes
import { db } from "./db";

export async function Post({ id }: { id: string }) {
  const post = await db.posts.find(id); // direct DB access, awaited in render
  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: renderMarkdown(post.body) }} />
      <LikeButton postId={post.id} initialLikes={post.likes} />
    </article>
  );
}
```

```tsx
// LikeButton.tsx
"use client"; // this module and its imports become client code

import { useOptimistic } from "react";

export function LikeButton({ postId, initialLikes }: { postId: string; initialLikes: number }) {
  const [likes, addLike] = useOptimistic(initialLikes, (n) => n + 1);
  // interactive: state, handlers, optimistic UI — all client-side
  return <button onClick={() => addLike(1)}>♥ {likes}</button>;
}
```

`Post` awaits the database and uses a heavy library, none of which ships. `LikeButton` is a small interactive island that does. The props crossing the boundary (`postId`, `initialLikes`) are plain data — they serialize.

## Walkthrough: a blog post page, end to end

We'll assemble a page that shows every boundary rule at once: a server data fetch, a heavy server-only dependency, a small interactive island, a Server Component rendered *inside* a Client Component, and a mutation via a Server Action.

### Stage 1 — the server shell fetches and renders

`Post` (above) is the root of this page: async, awaits data, renders the heavy markdown on the server, and drops in the `LikeButton` island. Everything except `LikeButton` ships as finished markup.

### Stage 2 — a Server Component inside a Client Component (the slot)

We want the comments (server-fetched, server-rendered) to live inside a *collapsible* panel, and collapsing is interactive — so the panel is a Client Component. But a Client Component **cannot import a Server Component** (it would pull server code, and its DB access, into the client bundle). The escape is composition: pass the Server Component as **children**, so it renders on the server and its *output* is handed to the client panel as a serialized child.

```tsx
// Collapsible.tsx
"use client";
import { useState } from "react";

export function Collapsible({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return (
    <section>
      <button onClick={() => setOpen((o) => !o)}>{open ? "Hide" : "Show"} comments</button>
      {open && children} {/* children is a server-rendered tree, passed as serialized JSX */}
    </section>
  );
}
```

```tsx
// Post.tsx (continued) — server composes client-wrapping-server via the children slot
<Collapsible>
  <Comments postId={post.id} /> {/* Server Component: fetches + renders on the server */}
</Collapsible>
```

`Collapsible` never imports `Comments`; it receives its rendered output. This is the owner-vs-parent / children-as-slot pattern from [`component-composition`](../foundations/component-composition.md), and it's the standard way to nest server content inside client interactivity. The client toggles visibility of markup it was handed; it never runs the comment-fetching code.

### Stage 3 — mutating with a Server Action

Adding a comment is a mutation, so it's a Server Action — the function runs on the server with DB access; the client only holds a reference.

```tsx
// actions.ts
"use server";
import { db } from "./db";

export async function addComment(postId: string, formData: FormData) {
  // PUBLIC ENDPOINT — authorize and validate here.
  const body = String(formData.get("body")).trim();
  if (!body) return { error: "Empty comment" };
  await db.comments.create({ postId, body });
  return { error: null };
}
```

```tsx
// CommentForm.tsx
"use client";
import { useActionState } from "react";
import { addComment } from "./actions";

export function CommentForm({ postId }: { postId: string }) {
  const [state, formAction] = useActionState(addComment.bind(null, postId), { error: null });
  return (
    <form action={formAction}>
      <textarea name="body" />
      {state.error && <p role="alert">{state.error}</p>}
      <button>Post</button>
    </form>
  );
}
```

The client API here is exactly [`actions`](../concurrent/actions.md)'s `useActionState` + `.bind` for the extra `postId` arg — and it pairs with the [`double-submit-and-optimistic-like`](../recipes/forms-and-ux/double-submit-and-optimistic-like.md) recipe. What RSC contributes is that `addComment` is a *server* function: the `"use server"` reference crosses to the client, the call RPCs back, and the mutation runs with direct backend access.

### What actually shipped

Tally the client bundle for this page: the markdown library — **0 bytes** (server-only). `Comments`' fetching code — **0 bytes** (server-only; only its rendered output crossed). `LikeButton`, `Collapsible`, `CommentForm` — **shipped and hydrated**, because they're interactive islands. `addComment` — **0 bytes of logic**, just a reference. The interactive surface is a few small components; everything else is finished markup. That inversion — most of the tree server-only, interactivity pushed to the leaves — is the RSC design goal.

## Real-world patterns

**Push client boundaries to the leaves.** The default is a Server Component; add `"use client"` as low in the tree as possible, wrapping only what's genuinely interactive. A high boundary (mistake #4) drags everything below it into the client bundle. Small islands in a server sea.

**Server-inside-client via the children slot.** The Stage 2 pattern is the answer whenever interactive chrome must wrap server content — tabs with server-rendered panels, a client modal showing a server-rendered detail, a collapsible around a server list. Client can't *import* server, but it can *receive* server output as props/children.

**Async components for data, no effects.** Fetch in the component body with `await` — no `useEffect`, no loading-flag state, no client-server waterfall for initial data. Co-locate the fetch with the component that needs it, on the server. Use `cache()` to dedupe identical fetches within a request ([`use-and-promises`](../concurrent/use-and-promises.md) noted the server side of this).

**Stream a promise to the client for `use()`.** A Server Component can start a fetch and pass the *unawaited* promise to a Client Component, which reads it with `use()` inside a Suspense boundary ([`suspense`](../concurrent/suspense.md), [`use-and-promises`](../concurrent/use-and-promises.md)). The server streams the shell immediately and the data as it resolves — the client gets interactivity without blocking on the data.

**Server Actions for mutations.** Form-shaped mutations become Server Actions with direct backend access and no hand-written API route ([`actions`](../concurrent/actions.md)). Non-form server data reads/caching on the *client* side of an app are still TanStack Query's job ([`data-fetching-tanstack-query`](../ecosystem/data-fetching-tanstack-query.md)) — RSC covers the initial server render and mutations, not client-side cache management.

## API and type reference

| Marker / API | Meaning |
| --- | --- |
| *(no directive)* | Server Component — server-only, may be `async`, no client runtime. The default. |
| `"use client"` | Marks a module (and its import subtree) as client code: ships, hydrates, interactive. |
| `"use server"` | Marks **Server Actions** — server functions callable from the client via a serialized reference. Not "this is a Server Component." |
| `async function Component()` | Server Components may be async and `await` in the body. |
| `cache(fn)` | Request-scoped memoization of a server fetch/computation, to dedupe within one request. |

Type note: props crossing server→client must be serializable — TypeScript won't stop you from typing a function prop on a Client Component, but passing one from a Server Component fails at runtime. Treat "is this prop data or behavior?" as a design check the type system doesn't enforce.

## Common mistakes

**1. Confusing RSC with SSR.** `"use client"` does **not** mean "don't render on the server" — Client Components still SSR to HTML for first paint and then hydrate. `"use client"` means "this ships and becomes interactive." RSC is about what runs in the browser at all; SSR is about producing HTML. They compose.

**2. Passing a function across the boundary.** A callback, handler, or render prop passed from a Server Component to a Client Component fails to serialize. Define handlers inside the Client Component, or make it a Server Action if it must run server-side.

**3. Hooks/state/effects/handlers in a Server Component.** No client instance, no hydration, so none of these exist there. Move the interactive part into a `"use client"` component.

**4. `"use client"` too high.** Put it at the root and the entire app becomes client code — you've kept the RSC ceremony and lost all its benefit. Push boundaries down to the smallest interactive units.

**5. Importing a Server Component into a Client Component.** The client can't run server code (and importing it would pull server-only deps into the bundle). Pass the Server Component as `children`/props instead (the Stage 2 slot).

**6. Reaching for request/server data in a Client Component.** Secrets, the database, the filesystem, request headers — none exist client-side. Read them in a Server Component and pass down *serializable* results.

**7. Treating Server Actions as private.** A `"use server"` function is a public endpoint reachable by anyone with the reference. Authorize and validate inside it; "it's defined on the server" is not access control.

**8. Expecting Server Components to re-render on interaction.** They ran once and produced markup; they don't re-render in response to client state. Interactivity lives in Client Components; refreshing server data is a re-request/revalidation (a framework concern), not a Server Component re-render.

**9. Swapping `"use server"` and `"use client"`.** They're opposites. `"use client"` marks a client boundary; `"use server"` marks server *actions*. Neither declares "this is a Server Component" — that's just the unmarked default.

**10. Expecting RSC in a plain SPA.** RSC needs a framework/bundler that implements the server/client split and the payload protocol. It doesn't run in a vanilla Vite SPA (this project's baseline); it's the fenced-off model, realized through a framework ([`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md)).

## How this evolved

- **Client-only React.** Everything ran in the browser; every dependency shipped; data came from client fetches after mount (the waterfall the [`search-race-condition`](../recipes/data-fetching/search-race-condition.md) recipe dissects).
- **SSR ([`ssr-and-hydration`](./ssr-and-hydration.md)).** Moved *rendering* to the server for fast first paint — but the component code still shipped and hydrated. SSR reduced time-to-content, not bundle size.
- **RSC, experimental → real.** Demoed in 2020, matured through the React 18 era inside frameworks, and became a stable model in the React 19 line as the Next.js App Router shipped it in production. RSC moved *component execution itself* to the server — the next step after moving rendering there.
- **19 / 19.2 (this baseline).** Server Components, Server Actions, and the `use()`/promise-streaming bridge are the stable model; 19.2's `prerender` family ([`ssr-and-hydration`](./ssr-and-hydration.md)) extends it toward static-plus-dynamic output. The through-line across the whole arc: push work off the client — first rendering, then the components' code, then the mutations — leaving the browser only the interactivity it truly needs.

## Exercises

**1. Draw the boundaries.** Given a small page (header, article body, a like button, a comment form, a related-posts list), label each component Server or Client and place the minimal `"use client"` directives. Justify each client boundary by the interactivity it owns.
*Hint:* Default to Server; add `"use client"` only where there's state, an effect, a handler, or a browser API.

**2. Fix a serialization error.** A Server Component passes an `onSave` callback to a Client Component and it throws at the boundary. Fix it two ways: define the handler inside the Client Component, and (separately) make the save a Server Action.
*Hint:* Behavior can't cross as a plain function; it lives in the client, or returns as a `"use server"` reference.

**3. (Stretch) Server inside client.** Wrap a server-rendered `<Comments>` in a client `<Collapsible>` using the children slot. Confirm `Collapsible` never imports `Comments`, and that toggling shows/hides server-rendered markup without re-fetching.
*Hint:* The Server Component is passed as `children`; the client toggles visibility of output it was handed.

## Summary

- **Server Components run only on the server**, ship zero client JS, can be `async` and `await` server resources, and **never hydrate** — so they can't use state, effects, handlers, or browser APIs. **Client Components** (`"use client"`) ship, hydrate, and own all interactivity.
- **RSC ≠ SSR.** SSR makes HTML from components that also ship and hydrate; RSC decides which components run in the browser at all. They compose — Client Components in an RSC app still SSR and hydrate.
- Props crossing server→client must be **data, not behavior**: primitives, plain objects, JSX (including server output), promises, and Server Action references cross; functions, class instances, and closures die at the boundary.
- `"use client"` is a **module-graph boundary** — everything above it stays server-only (the zero-bundle mechanism) and each boundary is a code-split point. Push boundaries to the leaves.
- **Server Actions** (`"use server"`) are server functions the client invokes by reference (an RPC) — public endpoints, so authorize inside. `"use server"` marks actions, not Server Components (the unmarked default).
- RSC needs a framework/bundler that implements it; this project's Vite SPA teaches the model, and [`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md) runs it.

## See also

- [`ssr-and-hydration`](./ssr-and-hydration.md) — the HTML/hydration layer RSC composes with; keep the two distinct.
- [`use-and-promises`](../concurrent/use-and-promises.md) — `await` on the server, pass a promise to the client for `use()`; `cache()`.
- [`actions`](../concurrent/actions.md) — the client API for Server Actions (`useActionState`, form actions, `.bind`).
- [`component-composition`](../foundations/component-composition.md) — the children-as-slot pattern that lets server content nest inside client interactivity.
- [`suspense`](../concurrent/suspense.md) — streaming boundaries for server-started data.
- [`nextjs-and-rsc-in-practice`](./nextjs-and-rsc-in-practice.md) *(planned)* — RSC as a running production app: routing, caching, revalidation.

## References

- React docs — [Server Components](https://react.dev/reference/rsc/server-components)
- React docs — [Server Functions](https://react.dev/reference/rsc/server-functions)
- React docs — [`"use client"`](https://react.dev/reference/rsc/use-client)
- React docs — [`"use server"`](https://react.dev/reference/rsc/use-server)

## Demo source

_Demo source pending — StackBlitz link to be added in the demo pass._