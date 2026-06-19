---
type: rep
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [rep, arc1, ticketing, flash-sale, contention]
related: ["[[littles-law]]", "[[setting-timeouts]]", "[[backpressure-and-capacity]]", "[[load-shedding]]", "[[sli-slo-sla]]", "[[gray-failure]]", "[[metastable-failure]]"]
---

# Rep 1 — "The On-Sale" (Arc 1 capstone)

Scenario: TicketRush, 100–400× instantaneous spike for a stadium on-sale. 8 stateless API servers, Postgres (1 primary 100-conn pool, 1 replica). Buy = `BEGIN; SELECT inventory FOR UPDATE; UPDATE; INSERT order; COMMIT` (~40ms).

## The core problem: row-level contention

Everyone buys the *same event* → contends the *same inventory row*. `SELECT FOR UPDATE` serializes decrements → throughput ceiling = 1/40ms ≈ **25 sales/sec**, un-parallelizable by correctness (can't sell one seat twice). DB dies at ~30% CPU — **contention, not compute.** Generic "add cache/replica" does nothing (write-lock bottleneck).

Cascade: lock serializes → blocked txns hold connections → L=λW explodes → 100-conn pool exhausts → **blast radius hits unrelated events too** (missing bulkhead) → API timeouts → retries → metastable.

## The 25-minutes-to-live fix (stabilize with existing levers)

DON'T re-architect under time pressure. Config-only moves:
1. **`lock_timeout`/`statement_timeout` (~200ms)** on the buy txn → caps W → bounds L → protects the pool. Blocked txn releases its connection fast instead of rotting.
2. **Admission control / rate limit** on buy endpoint → converts crash into queue; only ~25/sec reach the lock, rest get cheap "try again."
3. **Virtual waiting room** (Ticketmaster/Shopify/Queue-it) → drip-feed demand to match DB pace; humane visible backpressure.

Win condition redefined: can't beat the 25/sec physics → **control who's in the line** so *some* users succeed + site stays up, instead of everyone crashing.

## Twist 1 — overselling despite the lock

Same seat sold 3×. **"Check before you write" is the race condition (TOCTOU), not the fix** — 3 buyers read "available", all proceed. Races are fixed by **atomicity**, not careful app-layer checks. Prime suspect given our own fix: the **lock timeout's error path** — a speed-fix can introduce a *correctness* bug (worse than the availability one it solved).

Durable fix: **atomic conditional write** `UPDATE seats SET status='sold' WHERE seat_id=? AND status='available'`, check affected-rows (0 = already taken, fail cleanly). + unique constraint on seat as DB backstop. No read-write gap → robust even when lock timeout fires.

## Twist 2 — atomicity ≠ idempotency (Ray connected etcd CAS / optimistic locking here)

- **Race condition** (different requests, same resource, concurrent) → **atomicity** (FOR UPDATE pessimistic, or conditional UPDATE / CAS optimistic). Optimistic often better on hot rows: no lock held while waiting → doesn't exhaust pool. (Same family as etcd resourceVersion CAS in k8s.)
- **Duplicate delivery** (same request twice, e.g. retry) → **idempotency key** (Module 5). Conditional write alone can falsely tell buyer "failed" on their retry of a *successful* buy.
- Ticketing needs BOTH.

## Twist 3 — "is the on-sale going well?" (Module 2)

99.92% success / p99 280ms = green, but **a "sold out" response is a fast clean HTTP 200 = a "success" that's a user failure.** 140K can't get tickets while the dashboard reads healthy — wrong SLI (cousin of [[gray-failure]]). Measure the **user's goal / business question**:
- sell-through rate & time-to-sellout (the CEO's real number)
- **checkout completion rate** (where real tech failure hides — people who *should* buy but fail at payment)
- purchase latency for buyers; held-seat timeouts
- **oversell count = 0** (correctness SLI, live)
- error rate **segmented** (sold-out 200s vs genuine 5xx)

Lesson: instrumenting the business question is a high-leverage *technical* deliverable (custom metrics, designed in advance) — not "nothing we can do technically."

## Scorecard

Strong: asked baseline first; contention-not-CPU; self-rejected batch idea; reinvented Redis-inventory; etcd↔SQL optimistic-locking transfer.
Edges: check-before-write = the race not the fix; idempotency vs atomicity conflation; reached for slow re-architecture under a 25-min clock; undersold business-metric instrumentation as "non-technical."
