---
title: "A pragmatic look at the Model Context Protocol"
date: 2026-04-19
reading_time: 6
description: "MCP gets talked about as the answer to every LLM-tool integration. After spending some time with it, I think it's a useful protocol with a narrow sweet spot, not something every project needs."
tags: [ai, llm, mcp, architecture]
---

The Model Context Protocol (MCP) gets described as *the* standard for connecting
LLMs to tools and data. Anthropic published it in November 2024, OpenAI, Google
and Microsoft added support during 2025, and the comparison to LSP (the Language
Server Protocol that let every editor talk to every language tool) gets made in
almost every conversation about it.

I spent a few evenings experimenting with it because I wanted an
opinion that didn't come from a launch post. This is where I ended up. Parts of
it may be wrong by the time you read this; the spec has had several dated
revisions already.

## What MCP actually is

Take away the framing and MCP is JSON-RPC 2.0 between a *host* (an IDE, a chat
app, an agent runtime) and a *server* that exposes capabilities. The model
never talks to the server directly. It lives inside the host, and the host turns
the model's tool calls into requests to the server and passes the results back.

There are two standard transports: stdio for servers running locally as a
subprocess, and Streamable HTTP for remote ones (it replaced the earlier
HTTP+SSE transport in the March 2025 revision). There's an `initialize`
handshake where both sides declare capabilities, and three main server
primitives:

- **Tools**: functions the model can call (`tools/list`, `tools/call`)
- **Resources**: data the host can read into context (`resources/list`, `resources/read`)
- **Prompts**: reusable templates the user can pick (`prompts/list`, `prompts/get`)

That's most of it. It's plumbing, and it's decent plumbing. Separating tools
(the model *does* something) from resources (the model *reads* something)
matches how models actually use external systems.

## Where MCP is worth it

I see two cases where it clearly pays for itself.

**Tools that should work in many hosts.** If you want the same tool to work in
Claude Desktop, VS Code, Cursor, and some agent framework, MCP means one server
instead of four integrations. This is where the LSP comparison holds up.

**Internal tooling at a company with several AI products.** If three LLM-powered
products all need to reach the same internal systems, defining those tools once
as MCP servers beats rebuilding the integration in each product.

In both cases the extra layer is paid for by reuse. That's the test I'd apply
before adding it.

## Where I think it's overused

**One app, one model, a handful of tools.** If your app calls one provider and
has five tools nobody else will use, MCP is overhead. The provider's native
tool calling (a JSON schema per tool in the request) is simpler, has one fewer
process to deploy, and has one fewer thing to debug. You'd be paying for
interoperability you won't use.

**Latency-sensitive paths.** A remote MCP server is an extra network hop per
tool call. In a chat agent you won't notice it. In an agent loop that makes
eight sequential tool calls behind an interactive UI, it adds up. Measure it
before you commit.

**As a way to avoid designing the integration.** This is the one I see most.
"Just expose it as an MCP server" gets used to skip the harder questions: what
should the model be allowed to do, with which arguments, with what error
behavior, and under whose permissions? MCP gives you a place to write the
answers down. It doesn't answer them for you.

## Auth

This is the part I'd watch most closely. The spec now defines authorization
for HTTP transports on top of OAuth 2.1, and the June 2025 revision made remote
servers OAuth resource servers with discoverable metadata (RFC 9728). That
helps a lot. In practice:

- **Local stdio servers** don't use any of it. They usually read an API key from
  an environment variable, and the security boundary is your machine and
  whatever that key can do.
- **Remote servers** vary in how fully they implement the spec, and
  enterprise identity providers don't always support the flows it assumes, such
  as dynamic client registration.

If you're building an enterprise integration, plan real time for auth. It will
take longer than the tools themselves.

## What surprised me

I expected tool calling to be the interesting part. It turned out to be
discovery. A host can ask a server what it offers (`tools/list`), show that to
the user, and let them turn individual tools on and off. The server can also
send a `notifications/tools/list_changed` notification when its tool set
changes. Most provider tool-calling APIs assume the application already knows
its full tool list at build time. MCP lets tools be connected at runtime, and
lets the user choose them.

That makes something possible I hadn't thought much about before: users
assembling their own agent's capabilities, server by server. Strictly speaking
that's a host feature, not part of the protocol, but MCP is what makes it
practical.

## What I'd do if I were building agent features today

- **A single product:** start with the provider's native tool calling. Add MCP
  if and when a second host or product needs the same tools.
- **Infrastructure other teams consume:** reach for MCP early. The interop
  pays off once you have more than one consumer.
- **Anything that connects to user-supplied tools or third-party services:**
  start with MCP. Runtime discovery is the main reason to use it.

I'd push back on "MCP is the standard, so use MCP." A standard you have no use
case for is just more code to maintain. The useful question is whether you
have the many-hosts or many-tools problem this protocol solves. If you don't,
you're paying for a layer you don't need.

## Will it last?

LSP worked because it solved an obvious N×M problem (N editors, M language
tools), it had workable governance, and the alternative of every editor
integrating every language natively was clearly worse.

MCP has the same N×M shape (N hosts, M tool providers), and on adoption it has
already won: every major model provider and most AI-enabled IDEs support it,
and Anthropic moved it to a Linux Foundation project at the end of 2025, which
answers some of the governance question. What I'm less sure about is the
*shape*. The spec is still changing in meaningful ways, especially around auth
and long-running operations, and servers written against the November 2024
version already needed updates.

What I'd bet on is that a structured, host-independent way for models to
discover and call tools is here to stay. Whether it ends up looking much like
today's MCP is a separate question.

---

*To be clear about my experience: I've experimented with MCP, not shipped a
production system on it. Some of the rough edges above may be worse in
practice than I saw, or already fixed. If you've run MCP in production, I'd
welcome corrections.*
