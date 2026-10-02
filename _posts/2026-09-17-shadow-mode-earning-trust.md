---
layout: post
title: "Shadow Mode: Earning a Skeptical Team's Trust Before a Model Makes a Single Decision"
date: 2026-09-17
tags: [ai, deployment, evaluation, reliability, python]
description: "Nobody should be asked to trust a model on day one. Run it silently beside the process it will replace, show the diff, and let the data pick the confidence threshold. With code that picks the threshold for you."
---

The fastest way to kill an AI rollout is to ask the people who own a process to
trust a model on day one. They know their process. They know the edge cases that
never made it into a spec. And they have usually seen at least one confident,
wrong demo.

The approach I keep coming back to is boring and effective: **run the model in
shadow mode first**. It sees every real input, makes every decision it would
make in production, and none of those decisions touch anything. A human keeps
doing the work exactly as before. You log both answers and compare.

This post covers how I set that up, what the comparison data is actually for,
and how to turn it into a confidence threshold you can defend in a room full of
skeptics.

## What shadow mode buys you

A demo answers "can the model do this?" Shadow mode answers the questions the
process owner actually cares about:

- **How often does it agree with us, on our real traffic?** Not on a curated
  eval set. On last Tuesday's inputs, including the weird ones.
- **Where does it disagree, and who was right?** Some disagreements are model
  errors. Some turn out to be inconsistencies in how humans were doing the work.
  Both are valuable, and the second kind tends to change the conversation.
- **Can we tell in advance which answers to trust?** This is the one that
  decides whether you ship at all.

It also changes the social dynamics. You are no longer asking for trust. You are
handing over evidence and asking the owners to judge it. People who would have
blocked a launch become the reviewers of the diff, and reviewers have a stake in
the outcome.

## The setup

The mechanics are simple, and most of the value is in being disciplined about
them:

1. **Fork the input, not the output.** The model receives the same request the
   human does, at the same point in the workflow. Nothing it produces is written
   anywhere a user or downstream system can see.
2. **Log a comparable record.** For each case: the input (or a reference to it),
   the model's answer, the model's confidence, the human's answer, and enough
   metadata to slice by (category, source, time).
3. **Decide what "agree" means up front.** For a classification task it is exact
   match. For free text you need a rubric, a reviewer, or an LLM judge that you
   have spot-checked. Write this down before you look at results, or you will
   move the goalposts without noticing.
4. **Run it long enough to see the unusual cases.** A week of traffic usually
   misses month-end, quarter-end and the odd seasonal spike. Ask the process
   owner when their weird weeks are, and make sure the shadow run covers at
   least one.

## Confidence-threshold routing

Shadow data rarely says "the model is good enough for everything." It usually
says something more useful: the model is very reliable on some slice of inputs
and unreliable on the rest. If the model's confidence score separates those
slices, you can route by it:

- **Above the threshold:** the model's answer is used automatically.
- **Below the threshold:** the case goes to a human, exactly as it does today.

That turns a yes-or-no launch decision into a dial. You can start with a high
threshold that automates a small share of cases at very high precision, and
lower it as trust and evidence accumulate.

The trap is picking the threshold by eye from a chart. With small samples at the
high-confidence end, a slice that looks 97% accurate might be 90% accurate in
reality. So I pick it with a lower confidence bound instead of the raw
precision, and require a minimum sample size per slice.

## Picking the threshold from shadow data

Here is the small script I use. It takes `(confidence, agreed_with_human)` pairs
from the shadow log and finds the lowest threshold whose auto-accepted slice
still clears a precision target **at the lower bound of a 95% Wilson interval**.

```python
import math

def wilson_lower(successes, n, z=1.96):
    """Lower bound of the 95% Wilson interval for a proportion."""
    if n == 0:
        return 0.0
    p = successes / n
    denom = 1 + z * z / n
    centre = p + z * z / (2 * n)
    margin = z * math.sqrt(p * (1 - p) / n + z * z / (4 * n * n))
    return (centre - margin) / denom

def pick_threshold(records, target=0.95, min_n=200):
    """records: (confidence, model_agreed_with_human) pairs from shadow mode.
    Returns the lowest threshold whose auto-accepted slice clears `target`
    precision at the Wilson lower bound, plus a table of every candidate."""
    total = len(records)
    table, chosen = [], None
    for t in [x / 100 for x in range(50, 100, 5)]:
        accepted = [ok for conf, ok in records if conf >= t]
        n, hits = len(accepted), sum(accepted)
        lower = wilson_lower(hits, n)
        table.append((t, n / total, hits / n if n else 0.0, lower, n))
        if chosen is None and n >= min_n and lower >= target:
            chosen = t
    return chosen, table
```

I ran it against 5,000 simulated shadow records from a model that is usually
confident and only roughly calibrated:

| Threshold | Coverage | Precision | Lower 95% bound | Cases |
|---|---|---|---|---|
| 0.50 | 89.8% | 82.3% | 81.1% | 4,492 |
| 0.70 | 57.8% | 87.2% | 85.9% | 2,892 |
| 0.80 | 35.5% | 90.1% | 88.7% | 1,773 |
| 0.90 | 11.4% | 94.7% | 92.6% | 568 |
| 0.95 | 3.4% | 97.0% | 93.2% | 168 |

Three things in that table are worth noticing:

- **A 95% target returns no threshold at all.** The 0.95 slice looks like 97%,
  but it holds only 168 cases, and its lower bound is 93%. Reading the chart
  by eye would have shipped it. The script refuses, and that refusal is the
  right answer to bring to the process owner.
- **At a 90% target it picks 0.90**, which automates about 11% of cases. That
  sounds small. It is also a real, defensible start, and every automated case
  frees a person for the cases that need them.
- **Coverage and precision trade off smoothly.** That gives the owner a real
  choice. "How much do you want automated, and what error rate can your
  process absorb?" is a business decision, and it should be theirs, made with
  the numbers in front of them.

## What changes when you present it this way

Bring the table, not a recommendation. In my experience the conversation shifts
from "should we trust the model?" to "which row do we start on?", and that is a
conversation the people who own the process are comfortable having.

Bring the disagreements too. Pull twenty cases where the model and the human
differed and walk through them together. Some will be model errors you need to
fix. Some will be cases where two humans would also have disagreed. Those
usually lead to a clearer rule for the humans as well, which earns the model
more goodwill than any accuracy number.

## After launch

Shadow mode does not end at launch. It narrows:

- **Keep sampling the auto-accepted slice.** A small random sample still goes to
  a human reviewer so precision is measured continuously, not assumed.
- **Re-run the threshold picker on fresh data** when inputs drift, the model
  changes, or the prompt changes. A threshold is a property of a model on a
  distribution, and both of those move.
- **Treat a prompt change like a model change.** Run it in shadow against the
  current version before it takes traffic.

None of this is sophisticated. It is mostly the discipline of not asking anyone
to take the model's word for it. That discipline is what lets a model move from
an impressive demo to something a team relies on every day.

---

*The numbers above come from simulated data, generated so the script has
something realistic to chew on. The pattern itself is one I've used on
production AI work; nothing here describes any employer's internal systems.*
