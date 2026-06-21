---
type: rep
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [rep, capstone, all-arcs, surge-pricing, hot-key, cascade]
related: ["[[cache-stampede-and-failure]]", "[[bulkhead]]", "[[circuit-breaker]]", "[[littles-law]]", "[[scaling-evolution]]", "[[observability]]", "[[incident-response-postmortems-chaos]]"]
---

# Rep 4 — "The Capstone" (all four arcs)

Scenario: Toll, real-time surge-pricing API on the critical path of every ride booking. 6 stateless Pricing API instances → Surge service → **single Redis** (live supply/demand per zone, 60s TTL) + Postgres (1 primary + 1 async replica). SLO 99.9% < 200ms.

## Opening read — all 4 arcs (Ray found risks in each)
1. **Single Redis = blast radius (Arc 1)** — on hot path of every price; pricing on critical path of every booking → Redis down = Toll 100% down (not degraded). ✅
2. **No circuit breaker / no fallback for Surge↔Redis (Arc 2)** ✅ — deeper: no *graceful degradation*.
3. **Skew + single Redis (Arc 3)** ✅ — sharp catch: "per-zone data isn't uniform" = hot-key/skew (downtown/airport/stadium); sharding by zone still lands the hot zone on one shard.
4. **Async replica lag + tail-beyond-SLO + need business metrics (Arc 4)** ✅✅ — silent-2xx (fast WRONG price) needs business SLIs.
**MISSED: cache stampede** — latent in his own facts (hot zone × 60s TTL): hot zone's TTL expires → thousands miss at once → all recompute → melt. *Lesson: worst risks live in the INTERACTION of two innocuous facts (he had both: "hot zones" + "60s TTL").*

## The cascade (Ray DERIVED the spine unprompted — the curriculum's punchline)
1. Why Redis pegged: **skew (trigger)** × **stampede on 60s TTL + write storm** (concert = flood of riders AND drivers → read spike + continuous-update write spike) = amplifiers. (Ray had skew; missed the stampede amplifier that explains the 100%.)
2. Why other zones slow too — **Ray's chain, verbatim, with names:**
   hot dependency (skew M8/M13) → **slow not down** (M6) → callers hold threads/conns (**Little's Law W↑→L↑** M3) → **shared pool exhausts** (M3) → unrelated zones starve (**NO BULKHEAD** M6) → local→global (**blast radius** M1).
   Root: all zones share ONE Pricing API pool → downtown holds every thread → quiet zones can't get a worker (not because busy, because starved).

## Triage
(a) **Immediate** (order by time-to-effect):
1. **Circuit breaker Surge→Redis** → frees shared pool → rest of city recovers in seconds.
2. **Serve last-known-good surge price** (the killer move). NOT base 1.0x — **failing open to 1.0x at peak = underpricing during a surge event = worsens supply/demand + bleeds revenue.** Graceful degradation for pricing is *business-aware*: degrade to stale-recent, not neutral. *"A fallback that fires during peak must be sensible during peak."*
3. **Shed** hot-zone load if still melting; **never cold-restart Redis** (stampede; gradual warm-up).
Accept degrading: price *accuracy* downtown (stale) ≫ no bookings.

(b) **Durable** (spans all 4 arcs — the capstone point):
- **Bulkhead** Pricing API pools (Arc 2) — #1 fix, stops spread permanently.
- **Redis HA (replicas) + shard by zone** (Arc 3) — fixes SPOF + capacity.
- **Hot-key/stampede (root cause, Arc 1+3)**: request coalescing (`SET NX` leases), TTL jitter + early refresh, replicate/local-cache the hot zone (= celebrity fan-out-on-read: treat hot zone unlike the uniform tail).
- **Ops (Arc 4)**: business metric (bookings/min/zone — catches it faster than latency), SLO burn-rate alerting, **chaos/load-test the concert-spike scenario** (rehearse the emergency path, M17).

## Scorecard
Strong: all 4 arcs in the opening read; **derived the cascade spine from first principles unprompted** (clearest "in your veins" evidence); right triage skeleton across all arcs.
Capstone-level sharpenings: (1) multiply two facts → spot the stampede; (2) *specificity* — name the mechanism (SET NX leases, TTL jitter) not just "fix hot-key"; (3) business-aware degradation — 1.0x-at-peak is a trap, degrade to last-known-good.
