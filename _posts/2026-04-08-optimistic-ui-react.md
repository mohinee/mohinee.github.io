---
title: "Optimistic UI in React, and the parts the tutorials skip"
date: 2026-04-08
excerpt: "Most optimistic-UI tutorials show you the happy path and stop there. The interesting questions start at the second tab open, the failed rollback, and the moment two updates race."
reading_time: 7
---

I built a small Todo app over a weekend recently — nothing novel, just a frontend
exercise wired up to JSONPlaceholder so I could practice some patterns end-to-end.
The interesting part wasn't the app itself. It was that I caught myself, halfway
through, copying patterns from optimistic-UI tutorials and realizing they all
stopped at exactly the point where the real work begins.

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

That's optimistic UI. It works. It's also where most posts end. But the moment
you put this in front of a real user — or even just yourself, two tabs open —
the questions get more interesting fast.

## The questions tutorials skip

**What does the temp ID look like?** I started with `Date.now()`. Then I clicked
"add" twice in a hundred milliseconds and got a duplicate React key. Switched to
`crypto.randomUUID()`. Fine for me, but it doesn't render the same on the server —
which matters if you ever want SSR. The eventual answer was a counter scoped to
the session: cheap, predictable, and impossible to collide.

**What if the user edits the temp todo before the server has acknowledged it?**
This was the one that bit me. I had an edit handler that did
`api.update(todo.id, ...)`. If the user typed fast enough, `todo.id` was still
the temp ID. The server returned a 404. I had to hold the edit until the create
resolved, or queue it.

That second option — a queue per pending item — is the actually-interesting
pattern. Something like:

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

This is the kind of thing that's basically invisible when it works and makes
your app feel broken when it doesn't.

**What does the rollback actually look like?** Every tutorial says "rollback on
failure," shows you `setTodos(prev => prev.filter(...))`, and moves on. Real
rollback is harder. If the user has continued working — added two more items,
toggled one — your stale closure-based filter doesn't compose with their
intervening changes. You need to either:

1. Track the rollback as a *patch* (an inverse op) rather than a snapshot, or
2. Use a reducer where every action is a structured event you can replay or undo.

I went with option 2 in this app. It made the rollback code embarrassingly small
and the logging — which matters more than I expected — basically free.

## The thing nobody warned me about

The single biggest source of bugs in optimistic UI isn't optimism. It's
**identity**. Your local representation of the thing has one ID, the server has
another, and every operation needs to know which one is current. Until I sat
down and drew the state machine for "what does an item know about itself
between client creation and server confirmation," I was just patching symptoms.

The states ended up being something like:

- `local-pending` — has temp ID, no server ID, create in flight
- `confirmed` — has server ID, no in-flight ops
- `mutating` — has server ID, update or delete in flight
- `failed` — last op rejected, awaiting retry or rollback

Once I wrote that down explicitly, half the bugs in my code became impossible
to express.

## What about the libraries?

The honest answer: TanStack Query and SWR both handle a chunk of this. Their
optimistic update APIs cover the simple cases well, and their cache invalidation
gives you sane defaults for the rollback question.

But — and this is the thing I want to be careful about — they handle the
*request* lifecycle, not the *entity* lifecycle. The temp-ID-becomes-real-ID
handoff, the queued edits, the structured rollback — those still need to live
in your code. The library makes the wire layer easier; it doesn't make the
identity problem go away.

## Accessibility, briefly

One thing I made sure to wire up properly: when an item is in the
`local-pending` state, it's still operable but visually distinct, and the
status is announced via an ARIA live region. When a rollback happens, the
focus returns to the input that originated the action. This is the kind of
thing that's trivial when you remember and miserable to retrofit.

Also: `aria-busy="true"` on the list during a rollback animation, so screen
readers don't try to keep up with a transient state. Small detail, easy to
forget.

## What I'd do differently

I'd reach for a reducer-based pattern from the start. The ad-hoc setState
sprinkled across handlers got out of control faster than I expected, and the
moment I refactored to a single reducer with named actions, the whole thing
got tractable. Optimistic UI is fundamentally about *modeling change* — and
reducers are just better at that than scattered setState calls.

I'd also be more honest, earlier, about which operations are actually worth
making optimistic. Adding a todo? Sure. Marking it complete? Definitely.
Deleting? Probably. But syncing a setting that affects the rest of the app?
That's a place where a 200ms loading state is fine, and pretending the change
took effect when it didn't will get you in trouble.

The mantra I keep coming back to: **optimism should be reserved for changes
the user expects to succeed and would barely notice if they didn't.** Anything
heavier than that needs the truth.

---

*If you've shipped optimistic UI in production, I'd love to hear what bit you.
The state machine post above is mine, and I'm sure I'm missing patterns other
people have figured out the hard way.*
