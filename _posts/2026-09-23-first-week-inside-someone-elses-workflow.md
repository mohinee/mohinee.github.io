---
layout: post
title: "The First Week Inside Someone Else's Workflow"
date: 2026-09-23
tags: [engineering, discovery, product, ai, stakeholders]
description: "The request you're handed is rarely the problem you need to solve. How I spend the first week with a team I'm building for: watching the work, mapping the waits, and finding the bottleneck before writing code."
---

Some of the most useful things I've built were not on anyone's request list.

A few years ago I noticed that a marketing team I worked alongside waited on
engineering for every personalised campaign landing page. Nobody had filed that
as a problem; it was simply how launches worked. Engineers weren't slow at
building the pages. Each campaign waited days in a queue for engineering time,
for work that was mostly configuration. The fix wasn't more engineering time.
It was a self-serve tool that took engineering out of the loop, so a campaign
could go live in about half an hour instead of days.

I think about that often. When you build for a team you don't belong to (an
internal customer, another department, an external client) the request you
receive is usually a proposed solution. Your first job is to find the problem
underneath it. This is how I spend the first week.

## Day one: separate the request from the outcome

Every request arrives as a solution: "we need a dashboard," "can you add a
field," "we want a chatbot for this." I write the request down verbatim, and
then I ask one question repeatedly until I reach something measurable:

> What would be different for you if this existed?

"We need a dashboard" becomes "we want to see which orders are stuck." That
becomes "we find out about stuck orders when the customer complains, two days
late." The real outcome is *finding out sooner*. A dashboard is one way to get
there. An alert might be a better one.

I don't argue with the proposed solution at this stage. I just make sure I
know which outcome it's supposed to deliver, so I can judge alternatives against
it later.

## Days two and three: watch the work, don't interview about it

Interviews tell you how people believe the process works. Watching tells you
how it actually works. The gap between the two is where most of the useful
discoveries are.

I ask to sit with someone while they do the real task, on real cases, and I
take notes on three things:

- **Waits.** Every point where the work stops because it is waiting for a
  person, a system, an approval or a batch job. Waits are usually far longer
  than the work itself, and they are invisible in process diagrams.
- **Workarounds.** Spreadsheets kept next to the official system, sticky notes,
  copy-paste between tabs, a message sent to "the person who knows." Each one
  marks a place where the official tooling doesn't fit the real job.
- **Judgment calls.** The moments where someone pauses and decides. These are
  the parts that are hardest to automate, and the parts people are most
  protective of. Knowing where they are early saves you from building something
  that tries to take them away.

## Day four: map the queue, not the process

By now I can draw the workflow as a sequence of steps, with the time spent
*working* and the time spent *waiting* at each one. It almost always looks the
same: a few minutes of work, separated by hours or days of waiting.

That picture usually points straight at the bottleneck, and it is rarely where
the request pointed. In the landing-page case, building pages faster would have saved
minutes. Removing the wait for an engineer saved days.

This is also where AI fits, or doesn't. Models are good at shrinking the
*work* inside a step: drafting, classifying, summarising, extracting. They
don't help with a wait caused by an approval that nobody needs or a hand-off
between teams. If the bottleneck is a wait, automating the work around it
changes very little. It's worth being honest about this before proposing an
AI feature.

## Day five: play it back, then propose

Before proposing anything, I play my understanding back to the team: here's
what I saw, here's where the time goes, here's what I think the real problem is.
Two things happen. They correct the parts I got wrong, which they always do,
and they start to trust that the eventual proposal is grounded in their actual
work rather than in a generic idea of it.

Then I propose the smallest thing that moves the outcome, and I tie it to a
number we agreed on. "Campaigns go live the same day" is a goal a team can
hold me to. "A better landing-page tool" isn't.

## Things that go wrong

- **Discovery with the wrong people.** Managers describe the process they
  designed. The people doing the work describe the process that exists. You
  need both, but the second matters more for what you build.
- **Discovery that never ends.** A week is usually enough to find the
  bottleneck. After that, a working prototype in their hands teaches you more
  than another round of interviews.
- **Falling in love with the elegant solution.** Sometimes the right fix is a
  form, a permission change or a deleted approval step. If it moves the
  outcome, ship it. Nobody downstream cares how interesting it was to build.

## Why this matters more with AI

AI makes it cheap to build something impressive, and that raises the risk of
building the wrong thing well. A slick agent that automates a step that was never the
bottleneck will demo beautifully and change nothing. A week of watching the
work is the cheapest insurance I know against that outcome.

---

*Examples are generalised from past projects and contain no confidential
details from any employer.*
