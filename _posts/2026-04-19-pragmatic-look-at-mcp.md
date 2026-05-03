---
title: "A pragmatic look at the Model Context Protocol"
date: 2026-04-19
excerpt: "MCP shows up in a lot of conversations as a kind of universal solvent for LLM-tool integration. After spending real time with it, I think it's better understood as a useful protocol with a narrow sweet spot — not a standard everyone needs to adopt."
reading_time: 8
---

The Model Context Protocol — MCP — has been getting attention as the standard
for connecting LLMs to tools and data. Anthropic published it, the ecosystem
has grown quickly, and the analogy to LSP (the Language Server Protocol that
made every editor speak the same language to every linter) has been doing a lot
of rhetorical work.

I spent a few evenings poking at it because I wanted to form an opinion that
wasn't downstream of a launch announcement. This is what I came away with.
None of it is hot takes; some of it might be wrong as the protocol evolves.

## What MCP actually is

Strip away the framing and MCP is a JSON-RPC protocol that defines how a host
(say, an IDE or a chat app) talks to a server (which exposes tools, resources,
or prompts) on behalf of an LLM. The client doesn't call the LLM. The LLM
sits inside the host, and the host translates between the model's requests and
the server's responses.

That's it. There's a transport layer, a capability negotiation, a few
primitives — `tools`, `resources`, `prompts` — and conventions for how each
behaves. It's not magic. It's plumbing.

The plumbing is well-designed. The schema is reasonable. The decision to model
*both* tool-calls (model takes an action) and resources (model reads context)
captures something real about how LLMs actually use external systems.

## Where MCP earns its keep

Two contexts where I think MCP is genuinely valuable:

**Tool ecosystems with many hosts.** If you're building tools you want to be
usable across IDEs, chat apps, and agent frameworks, MCP saves you from writing
six different integrations. This is the LSP analogy and it holds. The work
that's gone into editor support is real.

**Internal tooling at companies with many AI surfaces.** If your company has
three different LLM-powered products and they all need to reach the same set of
internal tools, defining those tools once via MCP is genuinely better than
redoing the integration per product. The auth story still has rough edges, but
the protocol gives you a sane structure.

In both of these, MCP is paying off the cost of an extra abstraction with real
reuse. That's the test I'd apply.

## Where I think people overuse it

**Single-product, single-LLM apps.** If you have one app calling one model and
five tools that only that app uses, MCP is overhead. Native tool-calling APIs
from the model providers are simpler, faster, and don't require running an MCP
server. You'd be paying for interop you'll never use.

**Anywhere latency matters more than flexibility.** MCP adds a hop. For a chat
agent, that's invisible. For something embedded in a tight UX loop, the
hop becomes part of your perceived latency budget. Worth measuring before
committing.

**As a substitute for thinking about the integration.** This is the one I see
most. "Just expose it as an MCP server" gets used as a way to avoid the harder
question of *what should the model actually be allowed to do, with what
arguments, with what error semantics, with what auth scope?* MCP doesn't answer
those — it gives you a place to put the answer. The questions are still yours.

## The auth story, briefly

MCP's auth story is the area I'd watch most closely if you're building on it.
The protocol allows for it but doesn't prescribe it tightly, which means
implementers have made different choices. For local MCP servers running on
the user's machine — fine, the security boundary is the OS. For remote MCP
servers, you're inheriting whatever auth model the server author chose, and
that varies. If you're building an enterprise integration, you'll spend real
time on this.

## What surprised me

The thing I didn't expect, going in, was how much of MCP's value is in the
*discoverability* primitives, not the tool-calling primitives. The
`list_tools` / `list_resources` flow — where a host can ask a server "what
can you do?" and present that to the user — is the part that feels genuinely
new. Most tool-calling APIs assume the host already knows what's available.
MCP says: tools can be discovered and connected on the fly, and the user can
choose.

This unlocks a UX pattern I hadn't fully appreciated: the user composing their
own agent capability surface. That's downstream of MCP, not strictly part of
it, but it's where the protocol gets interesting.

## What I'd do if I were building agentic features today

Honest answer:

- If I were building a *single product*, I'd start with the model provider's
  native tool-calling and add MCP only if I outgrew it.
- If I were building *infrastructure* — something other teams or products
  consume — I'd reach for MCP earlier, because the interop pays off across
  consumers.
- If I were building *anything that connects to user-supplied tools or
  third-party services*, MCP is the right starting point. The discoverability
  primitives are the unlock.

The framing I'd push back on is "MCP is the standard, so use MCP." Standards
without a use case are just code. The right question is: do you have the
multi-host or multi-tool problem this protocol solves? If yes, great. If not,
you might be paying for a layer you don't need.

## Will it become The Standard?

I don't know. LSP succeeded because it solved a clear N×M problem (N editors,
M language tools), it had reasonable governance, and the alternative — every
editor implementing every linter natively — was visibly worse than the
abstraction.

MCP has the same shape of problem (N hosts, M tool ecosystems) but the field
is moving faster, the competing approaches are evolving alongside it, and the
LLM space tends to consolidate around whatever the leading model providers
support. I think MCP has real momentum. I'd be surprised if some version of
it didn't end up as the de facto interface for a chunk of agentic tooling.
But "this exact protocol, in this exact form, by this time next year" is a
forecast I wouldn't bet on.

What I'd bet on: the *idea* of a structured, host-agnostic way for LLMs to
discover and call tools is here to stay, regardless of which protocol wins.
And that's actually the more important thing.

---

*Caveat I should add at the bottom: I've poked at MCP, not shipped a
production system on it. Some of the rough edges I'm describing might be more
serious in practice than I've experienced, or might already be fixed by the
time you read this. If you've actually shipped MCP-based tooling, I'd value
the corrections.*
