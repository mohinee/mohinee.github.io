---
layout: post
title: "Designing Resumable Large-File Uploads in the Browser"
date: 2026-08-28
tags: [frontend, architecture, javascript, system-design, reliability]
description: "Uploading a 5 GB file is not a bigger version of uploading a 5 MB file. Chunking, per-chunk checksums, bounded concurrency, resume, and abort, and the bug I introduced in my own first draft."
---

A single `fetch` with a `FormData` body is fine for a profile picture. It is the wrong tool for a 5 GB video. One dropped connection at 98% means starting over, the browser may try to hold the whole thing in memory, and the user has no way to pause or cancel meaningfully.

The design that fixes this is well known: split the file into chunks and upload them independently. The interesting part is everything that has to be true for that to actually work.

## The four constraints

1. **Memory must be bounded**, regardless of file size.
2. **Failure must be local.** A failed chunk retries alone. It does not restart the upload.
3. **The upload must be resumable**, including after a page reload.
4. **The user must be able to cancel**, and cancel must actually stop work and clean up.

Everything below serves one of these.

## Never read the whole file

A `File` is a `Blob`, and blobs are lazy. `file.slice(start, end)` returns a new blob without copying anything. Only when you call `.arrayBuffer()` on the slice do those bytes enter memory.

```js
const MiB = 1024 * 1024;

export function planChunks(size, chunkSize = 8 * MiB) {
  const chunks = [];
  for (let i = 0, start = 0; start < size; i++, start += chunkSize) {
    chunks.push({ index: i, start, end: Math.min(start + chunkSize, size) });
  }
  return chunks;
}
```

The plan is just numbers. No data is touched until a worker is ready to send a chunk. The anti-pattern to avoid is `await file.arrayBuffer()` "to make things simpler," which will hold the entire file in memory at once.

**Choosing a chunk size** is a tradeoff. Small chunks make retries cheap but multiply request overhead and the number of parts you track. Large chunks are efficient but make each failure expensive. If your storage backend is S3-style multipart, it constrains you: parts must be at least 5 MiB (except the last), an upload can have at most 10,000 parts, and an object can be at most 5 TiB. That means a fixed 5 MiB chunk cannot exceed about 48 GiB, so very large files need a larger chunk size computed from the file size.

## Integrity per chunk

`SubtleCrypto.digest` hashes a buffer in one call. It has no streaming interface. That is actually convenient here, because hashing one bounded chunk at a time is exactly what keeps memory flat.

```js
export async function sha256Hex(buf) {
  const d = await crypto.subtle.digest("SHA-256", buf);
  return [...new Uint8Array(d)].map(b => b.toString(16).padStart(2, "0")).join("");
}
```

Send the digest with each chunk, and have the server recompute and reject on mismatch. This catches corruption that TCP's checksum can miss and gives you a precise error: "chunk 41 failed verification," not "the file is broken."

A whole-file hash is a separate problem. Since the Web Crypto API cannot hash incrementally, you either use a JS library that supports incremental hashing, or let the server compute the final hash by combining chunk results. Decide up front which one your integrity guarantee depends on.

## Bounded concurrency is a memory limit in disguise

Uploading one chunk at a time wastes bandwidth. Uploading all of them at once defeats the point. A worker pool with a fixed size does both jobs: it keeps the pipe busy and it puts a ceiling on how many chunks are ever in memory.

```js
export async function uploadAll(file, send, { concurrency = 3, signal, alreadyDone = new Set() } = {}) {
  const queue = planChunks(file.size).filter(c => !alreadyDone.has(c.index));
  const ctl = new AbortController();
  signal?.addEventListener("abort", () => ctl.abort(signal.reason), { once: true });

  async function worker() {
    while (queue.length && !ctl.signal.aborted) {
      const chunk = queue.shift();
      try { await uploadChunk(file, chunk, send, ctl.signal); }
      catch (e) { ctl.abort(e); throw e; }   // first hard failure stops the siblings
    }
  }

  await Promise.all(Array.from({ length: concurrency }, worker));
  ctl.signal.throwIfAborted();               // a user abort must reject, not resolve quietly
}
```

With three workers and 8 MiB chunks, peak chunk memory is about 24 MiB whether the file is 100 MB or 100 GB. In a simulated run the pool never exceeded its cap of 3 in-flight chunks.

Why three? Browsers limit concurrent connections per host on HTTP/1.1 (typically six), and other requests on the page share that budget. HTTP/2 multiplexes, so the limit is softer. A fixed small number is a sensible default. If you want to go further, adjust concurrency up while throughput improves and errors stay low, and back off when either turns.

## Retries: know which failures are worth retrying

```js
const backoff = attempt => Math.random() * Math.min(10_000, 500 * 2 ** attempt);  // full jitter

async function uploadChunk(file, chunk, send, signal, maxAttempts = 4) {
  for (let attempt = 0; ; attempt++) {
    try {
      const buf = await file.slice(chunk.start, chunk.end).arrayBuffer();
      const checksum = await sha256Hex(buf);
      return await send({ ...chunk, body: buf, checksum, signal });
    } catch (e) {
      if (signal.aborted || !e.retryable || attempt + 1 >= maxAttempts) throw e;
      await sleep(backoff(attempt), signal);
    }
  }
}
```

- **Retry** network errors, timeouts, `5xx`, and `429`. Honor `Retry-After` if the server sends it.
- **Do not retry** `4xx` responses such as `403` and `413`. They will fail the same way every time, and retrying wastes the user's time.
- **Use jitter.** Without it, every client that failed together retries together, which is how a brief blip becomes a sustained outage.
- **Re-read the slice on each attempt.** Holding the buffer across retries keeps memory pinned during the backoff sleep.
- If the chunk URL is pre-signed, it can expire mid-upload. A `403` from an expired signature is one of the few `4xx` cases where you refresh credentials and try again.

## Resume

Resumability needs a server-side record of which chunks arrived, and a chunk endpoint that is **idempotent**: uploading chunk 7 twice must be harmless.

On resume, the client asks the server for the set of completed chunk indices, then feeds that set into `alreadyDone`. In a simulated run with chunks 0 to 3 already stored, only chunks 4, 5, and 6 were sent.

Two practical points:

- The upload needs a stable identity. Derive it from something like file name, size, and last-modified time, or issue a server-side upload ID and keep it in `localStorage`. If the user picks a different file, that must not resume the old upload.
- Trust the server's list, not the client's memory. The client may believe chunk 5 succeeded when the response never arrived.

## Cancel means cleanup

`AbortController` is how you make cancel real. Pass the signal into `fetch` so in-flight requests are actually torn down, and into the backoff sleep so a cancelled upload is not sitting in a timer.

Cancel also has a server half. Abandoned partial uploads consume storage forever unless something removes them. Have the client send an explicit abort (for S3-style multipart, the AbortMultipartUpload call), and back that up with a lifecycle rule that expires incomplete uploads after some number of days, because the client is not always around to clean up: tabs close, phones die.

## A bug from my own first draft

My first version of the pool looked correct and passed a happy-path test. When I simulated a user cancelling mid-upload, `uploadAll` **resolved successfully** with only some chunks sent.

The reason: on abort, each worker's `while` condition became false and the loop simply ended. `Promise.all` saw workers finish without error and returned normally. From the caller's perspective, a cancelled upload looked like a finished one. The fix is the single `throwIfAborted()` line at the end of the function above.

The general lesson is that "the loop stopped" and "the work completed" are different facts. Any code that stops early on a signal has to tell its caller which of the two happened.

## UI state is not transport state

One last design point that pays off later. Keep two models:

- **Transport state:** chunk indices, retry counts, checksums, upload IDs, signed URLs. This is internal and changes constantly.
- **UI state:** `queued`, `hashing`, `uploading`, `paused`, `completing`, `done`, `failed`, plus a single progress number.

The UI subscribes to the second model only. Otherwise every chunk completion triggers renders across the screen, and any change to your chunking strategy becomes a change to your components.

## Checklist

- Slice lazily and never read the whole file
- Chunk size chosen against the backend's limits
- Per-chunk SHA-256, verified server-side
- Fixed-size worker pool, which is also your memory ceiling
- Retry only retryable errors, with backoff and jitter
- Idempotent chunk endpoint and a server-side "what do you have?" call for resume
- `AbortController` end to end, plus server-side cleanup
- Aborts reject; they do not resolve
- Separate UI state from transport state
