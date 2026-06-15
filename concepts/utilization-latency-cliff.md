---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, load, queueing, capacity, arc1, module3]
related: ["[[metastable-failure]]", "[[percentiles-vs-averages]]", "[[littles-law]]"]
---

# The utilization–latency cliff

Latency is **not a slope, it's a cliff.** It stays nearly flat as load rises, then goes vertical around 70–85% utilization. The difference between 80% and 95% isn't "a bit slower" — it's "fine" vs "on fire."

## Why: queueing + variability (Ray's insight)

The server's *service speed never changes* — what explodes is **waiting in line**. And it explodes because **work doesn't arrive evenly.** Ray's framing: in the worst case all 29 customers/hour arrive at once, so the last waits behind 28 even though average arrival < capacity.

Generalized: even mild randomness creates brief queues. To drain a queue you need **idle moments** — and **high utilization is the absence of idle time.** At 50% the cashier is free half the time and queues drain fast; at 97% there's no slack, so each burst lands on a still-draining queue and waiting compounds.

## The formula (feel the cliff)

$$\text{wait} \propto \frac{\rho}{1-\rho}$$

| ρ | ρ/(1−ρ) | |
|---|---|---|
| 50% | 1.0 | baseline |
| 80% | 4.0 | 4× |
| 90% | 9.0 | 9× |
| 95% | 19.0 | 19× |
| 99% | 99.0 | 99× |

90%→99%: 9% more utilization → ~11× worse waiting. As ρ→1, denominator→0, wait→∞. (Ray's "all at once" = this formula at its limit.) More bursty arrivals / more variable service times → cliff hits **earlier and harder** at the same ρ.

## Money lesson

"Never run above ~70–80% utilization." The idle headroom is **not waste — it's the shock absorber** for bursts, failovers, retry storms. Average utilization hides tail latency exactly the way average latency hides slow requests ([[percentiles-vs-averages]]): the average has headroom while the *bursts* already hit the wall, and p99 lives in the bursts.

This is the *why* behind [[metastable-failure]]: the cliff isn't a special disaster mode, it's the normal physics of queues pushed past the edge.
