---
title: "What I learned building a job tracker while using it"
date: 2026-04-29
reading_time: 5
description: "Building a tool you depend on every day creates a fast feedback loop. The features you'd add as an engineer aren't the ones you need as a user, and you find out which is which quickly."
tags: [react, frontend, side-project]
---

I'm in the middle of a job search. After about two weeks of tracking
applications across a spreadsheet, my notes app, and three browser tabs, I did
what engineers do and built a tool instead.

It's a small React app: an application tracker with stages, company details,
notes, and CSV export. I gave it a dark editorial look (Fraunces for headings,
JetBrains Mono for data, amber accents), because if I'm going to open it ten
times a day it shouldn't look like a Jira board.

Here's what surprised me, both building it and using it.

## You build different things when you're the user

The first version had what you'd expect: a list of applications, a status
field, basic CRUD. It was fine. After one day of using it, the second version
looked completely different.

I didn't want a list as the main view. I wanted something like a Kanban board,
but only for applications still in progress. The ones I'd already heard back
on, good or bad, didn't belong on the main screen at all. They were noise.

So the main view became:

- Active pipeline: applied → screened → interviewing → offer
- Recently archived, so I can find a rejection again if I need to
- Everything else hidden by default

That isn't a general-purpose job tracker. It's mine, and that's the point. When
you're the only user, you can cut anything you don't use.

## The stages I needed weren't the ones I modeled

I started with this status enum:

```
APPLIED, SCREENING, INTERVIEW, OFFER, REJECTED, WITHDRAWN
```

Within a week I'd added three more:

```
ON_HOLD     // I paused this myself
GHOSTED     // no reply for more than two weeks
DECLINED    // I turned down an offer (not the same as withdrawing early)
```

`GHOSTED` turned out to be the most useful. "Company X stopped replying" feels
very different from "Company X rejected me," and filing silence under
`REJECTED` made my pipeline look bleaker than it was. Software you build for
yourself is allowed to account for how you feel using it.

I also added a `last_contact` date, which wasn't in the original design. The
question I asked the tracker most often was "who haven't I heard from in a
while?", and v1 didn't store the data to answer it.

## CSV export got the most use

By a wide margin, the feature I used most was CSV export:

- I needed to share a list with someone helping me with the search.
- I wanted to answer questions the UI couldn't, like which company sizes were
  calling me back most often.
- I wanted a backup that wasn't a single localStorage entry.

An export button is the cheapest way to ship a tool that doesn't have to do
everything. Whatever the app can't do, a spreadsheet can. Build export early.

## localStorage worked until it didn't

It's a single-user app with no backend, so localStorage was the obvious choice.
That was fine until I had around 80 entries and a habit of leaving the tab open
for days.

Two problems came up.

**Schema changes.** I changed the data model three times in the first week.
With no migration path, each change meant starting from an empty tracker, which
defeats the purpose. I wrote a small migration runner: store a version number
with the data, and on load, apply each migration from the stored version up to
the current one, then save. It's about fifteen lines and saved me a lot of
manual cleanup.

**Writing too often.** A `useEffect` serialized the whole applications array to
localStorage on every state change, and the Kanban board re-rendered on hover,
so I was running `JSON.stringify` on 80 records far more often than needed. I
debounced the write and added a dirty flag so it only saves when the data has
actually changed. A small fix with a noticeable improvement.

## What a real product would need

I don't want this to be a product. But asking what it would take is a useful
exercise, because it shows the gap between a tool for me and a tool for other
people:

- **Automatic import.** I add applications by hand. A product would pull them
  from Gmail confirmation emails, a browser extension, or job board APIs.
- **Calendar integration.** Interviews should land on my calendar with prep
  notes attached.
- **AI features that actually help.** Not "summarize my pipeline." Something
  like "draft a follow-up email for this company, using my notes from the last
  call." LLMs are good at that shape of task: messy input (job posts, notes, a
  CV) turned into specific output (a draft, a comparison).
- **Privacy users can check.** The main reason I built my own was that I didn't
  want my job search data on someone else's server. A product would need to
  make its privacy guarantees understandable to non-technical users, which is a
  design problem of its own.

## The broader lesson

Tools you build for yourself teach you how to build tools for other people.
The product rarely generalizes, but the experience does. Two weeks of using
your own software shows you which features you never open, which interactions
take one step too many, and what's missing, because you reach for it and it
isn't there.

If you're a frontend engineer and haven't built something for yourself
recently, try it. Pick something you're tracking in a spreadsheet, build the
tool, and use it for a month. You'll learn things about UX, state, and scope
that you don't learn at work, where the user is someone else and the
requirements arrive already written.

The job tracker won't become a product. The next thing I build for someone
else will be better because of it.

---

*The code is on my GitHub if you're curious. It's small and ordinary. The
useful part was the two weeks of using it, not the code.*
