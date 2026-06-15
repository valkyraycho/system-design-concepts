---
type: concept
status: active
created: 2026-06-11
updated: 2026-06-11
tags: [reliability, overload, admission-control, arc1, module1]
related: ["[[retry-storm-and-jitter]]", "[[aws-s3-2017-outage]]", "[[metastable-failure]]"]
---

# Load shedding

When offered more than capacity: **serve what you can well, reject the rest fast.** Ray's instinct both times: "a rate limiter at the front door, like a leaky bucket" — correct, the block is only giving yourself permission to be rude.

## The goodput arithmetic

Flood = 10N, capacity = N:
- **Accept everything** → every request gets a starving slice → everyone times out → goodput ≈ 0, and the timeouts feed more retries
- **Admit N, reject 9N instantly** → N succeed; rejections cost microseconds vs timeouts burning threads/memory for 30s → goodput = N even mid-storm

Rejecting isn't failure — each completed request permanently removes a client from the mob. It's the only path back to the healthy state.

## Refinements

- **`Retry-After: n`** on rejections — the server conducts its own comeback traffic
- **Gradual admission** — 10% → 25% → 100%, warming caches (Facebook's manual power-ramp, built into the front door)
- Mechanisms: leaky/token bucket, or simple concurrency caps ("max 5000 in flight, 503 beyond")

## The symmetry

Client side and server side are the same idea on opposite ends of the wire: client spreads demand over time (backoff + jitter + budget); server caps admitted load to capacity (shed + schedule + ramp). Either side behaving keeps the system out of the metastable trap; reappears as circuit breakers vs backpressure in Arc 2.
