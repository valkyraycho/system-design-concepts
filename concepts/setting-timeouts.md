---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, timeouts, latency, capacity, arc1, module3]
related: ["[[littles-law]]", "[[percentiles-vs-averages]]", "[[metastable-failure]]", "[[retry-storm-and-jitter]]"]
---

# How to set a good timeout

A timeout is a statement about **normal**, not worst case: "past what point is this request almost certainly doomed, so stop holding an L slot for it?" Not "how long could this legitimately take?"

## Method

1. **Measure the dependency's latency percentiles** (p99/p99.9, NOT average).
2. **Set timeout a bit above the high percentile** you'll wait for (~p99.9 × small factor). A request still running past p99.9 is almost certainly doomed (struggling dep, dropped packet, GC) — waiting longer just wastes an L slot.
   - Trade-off: too low → abort healthy-but-slow requests (false positives, wasted work, retries); too high → hold L slots for doomed requests → exhaustion. Choosing the percentile = choosing where to stand on that trade-off.
3. **Little's Law ceiling check**: timeout × λ ≤ pool size (with headroom). Latency data sets a *floor* (don't go below normal perf); [[littles-law]] sets a *ceiling* (don't allow more concurrency than you can hold). If they conflict → grow pool, lower timeout, or fix the dependency.

## Timeout budget (the senior skill — derive TOP-DOWN)

Timeouts are **nested like Russian dolls**, not independent knobs. Rule: **a parent's timeout on a child must be ≥ the child's total need (its own work + its own downstream timeouts), and the whole chain must fit the user budget.**

Start from the user-facing budget and **subtract going down**; each hop leaves margin. If a child's timeout ≥ parent's, the parent gives up while the child still works → child's effort becomes wasted work (caller already left → [[metastable-failure]] pattern). gRPC propagates deadlines natively; often manual otherwise.

### Worked trap (Ray, 2026-06-13)
Checkout, 4s user budget. Ray set BankAPI=3s but OrderService→PaymentService=200ms — **contradiction**: Payment needs 3s+ but Order kills it at 200ms → 100% failure. Correct derivation: 4000ms user − gateway margin → Order ~3600ms − own work → Payment call ~3.4s ⊃ BankAPI ~3.0s. Inventory (parallel path) ~150ms ✅.
Then Little's check: 3.4s × 200 req/s = **680 > pool 300** → synchronous design *cannot* be tuned safe → architecture must change (→ Q3).

## When a slow critical dependency eats the budget (e.g. external bank, p99.9=2.5s)

Two strategies (payments often use both):
1. **Async / decouple** — accept, return "processing…", queue the charge ([[queues]] M7), notify via webhook/poll. Fast ack satisfies user budget; cost = eventual consistency, UX must show "pending." (Can't literally "confirm" before money moves.)
2. **Sync but bounded** — tight timeout + circuit breaker (M6, fail fast when dep is down) + **idempotency keys** (M5, so timeout-retry doesn't double-charge).

## Gotchas

- **Default-infinity trap**: many HTTP clients / DB drivers / pools ship with no (or 30–60s) timeout → the 30s-Service-A disaster. Audit every client lib.
- **Connection timeout ≠ request timeout** — set both.
- Make timeouts **configurable, not hardcoded** — you'll tune them mid-incident.
- A timeout doesn't reduce demand; pair with client-side backoff+jitter+retry budget ([[retry-storm-and-jitter]]) or you trade exhaustion for a retry storm.
