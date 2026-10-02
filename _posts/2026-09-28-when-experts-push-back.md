---
layout: post
title: "When the Experts Push Back: Turning Objections Into Requirements"
date: 2026-09-28
tags: [ai, adoption, stakeholders, code-review, engineering]
description: "When experienced people resist an AI tool, the objection is usually a requirement they haven't written down. A case study from introducing an LLM reviewer into CI, and how a design change turned resistance into adoption."
---

The hardest audience for an AI tool isn't the people who don't understand it.
It's experienced people who understand their own work very well and can see
exactly what the tool would get wrong.

I learned this introducing an LLM-based code reviewer into a team's CI
pipeline. The goal was modest: catch untested edge cases before merge and take
some load off a review queue where pull requests waited days for attention. The
pushback came from the people whose time it was meant to save.

## The objection

The senior engineers didn't argue that the reviewer would be inaccurate. Their
concern was sharper than that. Paraphrased, it went like this:

> Experienced reviewers look at code differently. One of us might focus on
> concurrency, another on API compatibility, another on how a change behaves
> during an incident. A single agent will push the whole codebase toward one
> default opinion, and we'll lose the differences in perspective that catch
> real bugs.

It would have been easy to treat that as resistance to change and push through
it with metrics. That would have been a mistake, because they were right. A
reviewer configured only with general guidelines *does* produce one generic
opinion. The objection described a real failure mode of the design.

## Reading an objection as a requirement

When experts push back, I've found it useful to translate the objection into
the requirement it implies. This one translated cleanly:

- **Objection:** "It will flatten our different perspectives."
- **Requirement:** "The reviewer must be able to represent a specific
  reviewer's standards, not only a team-wide average."

Once it was written as a requirement, it stopped being an argument and became a
design problem, and a better design than the one I'd started with.

## The change

I added a way to configure the agent with an individual reviewer's perspective
as its source of truth: the things that person consistently checks, the
patterns they reject, and the reasoning they usually give. Instead of one
generic reviewer, the team could run reviews in the voice of the specific
expertise a change needed.

If you build something similar, a few design choices make the difference
between a tool experts tolerate and one they adopt:

- **Let experts own their own perspective.** Provide the structure, but let the
  content come from the people whose judgment it represents, in their words.
  That keeps them in control of what the agent claims to know.
- **Make each comment traceable.** When a review comment points back to the
  rule or standard it came from, it is obvious whether a bad comment came from
  the configuration or the model, and the right person can fix it.
- **Keep the gate narrow.** Decide explicitly which few things the agent may
  block on, and make everything else advisory. The approval decision stays with
  people.

## What happened next

The objection stopped being a reason to block the rollout and became part of
the design. Review turnaround dropped sharply, fewer bugs escaped to
production, and the tool was later rolled out well beyond the original team.
That expansion was possible because the people with the most to lose had their
concern built into the tool rather than overruled.

## What I took from it

- **Experts usually object to a specific failure mode, not to change itself.**
  Ask what exactly would go wrong, and listen for the requirement inside the
  answer.
- **Give control to the people with the most to lose.** A tool that encodes
  someone's judgment, under their control, is an extension of their expertise.
  A tool that replaces it is a threat.
- **Make errors attributable.** When people can see why the system said
  something, they correct it. When they can't, they stop trusting it.
- **Keep the decision with humans until the data says otherwise.** Automation
  can expand later, as evidence builds up. Starting narrow is what makes that
  expansion possible.

The general lesson goes beyond code review. Whenever I'm putting an AI system
into someone else's workflow, the loudest objection is usually the most
valuable input I'll get, as long as I treat it as a spec rather than an
obstacle.

---

*Generalised from my own work. No internal tooling, names or confidential
details from any employer are described here.*
