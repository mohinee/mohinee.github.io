---
title: "The three-panel workspace: why UI structure is an architecture decision"
date: 2026-06-14
reading_time: 6
tags: [ai, ux, frontend, architecture, vue, security, design]
description: "The Genesis workspace (chat, code editor, live preview) was designed as three separate failure domains with different trust levels, not three panes onto one app."
---

Most split-pane developer tools treat their panels as a layout question: how
wide each one is, how resizing works, how to theme them consistently. There's
one application underneath, and the panes are windows onto it.

The Genesis workspace has three panels: chat, a Monaco code editor, and a live
preview. I designed them as three separate failure domains that happen to share
a screen, not as three views of one app.

That choice set the security model, the error handling, and what happens when
any one part breaks.

## The three panels

**Chat** is where the user talks to the agent pipeline. They describe what they
want, and the agents reply with questions, status updates, and progress. This
is the control surface. It's fully trusted and runs in the main Vue app.

**Code editor** is a Monaco instance showing the current generated files. The
user can read and edit the code before it runs. I treat it as semi-trusted: the
content came from a model and may be wrong, but it's only text. Nothing in the
editor executes.

**Live preview** is an iframe running the generated app. It's **explicitly
untrusted** and isolated accordingly.

## The preview sandbox

The preview iframe uses `sandbox="allow-scripts"` without `allow-same-origin`.
Scripts can run, so the generated app works, but the frame gets an opaque
origin, so it can't touch the parent page's DOM, cookies, storage, or the
user's session. I go through the details, and the gaps the sandbox doesn't
cover, in [the Genesis security post]({% post_url 2026-06-01-building-secure-ai-app-builder %}).

What matters for this post is the result: whatever the generated code does, it
can't reach into the chat or the editor. If it crashes, hangs, or turns out to
be malicious, the damage stays inside the preview.

## Separate failure domains

The broader principle is that each panel should be able to fail on its own.

If the preview crashes (an unhandled exception, an infinite loop, a blank
render), chat and the editor keep working. The user can read the error, fix the
code, and regenerate the preview without losing the conversation or the
pipeline state.

If the agent pipeline fails or stops responding (a timeout, a rate limit, an
instruction it can't resolve), the editor still shows the last good output and
the user can still edit it.

If the editor loses state (a dropped connection, a sync failure with
Firestore), the conversation can be recovered separately, because it lives in
its own Firestore collection and isn't derived from the editor.

Ships use the same idea with bulkheads: if one compartment floods, the others
stay dry. The goal is to contain failures, not only to prevent them.

## How the panels communicate

Separate failure domains still need to coordinate. In Genesis the panels never
hold references to each other. They communicate through defined channels.

**Chat to editor.** Chat doesn't modify the editor's content. Pipeline output is
written to Firestore, and the editor subscribes to those documents. If a write
fails, the editor doesn't receive a partial update. It just doesn't change until
a write succeeds.

**Editor to preview.** The editor doesn't push into the iframe. When the user
asks for a preview, the current files are serialized, sent through a Cloud
Function that checks they're well-formed, and rendered into the iframe. That
check means malformed output produces a clear error instead of an unpredictable
crash.

**Preview to anything.** Nothing. The preview has no channel back to chat or
the editor, on purpose. It's output only. If the user wants to change something
they see in the preview, they say so in chat.

Data flows chat → editor → preview and never backwards. That's what makes the
isolation easy to reason about: for any component, you can list exactly what
it's able to affect.

## Why Monaco, and what it costs

The editor is Monaco, the editor component from VS Code. I chose it for its
text model. Monaco keeps the document in its own model, separate from the DOM,
applies changes as edits to that model, and only renders the lines currently on
screen. When the pipeline streams several hundred lines into a file, Monaco
applies them incrementally instead of re-rendering the whole document. For
large files and fast streaming, that's the difference between smooth and janky.

The cost is size. Monaco is several megabytes of JavaScript plus web workers
for language services, so it's worth loading lazily. It's worth it for Genesis because
the editor is a main surface, open for most of the session. If reading code were
occasional, something lighter like CodeMirror 6 would be the better choice.

## Streaming into an editor the user can also type in

The pipeline streams over Server-Sent Events, so as the Developer agent writes
code, it shows up in the editor live instead of all at once at the end.

That raises a question: what happens when the stream and the user edit the same
file at the same time?

Genesis avoids the question with a simple rule. While a pipeline run is active,
the editor is read-only and only the stream writes to it. When the run
finishes, the editor becomes editable again, and the user can edit the code or
steer the next run from chat.

That isn't clever conflict resolution. It removes the conflict. When there's no
good way to resolve a problem, sometimes the right move is to make sure it
can't happen.

## A typical session

1. The user types a feature request in chat.
2. The Planner and Architect run, and chat shows their progress.
3. The Developer runs, and code streams into the editor, which is read-only for now.
4. The Developer finishes, and the Tester and Reviewer run in the background.
5. If the Reviewer approves, the editor becomes editable and the preview
   refreshes with the new code.
6. If the Reviewer blocks, chat lists the critical issues, the editor is
   editable, and the user can change the code and run the pipeline again.

A failure at any step stays in its panel. If the preview crashes at step 5,
the chat notification from step 6 still arrives. If chat loses its SSE
connection during step 2, the pipeline state is fine, because the agents write
to Firestore, not to the stream.

## The principle

Splitting a UI into panels is easy. Making those panels separate failure
domains, with defined channels between them, one-way data flow, and trust
levels enforced at each boundary, means treating the UI as an architecture
problem rather than a layout problem.

The sandbox setting on the preview is one attribute. The reason for it, that
AI-generated code is untrusted and has to be contained, shapes everything else.

The most important UI decisions in Genesis weren't about fonts, colors, or
breakpoints. They were about where the trust boundaries go and what happens
when something fails.
