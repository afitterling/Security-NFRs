# Web write-back to a last-write-wins sync

The web client edits and deletes records that the mobile app also syncs. To not
clobber the app's data — and to make a delete *stick* across devices — every web
mutation is a **full-list write-back** (fetch all → change one → PUT all) and a
delete is a **soft-delete tombstone**, mirroring the native client.

Reference impl: `mutateTask` and the `toggle`/`edit`/`delete` actions in
`sst/landing page/app/routes/app.tsx`; the slim shared model in
`…/app/lib/tasks.ts`.

## The fetch-all → mutate-one → PUT-all pattern

The sync API's `PUT /tasks` **replaces the whole list** (anything omitted is
deleted). So a web edit can't PUT just the changed row — it must read the
current set, transform the one target, and push everything else back untouched.

```ts
// Returns a Response to short-circuit on 401 (session expired), else null.
async function mutateTask(id: string, change: (t: Task) => Task): Promise<Response | null> {
  const res = await fetch(`${base}/tasks`, { headers });
  if (res.status === 401) return redirect("/app", { headers: clearCookie() });
  const all = (await res.json().catch(() => [])) as Task[];
  if (!Array.isArray(all)) return null;
  const next = all.map((t) => (t.id === id ? change(t) : t));      // touch only the target
  const put = await fetch(`${base}/tasks`, {
    method: "PUT",
    headers: { ...headers, "content-type": "application/json" },
    body: JSON.stringify(next),
  });
  if (put.status === 401) return redirect("/app", { headers: clearCookie() });
  return null;
}
```

Each action is `mutateTask` with a different transform, and every transform
bumps `updatedAt` so it wins last-write-wins on other devices:

```ts
// toggle done
(t) => ({ ...t, done: !t.done, updatedAt: Date.now() })

// soft-delete (NOT array removal) — see below
(t) => ({ ...t, deleted: true, updatedAt: Date.now() })

// edit: touch ONLY the web-editable fields; spread preserves everything else
(t) => ({ ...t, title, priority, recurrence, tags, updatedAt: Date.now() })
```

## Why soft-delete instead of dropping the row

If the web client deleted by *omitting* the row from the PUT, another device that
still holds the task would just push it back on its next sync and it would
**reappear**. A tombstone (`deleted: true` + fresh `updatedAt`) propagates the
removal as a normal update that wins last-write-wins, exactly like the iOS app.
See [SYNC-001 — per-record merge on sync](
../general-specs/sync/SYNC-001-conflict-free-merge.md).

## Rules

- **Preserve unknown fields verbatim.** The web model
  (`tasks.ts`) is a deliberate subset; the app pushes more (reminders, Apple
  links, `order`, `createdAt`). The spread (`...t`) carries them through
  untouched — never reconstruct a record from only the fields the web knows.
- **Always bump `updatedAt`** on any change so the mutation wins last-write-wins.
- **Delete = tombstone, never array removal**, so removals survive other
  devices' pushes.
- **Handle 401 mid-mutation** — the session can expire between the GET and the
  PUT; clear the cookie and bounce to a re-auth, don't write with a dead token.
- **Full-list PUT is read-modify-write** — there's a small lost-update window if
  two clients PUT concurrently; acceptable for a single-user task list, but note
  it. A per-record PATCH API would remove the window (and the fetch-all step).
- **Mirror the native client's semantics exactly** — same tombstone shape, same
  `updatedAt` discipline — so web and app converge instead of fighting.

## Checklist

- [ ] Web mutation reads the full list, changes only the target, PUTs all back.
- [ ] Unknown/native-only fields preserved via spread.
- [ ] `updatedAt` bumped on every change.
- [ ] Delete writes a `deleted:true` tombstone, not a row removal.
- [ ] 401 between read and write clears the session and re-auths.
- [ ] Web and native share the same record shape + LWW rules.
