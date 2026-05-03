---
title: "What shipping AI agents into a large enterprise platform taught me"
date: 2026-04-22
excerpt: "Most writing about AI agents is from the perspective of building one. Less of it is about what happens when you put one in front of millions of users inside a platform they already depend on. The lessons are different."
reading_time: 9
---

I've spent the last couple of years architecting AI agents that ship inside a
large enterprise productivity platform. The specifics aren't the point of this
post — and aren't mine to share publicly — but the *shape* of the work is
something I think more people building agents would benefit from hearing about.

Most of the agent content I read is written from the perspective of someone
building a standalone agent. The user opens a new app, has a conversation,
gets a result. That's a real category of work, but it's the easier one. The
harder problem — and the one I keep coming back to — is integrating an agent
into a platform people already use, where the bar isn't "is this impressive"
but "does this make my workday better, every day, without surprising me."

These are notes from that work, generalized.

## The novelty budget is much smaller than you think

When you build a standalone AI app, users come to it with their expectations
calibrated to "this is an AI thing." They tolerate delays, hallucinations, and
strange behavior because they came for the magic.

When you embed an agent inside a tool people use for their actual jobs, none
of that tolerance carries over. Users come with expectations calibrated to
the *host platform*. If the host responds to clicks in 100ms, your agent's
800ms first-token latency reads as "this thing is broken." If the host has
a polished UI, your streaming-text-into-a-bubble interface looks like a
prototype someone forgot to finish.

The implication: the more established the platform you're embedding in, the
more your agent has to feel like it belongs there. Reuse the host's design
system religiously. Match its latency profile, even if that means simpler
features. Inherit its error states. Boring is a feature.

## "Where does the agent live in the workflow" is the actual hard question

Building the agent's capabilities is, surprisingly, not the hard part anymore.
The model has gotten good. Tool-calling is well-supported. RAG patterns are
documented. You can stand up a competent agent in a week.

The hard part is figuring out where in the user's existing workflow the agent
should *show up*. This is the question that took me the longest to learn to
ask early enough:

- Is the agent a button users press when they want help?
- Is it a passive presence that watches and offers suggestions?
- Is it a separate surface they navigate to?
- Does it interrupt? Does it wait to be summoned?
- What's its relationship to the things the user was already doing?

Each of these is a different product. Each is a different engineering
investment. And each fails in different ways. I've watched smart teams build
technically beautiful agents that nobody used, because the answer to "where
does this fit in my day" was "nowhere obvious." The model quality wasn't the
problem. The integration site was.

## Trust is built in two stages, and the second one is where it actually matters

The first stage is initial trust — does the user try it once. This is mostly
about discoverability, the on-ramp UX, and the demo-quality of the first
interaction. It's important and it gets a lot of attention.

The second stage is *sustained* trust — does the user reach for it again next
week, next month. This is where most of the actual product work lives, and
it's bottlenecked on something boring: predictability.

A user can tolerate an agent that's right 70% of the time *if they can predict
which 70%*. They can't tolerate an agent that's right 90% of the time but the
failures are random. The first is a tool with known limits. The second is a
slot machine.

The implication for design is uncomfortable: it's often better to make your
agent *less ambitious in scope* and more reliable within that scope, than to
push for capability and hope users learn the failure modes. The version that
says "I can do A, B, C, and I'll tell you when something's outside that" beats
the version that tries everything and is often, mysteriously, wrong.

## Tool surfaces get designed for the model, not the human

When I started, I designed tools the way I'd design APIs for a developer to
use: clear semantics, thorough parameters, sensible defaults. The model
struggled with them.

I'd designed them for *me*. The model needed something different.

What worked better, after a lot of iteration:

- **Narrower tools, more of them.** A single "search" tool with twelve
  parameters lost to three tools with two parameters each, even though the
  combined surface was strictly smaller.
- **Names written for the model.** `find_documents_by_keyword` beats `search`.
  The verbosity helps the model pick correctly. Cosmetic to a human reader,
  load-bearing for the LLM.
- **Examples in descriptions.** Tool descriptions that include concrete
  examples of valid arguments dramatically reduced bad calls. The model
  treats descriptions like documentation, and writing them like *good*
  documentation pays off.
- **Errors as feedback, not failures.** When the model called a tool wrong,
  the question wasn't "how do I show the user a graceful error" — the user
  shouldn't see the error at all. The question was "how do I return something
  to the model that helps it self-correct on the next loop iteration."

This last one was the biggest mental shift for me. Tools aren't called once
and consumed. They're part of an inner loop where the model is constantly
adjusting its plan. Treating tool errors as a structured signal back into
that loop, rather than as exceptions to handle, made my agents materially
more capable.

## The expensive bug is the one nobody notices

In a chat agent, when something goes wrong, the user sees it. They thumbs-down
it. They retry. The signal is loud.

In an embedded enterprise agent, the failures that matter most are often
*silent*. The agent looks like it succeeded. It returned a confident-sounding
answer. The user accepted it. And it was wrong in a way that won't surface for
a week, when whoever depends on the output discovers something doesn't add up.

The implication: evaluation has to look at outputs, not just at user
feedback. Thumbs ratings are a useful signal but they're heavily biased
toward the loud failures. The quiet ones — the confidently wrong ones — are
the ones that erode trust hardest, and they're the ones you have to actively
hunt for. We invested a lot more in offline evaluation, golden-set replay, and
LLM-as-judge pipelines than I'd have predicted upfront. It paid off every
time.

## Latency is mostly perceived, but not entirely

There's a popular framing — true and useful — that perceived latency matters
more than actual latency. Stream the response, show progress, give the user
something to read while the rest arrives, and a 4-second total turnaround
feels like 1.

This is right, and it's the first lever to pull. But there's a floor it can't
get you under. If your agent does a multi-step plan with three tool calls
before the first user-visible output, *no amount of streaming polish* will
cover the dead air. The user is staring at a thinking indicator with nothing
to read.

The fix isn't streaming, it's architectural. Either:

- The first model output is text, before any tool calls — even if it's just
  acknowledging the request and stating the plan
- The tool calls happen in parallel where possible, not sequentially
- The agent is smaller-scoped so the multi-step plan isn't necessary in the
  first place

I default to the third. A scope-tightened agent is almost always faster than a
scope-broad one with clever orchestration.

## Mentorship and ramp-up: the hidden cost

Something I didn't expect: bringing engineers up to speed on building AI
features takes longer than bringing them up to speed on a normal codebase. The
mental model is different. The debugging loop is different. The reasoning
about "why did the model do that" requires a different muscle than "why did
my function return that."

The teams that absorbed AI work fastest weren't the teams with the most ML
background. They were the teams where someone had taken the time to write
down the patterns, the gotchas, the eval workflow, the prompt-versioning
conventions — all the unwritten knowledge that would otherwise have to be
acquired by trial and error.

Mentorship, in this domain, is partly about codifying tacit knowledge that
isn't in any tutorial yet. The investment is high. It pays back in team
velocity within a quarter.

## The thing that kept surprising me

After all of this, the part that still catches me is how much agent work is
actually *product work*, not ML work. The hard questions are about scope,
trust, integration, latency, error handling, what the user sees when something
goes wrong, what the user sees when something goes right. These are the
questions every other product surface has to answer. Agents don't get to skip
them just because there's a model under the hood.

If anything, the model's flexibility makes them harder. With a deterministic
feature, you know what it does and you design the UX to match. With an agent,
the same UI surface might be doing very different things on different inputs.
The user experience has to hold up across all of those.

The good news: the discipline that makes a good product engineer — caring
about what the user feels, sweating the details, taking failure modes
seriously — translates directly to making good agents. The mental model
upgrade is real, but it's an upgrade, not a replacement.

---

*Caveat to head off the obvious question: I work on AI agent systems
professionally. Nothing in this post is specific to my employer's
architecture, internal tooling, or unreleased work — these are patterns I'd
recognize across enterprise agent projects in general, written at the level
where they're useful to other engineers thinking about similar problems. If
you've shipped agents in a different context and your lessons differ, I'd be
genuinely curious to compare.*
