---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, queues, async, decoupling, arc2, module7]
related: ["[[backpressure-and-capacity]]", "[[utilization-latency-cliff]]", "[[idempotency-keys]]", "[[off-serving-path-failures]]", "[[setting-timeouts]]"]
---

# Queues & async as reliability tools

Shift from **call-and-wait** (sync, A's fate tied to B's) to **send-and-forget** (async, A returns immediately): `A → [queue] → B`.

A queue = a **buffer that decouples producer from consumer in time.** That temporal decoupling delivers 4 benefits:
1. **Independent failure** — B can be fully down; messages wait until B recovers. "Both up at once" → "fail independently."
2. **Burst absorption** — 50× spike fills the queue, not B; B drains at steady rate → keeps B off the [[utilization-latency-cliff]]. (Generalized upload-pipeline fix; a *third* burst defense beyond headroom/autoscaling.)
3. **Load leveling** — size B for the *average*, not the peak.
4. **Independent scaling** — add consumers to drain faster.

## The cost (the half juniors skip) — all from "producer stopped waiting"

- **A doesn't know if B succeeded** — A got "queued", not "done."
- **Eventual consistency** — correct, but not *yet*. UX must show "processing…" honestly.
- **Ordering not guaranteed** (parallel/redelivered consumers).
- **At-least-once delivery → duplicates** → needs idempotent consumer ([[idempotency-keys]]). Module 5 is the *prerequisite*: at-least-once + idempotent = exactly-once effect.

## The deciding question (every sync-vs-async choice)

**Does the caller need the result to continue?**
Yes → **sync** (+ timeout/breaker/bulkhead). No → **async** (+ idempotent consumer + DLQ + pending UX).

Apply it to **each sub-operation separately**, not the request as a lump:
- login password check → sync (can't be logged in "eventually")
- monthly invoice PDF+email → async (no one waiting)
- photo upload: **accept = sync** (user needs confirmation), **resize/transcode = async** (downstream work). YouTube: upload "done" in seconds, 4K transcode minutes later. *(Ray initially called resize sync by lumping accept+process — the M3 split, new costume.)*
- ticket inventory decrement → **sync + atomic** (correctness-critical & contended; async opens an oversell window)

## Adopting a queue is a BUNDLE, not just a queue

queue + idempotent consumer + **dead-letter queue** + honest pending UX.

**Dead-letter queue (DLQ):** after N failed attempts, move the message to a separate queue → inspect/alert/manual-retry. Without it: a **poison message** retries forever (blocks the queue) or is silently dropped (user never emailed, nobody knows). Async work is now [[off-serving-path-failures]] — failures rot silently unless you build explicit visibility (DLQ + alerting).

Lost in the sync→async move: the **synchronous failure signal**. Failures now happen out-of-band; someone must deliberately notice.
