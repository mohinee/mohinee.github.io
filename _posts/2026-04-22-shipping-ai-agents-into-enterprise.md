---
title: "What shipping AI agents into a large enterprise platform taught me"
date: 2026-04-22
reading_time: 8
description: "Most writing about AI agents comes from people building standalone ones. Putting an agent in front of millions of users inside a product they already depend on teaches different lessons."
tags: [ai, agents, product, enterprise]
---

For the last three years I've been designing AI agents that ship inside
Microsoft Teams. This post is about the product side of that work: where an
agent should live, how users come to trust it, and how to design its tools. (I
wrote about the backend side [separately]({% post_url 2026-05-25-backend-for-ai %}).)
Most of what I read about agents assumes a different situation.

That writing usually describes a standalone agent: the user opens a new app,
has a conversation, gets a result. Building an agent into a product people
already use every day is a different problem. Nobody is asking whether it's
impressive. They're asking whether it makes their workday better without
surprising them.

These are my notes from that work.

## Users won't forgive much

People who open a standalone AI app expect an AI app. They put up with delays,
wrong answers, and odd behavior because they came to try the new thing.

Put an agent inside a tool people use for their jobs and that patience
disappears. Their expectations come from the *host product*. If the host
responds to a click in 100ms, an agent that takes 800ms to show its first
token feels broken. If the host is polished, a box of streaming text looks
like a prototype someone forgot to finish.

So the more established the host, the more the agent has to look and behave
like part of it. Use the host's design system, with no exceptions. Match its
latency, even if that means cutting features. Use its error states rather than
inventing your own. Users shouldn't be able to tell where the host ends and the
agent begins.

## The hard question is where the agent lives

Getting an agent to *do* things is no longer the hard part. The models are
good, tool calling is well supported, and retrieval patterns are well
documented. A competent team can have a working agent in a week.

What's hard is deciding where in the user's existing workflow the agent shows
up. It took me a long time to start asking this early enough:

- Is it a button people press when they want help?
- Does it sit in the background and offer suggestions?
- Is it a separate place they navigate to?
- Does it interrupt, or wait to be asked?
- How does it relate to what the user was already doing on that screen?

Each answer is a different product, needs different engineering work, and
fails in different ways. I've seen strong teams build technically excellent
agents that nobody used, because the honest answer to "where does this fit in
my day?" was "nowhere obvious." The model wasn't the problem. Where it was
placed was.

## Trust comes in two stages

The first stage is getting someone to try it once. That depends on
discoverability, onboarding, and how good the first interaction feels. It gets
most of the attention.

The second stage is getting them to come back next week and next month. That's
where most of the product work is, and it depends mostly on predictability.

People can work with an agent that's right 70% of the time *if they can tell
which 70%*. They won't keep using one that's right 90% of the time when the
failures look random. The first is a tool with known limits. The second is a
slot machine.

That leads to an uncomfortable design choice: it's often better to narrow what
the agent attempts and be reliable within that scope than to push capability
and hope users learn where it fails. An agent that says "I can do A, B, and C,
and I'll tell you when something is outside that" beats one that tries
everything and is sometimes wrong for no visible reason.

## Design tools for the model, not for yourself

At first I designed tools the way I'd design an API for another developer:
clear semantics, complete parameters, sensible defaults. The model handled
them badly. I had designed them for me.

What worked better, after a lot of iteration:

- **More tools, each narrower.** One `search` tool with twelve parameters did
  worse than three tools with two parameters each, even though the three
  covered less ground.
- **Names written for the model.** `find_documents_by_keyword` beats `search`.
  The longer name looks redundant to a person but helps the model pick the
  right tool.
- **Examples in the descriptions.** Adding concrete examples of valid arguments
  to tool descriptions cut bad calls noticeably. The model reads descriptions
  as documentation, so it pays to write them like good documentation.
- **Errors written for the model, not the user.** When the model called a tool
  wrong, the user shouldn't see an error at all. The useful question was what
  to send back so the model could correct itself on the next step: which
  argument was invalid, what the valid values were, what to try instead.

That last point changed how I thought about tools. A tool isn't called once
and finished. It's part of a loop in which the model keeps revising its plan.
Returning tool errors as structured feedback into that loop, instead of
treating them as exceptions to handle, made the agents noticeably more capable.

## The costly failures are the silent ones

In a chat product, a bad answer is visible. The user clicks thumbs-down or
retries, and you get a clear signal.

In an agent built into an enterprise product, the failures that matter most
are often silent. The agent appears to succeed. It gives a confident answer,
the user accepts it, and the mistake surfaces a week later when someone who
relied on the output finds that something doesn't add up.

So evaluation has to look at the outputs themselves, not only user feedback.
Thumbs ratings are skewed toward obvious failures. The confidently wrong
answers do the most damage to trust, and you have to go looking for them. We
put far more into offline evaluation, replaying golden sets, and LLM-as-judge
pipelines than I would have guessed at the start, and it paid off every time.

## Perceived latency only gets you so far

The usual advice is that perceived latency matters more than actual latency:
stream the response, show progress, give the user something to read, and a
4-second response can feel like 1.

That's right, and it's the first thing to try. But it has a limit. If the agent
runs three tool calls before producing any visible output, streaming can't
cover the gap. The user just watches a spinner.

The fix is in the design, not the streaming:

- Make the model's first output text, before any tool calls, even if it only
  restates the request and the plan.
- Run independent tool calls in parallel rather than one after another.
- Narrow the agent's scope so it doesn't need a multi-step plan in the first place.

I usually choose the third. A narrowly scoped agent is almost always faster
than a broad one with clever orchestration.

## Onboarding engineers costs more than you expect

I didn't expect getting engineers productive on AI features to take longer
than getting them productive on an ordinary codebase. But the debugging loop is
different. Working out why a model did something is a different skill from
working out why a function returned a value.

The teams that picked up AI work fastest weren't the ones with the most ML
background. They were the ones where someone had written down the patterns,
the common mistakes, the evaluation workflow, and the prompt-versioning
conventions: everything people would otherwise learn by trial and error.

In this area, a lot of mentoring is writing down knowledge that doesn't exist
in any tutorial yet. It takes real time, and in my experience it paid back in
team speed within a quarter.

## Agent work is mostly product work

What still surprises me is how much of this is product work rather than ML
work. The hard questions are about scope, trust, placement, latency, error
handling, and what the user sees when things go wrong or right. Every other
product feature has to answer the same questions. Agents don't get to skip
them because there's a model involved.

The model's flexibility makes them harder. With a deterministic feature, you
know what it does and design the UI to match. With an agent, the same UI might
be doing very different things depending on the input, and the experience has
to hold up across all of them.

The good news is that the habits of a good product engineer carry over:
caring how the user experiences the product, getting the details right, and
taking failure modes seriously. You need a new mental model on top, but you
don't have to throw out the old one.

---

*If you've shipped agents somewhere different and learned different lessons,
I'd like to compare notes.*
