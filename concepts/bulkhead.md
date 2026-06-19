---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, failure-isolation, bulkhead, oltp-olap, arc2, module6]
related: ["[[circuit-breaker]]", "[[littles-law]]", "[[load-shedding]]", "[[rep1-the-on-sale]]", "[[correlated-failure]]"]
---

# Bulkhead (failure-domain isolation)

From shipbuilding: watertight compartments so a hull breach floods *one* compartment, not the whole ship. (Titanic's bulkheads didn't reach the top → water spilled over = incomplete isolation.)

Software: **partition resources so the failure of one part can't consume what the others need.** Ray met the need in Rep 1 — one event's lock drained the *shared* 100-conn pool, killing unrelated events = a missing bulkhead.

## Canonical example

One shared 100-thread pool for PaymentAPI + RecommendationsAPI + ShippingAPI → Recommendations gets slow → its calls pile up (Little's Law) → consume ALL 100 threads → Payment can't get a thread → checkout dies because a *non-critical* dep ate the shared pool.

Fix: **separate pool per dependency** (Payment 40, Recs 20, Shipping 40). Recs slow → fills its own 20, fails (fine, enhancement), Payment's 40 untouched → checkout survives. Flood contained to one compartment.

## Bulkhead vs circuit breaker (complementary)

- **Circuit breaker** = reactive/temporal — detects failure over time, stops calling. Damage accrues before it trips.
- **Bulkhead** = structural/spatial — caps blast radius *by construction*, before any failure. A slow dep can only eat its own compartment even if nothing notices.

Pair them: bulkhead bounds the damage while the breaker is making up its mind.

## The three directions of failure isolation (Module 6 family)

- **Circuit breaker** → protects you from a failing *dependency* (downstream)
- **Load shedding** ([[load-shedding]]) → protects you from your *callers* (upstream)
- **Bulkhead** → protects your *parts from each other* (internal)

## Scales (fractal — "isolate failure domains")

pools per dependency · pools per database · cell-based architecture (partition *users*, Module 12) · Rep's per-event inventory budget.

## OLTP/OLAP — the ultimate bulkhead (Ray went straight here)

Fast user reads (~10ms) + slow reporting (~5s) sharing one 100-conn pool: one report holds a connection as long as **500 fast queries** → shared pool gives equal *claim* to unequal *cost* → expensive workload starves the cheap high-volume user-facing one.

Root fix: separate **OLTP** (row store: Postgres/MySQL — many small fast txns) from **OLAP** (column store: Redshift/BigQuery/ClickHouse — few large scans). Analytical scan on the OLTP primary is wrong twice: starves connections AND wrong engine. Bulkhead *between systems* (vs pool-split = bulkhead *within* a system). Bridged by replication/CDC/ETL (Module 9) which decouples so analytics never touches the transactional path.

> Most common real-world bulkhead lesson: a dashboard pointed at the production primary takes down checkout.
