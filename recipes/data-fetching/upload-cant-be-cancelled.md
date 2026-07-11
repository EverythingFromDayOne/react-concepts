---
recipe_id: upload-cant-be-cancelled
track: data-fetching
primary_concept: ecosystem/data-fetching-tanstack-query
difficulty: intermediate
react_baseline: "19.2"
related:
  - ecosystem/data-fetching-tanstack-query
  - recipes/data-fetching/cancel-vs-invalidate-confusion
  - recipes/data-fetching/search-race-condition
  - effects/effects-and-synchronization
status:
  drafted: true
  reviewed: false
---

# The Cancel button on a large upload does nothing

> **What you'll build:** a mutation that can be aborted from the UI — because unlike `queryFn`, `mutationFn` does **not** receive Query's auto `signal` — by threading your own `AbortController` through `mutateAsync`/`mutate` variables into `fetch`/`axios`.

## The scenario

A creator uploads a 400MB video with `useMutation`. UX includes Cancel. The handler calls `queryClient.cancelQueries()` — wrong verb, and mutations aren't queries. Or they call `mutation.reset()` — clears mutation state, **HTTP keeps uploading**. Network panel still shows the request until it finishes; server may still process the file. Users mash Cancel; support gets duplicate half-uploads.

**Why it escaped QA:** small fixture files finish before Cancel is clicked; cancel "clears the progress UI" via `reset()` so it looks cancelled.

## Walkthrough

### Stage 1 — Name it: mutations don't auto-abort

[Article note](../../ecosystem/data-fetching-tanstack-query.md#mutate-vs-mutateasync): `queryFn` gets `{ signal }` and Query aborts on unmount/key change. **`mutationFn` gets variables only.** Writes are not cancelled when you navigate away — by design (avoid "pay button → navigate → silent abort"). UI cancel is **opt-in**.

### Stage 2 — Reject the wrong cancels

- **`cancelQueries`** — stops reads; upload continues ([cancel vs invalidate](./cancel-vs-invalidate-confusion.md)).
- **`mutation.reset()`** — UI state only.
- **Ignoring the body stream** — must pass `signal` into `fetch`/`xhr`/`axios`.

### Stage 3 — Own the AbortController

```tsx
import { useMutation } from "@tanstack/react-query";
import { useRef, useState } from "react";

type UploadVars = { file: File; signal: AbortSignal };

async function uploadVideo({ file, signal }: UploadVars) {
  const res = await fetch("/api/videos", {
    method: "POST",
    body: file,
    signal,
    headers: { "Content-Type": file.type },
  });
  if (!res.ok) throw new Error("upload failed");
  return res.json() as Promise<{ id: string }>;
}

export function VideoUploader() {
  const controllerRef = useRef<AbortController | null>(null);
  const [progress, setProgress] = useState(0); // wire XHR if you need real %

  const upload = useMutation({
    mutationFn: uploadVideo,
  });

  function start(file: File) {
    controllerRef.current?.abort();
    const controller = new AbortController();
    controllerRef.current = controller;
    upload.mutate(
      { file, signal: controller.signal },
      {
        onSettled: () => {
          if (controllerRef.current === controller) controllerRef.current = null;
        },
      },
    );
  }

  function cancel() {
    controllerRef.current?.abort();
  }

  return (
    <div>
      <input
        type="file"
        accept="video/*"
        onChange={(e) => {
          const file = e.target.files?.[0];
          if (file) start(file);
        }}
      />
      <button type="button" disabled={!upload.isPending} onClick={cancel}>
        Cancel
      </button>
      {upload.isPending && <p>Uploading…</p>}
      {upload.isError && upload.error.name !== "AbortError" && <p>{upload.error.message}</p>}
    </div>
  );
}
```

Treat abort as non-failure in UX (no error toast). Server should tolerate aborted uploads (temp GC).

### Stage 4 — Harden + verify the loop

- Don't abort **payments** on route change automatically — only explicit Cancel.
- Axios: `axios.post(url, data, { signal })`.
- For progress, `XMLHttpRequest` + `signal` via `AbortController` + `xhr.abort()` on abort event.

**Verify the loop.** Start a large upload, click Cancel: Network shows **(canceled)**; `isPending` false; no success toast; server receives connection close. `cancelQueries` alone: upload continues (control experiment).

## Variations

1. **`mutateAsync` + try/catch** — catch `AbortError` separately.
2. **Multi-file** — one controller per file or one shared abort-all.
3. **TanStack Router nav block** — confirm before leaving mid-upload.
4. **Resumable uploads (TUS)** — abort pauses; cancel ≠ delete server draft (product decision).
5. **Query downloads** — use `cancelQueries` + `signal` in `queryFn` instead (reads).

## Trade-offs and common pitfalls

1. **Expecting mutation auto-signal** — doesn't exist.
2. **`reset()` as cancel** — cosmetic.
3. **`cancelQueries` for uploads** — wrong API.
4. **Not forwarding `signal`** — abort is a no-op ([search-race](./search-race-condition.md) same footgun).
5. **Toasting AbortError as failure** — scares users.
6. **Aborting money transfers on blur** — data inconsistency risk.
7. **Shared controller across sequential uploads** — abort kills the wrong one; per-attempt controller.
8. **No server-side temp cleanup** — orphan blobs.
9. **Testing with tiny files** — Cancel never races completion.
10. **Service worker caching the POST** — rare; verify canceled in Network.

### When NOT to abort mutations

Idempotent "save draft" that already committed server-side — Cancel only stops **in-flight** bytes; it won't undo a finished write. For payments and irreversible deletes, prefer disable-nav + pending UI over abort-as-undo.

## See also

- [`mutate` vs `mutateAsync`](../../ecosystem/data-fetching-tanstack-query.md#mutate-vs-mutateasync)
- [Cancel vs invalidate](./cancel-vs-invalidate-confusion.md)
- [Effects AbortController pattern](../../effects/effects-and-synchronization.md) — same signal discipline for reads

## References

- MDN — [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- TanStack Query — [Mutations](https://tanstack.com/query/latest/docs/framework/react/guides/mutations)

## Demo source

- `demos/data-fetching/upload-cant-be-cancelled/` — Cancel aborts HTTP vs reset-only. *(Demo host TBD)*
