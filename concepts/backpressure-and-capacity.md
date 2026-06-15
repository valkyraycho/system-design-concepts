---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, load, backpressure, capacity, arc1, module3]
related: ["[[littles-law]]", "[[utilization-latency-cliff]]", "[[load-shedding]]", "[[metastable-failure]]", "[[setting-timeouts]]"]
---

# Backpressure & capacity planning

## Backpressure: the system saying "slow down"

An overwhelmed component **pushes a signal back upstream** ("ease off"), and the upstream *actually slows down* — instead of silently accepting work it can't handle and collapsing (Little's Law → OOM). The active, upstream counterpart to [[load-shedding]] (which is the blunt last-ditch reject-at-the-door).

From plumbing: water in faster than it drains → pressure builds backward. Software backpressure makes that pressure **visible and actionable** instead of silent queue growth.

## Bounded vs unbounded queue — THE decision

- **Unbounded queue** *feels* safe ("never reject!") but is a trap: it doesn't prevent overload, it **hides** it until OOM — converting a recoverable "slow down" into an unrecoverable crash. Queue depth = L; unbounded queue = unbounded L = guaranteed exhaustion.
- **Bounded queue is a feature**: its limit is exactly the mechanism that *generates* the backpressure signal (full → producer blocks/gets rejected → slows).

## Where the pressure goes

It propagates to the only place that can absorb it — the **client** (the source of demand): DB writer slow → its bounded queue fills → feeding service blocks → its pool fills (Little's Law) → LB sees unavailable → 503 + `Retry-After` → client backs off. Everything between just *propagates*; backpressure terminates in load shedding + client backoff. (TCP flow control = textbook backpressure: slow receiver shrinks its window, fast sender's OS throttles automatically.)

## The denial trap (Ray, 2026-06-13)

A friendlier error that **still accepts the work is NOT backpressure — it's denial.** Backpressure must change *behavior*, not just *presentation*. Test: "if the spike continued forever, would memory stay bounded?" A spinner that still enqueues fails (memory grows); a spinner that holds the user until capacity exists passes.

## Upload pipeline pattern (the fix)

Naive bug: one in-memory queue for *both* "accept" and "process." Separate them:
1. **Accept bytes to cheap deep storage (S3) immediately** — queue only a *reference*, never the payload (Ray's "items are large" intuition, applied as cure). RAM no longer holds images.
2. **Bound the work queue**; full → honest "accepted, queued for processing" (image is safe in storage).
3. Async decouple acceptance (cheap) from processing (expensive) — fast ack, slow work behind a buffer.
4. If even acceptance outpaces capacity → shed gracefully (503 + Retry-After). Be generous on acceptance, strict on processing.

## Capacity planning (Little's Law)

Workers needed = L = λW. Ray's example: 20 uploads/s × 2s = **40 workers** to *barely* keep up — but "barely" = 100% utilization = on the cliff ([[utilization-latency-cliff]]).
- Target ~75% utilization: 40 / 0.75 ≈ **53 workers**. Headroom is also your redundancy (lose 5 to a deploy → still above the 40 floor).
- "Peak 20/s" is itself bursty → more reason for headroom.

## Autoscaling: necessary but not sufficient

**It reacts with a delay (minutes); bursts fill a queue in seconds.** During the lag you're underprovisioned. So pair it with: (1) a **baseline** that absorbs a sudden burst while scaling catches up (don't scale to zero on latency-sensitive paths), and (2) **backpressure/bounded queues** so a burst that outruns the scaler degrades gracefully instead of crashing. Autoscaling handles *sustained* load; backpressure handles the *transient* spike. (Full autoscaling treatment: Module 11.)
