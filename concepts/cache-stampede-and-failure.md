---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, caching, thundering-herd, arc2, module8]
related: ["[[caching-strategies]]", "[[retry-storm-and-jitter]]", "[[load-shedding]]", "[[metastable-failure]]", "[[utilization-latency-cliff]]", "[[idempotency-keys]]"]
---

# Cache stampede & the death of the shield

Two dramatic cache failure modes — the operating lessons (Facebook memcache paper).

## Failure 1: cache stampede (thundering herd)

Cache-aside populates only on a **miss**. A hot key (e.g. 500K req/s) expires → before any one request repopulates, **all concurrent requests miss at once** → all hit the DB (sized for ~5% of reads) simultaneously → pool exhausted → CPU cliff → queue → timeout → retry → [[metastable-failure]].

Same shape as the [[retry-storm-and-jitter]] (a synchronizing event → mob hits a dependency). Cures (same family):
- **Request coalescing / lease / per-key lock** — only the FIRST miss goes to the DB; others wait + re-read the now-populated cache. 500K misses → 1 DB query. Impl: `SET lock:key <id> NX EX 10` (the same `SET NX` atomic primitive as [[idempotency-keys]] + distributed locks — "exactly one winner"). Facebook **leases** refine this + also arbitrate the invalidation race.
- **Desynchronize expiry** — TTL **jitter** (don't expire all hot keys together) + **early/probabilistic refresh** (recompute hot key in background before it expires → never misses under load).

## Failure 2: the shield dies (the company-killer)

Cache absorbs ~95% of reads. **When the whole cache dies → 100% of reads hit the DB instantly, zero warmup → ~20× normal load in one moment.** DB was never provisioned for it (that was the cache's whole job) → melts.

**Vicious recovery trap:** restart cache → comes back **cold/empty** → every request still misses → still hammers the dying DB → DB can't answer → cache can't populate → can't recover by just restarting the cache. Cache needs DB to warm up; DB needs cache to survive. → multi-hour outages.

**Lesson: a load-bearing cache is NOT an optimization — it's critical infrastructure.** The question is never "how fast when it works?" but **"what happens to everything behind it when it doesn't?"**

## Defense in depth (two layers — Ray proposed layer 1; senior answer = both)

- **Make the cache hard to fully lose** — replication/HA pools (reduce *probability*). [Ray's "treat Redis like real infra + replicas + LB".]
- **Make the DB survive total cache loss** — [[load-shedding]] at the DB/proxy (admit 5K/s, fast-fail the rest) + **gradual warm-up** on recovery (ramp traffic so cache repopulates while DB serves a survivable trickle) (limit *consequence*).

Recurring Arc 1 pairing: reduce probability AND limit blast radius. Never rely on "it won't fail" alone.
