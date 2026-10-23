---
title: "Why I built a five-agent pipeline instead of one big prompt"
date: 2026-06-09
reading_time: 7
tags: [ai, agents, architecture, llm, multi-agent, backend]
description: "The reasoning behind Genesis's five-agent pipeline: why splitting the work by kind of thinking beats one large prompt, and why plain JSON files beat an orchestration framework for handoffs."
---

The obvious way to build an AI app builder is one long, carefully written
prompt: pass in the user's request and let the model write the code. It's quick
to prototype and it works for simple apps. As apps get bigger, it fails in ways
that are hard to debug and harder to fix.

Genesis uses a pipeline of five agents instead: Planner, Architect, Developer,
Tester, and Reviewer. Each has a narrow job, defined inputs, and a defined
handoff. This post explains why, and what I'd change if I started again.

## The problem with one prompt

If one prompt has to plan, design, implement, test, and review an application,
a single context window has to switch between five very different kinds of
thinking:

- **Planning is about scope:** what's in, what's out, what comes first, what
  depends on what.
- **Architecture is about contracts:** what each component exposes, what it
  consumes, and what must stay true at the boundary.
- **Development is about implementation:** how it works in code, the happy
  path, the error path, and whether it matches the contract.
- **Testing is about breaking things:** which inputs make it fail and which
  edge cases the developer missed.
- **Review is about consistency:** whether the code matches the spec, whether
  there are security holes, and whether it can be maintained.

On human teams these are often different people. Asking one model call to do
all five gives you mediocre results on each, because the prompt can't go deep
on any of them, and the goals pull against each other. The developer wants the
code to work. The tester wants to show it doesn't.

## The five agents

### Planner

The Planner takes a product description in plain language and produces:

- a map of the system's domains
- a feature backlog with priority, size, and dependencies
- a phased plan (Phase 0: foundation, Phase 1: core loop, Phase 2+: extensions)

It doesn't write code, make architecture decisions, or estimate in lines of
code. Its only job is to decide what to build, in what order, and why.

Output: `docs/plan/` and `.pipeline/planner-output.json`

### Architect

The Architect reads the plan and produces the design:

- `spec.md` with component responsibilities and data flow
- architecture decision records (ADRs) for anything non-obvious
- API contracts between components
- a technology map

It isn't allowed to write implementation code. It works at the interface level:
what each module exposes, what must hold true, and what shape payloads take.
That can sound like overkill for a side project. In practice, the spec is what
keeps the Developer and the Reviewer from talking past each other.

Output: `docs/spec.md`, `docs/adr/`, `docs/contracts/`, `.pipeline/architect-output.json`

### Developer

The Developer reads the Architect's output and writes code in `src/`. It only
works on the current slice, not the whole system.

It isn't allowed to redesign the architecture while implementing. If it finds a
contradiction in the spec, it records a flag in its handoff JSON, makes a
reasonable assumption, and continues. The Reviewer will catch it.

Output: `src/`, `.pipeline/developer-output.json`

### Tester

The Tester reads the Developer's output and the Architect's contracts and
writes tests in `tests/`. It isn't allowed to write tests by copying the
structure of the source code, because those tests pass by construction. It
starts from the contract instead: given this interface, which inputs should
break it?

Output: `tests/`, `.pipeline/tester-output.json`

### Reviewer

The Reviewer is the only agent that reads everything: plan, spec, code, and
tests. It sorts issues into Critical (blocks the merge), Major (fix before the
next slice), and Minor (log it, fix during polish). If there are critical
issues, the pipeline loops and the Reviewer's output becomes the Developer's
input for the next pass.

Output: `.pipeline/reviewer-output.json`

## Why plain JSON files instead of a message bus

The most common question about this design is why the agents hand off through
JSON files in a `.pipeline/` directory instead of an orchestration framework or
an event bus.

I have three reasons.

**Debugging.** When a run goes wrong, `cat .pipeline/architect-output.json`
shows exactly what the Architect decided and why. I can diff it against what the
Developer assumed and find where things went off track without setting up
tracing or searching logs.

**Resuming.** Files survive restarts. If a run dies halfway through the
Developer pass, the Planner and Architect outputs are still on disk and the
pipeline picks up from the last agent that finished. An in-memory queue doesn't
give you that, and a message bus only does with extra setup.

**History.** Every handoff is a file I can commit, diff, review, and revert. If
the Reviewer blocks a merge, the reason is in the file. If I want to know why
the Architect chose a pattern, the ADR is in `docs/adr/`.

The downside is that this doesn't scale to many pipelines running in parallel.
Genesis runs one pipeline for one user at a time, so that's fine.

## What I got wrong the first time

In the first version I defined the agents with frontmatter fields I'd made up,
like `trigger`, `pipeline_position`, and `reads_from`. The tool running the
agents (Claude Code) didn't recognize any of them, couldn't discover the
agents, and the pipeline never ran cleanly.

The fix was obvious in hindsight: use the format the tool actually expects.
Subagents take `name`, `description`, `tools`, and `model`. Skills take
`description` and `allowed-tools`. Don't invent schema in config files and
hope something reads it.

More generally, the first version tried to be clever about discovering and
routing agents automatically. The version that works is explicit. Each agent
knows exactly which files it reads and which it writes.

## The review loop

The pipeline doesn't run straight through. The Reviewer's verdict decides what
happens next:

- **Approved:** the slice is done.
- **Critical issues:** the Developer runs again, with the Reviewer's critical
  issues as extra input.
- **Blocked:** the pipeline stops and asks me to resolve a contradiction or
  ambiguity.

The loop is capped. A well-defined slice should pass in one or two rounds. If
it doesn't, either the Planner made the slice too large or the Architect's
contracts were too vague. That's frustrating, but it tells me where the problem
is.

## What this costs

The costs are real, so here they are.

**Time.** Five agents running one after another means at least five long LLM
calls. A slice takes clearly longer than a single-prompt generation. For a
complete, tested, reviewed slice of an app, I'm fine with that. For code
autocomplete it would be ridiculous.

**Context.** Each agent reads its predecessors' output before it starts, so
the further down the pipeline, the more tokens are spent before any new work
happens. The Reviewer, which reads everything, is the most expensive.

**Maintenance.** Five agents means five system prompts, five sets of rules to
keep consistent, and five handoff formats to keep in sync. Change the Planner's
output schema and you have to update the Architect's input handling too.

The design suits Genesis because generating a whole tested application needs
it. For a simpler task, one good prompt is probably the right tool.

---

Multi-agent pipelines aren't new. What's worth looking at is *when* they're
worth it and how the details work in practice. For Genesis, three decisions
made the difference: splitting agents by the kind of thinking they do, making
every handoff explicit, and keeping pipeline state in plain files.
