---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, caching, consistency, invalidation, arc2, module8]
related: ["[[externalized-state]]", "[[queues-async]]", "[[rep1-the-on-sale]]", "[[cache-stampede-and-failure]]"]
---

# Caching strategies & invalidation

**A cache is a second copy of data, kept faster/closer, to *absorb load* that would otherwise hit the source of truth.** Reliability lens: it's a **shield** in front of the DB (95% hit rate → DB sees 5% of reads → cache does 20× the work). → raises the climax question: what happens when the shield vanishes? (see [[cache-stampede-and-failure]]).

**A cache is a deliberate, managed inconsistency** — you keep a copy you *know* may diverge, trading staleness for speed/load-absorption. So every caching choice is a consistency trade-off: **choose per-data-type by staleness tolerance.** No universal "right" cache.

## Cache-aside (lazy loading) — the default, likely my-redis's pattern

```
read(key): v = cache.get(key); if hit return v
           else v = db.get(key); cache.set(key, v); return v
```
App manages cache; cache doesn't know the DB. Populated **on-demand by misses** (the seed of [[cache-stampede-and-failure]]).

## Write/invalidation strategies

- **Cache-aside + TTL** — no active invalidation; entry expires after TTL, next read reloads. Stale up to TTL. Knob: short = fresher + more DB load; long = less load + staler.
- **Write-through** — write DB + cache together, synchronously. Always fresh; slower writes; caches maybe-never-read data.
- **Write-behind (write-back)** — write cache, return, async flush to DB. Fast writes but **dangerous**: cache dies before flush → lost acknowledged write (Module 7 durability trade in cache clothing). Right for high-write counters (likes), not for durable data.

## Invalidate vs update on write (Ray Q1: write-through is right; this is the safer flavor)

On a write, **delete the cache key** > update it:
1. If you *update* cache then the DB write rolls back → cache holds an uncommitted value (worst-case disagreement). Delete → next read reloads committed DB state.
2. Concurrent "update cache" ops can race and leave the *older* value winning. Delete sidesteps it.
Pattern: **write DB, then delete key.** (Not perfectly race-free — concurrent miss can repopulate stale → Facebook leases, see [[cache-stampede-and-failure]].)

## The decision-vs-display rule (the trap, Ray Q1b wallet)

Deciding question is NOT "is this data important?" but **"read for *display* or read to make a *state-changing decision*?"**
- **Read-for-display** (show balance on dashboard): seconds-stale fine → cache it.
- **Read-for-decision** (does balance cover this $50 charge?): must read authoritative current value, **transactionally, atomically with the spend** ([[rep1-the-on-sale]] oversell, wallet costume). **No cache ever — any TTL > 0 is a double-spend bug.** A "short TTL" does NOT save you.

Same data (balance), two staleness tolerances by *what the read is for*. Trap = treating "balance" as one cache policy.

## Decompose before caching (Ray Q1b post)
Celebrity post = content (write-once/read-millions → cache-aside, hot-key shield, easy invalidation) + counters (likes/replies, high-write → async aggregation, eventually consistent). Two data types, two policies.
