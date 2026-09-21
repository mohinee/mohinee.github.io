---
layout: post
title: "P90 Just Jumped: A Playbook for Latency Regressions"
date: 2026-08-13
tags: [performance, frontend, observability, debugging, engineering]
description: "When a latency graph steps up, the instinct is to open the profiler. The order you investigate in matters more than the tools. A sequence that finds causes faster and breaks fewer things."
---

A latency graph steps up after a release. Everyone opens the profiler.

That is the wrong first move. Profiling is slow, it answers "where is time going?" and not "what changed?", and while you do it, users are still having the bad experience. Here is the order I try to follow instead. It is boring, and it is the fastest path I know.

## 1. Confirm it, then slice it

Before theorizing, establish what you are actually looking at.

- **Which percentile?** A p50 shift usually means every request got slower: a code path, a dependency, a bundle. A p90 or p99 shift with a flat median usually means a *subset* got slower: a cohort, a cold path, a cache miss, a tail effect.
- **Which slice?** Break the metric down by release ring, region, browser, device class, tenant, route, and logged-in versus anonymous. A regression that appears in only one slice is already half-diagnosed.
- **Is the measurement itself sound?** Check that the instrumentation, sampling rate, and definition of the metric did not change in the same release. Some "regressions" are a new timer starting earlier.

The output of this step is one sentence: "p90 of X rose by Y for cohort Z starting at time T." Without that sentence you are debugging a feeling.

## 2. Correlate with change

Regressions are caused by something that changed. Line up the start of the step with:

- Deploys and their contents
- Feature flag flips and rollout percentage changes
- Dependency or infrastructure changes
- Traffic shape changes (a new customer, a campaign, a bot)

Flags deserve special attention because they change behavior *without* a deploy, so they don't appear in your release notes. If your flag system has an audit log, check it before you check the diff.

## 3. Mitigate before you diagnose

If the timing points at a specific change and you can revert it safely, revert it. Turn the flag off, roll back the release ring, halt the ramp.

This feels like giving up on understanding the bug. It is the opposite. Rollback buys you a quiet system to investigate in, and it gives you the most valuable piece of evidence there is: **if the metric recovers, you have confirmed the cause without profiling anything.** If it doesn't recover, you have eliminated a suspect just as cheaply.

## 4. Localize: client or server?

Only now do you need to find where the time went. The question at this stage is a coarse one: is the added time before the request leaves the client, in the network and server, or after the response arrives?

- **Distributed tracing** with a shared trace ID from the browser through every service answers this directly.
- **`Server-Timing` response headers** let the server report its own processing time to the browser, so both numbers show up in the same waterfall.
- **Resource Timing and Navigation Timing** entries give you the client-side split between DNS, connect, request, and response.

If the server side is flat, the regression is in the client, which is where the rest of this post lives.

## 5. Client-side causes, in rough order of likelihood

When I have narrowed a regression to the frontend, this is the order I check, because it is the order things have most often turned out to be true:

1. **Bundle growth.** A new dependency, a barrel-file import that defeated tree-shaking, or a chunk that used to be lazy and is now loaded eagerly. Compare bundle analysis output between the good and bad builds. This is a five-minute check that explains a surprising share of regressions.
2. **A request waterfall.** Two calls that used to run in parallel now run in sequence, often because one now depends on the other's result, or because an `await` was added inside a loop.
3. **Broken memoization causing render explosions.** A selector or hook that used to return a stable reference now returns a new object on every call, so every consumer re-renders on every store update. The React Profiler's "why did this render?" view makes this easy to confirm.
4. **Unstable keys or remounting.** List items whose keys changed to something non-stable will unmount and remount instead of updating.
5. **Synchronous main-thread work.** A large JSON parse, a sort, a regex, or a synchronous layout read inside a loop. Look for long tasks in the Performance panel.
6. **Preload or prefetch coupling.** An optimization that starts speculative work on a user gesture, such as hover, but is tied so tightly to the interaction that it now delays the thing the user asked for.
7. **Memory growth.** Leaks show up as latency that worsens over the life of a session, not as a step change. If the graph is a slope rather than a cliff, look here.

The theme is that most frontend regressions are not slow code. They are *more* code, *more* work, or *the same work in a worse order*.

## 6. Measure before and after every fix

Whatever you change, capture the metric before and after, on the same slice, on the same kind of traffic. A fix you have not measured is a hypothesis. It is also worth checking that the fix didn't just move the cost elsewhere: an optimization that speeds up one interaction while quietly slowing a neighboring one is common when code paths are shared.

Shared code paths are also why a "quick patch" is not always quick. If the slow code is used by more than one surface, a change that helps one can regress another. A staged rollout, one surface at a time behind a flag, is slower to start and much faster to finish than a patch you have to un-ship.

## 7. Make the next one cheaper

The real payoff comes after the incident:

- **Performance budgets in CI.** Fail the build when bundle size or a key synthetic metric crosses a threshold, so regressions are caught in the pull request instead of on a dashboard.
- **Real-user monitoring alerts per release ring.** If canary users see a p90 step, the alert fires before the change reaches everyone.
- **Canary and ring-based rollouts.** Regressions found at 1% of traffic are a nuisance. At 100% they are an incident.
- **Profiler checks in tests** for the components you know are hot.

## The short version

1. Confirm and slice the metric.
2. Correlate with deploys, flags, dependencies, and traffic.
3. Roll back or switch off first, diagnose second.
4. Localize client versus server.
5. Check the likely client causes in order: bundle, waterfall, renders, keys, main thread, preloads, memory.
6. Measure before and after.
7. Add the guardrail that would have caught it.

The tool you use at each step barely matters. The order does.
