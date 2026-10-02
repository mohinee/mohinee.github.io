---
layout: post
title: "Did It Actually Work? Measuring an AI Deployment Without Fooling Yourself"
date: 2026-10-02
tags: [ai, measurement, deployment, analytics, python]
description: "A before-and-after chart is the easiest way to overstate what an AI deployment did. Baselines, control groups and defined terms, with a small simulation showing a naive estimate inflating a real effect by more than half."
---

Every AI deployment eventually gets the same question from someone with a
budget: *did it work?* The easiest answer is a before-and-after chart. Tickets
were here, we launched, tickets went down. It is also the answer most likely to
be wrong, and the wrong answer usually flatters you.

I care about this for an unglamorous reason. If you overclaim impact, the
next person to look closely at the numbers will discount everything else you
say. If you can show the method behind a number, it survives scrutiny, and so
does your credibility with the people who'll decide what you build next.

## Decide what you're measuring before launch

Most bad impact numbers are decided after the fact, by whoever picks the
chart. Before the first user sees the feature, I write down:

- **The outcome metric, in one sentence.** "Support tickets in the categories
  the agent handles," not "support load."
- **What counts as a success.** For a self-service agent, a resolved
  conversation is only a success if the person *didn't* raise a ticket on the
  same issue soon after. Define that window up front.
- **What counts as an incident.** A wrong answer, a bad answer and a production
  incident are different things. Decide which ones you're promising zero of.
- **The comparison.** What the metric would have done without the feature.
  This is the hard part, and it gets its own section.

## The counterfactual problem

A before-and-after chart quietly assumes nothing else changed. In practice,
something always does. Seasons, team changes, a product launch, a policy
change, a holiday quarter. Any of these can move the metric in the same
direction as your feature, and the chart will credit you for all of it.

The cheapest fix I know is a **control group that the feature doesn't touch**.
For a support agent that only handles some ticket categories, the categories it
*doesn't* handle make a natural control. They live through the same seasons and
the same organisational changes. If they also dropped after launch, part of
your drop isn't yours.

## A small simulation

To show how much this matters, here is a simulation where I know the true
answer. The agent genuinely removes **20%** of tickets in the categories it
covers. But the launch also happens to land at the start of a quieter quarter,
which reduces tickets in *every* category by about 15%.

```python
import random
from statistics import mean

def weekly(base, weeks, season, effect=1.0, noise=0.06):
    return [base * season[w] * effect * random.uniform(1 - noise, 1 + noise)
            for w in range(weeks)]

random.seed(11)
weeks = 12
pre_season  = [1.00] * weeks
post_season = [0.85] * weeks          # a quiet quarter hits every category
TRUE_EFFECT = 0.80                    # the agent really removes 20% of tickets

covered_pre    = weekly(400, weeks, pre_season)
covered_post   = weekly(400, weeks, post_season, effect=TRUE_EFFECT)
uncovered_pre  = weekly(250, weeks, pre_season)
uncovered_post = weekly(250, weeks, post_season)

naive = mean(covered_post) / mean(covered_pre) - 1
control_drift = mean(uncovered_post) / mean(uncovered_pre)
adjusted = (mean(covered_post) / mean(covered_pre)) / control_drift - 1
```

The output:

| Estimate | Change |
|---|---|
| Naive before/after on covered categories | −32.3% |
| Drift in uncovered categories (control) | −13.6% |
| Control-adjusted effect | −21.6% |
| True effect in the simulation | −20.0% |

The naive number overstates the real effect by more than half. Nobody lied to
produce it. It is exactly what the dashboard would have shown. The
control-adjusted estimate lands within two points of the truth. That gap is
noise, and you shrink it with more weeks of data, not with a fancier method.

The adjustment here is a simple ratio form of difference-in-differences. It
rests on one assumption you should say out loud: without the agent, covered and
uncovered categories would have moved together. If you have reason to doubt
that (say the covered categories are seasonal in a different way) look at
several pre-launch periods to check that they really do track each other.

## Other ways to fool yourself

- **Deflection that is really delay.** Someone gets an answer from the agent,
  doesn't raise a ticket today, and raises one next week. Count repeat contacts
  within your success window, or you're counting postponed tickets as solved.
- **Shifting volume to another channel.** If tickets drop but direct messages
  to the support team rise, the work moved. It didn't go away.
- **Time saved, multiplied carelessly.** "Hours saved" is usually tickets avoided
  times average handling time. Use the handling time *for the categories the
  agent covers*, which are often the quick ones. Using the overall average
  inflates the figure.
- **Survivorship in feedback.** Thumbs-up rates come from people who chose to
  rate. They are a signal, not a measurement of the whole population.

## Reporting it

When I report impact now, the number comes with its method attached: the metric
definition, the comparison used, the time windows, and the assumptions. It
reads less like marketing. It also holds up when a finance partner or an
executive asks "how did you calculate that?", and that question always comes
eventually.

That is the standard I want for anything I deploy. The work should hold up to
someone checking it, and so should the claims about it.

---

*All data in this post is simulated so the method can be shown end to end.
Nothing here reflects internal figures from any employer.*
