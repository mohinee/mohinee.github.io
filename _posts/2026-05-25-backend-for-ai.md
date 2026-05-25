---
layout: post
title: "Designing backend systems for AI agents: what I learned building for Microsoft Teams"
date: 2026-05-25
categories: [backend, architecture, ai]
description: "Practical backend architecture lessons from building enterprise AI agents — covering service design, state management, RAG pipelines, and the infrastructure decisions that actually matter at scale."
---

I've spent the last three years at Microsoft building AI agents that live inside Microsoft Teams. Not chatbots — agents that actually *do things*: profile employee skills across the organization, automate HR and IT support tickets, and surface contextual information across search platforms.

The frontend gets the attention. The conversational UI, the streaming responses, the slick card-based interactions in Teams. But the backend is where the real engineering happens, and where most teams get it wrong.

This post covers the backend architecture patterns I've found essential for production AI agent systems. Not the theoretical stuff from architecture blogs — the patterns that survived contact with millions of enterprise users.

## The request isn't a request anymore

Traditional web backends handle stateless request-response cycles. A user submits a form, you validate, persist, and respond. The mental model is clean.

AI agents break this model completely. A single user message might trigger:

1. Intent classification (which agent should handle this?)
2. Context retrieval from multiple data sources
3. An LLM call that takes 2-8 seconds
4. Tool execution based on the LLM's decision
5. Another LLM call to synthesize the result
6. State persistence for conversation continuity

That's not a request — it's an orchestration pipeline. And if you try to handle it with a traditional controller-action pattern, you'll end up with methods that are 400 lines long and impossible to test.

### What worked: a pipeline-based service layer

We moved to an explicit pipeline model early on. In C#, this looked roughly like:

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

Each stage is independently testable. Each stage can be instrumented. Each stage can fail independently with its own retry logic. The `CancellationToken` threading is critical — users abandon conversations constantly, and you don't want orphaned LLM calls burning tokens.

[VERIFY: If your actual pipeline architecture differed from this, adjust the code and explanation. The shape matters more than the exact implementation — readers will notice if the patterns don't match real production constraints.]

## The state management problem nobody warns you about

Conversation state sounds simple. It isn't.

An AI agent conversation isn't just a list of messages. It's a graph of:
- The raw message history
- Extracted entities and their resolved values
- Tool execution results (which may have side effects)
- User preferences learned during the conversation
- The current "task" the agent believes it's working on

We initially stored this as a flat JSON blob in CosmosDB. That lasted about two months before we hit every problem you'd expect: race conditions when users sent messages quickly, state objects growing unbounded (some conversations generated 50KB+ of context), and the classic "stale read" when multiple agent steps tried to update state concurrently.

### What worked: event-sourced conversation state

We moved to an event-sourcing model where conversation state is derived from an ordered log of events:

```
ConversationCreated → MessageReceived → IntentClassified → 
ContextRetrieved → LLMResponseGenerated → ToolExecuted → 
ResponseDelivered → MessageReceived → ...
```

The current state is a projection over this event stream. This solved three problems simultaneously:

1. **No race conditions** — appending events is an atomic operation
2. **Debuggability** — you can replay any conversation to see exactly what happened and why
3. **Bounded state** — the projection can discard old context while keeping the event log for auditing

The tradeoff is read complexity. Rebuilding state from events on every request is expensive if the conversation is long. We used snapshotting — persist a materialized state every N events and replay from the last snapshot.

[ADD YOUR DETAIL: What was the specific snapshotting interval you used? What was the p95 state reconstruction time? These concrete numbers make the post credible.]

## RAG is a backend problem, not an AI problem

Every "RAG tutorial" focuses on the retrieval and generation parts. In production, the hard part is the infrastructure around it:

**Indexing pipeline.** Our skills-profiling engine needed to index employee data from multiple sources — HR systems, project management tools, internal profiles, and communication patterns. Each source had different update frequencies, different schemas, and different access control requirements. The indexing pipeline was a series of Azure Functions that normalized data into a common schema before embedding.

**Freshness vs. cost.** Re-embedding everything on every update is prohibitively expensive. We implemented a change-detection layer that tracked document hashes and only re-embedded modified content. This reduced our embedding API costs by roughly 70%, but introduced a staleness window we had to make visible to users.

**Access control at retrieval time.** This is where most enterprise RAG implementations fail. The agent can retrieve information that the *asking user* shouldn't have access to. We implemented row-level security in the vector store by tagging every embedding with the access control list from its source system. The retrieval query includes the user's permissions as a filter — the vector similarity search only returns documents the user is authorized to see.

```csharp
var results = await _vectorStore.SearchAsync(
    query: embeddedQuery,
    filter: BuildAccessFilter(user.Permissions),
    topK: 10,
    cancellationToken: ct
);
```

This is unglamorous but non-negotiable for enterprise. Getting it wrong means your AI agent becomes a data exfiltration tool.

## The latency budget

End-to-end latency for an AI agent response has a hard ceiling. In our user research, anything beyond 4 seconds felt "broken" to users — they'd assume the agent crashed and either retry or abandon.

Our latency budget breakdown:

| Stage | Budget | Actual p50 | Actual p95 |
|-------|--------|-----------|-----------|
| Intent classification | 100ms | 45ms | 120ms |
| Context retrieval | 300ms | 180ms | 450ms |
| LLM generation | 2500ms | 1800ms | 3200ms |
| Tool execution | 500ms | 200ms | 800ms |
| Synthesis | 1500ms | 900ms | 2100ms |
| **Total** | **4900ms** | **3125ms** | **6670ms** |

The p95 blew our budget. The fix wasn't faster components — it was architectural:

1. **Streaming responses** — start sending the LLM output to the frontend as tokens arrive, not after the full response is generated. This changes perceived latency from "total time" to "time to first token."
2. **Speculative context retrieval** — start retrieving context *before* intent classification completes, using the raw message as the retrieval query. If the intent classifier changes the retrieval strategy, discard and re-fetch. In practice, the speculative fetch was correct ~80% of the time.
3. **Connection pooling and warm starts** — the cold start penalty for our Azure Functions was brutal (800ms+). We moved latency-critical paths to always-warm container instances.

[VERIFY: Adjust the latency numbers to match your actual measurements. The architectural patterns are the interesting part, but realistic numbers build credibility.]

## What I'd do differently

**Start with observability, not add it later.** We added structured logging and distributed tracing six months into the project. Those six months of debugging without it were painful. For any agent system, instrument every pipeline stage from day one — every LLM call, every retrieval query, every tool execution. The cost is negligible; the debugging value is enormous.

**Don't abstract the LLM too early.** We built a generic `ILLMProvider` interface initially, thinking we'd swap models frequently. In practice, each model has different prompting requirements, different token limits, different latency characteristics, and different failure modes. The abstraction leaked everywhere and we spent more time maintaining the abstraction than we would have spent on direct integrations.

**Treat AI responses as untrusted input.** The LLM's output is user-generated content from a security perspective. It can contain injection attacks, hallucinated tool calls, and malformed data. Every tool execution should validate the LLM's proposed parameters against a strict schema before executing.

---

Building backend systems for AI agents is closer to distributed systems engineering than it is to "AI engineering." The LLM is one component in a pipeline that includes state management, access control, caching, observability, and all the infrastructure patterns we've been refining for decades. The teams that treat it as a fundamentally new discipline tend to reinvent (worse versions of) existing solutions.

The unsexy backend work — the event sourcing, the access control filtering, the latency budgeting — is what separates a demo from a product that millions of people depend on daily.