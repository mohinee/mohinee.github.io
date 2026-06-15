---
layout: post
title: "Building a Secure AI App Builder: Architecture Notes from Genesis"
date: 2026-06-01
tags: [ai, agents, security, architecture, multi-agent, fullstack, firebase, vue]
description: "What I built, why I built it the way I did, and how security shaped every major architectural decision in Genesis — a multi-agent AI-powered app builder."
---

There's a category of engineering problems where the architecture *is* the product. The user never sees the architecture directly. They see the output — the generated app, the streamed code, the live preview. But every decision about how components are separated, how agents communicate, how trust is established between layers: that all becomes visible the moment something goes wrong, and you either had it right or you didn't.

Genesis is an AI-powered app builder. You describe an app in plain language. A pipeline of AI agents figures out what to build, designs it, writes the code, tests it, and reviews it. A three-panel workspace gives you a chat interface, a Monaco code editor, and a live preview — all at once. The backend runs on Firebase with Cloud Functions, Firestore, and Cloud Storage. The frontend is Vue 3.

Here's what I learned building it, with a specific focus on the security decisions that quietly shaped everything.

---

## The Core Problem: AI-Generated Code Is Untrusted Code

Before agents or UI panels, there's one premise that governs the entire system:

**Code produced by an LLM is untrusted. Always. Without exception.**

This is obvious when stated plainly, but most AI-assisted development tools handle it poorly in practice. They render generated code in the same execution context as the rest of the application. They pass user tokens into the same scope. They treat the model's output as if it went through a review process it didn't.

Genesis starts from the opposite assumption: the generated code is hostile until proven otherwise. The architecture is built around that.

---

## The Sandboxed Preview Panel

The live preview renders generated applications in an `<iframe>` with a carefully chosen sandbox configuration:

```html
<iframe
  sandbox="allow-scripts"
  srcdoc="..."
/>
```

The key decision here is what's *absent*: `allow-same-origin`.

When `allow-same-origin` is omitted, the iframe is treated as a cross-origin document regardless of where it's actually hosted. This means the generated application code:

- Cannot access the parent window's DOM
- Cannot read cookies set on the parent origin
- Cannot access `localStorage` or `sessionStorage` from the parent context
- Cannot make credentialed requests using the user's existing session

`allow-scripts` is present because generated applications need JavaScript to run. But without `allow-same-origin`, that JavaScript executes in an isolated sandbox with no access to anything the user cares about.

If a model produces malicious JavaScript — through a prompt injection attack, a compromised response, or a subtle coding error that happens to have dangerous behavior — the damage is contained. The preview can break. It cannot escalate into the parent application or steal credentials.

This is one line of HTML. It's also one of the most important security decisions in the entire system.

---

## The OAuth Proxy Pattern

Genesis integrates with external APIs. The naive implementation of this passes API tokens to the frontend, which then calls the external API directly. This is wrong for two reasons: the token is exposed to the browser (and therefore to the untrusted iframe context), and it bypasses any server-side validation or rate limiting you might want to enforce.

The correct pattern: the frontend never holds the token.

Instead:

1. The user authenticates via Firebase Auth. The frontend holds a Firebase ID token, which is scoped to your application and expires in one hour.
2. When the frontend needs to call an external API, it calls a Cloud Function endpoint, passing the Firebase ID token in the `Authorization` header.
3. The Cloud Function validates the ID token using the Firebase Admin SDK, confirms the user is authorized to make this request, and then makes the external API call using the API key that lives in server-side environment variables.
4. The result is returned to the frontend.

The API key for external services never touches the client. The Cloud Function is the only entity that knows it. The frontend can be compromised entirely and the external API key remains protected.

---

## SSE Streaming and Authentication

The agent pipeline streams its output via Server-Sent Events. Streaming is the right choice for long-running agent passes — you want the user to see code appearing in the editor in real time, not stare at a spinner for 30 seconds.

SSE introduces a specific authentication problem: unlike standard fetch requests, SSE connections are established with the `EventSource` API, which doesn't support custom request headers. You can't pass a `Bearer` token the normal way.

The solution in Genesis: the Firebase ID token is passed as a URL query parameter when establishing the SSE connection. The Cloud Function that serves the SSE stream validates the token on connection before opening the stream. If the token is invalid or expired, the connection is rejected immediately.

This means the token is briefly visible in the URL, which isn't ideal. The mitigation: the SSE endpoint URL is never logged, the token expires in one hour, and HTTPS ensures it's not visible in transit. For a production system with higher security requirements, the better solution is to exchange the ID token for a short-lived SSE-specific token before establishing the connection — but that adds latency and complexity that wasn't warranted for this system.

---

## Firestore Security Rules

Firebase applications often underinvest in Firestore security rules. The default rule that ships in development mode — allow all reads and writes — has made it to production in more projects than anyone would like to admit.

Genesis's Firestore security model is organized around one principle: **users can only read and write their own data**. No shared collections, no public documents, no cross-user reads.

The baseline rule:

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

Every user's data lives under `users/{userId}/`. The rule binds the authenticated UID to the path parameter. A user with UID `abc` can only reach `users/abc/**`. Attempting to read `users/xyz/**` returns a permission denial at the Firestore layer, before the Cloud Function is ever involved.

This is defense in depth: the Cloud Function also enforces user-scoped access, but the Firestore rule means a bug in the Cloud Function authorization logic doesn't automatically become a data breach.

---

## Cloud Storage Versioning as Audit Trail

Generated files — the code produced by the agent pipeline — are stored in Cloud Storage rather than as Firestore documents. This is partly a size decision (Firestore documents have a 1MB limit; generated applications can exceed that), but it also enables something valuable: versioning.

Cloud Storage versioning keeps every historical version of each file. This serves two purposes in Genesis:

**Recovery.** If a pipeline pass produces bad output and overwrites a good version, the previous version is retrievable. This is a practical safeguard when you're iterating on a generated application — the agent might regress something that was working.

**Audit.** Every version has a timestamp and metadata. If a user wants to understand what the system produced at a particular point in time — or if you need to investigate a support issue — the history is there. You're not reconstructing it from logs; you're reading it directly.

The cost is storage. For a single-user development tool, this is negligible. For a multi-tenant system at scale, you'd want a retention policy.

---

## The Failure Isolation Principle

All of these decisions connect to a single architectural stance: **components should fail independently**.

The preview iframe can crash without affecting the chat panel. The agent pipeline can error out without corrupting the code editor state. The Cloud Function handling SSE can time out without losing the conversation history.

This is enforced by the directional data flow: chat → Firestore → editor → Cloud Storage → preview. Nothing flows backward. The preview has no write path into the editor. The editor has no write path into the conversation. Each component reads from a stable source and writes to a well-defined destination.

When you build it this way, failures are local. The user loses the preview but not the session. They lose the pipeline run but not the code. The system degrades gracefully rather than failing completely.

---

## What I'd Change

One decision I'd revisit: using `srcdoc` to pass generated HTML directly into the iframe. For large applications, `srcdoc` can hit browser limits on attribute length. A more robust approach is to write the generated output to a Cloud Storage object, generate a signed URL scoped to the user, and set the iframe `src` to that URL — still sandboxed, but not limited by attribute size. The tradeoff is a roundtrip to Cloud Storage on every preview refresh.

I'd also implement the short-lived SSE token exchange I mentioned above rather than putting the Firebase ID token in the URL parameter. It's the cleaner solution; I just didn't do it.

---

Security in an AI-generated code system is not a feature you add after the core experience works. It's a constraint you design around from the beginning, because the attack surface — untrusted code executing in the browser, external API credentials, user data in a shared backend — is real, and the LLM doesn't know or care about it. The architecture has to.

---

*All architectural decisions described here are from my own project work. Nothing is specific to any employer's internal systems.*
