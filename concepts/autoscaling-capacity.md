---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [scaling, autoscaling, capacity, arc3, module11]
related: ["[[utilization-latency-cliff]]", "[[littles-law]]", "[[backpressure-and-capacity]]", "[[metastable-failure]]", "[[cache-stampede-and-failure]]"]
---

# Autoscaling & capacity

Promise: don't provision for peak, scale to load. Catch: it doesn't dissolve the capacity problem — it *moves* it, and the catch is **time**.

## Reaction lag — the whole story

spike → metrics collected (~30-60s) → threshold evaluated → instances requested → **boot (30s–min)** → app warmup + readiness → LB/discovery routes. Total = **minutes**.

But a burst hits the cliff in **seconds** ([[utilization-latency-cliff]]). So:
**Autoscaling can't save you from a sharp SPIKE — only from a sustained INCREASE.** New capacity arrives after the existing servers are already over the cliff / in [[metastable-failure]] — possibly into an already-down system. Autoscaling tracks *trends*, doesn't absorb *shocks*. For shocks → Module 3 tools (headroom, queues, load shedding); autoscaling is the slow layer refilling behind them. **Complement to a survivable baseline, not a replacement for capacity planning.**

## What to scale ON (the signal problem)

- **CPU** — classic default, but **wrong for contention/IO-bound systems** that die at low CPU (locks, connection pools, I/O waits). CPU stays low while you die → scaler never fires. Worse than no scaling (false confidence).
- **Request rate / concurrency (L)** — scales on actual load.
- **Queue depth / lag** — THE signal for queue-worker systems.
- **Custom/business metrics.**

**Scale on the OUTCOME, not a resource proxy.** Queue depth is an *outcome* signal (work piling up) sitting *downstream of all bottlenecks* — robust no matter *why* workers are slow. CPU is a *resource* signal, only valid if CPU is the bottleneck (often isn't). Signal choice = bottleneck-identification (the Module 3 work).
*(Ray Q1a: IO-bound workers, CPU-scaled → never fires → queue backs up. Fix: scale on queue depth.)*

## Cold start — scaling can make it WORSE

New instance = cold caches, cold pools, runtime warmup, must open connections → slow at first, can fall over if hit with full traffic immediately.
**Scale-up under load can deepen an outage:** 20 new app instances each open fresh DB connections → floods the already-struggling DB → accelerates its death. Scaled the non-bottleneck tier, hammered the bottleneck tier. (Common: app-tier autoscaling DDoSing the DB.)

## Scaling down traps

Too aggressive → removes capacity before a spike. Killing in-flight work → need graceful drain (M4). **Flapping** (up/down thresholds too close) → oscillation → fix w/ cooldowns + hysteresis. **Scale-to-zero** → cheapest but first request pays full cold-start (serverless cold start) — fine for dev, bad for latency-sensitive.

## Predictive/scheduled scaling (Ray Q1b — known event)

Known spike time (flash sale) → **pre-scale before it** (e.g. 30 min early) → converts reaction-lag into a non-problem.
- **Pre-WARM, not just pre-count** — boot early AND warm caches/pools/shadow-traffic so capacity is *hot* at T0, not just present.
- **Scheduled = floor, not ceiling** — pre-scale to expected peak; keep **reactive on top** for the surprise delta; keep Module 3 shock absorbers (queue/shed/waiting-room) for spikes that outrun both. (How Shopify survives Black Friday.)

**Portfolio principle:** the more you know about *when* load arrives, the less you must *react* (the dangerous part). Reactive (cheap, dangerous) → scheduled/predictive (pre-empt the known) → over-provisioned (safe, expensive). Match strategy to knowledge of the load pattern; combine all three.
