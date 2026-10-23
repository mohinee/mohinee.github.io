---
title: "Building a secure AI app builder: architecture notes from Genesis"
date: 2026-06-01
reading_time: 8
tags: [ai, agents, security, architecture, multi-agent, fullstack, firebase, vue]
description: "How security shaped the main architectural decisions in Genesis, a multi-agent app builder: the sandboxed preview, a server-side API proxy, authenticated streaming, Firestore rules, and versioned storage."
---

Genesis is an AI app builder. You describe an app in plain language, and a
pipeline of AI agents plans it, designs it, writes the code, tests it, and
reviews it. The workspace has three panels: chat, a Monaco code editor, and a
live preview. The backend is Firebase (Cloud Functions, Firestore, Cloud
Storage), and the frontend is Vue 3.

Users never see the architecture. They see the generated app, the code
streaming into the editor, and the preview. But how components are separated
and what each one is allowed to touch becomes very visible the first time
something goes wrong.

This post covers the security decisions that shaped most of the rest of the
design.

## The starting assumption: generated code is untrusted

Everything follows from one rule: **code an LLM produced is untrusted.**

Stated like that it sounds obvious, but a lot of AI coding tools don't act on
it. They run generated code in the same origin as the app itself, with the
user's session in reach, as if someone had reviewed it.

Genesis assumes the generated code is hostile and builds around that.

## The sandboxed preview

The preview renders the generated app in a sandboxed `<iframe>`:

```html
<iframe
  sandbox="allow-scripts"
  srcdoc="..."
/>
```

The important part is what's missing: `allow-same-origin`. Without it, the
browser gives the iframe a unique opaque origin, whatever URL it was loaded
from. The generated code:

- can't reach into the parent page's DOM
- can't read the parent origin's cookies, `localStorage`, or `sessionStorage`
- can't make requests with the user's session credentials
- can't navigate the top-level page or open popups, because
  `allow-top-navigation` and `allow-popups` are also absent

`allow-scripts` is there because generated apps need JavaScript. Without
`allow-same-origin`, that JavaScript runs isolated from everything the user
cares about.

So if the model produces malicious code, whether from prompt injection, a
compromised response, or an innocent bug that happens to be dangerous, the
damage stays in the frame. The preview can break. It can't take over the
editor or steal tokens.

Two limits worth knowing. The sandbox doesn't block network access: generated
code can still `fetch` any URL it likes, so anything inside the frame (including
the generated code itself) can be sent out. And it can draw a convincing fake
login form inside the preview. A `Content-Security-Policy` with a restrictive
`connect-src` covers the first. Making the preview look unmistakably like a
preview helps with the second.

It's one attribute, and it's one of the most important security decisions in
the project.

## A server-side proxy for third-party APIs

Genesis calls external APIs. The naive approach gives the frontend an API key
and lets it call the service directly. That's wrong for two reasons: the key
ends up in the browser, and you lose any chance to validate or rate-limit
requests on the server.

Instead, the frontend never holds a third-party credential:

1. The user signs in with Firebase Auth. The frontend holds a Firebase ID
   token, which is scoped to this project and expires after an hour.
2. To call an external API, the frontend calls a Cloud Function and passes the
   ID token in the `Authorization` header.
3. The function verifies the token with the Admin SDK (`verifyIdToken`), checks
   that this user may make this request, and calls the external API with a key
   kept in server-side configuration.
4. The function returns the result to the frontend.

The external API key never reaches the client. Even a fully compromised
frontend can't leak it.

## Authenticating the SSE stream

The agent pipeline streams its output over Server-Sent Events, so users watch
code appear in the editor instead of staring at a spinner for 30 seconds.

SSE has an authentication problem: the browser's `EventSource` API doesn't let
you set request headers, so you can't send a `Bearer` token the usual way.

In Genesis, the Firebase ID token goes in a query parameter when the stream
opens. The function validates it before writing a single byte and rejects the
connection if it's invalid or expired.

I originally argued this was acceptable because the token expires within an
hour and HTTPS protects it in transit. That reasoning missed something: Cloud
Functions (2nd gen) run on Cloud Run, and Cloud Run's request logs record the
full request URL, query string included. That means the ID tokens end up in
Cloud Logging. Anyone with log access could replay one until it expired.

There are two better options. One is to swap the ID token for a single-use
stream ticket that expires in seconds, so a logged ticket is useless. The other
is to drop `EventSource` and read the stream with `fetch()` and a
`ReadableStream`, which can send an `Authorization` header like any other
request. Libraries such as `@microsoft/fetch-event-source` handle the SSE
parsing and reconnection for you. I'd now choose the second: no extra endpoint,
and the token never appears in a URL.

## Firestore security rules

Firebase projects often under-invest in security rules. The test-mode rules
that allow all reads and writes until a date 30 days out have ended up in
production more often than anyone would like.

The Genesis rules follow one principle: **users can only read and write their
own data.** No shared collections, no public documents, no reads across users.

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null
        && request.auth.uid == userId;
    }
  }
}
```

Each user's data lives under `users/{userId}/`, and the rule ties the path to
the authenticated UID. User `abc` can reach `users/abc/**` and nothing else.

Be precise about what this protects. Security rules apply to the *client* SDKs.
Cloud Functions use the Admin SDK, which bypasses rules completely. The rules
stop someone who opens devtools and queries Firestore directly with the web
SDK. They do nothing if a Cloud Function has an authorization bug. So every
function that reads user data has to check `uid` against the path itself. The
rules and the function checks protect two different entry points; neither one
backs up the other.

## Versioned Cloud Storage as a history

Generated files live in Cloud Storage, not in Firestore documents. Partly
that's size: a Firestore document is capped at 1 MiB, and a generated app can
exceed that. It also means I can turn on Object Versioning.

With versioning on, overwriting a file keeps the previous version as a
noncurrent object. That gives Genesis two things.

**Recovery.** If a pipeline run produces worse code and overwrites a good
version, the old one is still there. When you iterate on a generated app, the
agents regress working code more often than you'd hope.

**History.** Every version has a timestamp and generation number. To see what
the system produced at a given point, for a bug report or my own debugging, I
read it directly instead of reconstructing it from logs.

The cost is storage. For a single-user tool it's negligible. For a multi-tenant
system you'd add a lifecycle rule, for example keeping the last N noncurrent
versions or deleting them after 30 days.

## Failures stay local

All of these decisions come from the same goal: **each component should be
able to fail without taking others down.**

The preview can crash without affecting chat. The pipeline can fail without
corrupting the editor. The SSE function can time out without losing the
conversation history.

Data moves in one direction: chat → Firestore → editor → Cloud Storage →
preview. Nothing flows back. The preview can't write to the editor, and the
editor can't write to the conversation. Each component reads from a stable
source and writes to one defined destination.

Built this way, failures stay small. The user loses the preview but keeps the
session, or loses a pipeline run but keeps the code.

## What I'd change

**The SSE token,** as described above. That one is a real fix, not a nicety.

**`srcdoc`.** Passing generated HTML through `srcdoc` was the quickest way to
get a working preview, but a `srcdoc` frame without `allow-same-origin` has an
opaque origin, so any generated app that uses `localStorage`, `IndexedDB`, or
cookies throws a `SecurityError` in the preview and fails there even though it
would work once deployed. The more robust setup is the one CodePen and
CodeSandbox use: serve previews from a separate, throwaway domain. The preview
can then have `allow-same-origin` (it's same-origin with nothing that matters)
and storage works, while the isolation from the main app still comes from the
different domain. The cost is another domain to run and a Cloud Storage or
hosting round trip on every refresh.

---

Security in a system that runs AI-generated code isn't something you add once
the core experience works. Untrusted code runs in the browser, third-party
credentials exist, and user data shares a backend, and the model has no idea
about any of it. The architecture has to handle all of that from the start.
