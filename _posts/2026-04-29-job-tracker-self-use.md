---
title: "What I learned building a job tracker (in the middle of using it)"
date: 2026-04-29
excerpt: "Building a tool while you're actively depending on it is a strange feedback loop. The features you'd add as an engineer are different from the ones you actually need as a user, and you find out which is which fast."
reading_time: 6
---

I'm in the middle of a job search right now, and after about two weeks of
managing applications across a spreadsheet, my notes app, and three browser
tabs, I did the thing engineers do: I built a tool for it instead.

It's a small React app — a job application tracker with stages, company
metadata, notes, and CSV export. Dark editorial styling, Fraunces and JetBrains
Mono, amber accents, because if I have to look at it ten times a day it might
as well not feel like a Jira board.

This is a quick set of notes on what surprised me, both about building it and
about using it.

## You build different things when you're the user

The first version of the tracker had everything you'd expect: a list of
applications, status enums, the standard CRUD. It was fine. I used it for a
day and the second version looked completely different.

The thing I hadn't anticipated: I didn't actually want a list view as the
primary interface. I wanted a *Kanban* — but only sort of, and only for the
in-flight ones. The applications I'd already heard back on (positive or
negative) didn't belong in my main view at all. They were noise.

So the actual primary view became:

- Active pipeline (applied → screened → interviewing → offer)
- Recently archived (so I can re-find a rejection if I need to)
- Everything else hidden by default

That's not a generic job tracker. That's *my* job tracker. Which is the point.
Building something for yourself means you can be ruthless about cutting things
the user (you) doesn't actually use.

## The stages I cared about weren't the stages I'd modeled

I started with a status enum like:

```
APPLIED, SCREENING, INTERVIEW, OFFER, REJECTED, WITHDRAWN
```

Within a week I'd added — and this is where the real pain points showed up:

```
ON_HOLD     // I paused this myself
GHOSTED     // they've gone silent for >2 weeks
DECLINED    // I withdrew (different from rejection!)
```

`GHOSTED` was the most useful one. The emotional content of "company X went
silent" is very different from "company X rejected me," and conflating them
made me anxious in a way that lumping them together in `REJECTED` did not
when I was less rigorous about it. Software for yourself is allowed to have
emotional UX considerations. Mine does.

I also added a `last_contact` date field — not in the original design — once
I realized the question I asked the tracker most often was "who haven't I
heard from in a while?" That's a follow-up nudge, and the data needed to
support it wasn't in v1.

## CSV export was the unsung hero

The most-used feature, by a margin I didn't expect, was CSV export. Reasons:

- I needed to share a list with someone helping me with the search
- I wanted to do ad-hoc analysis that the UI didn't support (which company
  size am I getting the most callbacks from?)
- I wanted to back up my data somewhere that wasn't a single localStorage
  blob

The lesson: an export hatch is the cheapest way to ship a tool that doesn't
have to do everything. Whatever your app can't do, the spreadsheet can. Build
the export early.

## State, persistence, and the localStorage cliff

This is a single-user app with no backend, so localStorage was the obvious
choice. It worked great until I had ~80 entries and a habit of leaving the
tab open across browser sessions.

The two things I hit:

**Schema migrations.** I changed the data model three times in the first week.
Without a migration story, I was rebooting from a clean slate every time, which
defeats the point. I ended up writing a tiny version-tagged migration runner —
read the stored version, apply each migration up to current, persist. It's
fifteen lines and saved me from a lot of manual cleanup.

**No-op writes thrashing the JSON serialize.** I had a `useEffect` that
serialized the entire applications array to localStorage on every state
update. With a Kanban that re-renders on hover, this got expensive fast. I
moved it behind a debounced write and added a "dirty" flag — only persist
when the data actually changed in a way that matters. Cheap fix, big perf
improvement.

## What I'd build if this were a real product

It's not, and I don't want it to be. But the question is useful as a framing
exercise: what *separates* a tool I built for myself from a tool other people
would use?

The answers I came up with:

- **Multi-source ingestion.** Right now I add applications by hand. A real
  product would scrape from Gmail (the auto-replies), browser extensions, or
  job board APIs.
- **Calendar integration.** Interview slots should land on my calendar, with
  prep notes attached.
- **Genuinely useful AI features.** Not "summarize my pipeline" — boring. But
  "draft a follow-up email tuned to this company's stated values, given my
  notes from the last call" — that's a thing I'd actually use, and it's a
  good fit for an LLM. The structure matters: take in messy unstructured
  input (job posts, notes, my CV), produce specific, structured output (a
  draft, a suggestion, a comparison).
- **Privacy you can verify.** The single biggest reason I built this for
  myself is that I didn't want my pipeline data on someone else's server.
  A real product version of this would have to make the privacy guarantees
  legible to non-technical users, which is its own design problem.

## The meta-lesson

The thing I keep coming back to: tools you build for yourself are useful
training data for tools you might build for others. Not because the product
is generalizable — most aren't — but because the *process* of using your own
software for two weeks teaches you things no amount of user research will.
You see exactly which features you don't open. You feel which interactions
have one beat too many. You notice what's missing because you reach for it
and it isn't there.

If you're a frontend engineer who hasn't built something for yourself
recently, I'd recommend it. Pick a thing in your life you're using a
spreadsheet for, build the tool, use it for a month. The lessons you'll get
about UX, about state, about scope, about what "done" actually looks like —
they're not the same lessons you get from work projects, where the user is
someone else and the requirements come prepackaged.

The job tracker isn't going to be a product. But the next thing I build for
someone else will be quietly better because of it.

---

*Source code lives on my GitHub if you're curious. It's small and unsurprising.
The interesting part wasn't the code, anyway — it was the two-weeks-of-using-it
part.*
