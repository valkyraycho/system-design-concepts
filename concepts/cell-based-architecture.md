---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [scaling, cells, blast-radius, ha, chaos, arc3, module12]
related: ["[[bulkhead]]", "[[correlated-failure]]", "[[off-serving-path-failures]]", "[[utilization-latency-cliff]]", "[[multi-region]]", "[[rep1-the-on-sale]]"]
---

# Cell-based architecture

## Problem: scaling UP makes the blast radius BIGGER
One big fleet + one (sharded) DB serving all users = a single giant **failure domain.** A bad deploy / poison record / runaway query / config error hits *everyone at once*. (Rep 1: one event's lock drained the shared pool → killed unrelated events.)

## Idea: partition the whole system into independent cells
A **cell** = a complete self-contained stack (own servers, DB, queue) serving a *subset* of users (by ID hash / geo / tenant). Cells **share nothing**.

**Converts "everything fails together" → "one cell fails, rest fine."** = [[bulkhead]] applied to the ENTIRE system at the largest granularity. Blast radius of ANY failure capped at **1/N users**. Composes:
- **Cell-level canary** — deploy to one cell first; breaks → only 1/N affected, roll back before touching others.
- **Isolate a noisy tenant** — migrate them to their own cell (Rep hot-event fix).
- **Unit of capacity** — "1 cell = 100k users; need 1M → 10 cells."

Cost: complexity (routing users→cells, cross-cell ops hard, per-cell overhead) + **the cell router is a critical shared component** (keep it tiny & bulletproof — the one thing all cells share). AWS builds core services this way. Largest expression of "un-share fates."

## The hardest part: data that doesn't partition cleanly (Ray Q2a)
Cells shine when data partitions cleanly by the cell key. But **global/cross-tenant data** (shared catalog, multi-tenant user, search-across-all, billing rollups) either:
- (a) **replicated into every cell** → cross-cell consistency problems ([[multi-region]] trade in miniature), or
- (b) **lives in a shared service outside cells** → re-introduces a shared failure domain (the thing cells eliminate).
Art of cell design = maximize what partitions cleanly, minimize the global remainder.

## Regional evacuation (Netflix) — the operational payoff
Partitioned system → when one cell/region is sick, **evacuate** it (shift traffic to healthy ones) instead of debugging live under fire. Netflix fails out a whole AWS region in minutes.
- Old way: region sick → scramble to fix while users suffer.
- Cell way: region sick → evacuate → users recover immediately → debug calmly, zero user impact.
= blast-radius-limiting + fail-static + graceful degradation as an operational practice: **route around the sick thing, don't fix it under pressure.**

### Evacuation needs BOTH (Ray Q2b: got capacity, missed rehearsal)
1. **Standing capacity / headroom** — every region must run with enough slack to absorb an evacuated region's load (else evacuation → survivors overload → total outage). [[utilization-latency-cliff]]. Cost: permanent over-provisioning.
2. **Continuous rehearsal** — an evacuation capability you never exercise is silently broken when needed ([[off-serving-path-failures]] / "backups worthless until restored"). Netflix = **Chaos Engineering / Chaos Kong**: deliberately evacuate regions in prod regularly so the real one is a well-worn path.

**Capacity without practice = room but broken mechanism (GitLab backups). Practice without capacity = mechanism fires but survivors melt.** Routing (DNS/anycast/cell-router, M10) is just the plumbing; headroom + rehearsal make evacuation *trustworthy*. "We have a DR region" ≠ "we can fail over" — the latter is only true if you've done it recently under realistic conditions.
