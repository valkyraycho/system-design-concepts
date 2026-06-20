---
type: rep
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [rep, arc3, scaling, evolution, premature-scaling]
related: ["[[scaling-evolution]]", "[[multi-region]]", "[[failover-split-brain-sharding]]", "[[replication-rpo-rto]]", "[[load-balancing]]", "[[autoscaling-capacity]]"]
---

# Rep 3 — "From Garage to Planet" (Arc 3 capstone)

Format: grow ONE system (Plantpal — Instagram for plants) through orders of magnitude; make each scaling decision AND resist the premature ones. Tests Module 13 "don't scale prematurely" judgment as much as the techniques.

## Stage 1 — Day 1, ~hundreds of beta users
Ray: POC but production-hygienic; simple (few servers, 1 LB, 1 DB); **observability early** (data-driven scaling later); **CI/CD + dev env early**. Strong — drew the key line:
**Invest early in what makes you FASTER (CI/CD, observability, clean arch); defer what only matters at SCALE (sharding, multi-region, microservices).** Test: "does it help NOW or only at a scale we haven't reached?"
- Sharpenings: slightly over-built (LB + multiple instances unneeded at hundreds of users → 1 app server + managed Postgres + **S3+CDN for photos**). **CDN-for-images is NOT premature** — it's the *simplest correct* design (serving image bytes through the app server is MORE work). "Simple ≠ naive": some scale-friendly choices are also simplest (CDN→do it) vs pure scale-debt (sharding→defer).

## Stage 2 — 200K users, READ bottleneck (writes fine, feed slow, DB 70-80% CPU)
Ray: one-step-at-a-time; replica / cache / **vertical scale first** / explicitly NOT multi-region/autoscaling. Correct ordering (cheapest+safest first, measure after each):
1. **Vertical scale** (hours, reversible, no new failure modes) — buys time to do next step calmly. Boring on purpose.
2. **Read replica** — symptom-matched (reads slow, writes fine = textbook replica signal). YouTube-order step 2.
3. **Cache (Redis)** hot reads.
4. **Fan-out-on-write — NOT YET** (Ray floated it; flagged as premature — it's a feed *redesign*, far bigger than replica/cache; bringing a crane to hang a picture).
Discipline: pick the *cheapest rung that addresses the symptom*, re-measure, stop when cleared. Replica inherits **read-your-writes lag** (M9) → budget for it (route own-post-read to primary).

## Stage 3 — 40M users, three distinct problems
1. **Writes bottleneck → sharding.** Ray flagged "key is hard" ✅. Made him choose: shard by author = writes + "my posts" cheap, but **feed = cross-shard scatter-gather**. No key makes BOTH write & read cheap — sharding optimizes one access path, penalizes others. Shard key + feed architecture must be designed *together* (fan-out makes feed a single-key lookup → shard-friendly). This is why sharding is the point of no return.
2. **EU latency → multi-region.** Ray named conflicts + lag ✅. Refinement: Plantpal data is user-partitionable → **partition-by-home-region** (each datum one writer → dodges write conflicts) > full active-active. Residual: cross-region interactions (US follows EU / traveling user) pay the tax. Choose the topology that *minimizes* the problems, don't accept active-active conflicts as inevitable.
3. **Celebrity feed lag + load spikes → hybrid fan-out.** Ray ASKED "is the Twitter solution fair at this stage?" — **the key judgment moment.** Answer: YES now (40M, measured celebrity pain, cheaper tools exhausted) — same technique that was WRONG at 200K is RIGHT here. Technique didn't change; scale + evidence did. New problem: the celebrity/normal **threshold is fuzzy & shifts** (users cross it as they grow; a sudden-viral normal user mini-write-storms before reclassification).

## The growth being measured: judgment vs Rep 1
Rep 1: reached for a big re-architecture under time pressure (premature). Rep 3: **checked each technique against the current scale before reaching** ("is fan-out fair at THIS stage?") — unprompted. That instinct (technique-vs-scale-and-evidence, not technique-because-scalable) IS the Module 13 discipline, now internalized.

## Recurring theme: every scaling step is a deliberate trade taken in order, only when forced — and skew is the enemy at every level (celebrity = hot key, head/tail split is the fix).
