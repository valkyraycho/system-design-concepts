---
type: rep
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [rep, arc2, resilience, cascade, bulkhead, redis]
related: ["[[bulkhead]]", "[[circuit-breaker]]", "[[queues-async]]", "[[idempotency-keys]]", "[[caching-strategies]]", "[[cache-stampede-and-failure]]", "[[replication-rpo-rto]]", "[[littles-law]]"]
---

# Rep 2 — "The Thundering Monday" (Arc 2 capstone)

Scenario: Snapcart grocery delivery. 12 "stateless" API servers, **one Redis holding BOTH cache + sessions**, Postgres primary + 1 async replica, external payment provider (sync) + inventory svc. Peak ~4000 req/s, 94% Redis hit rate.

## Opening latent-risk read (Ray found all 3)
1. 2-node data tier (primary + 1 async replica) — Ray said split-brain; **correction: split-brain needs auto-failover (not stated). Certain risks = async data-loss (RPO) on failover + slow/manual RTO + stale reads from lagging replica.** Match the named failure to the mechanism present.
2. Cache + sessions in one Redis — Ray said "latency contention"; **real risk = missing bulkhead between regenerable (cache) and authoritative (session) state + LRU eviction crossing the two.**
3. Single Redis, no replica — load-bearing cache AND session store AND no redundancy = fattest SPOF. (Ranked worst — correct.)

## The cascade (slow payment → whole app frozen)
Payment W: 200ms→6s (30×), λ unchanged at peak → **L=λW explodes 30×** → blocked checkout requests squat the **shared API worker pool** → pool exhausts → no workers for ANY request incl. **browsing** (doesn't touch payments). = missing **bulkhead**.
- **"Slow is worse than down"**: instant errors would free workers; 6s hangs hold them hostage.
- **Trigger vs root cause**: payment provider slowness is the *trigger* (inevitable, 3rd party); **root cause = no isolation around an untrusted dependency** (Swiss cheese — their trigger hit a hole *you* left). Blaming the 3rd party = junior; "we failed to contain it" = senior → points at fixes in your control.
- 2nd order: naive retries amplify load on the provider (retry storm).

## Triage (immediate vs durable — sort by TIME-TO-EFFECT, not impact)
Immediate (existing levers): **circuit breaker / slash payment timeout** → frees pool → browsing back in seconds; then disable checkout w/ honest message. (Ray put bulkhead here — **bulkhead is architectural = durable, not immediate.**)
Durable (build this week, layer ALL three — complementary not alternatives):
- **Bulkhead** — separate checkout vs browsing pools/services (contains blast radius; #1 root-cause fix)
- **Circuit breaker + timeout on every external/cross-service call** (stops the waiting)
- **Async payment pipeline** — queue + transactional outbox + idempotent worker + honest "processing→confirmed/failed" UX (removes untrusted dep from hot path)

**Landmine: async ≠ fire-and-forget.** Payments need exactly-once *effect* (at-least-once + idempotency); never "forget" the result. "Processing…" UX is fine ONLY with a reliable pipeline behind it.

## Twist — shared Redis hits 100% memory
Recovery surge floods cache writes → Redis maxmemory → **`allkeys-lru` evicts coldest keys, can't tell sessions from cache** → cold session keys evicted (mass logout) AND cache keys evicted (hit rate 94%→60% → Postgres read load climbs). One eviction policy, two data types, both harmed.

**Landmine: do NOT `FLUSHALL` to "start fresh"** → deletes ALL sessions (total logout) + empties cache (0% hit → cache-death stampede → DB melts). In an incident, before any destructive action: *what am I holding, who depends on it now?* Ray sensed the danger (good instinct), didn't have the safe move.
- Immediate (non-destructive): **raise maxmemory** / bigger instance (stops eviction, halts logout bleeding); reduce cache inflow; maybe `volatile-lru` if sessions have no TTL.
- Durable: **separate Redis for sessions (authoritative: replicated, persistent, noeviction) vs cache (disposable: evictable, flushable)**. Stronger: sessions as JWT/DB-backed so losing Redis degrades not logs-out (Module 4). Cache Redis gets its own HA + DB load-shedding (Module 8).

## One-sentence theme: SHARED FATE
Shared worker pool → payment froze browsing. Shared Redis → cache spike logged users out. **Resilience = deliberately un-sharing fates between what must survive and what's allowed to fail.**

## Scorecard
Strong: found all 3 latent risks + ranked worst; clean cascade chain; Redis-LRU eviction mechanism; sensed flush danger.
Edges: name-the-failure-to-the-mechanism (split-brain assumed); async≠fire-and-forget (payments); immediate-vs-durable sorting (bulkhead misfiled); never-destroy-state-to-reset (flush trap).
