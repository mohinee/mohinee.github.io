---
layout: post
title: "Using AI to Learn a New Codebase Fast (Without Trusting It Blindly)"
date: 2026-09-11
tags: [ai, developer-productivity, engineering, onboarding, code-review]
description: "AI coding tools can compress the first weeks in an unfamiliar repository into days, if you use them as hypothesis generators rather than oracles. A five-phase workflow."
---

Every experienced engineer knows the feeling of opening a large, unfamiliar repository. You don't know what matters, what is load-bearing, and what is legacy nobody dares touch. Reading top to bottom doesn't work, because the thing you need to understand is rarely in one place.

AI coding agents change the economics of this a lot. They can read a hundred files while you read three. But they also produce confident, fluent, plausible-sounding descriptions of code that are sometimes wrong, and in a codebase you don't know yet, you are the worst-placed person to notice.

The workflow below tries to get the speed without the risk. The recurring rule is simple: **the model proposes, the repository and its history decide.**

## Phase 1: A metadata pass

Before any code, gather the material people wrote *about* the code, and have the AI synthesize it: README and docs, CONTRIBUTING, architecture decision records, CI configuration, the build files, dependency manifests, deployment config, and issue templates.

Ask for concrete artifacts, not summaries:

- What are the deployable units, and how are they built and released?
- What are the external dependencies (databases, queues, third-party APIs)?
- What is the test strategy, and how do I run it locally?
- What conventions does this repository state explicitly?

This phase is cheap and the failure mode is mild: docs go stale, so treat the output as a claim to check. Anything it says about *how things are supposed to work* is a lead, not a fact.

## Phase 2: Trace flows, don't read modules

Reading a codebase module by module gives you a catalog of parts. What you need is how a request moves through them.

Pick two or three concrete scenarios, such as "a user signs in," "an order is placed," or "a background job retries," and have the agent trace each from entry point to persistence, naming every file it passes through.

Two practices help:

- **Use read-only exploration.** Give the tool permission to search and read, not to edit. Onboarding is not the time for surprise diffs.
- **Parallelize.** Independent questions ("trace the auth flow," "map the module boundaries," "where is configuration loaded?") can run as separate agent sessions at the same time, then you merge the answers. If your tool supports subagents for exploration, this is what they are for.

Ask for file paths and line references with every claim. A trace you can click through is checkable. A trace written as prose is not.

## Phase 3: Mine the history for the *why*

Code tells you what the system does. It rarely tells you why it is shaped that way, and "why" is what stops you from breaking something.

Version control is the best source. A few commands that reliably pay off:

```bash
# Which files change most? (a proxy for where the action is)
git log --since="12 months ago" --name-only --pretty=format: \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -20

# Who has been working here?
git shortlog -sn --no-merges --since="12 months ago" HEAD

# When did this string or symbol appear or disappear?
git log -S'someSymbol' --oneline -- path/to/file

# Why is this line the way it is?
git blame -w -C path/to/file
```

The pickaxe search (`-S`) is especially good: it shows the commits where the *number of occurrences* of a string changed, so you find the commit that introduced a constant or removed a check. Then read that commit message and the pull request behind it.

Feed the AI those results, along with postmortems, incident write-ups, and long-running issue threads, and ask it to reconstruct causal stories: "this retry logic exists because of X." Then verify each story against the actual commit or document it cites.

## Phase 4: Build a risk map and write it down

By now you have enough to identify where a change is dangerous. A useful map has four signals:

- **High fan-in.** Files or functions that many others depend on.
- **High churn.** Files that change constantly, which often means unstable design or a merge-conflict magnet.
- **Low test coverage,** particularly where fan-in is high.
- **Sparse or contradictory comments,** which often mark places where the original author was uncertain.

The overlap of high fan-in and low coverage is the part to be careful with.

Persist what you learned. Most agent tools support a project-level memory or instructions file that they load on every session, such as a `CLAUDE.md` or `AGENTS.md`. Put the verified findings in there: how to build and test, the request flow, the areas to be careful with. That turns a one-time exploration into something every later session, yours or a teammate's, starts from. Keep it to things you have confirmed.

## Phase 5: Choose a low-blast-radius first change

The fastest way to test whether you really understand a codebase is to change it in a small way and see what happens. Good first contributions have a few properties:

- Isolated, with well-defined inputs and outputs
- Covered by existing tests, so you get fast feedback
- Reviewed by someone who knows the area
- Similar enough to existing code that you can copy its idioms

Let the AI help you find the *idiomatic* way to do it here, by showing you three existing examples of the same kind of change. The goal is to look like you have always worked in this repository, which means matching local conventions instead of importing your own.

## The critique pass, and where it goes wrong

Once you have a working model, an AI can quickly generate a list of code smells, inconsistencies, and things that look wrong. That list is useful, and it is also where newcomers get into trouble.

Every item on it is a **hypothesis**. Some are real problems. Many are decisions with context you don't have yet: a workaround for a vendor bug, a constraint from another team, a deliberate trade-off that was argued over in a design review two years ago.

Before you repeat any of them in a code review, a design discussion, or a standup:

1. Check the history for the reason (Phase 3 again).
2. Ask the people who own the area.
3. Only then decide whether it is a real issue.

Nothing damages a newcomer's credibility faster than confidently calling something a mistake that turns out to be deliberate. An AI is very good at producing that kind of confident claim, and it has no idea what your team decided last spring.

## Two practical cautions

- **Check your organization's policy on AI tools first.** Pointing an external service at a private repository may be restricted, or allowed only for approved tools. This is a two-minute question that avoids a serious problem.
- **Don't let the tool's summary replace reading.** For the small number of files that are truly central, read them yourself. The summary is a map, not the territory. You still need to walk the ground where it matters.

## The workflow in one glance

| Phase | What you ask for | How you verify |
|---|---|---|
| 1. Metadata | Build, deploy, dependencies, conventions | Cross-check against CI config |
| 2. Flow tracing | End-to-end paths with file references | Click through the references |
| 3. History | Causal stories for surprising code | Read the cited commits and PRs |
| 4. Risk map | Fan-in, churn, coverage overlap | Compare with actual coverage and git data |
| 5. First change | Idiomatic examples to copy | Existing tests and reviewer feedback |

AI makes it much faster to form a mental model of a new codebase. The verification steps are what keep that model from being wrong, and making them habitual matters more than which tool you use.
