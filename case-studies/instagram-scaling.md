---
type: case-study
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [case-study, instagram, sharding, feed, fan-out, ranking, real-world]
related: ["[[scaling-evolution]]", "[[dns-anycast-service-discovery]]", "[[backpressure-and-capacity]]", "[[caching-strategies]]", "[[rep3-garage-to-planet]]"]
---

# Instagram: 14 engineers, 30M+ users

Launched 2010, 1M users in 3 months, 30M+ users / ~14 engineers by 2012 acquisition. Philosophy: "keep it simple, don't reinvent the wheel, use proven tech" — Django + PostgreSQL + Redis/Memcached on AWS. (Validates Ray's Rep 3 Day-1 instinct: boring, hygienic, scale-minimal.)

## Sharded ID generation (Ray independently derived the shape)

Sharding Postgres broke auto-increment (only unique per-DB). Real scheme, 64-bit ID:
```
41 bits: timestamp (custom epoch, not 1970 — maximizes useful range)
13 bits: logical shard ID (up to 8,192 — deliberately >> physical machine count)
10 bits: auto-increment sequence per shard (mod 1024)
```
Ray proposed timestamp + machine-segment + auto-increment unprompted — same shape, right instinct that time-sortability matters (feed = chronological list; sortable IDs mean ordering a feed needs zero extra lookup/index vs a random UUID + separate `created_at`).

**The trick Ray didn't have: logical shards ≫ physical machines.** Lets you move logical shards to new hardware later (rebalance) WITHOUT regenerating any ID — the shard bits still point to the same logical shard, which now just lives elsewhere. = [[dns-anycast-service-discovery]] "address the stable abstraction, not ephemeral physical reality," applied to sharding. Also solves Rep 3's "re-sharding live data is brutal" problem by over-provisioning logical shards from day one.
10-bit sequence (1024/ms/shard) = a capacity number sized to expected per-shard write rate — headroom, not edge-of-cliff (M3).

## Feed generation: hybrid fan-out (Ray transferred Twitter's pattern unprompted)

Feed = per-user list in Redis (post IDs from followees, time-ordered). Normal users: **fan-out-on-write** (push to every follower's list on post). Very-high-follower accounts: **fan-out-on-read** (merged in at read time) — same shape as Twitter's celebrity problem (M13), independently transferred by Ray to a new company.

**Wrinkle beyond clean theory: the feed list must be BOUNDED**, or it grows unboundedly per active user forever — silent, not a crash, but an ever-growing infra cost. = [[backpressure-and-capacity]] bounded-queue principle, generalized: applies to any accumulating cache/list, not just literal request queues.

## The 2016 chronological → algorithmic shift (the real wrinkle, Ray's one redirect needed)

A personalized ranking score is NOT a property of the post — it's a property of **(post, viewer, moment)**: different per viewer, decays over time, changes with new signals. **Can't fan-out-on-write a score** the way you fan-out a stable timestamp — there's no single correct value to precompute and store. = read-for-display (timestamp, cacheable) vs read-for-decision ([[caching-strategies]] wallet trap) — a ranking score is decision-like: personalized + time-sensitive, wrong to cache globally.

**Real architecture: two-stage pipeline.**
1. **Candidate generation** — still uses fan-out machinery (cheap, precomputed pool: "what's plausible to show").
2. **Ranking** — MUST happen at read time, per request, ML-scored on live features (recency, engagement, session context). Work shifts from write-time (amortized) to read-time (per-view) — why ranked feeds need much heavier read infra (feature stores, real-time inference) than chronological ones.

**Recurring pattern, new costume:** decompose one blended operation into a cacheable/stable half (candidates) + a must-recompute-fresh half (ranking) — same instinct as accept-vs-process (M3/M7) and content-vs-counters (M8 celebrity post). Question order: "is this value stable enough to store, or must it be computed per-reader?" comes BEFORE "what data structure sorts it efficiently."
