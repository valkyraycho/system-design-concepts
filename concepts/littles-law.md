---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, load, queueing, capacity, arc1, module3]
related: ["[[utilization-latency-cliff]]", "[[metastable-failure]]", "[[load-shedding]]", "[[retry-storm-and-jitter]]"]
---

# Little's Law (L = λW)

In any **stable** system: **L = λ × W**
- **L** = avg items in the system (in-flight: processing + queued)
- **λ** = arrival rate (req/s)
- **W** = avg time each item spends in system (end-to-end latency)

Number inside = arrival rate × how long each stays. **No assumptions** about arrival/service distributions — holds exactly as long as the system isn't growing to infinity. Lets you solve for **L (concurrency)**, which is hard to observe, from λ and W, which are in your metrics.

## Capacity weapon: sizing

λ=500 req/s, W=0.2s → L = 100 in-flight. Blocking server with a thread-per-request needs a pool ≥ 100 (and well above, for bursts). Pool of 50 → requests queue for a thread even though the DB keeps up.

## The killer mechanism: latency↑ → concurrency explosion → exhaustion

Normal: λ=1000, W=0.05 → L=50. Dependency slows: W→2s, λ unchanged → **L=2000 (40×).** Ray's worked example.

Precise chain (the middle links are where the server actually dies):
**W↑ → L explodes → pool/connection/memory exhaustion → server refuses new work / OOMs → THEN clients timeout & retry → [[metastable-failure]] amplification**

Key decoupling: **traffic (λ) never rose.** A slow dependency consumes your connection pool exactly like a traffic spike. Junior stares at the flat request-rate graph and says "not a load problem" — but **L (concurrency), not λ, is the resource killer.** Most occupied slots are held by requests whose clients already left (wasted work).

## Timeouts ARE concurrency control

Capping W caps L. Force W ≤ 0.2s → L ≤ 1000×0.2 = 200. The timeout isn't just "give up on slow"; it's a **ceiling on concurrency.**

Two services, same struggling DB, λ=1000:
- A, 30s timeout → L up to 30,000 → falls over immediately
- B, 500ms timeout → L capped at 500 → degraded but survives

Caveat: a timeout doesn't reduce demand — the caller comes back. Short timeout protects **server-side L**; must pair with client-side **backoff + jitter + retry budget** ([[retry-storm-and-jitter]]) or you convert a concurrency-exhaustion outage into a retry-storm one.

## Master-variable framing

Every resilience lever controls L: headroom (caps L's fluctuations off the cliff), timeouts (cap L's ceiling), [[load-shedding]] (reject to keep L < pool), bulkheads (partition L). All answer: "stop simultaneous in-flight requests from exceeding what I can hold."
