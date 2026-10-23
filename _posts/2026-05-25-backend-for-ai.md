---
title: "Designing backend systems for AI agents: what I learned building for Microsoft Teams"
date: 2026-05-25
reading_time: 7
tags: [backend, architecture, ai]
description: "Backend lessons from building enterprise AI agents: pipeline-shaped services, event-sourced conversation state, permission-aware retrieval, and a latency budget that the p95 blew through."
---

For the last three years I've worked at Microsoft on AI agents that run inside
Microsoft Teams. These agents take actions, not just answer questions: they
profile employee skills across the organization, handle HR and IT support
tickets, and pull up relevant information from across search platforms.

The frontend gets the attention: the chat UI, the streaming responses, the
adaptive cards. Most of the engineering, and most of the mistakes, are in the
backend.

This post covers the backend patterns that held up for us in production, with
a large enterprise user base, and a few we had to replace.

## One message, many steps

A traditional web backend handles stateless request-response. A form comes in;
you validate it, save it, and respond.

An agent doesn't fit that model. One user message can trigger:

1. Intent classification (which agent should handle this?)
2. Context retrieval from several data sources
3. An LLM call that takes 2–8 seconds
4. A tool call, if the model decides it needs one
5. A second LLM call to turn the tool result into an answer
6. Saving state so the conversation can continue

That's an orchestration pipeline, not a request. Handle it in a standard
controller action and you end up with 400-line methods you can't test.

### What worked: a pipeline-shaped service layer

We moved to an explicit pipeline early. In C# it looked roughly like this:

```csharp
public class AgentPipeline
{
    private readonly IIntentClassifier _classifier;
    private readonly IContextRetriever _contextRetriever;
    private readonly ILLMOrchestrator _llm;
    private readonly IToolExecutor _toolExecutor;

    public async Task<AgentResponse> ProcessAsync(
        UserMessage message,
        ConversationState state,
        CancellationToken ct)
    {
        var intent = await _classifier.ClassifyAsync(message, ct);
        var context = await _contextRetriever.RetrieveAsync(intent, state, ct);
        var llmResult = await _llm.GenerateAsync(message, context, ct);

        if (llmResult.RequiresToolExecution)
        {
            var toolResult = await _toolExecutor.ExecuteAsync(
                llmResult.ToolCall, ct);
            return await _llm.SynthesizeAsync(
                message, toolResult, context, ct);
        }

        return llmResult.ToResponse();
    }
}
```

Each stage can be tested on its own, instrumented on its own, and given its own
retry policy. Passing the `CancellationToken` all the way down matters more than
it looks. Users abandon conversations constantly, and without cancellation you
keep paying for LLM calls whose results nobody will see.

## Conversation state is harder than it looks

An agent conversation isn't just a list of messages. It includes:

- The raw message history
- Extracted entities and their resolved values
- Tool results, some of which had side effects
- Preferences the user expressed along the way
- The task the agent thinks it's currently working on

We started by storing all of that as a single JSON document in Cosmos DB. It
lasted about two months before we hit the problems you'd expect: race
conditions when users sent messages in quick succession, documents that kept
growing (some conversations passed 50 KB of context), and stale reads when two
pipeline steps updated state at the same time.

### What worked: event-sourced conversation state

We switched to deriving conversation state from an ordered log of events:

```
ConversationCreated → MessageReceived → IntentClassified →
ContextRetrieved → LLMResponseGenerated → ToolExecuted →
ResponseDelivered → MessageReceived → ...
```

Current state is a projection over that log. This fixed three things:

1. **Concurrent writes.** Two steps no longer overwrite each other's version of
   a shared document. Each one appends an event with the next sequence number
   for that conversation, and a duplicate sequence number is rejected, so
   conflicts become retries instead of lost updates. Partitioning by
   conversation ID keeps each log inside one Cosmos DB logical partition,
   where writes can be transactional.
2. **Debugging.** You can replay any conversation and see exactly what happened
   and in what order.
3. **Size.** The projection can drop old context while the full log stays
   available for auditing.

The cost is on reads. Rebuilding state from scratch on every message gets
expensive for long conversations, so we stored a snapshot every N events and
replayed only the events after it.

## RAG is mostly a backend problem

RAG tutorials focus on retrieval and generation. In production, most of the
work is the infrastructure around them.

**The indexing pipeline.** The skills-profiling agent had to index employee
data from several sources: HR systems, project management tools, internal
profiles, and collaboration activity. Each source updated on its own schedule,
had its own schema, and had its own access rules. A set of Azure Functions
normalized everything into a common schema before embedding.

**Freshness versus cost.** Re-embedding everything on every change is too
expensive. We tracked a content hash per document and re-embedded only what had
changed, which cut embedding costs by roughly 70%. It also created a staleness
window, which we had to make visible to users.

**Access control at query time.** This is where many enterprise RAG systems go
wrong: the agent retrieves content the *person asking* isn't allowed to see. We
tagged every embedding with the access control list from its source system and
added the user's permissions as a filter on the vector query, so the similarity
search only returns documents that user can already open.

```csharp
var results = await _vectorStore.SearchAsync(
    query: embeddedQuery,
    filter: BuildAccessFilter(user.Permissions),
    topK: 10,
    cancellationToken: ct
);
```

It's not exciting work, but in an enterprise you can't skip it. Get it wrong
and your agent becomes a tool for leaking data.

## The latency budget

In our user research, people started assuming the agent had crashed after about
4 seconds with nothing on screen. They retried or gave up.

Here's how the budget broke down:

| Stage | Budget | p50 | p95 |
|-------|--------|-----|-----|
| Intent classification | 100ms | 45ms | 120ms |
| Context retrieval | 300ms | 180ms | 450ms |
| LLM generation | 2500ms | 1800ms | 3200ms |
| Tool execution | 500ms | 200ms | 800ms |
| Synthesis | 1500ms | 900ms | 2100ms |
| **Total** | **4900ms** | **3125ms** | **6670ms** |

The p95 was well over budget. Making individual stages faster wasn't going to
close a 1.8-second gap, so we changed the design:

1. **Streaming.** Send LLM tokens to the client as they're generated. What the
   user waits for is now time to first token, not total time.
2. **Speculative retrieval.** Start fetching context with the raw message as
   the query *before* intent classification finishes. If the classifier picks a
   different retrieval strategy, throw the result away and fetch again. The
   speculative fetch turned out to be usable about 80% of the time.
3. **No cold starts on the hot path.** Cold starts on our Azure Functions
   regularly cost 800ms or more. We moved the latency-critical paths to
   always-warm instances.

## What I'd do differently

**Add observability on day one.** We added structured logging and distributed
tracing six months in, and those six months of debugging without them were
painful. Instrument every pipeline stage from the start: every LLM call, every
retrieval query, every tool call. It costs very little and saves a lot of time.

**Don't abstract the LLM too early.** We built a generic `ILLMProvider`
interface up front, expecting to swap models often. In practice each model had
different prompting needs, token limits, latency, and failure modes. The
abstraction leaked everywhere, and maintaining it cost more than direct
integrations would have.

**Treat model output as untrusted input.** From a security standpoint, LLM
output is user-generated content. It can contain injected instructions, tool
calls the model made up, and malformed data. Validate every proposed tool call
against a strict schema before running it.

---

Building backends for AI agents has more in common with distributed systems
than with "AI engineering." The LLM is one component in a pipeline that also
needs state management, access control, caching, and observability, and those
are problems we already know how to solve. Teams that treat agents as an
entirely new discipline tend to rebuild worse versions of existing solutions.

The event log, the permission filter, and the latency budget aren't the
interesting parts of an AI product. They're what made ours usable in
production.
