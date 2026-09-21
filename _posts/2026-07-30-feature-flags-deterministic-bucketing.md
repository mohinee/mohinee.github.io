---
layout: post
title: "Feature Flags Done Right: Why Math.random() Breaks Your Rollouts"
date: 2026-07-30
tags: [architecture, feature-flags, typescript, backend, frontend]
description: "Percentage rollouts look trivial until users flicker between variants and your A/B metrics stop meaning anything. Deterministic hashing, salting, and evaluation order, with code you can run."
---

A feature flag service is one of those systems where the first version is 20 lines and the correct version is about 60. Almost all of the difference is in one function: how you decide whether *this* user is in the 10%.

## The naive version

```ts
const enabled = Math.random() < 0.10;
```

It passes every demo. It also fails in three ways that only show up in production:

1. **Flicker.** The same user gets a different answer on every evaluation. They see the new checkout on page load and the old one after a refresh.
2. **Corrupted experiments.** If a user can be in both arms across sessions, your A/B analysis is comparing populations that overlap, and the result is meaningless.
3. **Untestable.** You cannot write a test that says "user 42 gets the new flow" because the answer is different each run.

The requirement is that the assignment is a pure function of stable inputs, so the same user and flag always produce the same answer with no stored state.

## Deterministic bucketing

Hash the user into one of N buckets and compare the bucket to the rollout percentage.

```ts
const enc = new TextEncoder();

// 32-bit FNV-1a over UTF-8 bytes
export function fnv1a(input: string): number {
  let h = 0x811c9dc5;
  for (const byte of enc.encode(input)) {
    h ^= byte;
    h = Math.imul(h, 0x01000193);
  }
  return h >>> 0;
}

// 0..9999, so rollouts can be as fine-grained as 0.01%
export const bucket = (flagKey: string, userId: string) =>
  fnv1a(`${flagKey}:${userId}`) % 10_000;

const inRollout = bucket(flag.key, user.id) < flag.rollout.percent * 100;
```

Three details that matter:

**Hash the UTF-8 bytes, not `charCodeAt`.** JavaScript strings are UTF-16. If you hash code units, a Go or Python SDK evaluating the same flag will compute a different bucket for any non-ASCII ID, and your server and client will disagree about who is in the rollout. Hashing bytes makes the algorithm portable.

**Salt with the flag key.** Without `flagKey:` in the hash input, a user's bucket is the same for every flag. Then "10% of users get new checkout" and "10% of users get dark mode" are the *same* 10% of users. Every experiment is confounded with every other, and the unlucky 10% bear the risk of every rollout at once.

**A non-cryptographic hash is fine here.** You want speed and uniformity, not collision resistance. FNV-1a is a few lines and has no dependencies, which is handy for a browser SDK. Any well-distributed hash works. Just pick one and never change it, because changing the hash reshuffles every user in every rollout.

## Measuring it

I ran 100,000 synthetic user IDs through this code to check the claims rather than assume them:

| Check | Result |
|---|---|
| Actual size of a 10% rollout | 10.09% |
| Same user evaluated twice | always identical |
| Users in at 10% who are still in at 20% | 100% |
| Overlap of two salted 10% flags | 1.07% (expected about 1%) |
| Overlap of two *unsalted* 10% flags | 10.04%, the same users every time |

The last two rows are the salting argument in numbers. The third row is a property worth calling out: because the bucket is fixed per user, raising a rollout from 10% to 20% only *adds* users. Nobody who already has the feature loses it mid-ramp.

## Evaluation order is a design decision

A flag can be disabled globally, targeted at specific users, and rolled out by percentage, all at the same time. The order you check these in decides what happens when they conflict:

```ts
export function evaluate(flag: Flag, user: User) {
  if (flag.killed) return flag.defaultValue;                    // 1. kill switch
  for (const rule of flag.rules ?? []) {                         // 2. targeting, first match wins
    if (rule.attr in user && user[rule.attr] === rule.equals) return rule.value;
  }
  if (flag.rollout) {                                            // 3. percentage rollout
    return bucket(flag.key, user.id) < flag.rollout.percent * 100
      ? flag.rollout.value
      : flag.defaultValue;
  }
  return flag.defaultValue;                                      // 4. default
}
```

- **The kill switch is first, and beats everything**, including explicit targeting. At 2 a.m. during an incident, "turn it off for everyone" must mean everyone.
- **Targeting beats the percentage.** If you have explicitly listed an internal tester, they should get the feature regardless of which bucket they hash into.
- **A rule referencing a missing attribute must skip, not crash.** The `rule.attr in user` guard handles the case where a caller did not supply that attribute.

## The part "in-memory, no database" hides

If you keep flag state in process memory, everything above works on a single instance. The moment you run two, you have a coordination problem: a write to instance A must reach instance B. "No database" does not remove that requirement. It moves it somewhere else.

Options, roughly in order of complexity:

- **Poll a source of truth** on an interval. Simple and robust, but changes take up to one interval to propagate.
- **Publish changes over pub/sub** (Redis, a message bus) so instances update within milliseconds, with periodic full reconciliation as a safety net for missed messages.
- **A single writer** that everything else follows.

Whichever you choose, say the limitation out loud. A design that quietly assumes one process is not a design.

## Getting flags to the browser

For a React app, a context provider holding a flag snapshot works well. The snapshot should be delivered in one of two ways:

- **Embedded in the initial HTML** (for example on `window.__FLAGS__`) when server-rendering, so the first paint is already correct and nothing flickers.
- **Streamed via Server-Sent Events** for live updates. Flag changes are one-directional, server to client, so SSE is a better fit than WebSockets: it reconnects automatically and works through ordinary HTTP infrastructure.

Bulk evaluation matters here. Have the server expose an `evaluateAll(user)` that returns every flag's value for that user in one call, rather than the client asking flag by flag.

## Failure behavior

Decide in advance what a flag returns when the flag service is unreachable, and encode it as each flag's compiled-in default, not as a global "fail open" or "fail closed." A kill switch for a risky feature should default to *off*. A flag guarding a long-shipped safety check should default to *on*. The SDK should serve the last known value if it has one, and the default if it does not.

## Checklist

- Deterministic hash over UTF-8 bytes, chosen once and never changed
- Flag key mixed into the hash input
- Kill switch, then rules, then percentage, then default
- Rollouts are monotonic: ramping up never removes anyone
- A multi-instance propagation story, stated explicitly
- Per-flag safe defaults for when the service is down

The code above is small enough to keep in one file, and the checks table takes a few lines of test code to reproduce. I'd suggest running it before trusting any flag system, including your own.
