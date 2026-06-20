---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [scaling, evolution, sharding, fan-out, skew, arc3, module13]
related: ["[[cache-stampede-and-failure]]", "[[failover-split-brain-sharding]]", "[[cell-based-architecture]]", "[[bulkhead]]", "[[delivery-guarantees-and-ordering]]"]
---

# Scaling evolution (case studies) & premature scaling

## Thesis: solve the bottleneck you ACTUALLY have, at the order of magnitude you're ACTUALLY at
Every scaling solution is also a cost (complexity, ops burden, lost dev speed). Right arch for 10K ≠ right arch for 10M, **and vice versa.** Companies *evolved* into their architectures one bottleneck at a time, reluctantly, later than felt comfortable. "Boring monolith + one Postgres" is correct far longer than juniors believe.

**Premature scaling = expensive mistake.** Complexity compounds: slows every future change, multiplies failure modes, demands ops maturity. Each Arc 3 technique is a *debt* taken to solve a specific bottleneck — take it too early and you pay interest on a loan you didn't need.

## Twitter timeline — fan-out on read vs write
- **Fan-out on READ** — compute timeline per view (query everyone you follow, merge). Simple, no precompute, but expensive query every read → melts (reads dominate).
- **Fan-out on WRITE** — on tweet, push into every follower's precomputed timeline. Cheap reads. Cost moves to write: 1 tweet = N writes.
- **Celebrity problem**: 100M followers → 1 tweet = 100M writes → write storm. Neither pure approach works.
- **Hybrid (the answer)**: fan-out-on-write for normal users + fan-out-on-**read** for celebrities, merged at read time. Cheap reads for 99.9%, no write-storm for the few. Celebrity = hot key ([[cache-stampede-and-failure]]); fix = treat skewed head differently from uniform tail.

## YouTube — scaling a relational DB (the canonical order; cheap→expensive)
1. **One DB** (works far longer than expected — Stack Overflow).
2. **Read replicas** (reads outgrow one box; but lag/read-your-writes).
3. **Caching** (Redis in front, absorb reads).
4. **Vertical scaling** (bigger box — underrated, often cheaper than next step).
5. **Sharding** — ONLY when *writes* outgrow a single primary. The expensive, point-of-no-return step (cross-shard queries die, distributed txns nightmare). YouTube built **Vitess** to manage it.

**Reads scale easily (replicas/cache); writes scale hard (sharding).** Exhaust cheap options first. Most teams that *think* they need sharding have a read problem (replica/cache) or one hot table — not true write volume. Sharding = last resort.

## Discord — MongoDB → Cassandra → ScyllaDB (match DB to access pattern)
Messages = enormous write volume, append-mostly, read-by-recency, partitioned by channel. → wide-column store (Cassandra), not document store. Cassandra's GC pauses + hot-partition (busy channel = hot key) pain → ScyllaDB (C++, no GC). **The DB must match the workload's access pattern; "scaling" sometimes = change the storage engine, not add boxes.** (Echoes OLTP/OLAP + DDIA storage engines.)

## Sharding-now-to-be-safe is the RISKY choice (Ray Q1a)
Not just unnecessary — actively risky: (1) **wrong shard key** (don't know real access patterns yet; re-sharding live data is brutal), (2) **complexity tax the whole time you don't need it**, (3) **more failure modes** (sharded cluster less reliable than one Postgres). Premature sharding trades a known/deferrable future cost for an immediate/compounding/irreversible one made with the least info you'll ever have.

## Skew is the universal enemy of horizontal scale (Ray Q1b)
**Horizontal scaling divides load by KEY; skew concentrates load WITHIN a key; a key is indivisible** → the unit of scaling (machine) and unit of load (key) are mismatched. One hot key sits on exactly one machine — add 100 more, it still melts while 99 sit idle. "Just add machines" assumes *even key distribution*; skew violates that premise structurally (not a tuning problem).
**Fix is never "more machines" — always "detect the hot element, give it special handling."** Horizontal scale handles the uniform tail; bespoke handling rescues the skewed head. Recurring shape: Twitter celebrity, Discord busy channel, cell whale-tenant, Rep hot row — all head/tail splits.
