---
layout: post
title: "Why I Built a Five-Agent Pipeline Instead of One Big Prompt"
date: 2026-06-09
tags: [ai, agents, architecture, llm, multi-agent, backend]
description: "The design thinking behind Genesis's five-agent pipeline — why task decomposition beats monolithic prompting, and why boring file-based handoffs beat clever orchestration frameworks."
---

The obvious way to build an AI-powered app builder is to write one large, well-crafted prompt, pass in the user's request, and let the model produce code. It's fast to prototype. It works for simple cases. And it will eventually fail you in ways that are hard to debug and harder to fix.

Genesis takes a different approach: a pipeline of five specialized agents — Planner, Architect, Developer, Tester, Reviewer — each with a narrow job, explicit inputs, and a defined handoff. Here's why that decision was made, and what I'd do differently if I built it again.

---

## The Core Problem with Monolithic Generation

When you give a single LLM prompt the job of planning, designing, implementing, testing, and reviewing an application, you're asking one context window to hold expert-level knowledge across four fundamentally different modes of thinking.

A planner needs to reason about **scope** — what's in, what's out, what comes first, what's a dependency of what.

An architect needs to reason about **contracts** — what does this component expose, what does it consume, what invariants must hold at the boundary.

A developer needs to reason about **implementation** — how does this actually work in code, what's the error path, what's the happy path, does this match the contract.

A tester needs to reason about **falsification** — where will this break, what input makes it fail, what edge cases did the developer not consider.

A reviewer needs to reason about **coherence** — does the implementation match the spec, are there security holes, is this maintainable.

These are not the same cognitive mode. In human teams, they're often different people. Asking one model pass to do all five produces output that is mediocre at all of them, because the prompt can't afford to go deep on any one. The context is split across concerns that actively pull in different directions.

---

## The Five Agents in Genesis

### Agent 0 — Planner

The Planner takes a natural-language product description and produces:

- A domain map of the system
- A sequenced feature backlog with priority, size, and dependencies
- An implementation phase plan (Phase 0: foundation, Phase 1: core loop, Phase 2+: extensions)

It does not write code. It does not make architectural decisions. It does not estimate in lines-of-code. Its entire job is to answer: *what should we build, in what order, and why*.

Output: `docs/plan/` and `.pipeline/planner-output.json`

### Agent 1 — Architect

The Architect reads the Planner's output and produces system design artifacts:

- A `spec.md` with component responsibilities and data flow
- Architecture Decision Records (ADRs) for non-obvious choices
- API contracts between components
- A technology map

The Architect is explicitly prohibited from writing implementation code. It reasons at the interface layer — what does each module expose, what are the invariants, what shape do payloads take. This sounds like over-engineering for a side project. In practice, it's the document that keeps the Developer and Reviewer from talking past each other.

Output: `docs/spec.md`, `docs/adr/`, `docs/contracts/`, `.pipeline/architect-output.json`

### Agent 2 — Developer

The Developer reads the Architect's output and writes production code in `src/`. Critically, it is scoped to the current pipeline pass — it doesn't try to implement the entire system. It implements what was planned for this slice.

It is explicitly prohibited from redesigning architecture mid-implementation. If it encounters a contradiction in the spec, it writes a flag to the handoff JSON and continues with reasonable assumptions. The Reviewer will catch it.

Output: `src/`, `.pipeline/developer-output.json`

### Agent 3 — Tester

The Tester reads the Developer's output and the Architect's contracts, then writes tests in `tests/`. It is not allowed to read the source code and write tests that mirror its structure — that produces tests that pass trivially. It reasons from the contract outward: given this interface, what inputs would break it?

Output: `tests/`, `.pipeline/tester-output.json`

### Agent 4 — Reviewer

The Reviewer is the only agent that reads everything — plan, spec, implementation, and tests — and produces a verdict. It classifies issues as Critical (block merge), Major (fix before next slice), or Minor (log, address in polish). If there are critical issues, the pipeline loops: the Reviewer's output becomes the Developer's input for the next pass.

Output: `.pipeline/reviewer-output.json`

---

## Why Plain JSON Files Over a Message Bus

The most common reaction to this design is: *why are agents communicating via JSON files in a `.pipeline/` directory instead of a proper orchestration layer or event bus?*

Three reasons.

**Debuggability.** When a pipeline fails, you can `cat .pipeline/architect-output.json` and read exactly what the Architect decided and why. You can diff it against the Developer's assumptions. You can see where the chain broke without setting up tracing infrastructure or digging through logs. The entire pipeline state is readable as text files in your project directory.

**Statefulness across sessions.** File-based handoffs persist across process restarts. If a Cloud Function times out halfway through a Developer run, the Planner and Architect outputs are not lost. The pipeline can resume from the last completed agent. A message bus or in-memory queue doesn't give you this for free.

**Auditability.** Every agent's decision is a record in a directory that can be committed to version control, diffed, reviewed, and rolled back. If a Reviewer blocks a merge and you want to understand why, the answer is in the file. If a user asks "why did the Architect choose this pattern", the ADR is in `docs/adr/`.

The tradeoff is that this pattern doesn't scale to high-volume parallel pipelines. Genesis is a single-user, single-pipeline system. The tradeoff is correct.

---

## What I Got Wrong the First Time

The first version of this system had agents defined with custom frontmatter fields I invented — `trigger`, `pipeline_position`, `reads_from`. They didn't correspond to anything the orchestrator actually understood. The pipeline never ran cleanly because the orchestrator couldn't discover the agents.

The fix was obvious in retrospect: use the formats the orchestrator actually expects. Subagents need `name`, `description`, `tools`, `model`. Skills need `description` and `allowed-tools`. Don't invent schema in configuration files.

More broadly: the first version tried to be clever about agent discovery and auto-routing. The working version is explicit. Each agent knows exactly what it reads and exactly what it writes. There is no magic.

---

## The Iteration Loop

The pipeline is not linear in practice. The Reviewer's verdict determines the next state:

- **All clear**: pipeline complete, output is ready
- **Major issues**: Developer reruns with Reviewer's critical issues as additional input
- **Blocked**: pipeline halts, user is prompted to resolve a contradiction or ambiguity

This loop is bounded. A well-specified slice should converge in one or two passes. If it's not converging, the Planner's slice was too large or the Architect's contracts were underspecified. That's useful signal even when it's frustrating.

---

## What This Approach Costs

Honesty requires listing the costs.

**Latency.** Five sequential agent passes is five LLM calls. For a single feature slice, this takes noticeably longer than a single-pass generation. For the problem Genesis is solving — generating a complete, tested, reviewed application slice — the latency is acceptable. For a code autocomplete use case, it would be absurd.

**Context overhead.** Each agent reads its predecessors' output files before starting. The further down the pipeline, the more context is consumed before the agent does any work. The Reviewer, which reads everything, has the most expensive context load.

**Prompt engineering surface area.** Five agents means five system prompts to maintain, five sets of constraints to keep consistent, and five handoff formats to keep synchronized. When you change the Planner's output schema, you have to update the Architect's reader. This is real maintenance cost.

The design makes sense for Genesis because the problem demands it. For a simpler task, a well-crafted single prompt is probably the right tool.

---

The pipeline approach is not novel — multi-agent systems are a well-established pattern. What's worth examining is *when* they're appropriate, and what the concrete implementation decisions look like. In Genesis, specialization by cognitive mode, explicit handoffs, and file-based state persistence are the three decisions that made the system actually work.

---

*All architectural decisions described here are from my own project work. Nothing is specific to any employer's internal systems.*
