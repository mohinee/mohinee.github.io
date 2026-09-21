---
layout: post
title: "Learning Go by Fanning Out to Many APIs at Once"
date: 2026-07-14
tags: [go, concurrency, backend, learning, llm]
description: "A small Go CLI that calls several LLM APIs concurrently teaches goroutines, channels, context cancellation and error handling in one sitting. Here are the design decisions that mattered."
---

If you already write TypeScript, C# or Java, Go's syntax takes an afternoon. What takes longer is unlearning habits. Go has no exceptions, no `async/await`, and no inheritance. It has small structural interfaces, errors as ordinary values, and goroutines.

The fastest way I know to force all of that into your head is a small project that touches every one of them. I picked a CLI that sends one prompt to several LLM provider APIs at once, waits for whoever answers, and reports the successes and the failures separately. It is roughly 100 lines, and almost every line is a decision.

## The shape of the problem

Fan-out with partial failure is a real pattern, not a toy. Anything that queries multiple backends (search shards, price feeds, model providers) has the same three requirements:

1. Run all calls concurrently, so total latency is roughly the slowest call, not the sum.
2. Bound the whole operation with one deadline.
3. One provider failing must not discard the answers that succeeded.

## Decision 1: Put the error inside the response

Coming from exception-based languages, my first instinct was two channels, one for results and one for errors. That makes aggregation awkward, because you now have to `select` across both and keep counts in sync.

The idiomatic move is to make the failure part of the data:

```go
type Response struct {
	Provider string
	Text     string
	Latency  time.Duration
	Err      error
}

type Provider interface {
	Name() string
	Complete(ctx context.Context, prompt string) (string, error)
}
```

One channel now carries every outcome, and the receiver reads exactly `len(providers)` values. Note that `Provider` is an interface any type can satisfy without declaring it, exactly like a TypeScript structural type. That is what makes the fake provider in the tests trivial.

## Decision 2: Size the channel so nobody can get stuck

```go
func FanOut(ctx context.Context, providers []Provider, prompt string) []Response {
	out := make(chan Response, len(providers)) // buffered

	for _, p := range providers {
		go func() {
			start := time.Now()
			text, err := p.Complete(ctx, prompt)
			out <- Response{p.Name(), text, time.Since(start), err}
		}()
	}

	results := make([]Response, 0, len(providers))
	for range providers {
		results = append(results, <-out)
	}
	return results
}
```

The buffer is not a performance tweak. It is what guarantees every goroutine can complete its send and exit, even if the receiver stops reading.

I confirmed that by breaking it on purpose. I made the channel unbuffered and had the collector return early when the context expired. After the function returned, the goroutine count went from 2 to 5 in a test. Three senders were blocked on a send that would never be received. That is a goroutine leak, and in a long-running service it is a slow memory leak with no stack trace pointing at it.

Two smaller things in that snippet:

- `for _, p := range providers { go func() { ... p ... }() }` is only safe on Go 1.22 or later, where each iteration gets its own copy of the loop variable. On older versions all goroutines would see the last provider. The behavior is controlled by the `go` directive in `go.mod`, not by which compiler you have installed. If a module still says `go 1.21`, you get the old semantics.
- `for range providers` iterates purely for the count. There is no need for an index variable.

## Decision 3: Cancellation is cooperative, so build the fake first

`context.Context` does not kill anything. It is a signal, and the callee has to watch for it. A provider that ignores the context will happily run past your deadline.

So before touching a real HTTP client, I wrote a fake provider that honors the context the way a real one must:

```go
func (f Fake) Complete(ctx context.Context, prompt string) (string, error) {
	select {
	case <-time.After(f.Delay):
		if f.Fail {
			return "", ErrBoom
		}
		return f.ID + " says: " + prompt, nil
	case <-ctx.Done():
		return "", ctx.Err()
	}
}
```

When you later swap in real HTTP, the equivalent is `http.NewRequestWithContext(ctx, ...)`. The pattern is the same: the slow thing races the context, and the context wins.

I would also write the sequential version first and time it. Then the concurrent version has a baseline to beat, and you can see the difference instead of assuming it.

## Decision 4: Partial failure is data, and errors should still compose

After the fan-out, I split results into successes and one combined error:

```go
func Summarize(results []Response) (ok []Response, err error) {
	var errs []error
	for _, r := range results {
		if r.Err != nil {
			errs = append(errs, fmt.Errorf("%s: %w", r.Provider, r.Err))
			continue
		}
		ok = append(ok, r)
	}
	return ok, errors.Join(errs...)
}
```

`%w` wraps the cause and keeps the provider name as context. `errors.Join` (Go 1.20+) bundles them, and `errors.Is` still sees through the whole thing. That lets the CLI decide what to do with `errors.Is(err, context.DeadlineExceeded)` without string matching.

The test that pins this behavior uses three fakes: one fast, one that fails, one that is far slower than the deadline.

```go
ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
defer cancel()

results := FanOut(ctx, []Provider{
	Fake{ID: "fast", Delay: 10 * time.Millisecond},
	Fake{ID: "broken", Delay: 10 * time.Millisecond, Fail: true},
	Fake{ID: "slow", Delay: 2 * time.Second},
}, "hi")

ok, err := Summarize(results)
// ok has exactly one entry ("fast")
// errors.Is(err, ErrBoom)                  == true
// errors.Is(err, context.DeadlineExceeded) == true
// and the whole call returned in ~100ms, not 2s
```

Run it with `go test -race`. The race detector costs almost nothing to enable and is the closest thing Go has to a safety net for this kind of code.

## What transferred, and what didn't

Mapping from what I already knew:

| Familiar idea | Go's version |
|---|---|
| `Promise.allSettled` / `Task.WhenAll` | goroutines + a buffered channel, collected by count |
| `AbortController` / `CancellationToken` | `context.Context`, passed as the first argument by convention |
| `try/catch` | `if err != nil`, plus wrapping with `%w` |
| Structural typing (TS) | Interfaces, satisfied implicitly |

The thing that did not transfer is the mindset about lifetimes. In a language with a runtime that manages async work for you, "what happens to the losers when I stop waiting?" is rarely your problem. In Go it always is. Every goroutine you start needs a story for how it ends.

## If you want to build it

A sequence that worked for me:

1. Interface and fake provider (with `select` on `ctx.Done()`).
2. Sequential baseline, timed.
3. Concurrent `FanOut` with the buffered channel.
4. `context.WithTimeout` at the top level.
5. `Summarize` with `errors.Join`.
6. Real HTTP providers behind the same interface.
7. Table-driven tests under `-race`, then break the buffer deliberately and watch the leak.

Step 7 is the one that makes the rest stick.
