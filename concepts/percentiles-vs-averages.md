---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, measurement, latency, arc1, module2]
related: ["[[sli-slo-sla]]", "[[metastable-failure]]", "[[gray-failure]]"]
---

# Percentiles vs averages (measuring latency)

**Averages answer "how is my system doing?" Percentiles answer "how is my worst-served user doing?"** Nobody experiences the average.

## Why averages lie about latency

99 requests at 100ms + 1 request at 8000ms → average = 179ms (Ray computed it). One disaster dragged the average almost nowhere — it *hides* the bad experience.

## Why not standard deviation (Ray's first guess)

- Assumes a symmetric bell shape; latency is **skewed** — hard floor on the left, long ugly tail right (timeouts, GC, queue waits)
- Not actionable — can't contract/alert/explain "stddev 800ms"

## Percentiles: read values off the sorted list

- **p50 / median** — half of requests are at least this fast; ignores freak values
- **p99** — the experience of your worst-off 1% of requests; a real number a real request got
- A percentile is a *commitment* you can write down ("99% under 300ms"); stddev is just a description.

## Why the tail is the MAIN event, not a curiosity

1. **One user, many requests** — a page firing 50 API calls with 1% slow each → `1 − 0.99^50 ≈ 40%` chance the page is slow. A 1% request problem = 40% page problem.
2. **Fan-out / "The Tail at Scale"** — front-end waits for the *slowest* of dozens of backends → user latency ≈ max, not average. Scaling out makes tail latency *worse*. The tail becomes the typical experience.

This is why "average 180ms, green dashboard" coexists with a furious top customer: the biggest customer makes the most requests → rolls the dice most → lives in your tail.
