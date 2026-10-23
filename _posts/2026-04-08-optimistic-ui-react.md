---
title: "Optimistic UI in React, and the parts the tutorials skip"
date: 2026-04-08
reading_time: 6
description: "Most optimistic-UI tutorials show the happy path and stop. The bugs start when you open a second tab, when a rollback fails, or when two updates race."
tags: [react, frontend, state-management, accessibility]
---

I built a small Todo app over a weekend recently. Nothing novel: a frontend
exercise wired up to JSONPlaceholder so I could practice some patterns end to
end. Halfway through, I noticed I was copying patterns from optimistic-UI
tutorials, and that every one of them stopped right before the hard part.

Tutorials show you this:

```jsx
async function addTodo(title) {
  const tempTodo = { id: tempId(), title, completed: false };
  setTodos(prev => [...prev, tempTodo]);

  try {
    const saved = await api.create({ title });
    setTodos(prev => prev.map(t => t.id === tempTodo.id ? saved : t));
  } catch (err) {
    setTodos(prev => prev.filter(t => t.id !== tempTodo.id));
    showError(err);
  }
}
```

That works, and it's where most posts end. Put it in front of a real user, or
just open it in two tabs yourself, and you run into problems quickly.

## The questions tutorials skip

**What does the temp ID look like?** I started with `Date.now()`, hit "add" twice in
quick succession, and got a duplicate React key warning: two clicks can land in
the same millisecond more often than you'd expect.
`crypto.randomUUID()` fixes the collision, but it only exists in secure
contexts (HTTPS or `localhost`), so it's `undefined` if you test from your
phone over a LAN IP. I settled on a module-level counter that produces `tmp-1`,
`tmp-2`, and so on. It's cheap, easy to read in logs, and can't collide inside
a tab, which is the only place a temp ID ever exists.

A JSONPlaceholder-specific wrinkle: it doesn't persist writes. Every `POST
/todos` returns `id: 201`, so if you swap the temp ID for the "real" one, every
new item ends up with the same key. I had to keep the temp ID as the React key
and store the server ID next to it.

**What if the user edits the temp todo before the server has acknowledged it?**
This one bit me. My edit handler called `api.update(todo.id, ...)`. If I typed
fast enough, `todo.id` was still the temp ID and the server returned a 404. The
edit had to wait until the create resolved, either by blocking it or by queueing it.

Queueing is the better option, and it's a small amount of code. One queue per
pending item:

```jsx
const pendingOps = useRef(new Map());

function enqueueEdit(tempId, payload) {
  const queue = pendingOps.current.get(tempId) ?? [];
  queue.push(payload);
  pendingOps.current.set(tempId, queue);
}

// when the create resolves and we know the real id:
async function flushPendingFor(tempId, realId) {
  const queue = pendingOps.current.get(tempId) ?? [];
  pendingOps.current.delete(tempId);
  for (const payload of queue) {
    await api.update(realId, payload);
  }
}
```

When this works, nobody notices it. When it's missing, the app looks broken.

**What does the rollback actually look like?** Every tutorial says "roll back on
failure," shows `setTodos(prev => prev.filter(...))`, and moves on. If the user
has kept working in the meantime, say by adding two more items and toggling one,
a rollback written against the state at the time of the click doesn't fit with
those later changes. You need one of two things:

1. Store the rollback as a *patch*, an inverse operation, instead of a snapshot.
2. Use a reducer where every action is a structured event you can replay or undo.

I went with the reducer. The rollback code shrank to a few lines, and I got
logging almost for free, because every state change was already a named action.

## Identity is the real problem

Most of my bugs didn't come from being optimistic. They came from **identity**.
The client knows the item by one ID, the server knows it by another, and every
operation has to know which one is current. I was patching symptoms until I
drew out what an item knows about itself between client creation and server
confirmation.

I ended up with four states:

- `local-pending`: temp ID only, create in flight
- `confirmed`: has a server ID, nothing in flight
- `mutating`: has a server ID, an update or delete in flight
- `failed`: the last operation was rejected, waiting for retry or rollback

Once those were explicit in the reducer, about half my bugs could no longer
happen. You can't send an update for an item in `local-pending`, because the
reducer won't produce one.

## What about the libraries?

TanStack Query and SWR both handle part of this. TanStack Query's `onMutate` /
`onError` / `onSettled` pattern covers the simple cases well: snapshot the
cache, apply the change, restore the snapshot on error, refetch when settled.

But they manage the *request* lifecycle, not the *entity* lifecycle. Swapping a
temp ID for a real one, queueing edits against an item that doesn't exist on the
server yet, and rolling back without wiping out later changes all still live in
your code. The library makes the network layer easier. The identity problem is
still yours.

## Accessibility

A few things I wired up from the start, because they're painful to add later:

- An item in `local-pending` is still usable but visually distinct, and its status is announced through an
  `aria-live="polite"` region.
- When a rollback happens, focus goes back to the input that started the action,
  so a keyboard user isn't left on an element that just disappeared.
- The list gets `aria-busy="true"` during the rollback animation, so screen
  readers don't announce a state that only exists for 200ms.

## What I'd do differently

I'd start with a reducer. Separate `setState` calls in each handler got out of
hand faster than I expected. Moving to a single reducer with named actions
(something like `ADD_PENDING`, `CONFIRM`, `ROLLBACK`) made the whole thing
manageable. Optimistic UI is about modeling change over time, and a reducer
does that better than state updates spread across handlers.

I'd also decide earlier which operations are worth making optimistic. Adding a
todo, yes. Marking it complete, definitely. Deleting, probably, with an undo
toast. Changing a setting that affects the rest of the app, no. A 200ms loading
state is fine there, and showing a change that didn't actually happen will
cause trouble.

My rule: **only be optimistic about changes the user expects to succeed and
would barely notice if they didn't.** For anything heavier, wait for the server.

---

*If you've shipped optimistic UI in production, I'd like to hear what went wrong
for you. I'm sure there are cases my four-state model doesn't cover.*
