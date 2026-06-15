---
type: concept
status: active
created: 2026-06-11
updated: 2026-06-11
tags: [reliability, retries, overload, arc1, module1]
related: ["[[aws-s3-2017-outage]]", "[[load-shedding]]", "[[time-diversity]]", "[[correlated-failure]]"]
---

# Retry storms & jitter

**Retries amplify failures** (recurring lesson 2). A retry is a client voting to increase load on a system that's failing because of load.

## Amplification layers (they multiply)

SDK retries × app-level retries × platform retries (Lambda) × human refresh × accumulated cron/queue backlog. One real request → dozens of attempts. A recovering system faces current demand × retry multiplier + hours of backlog, **with cold caches** — the heaviest load of its life, in its first sixty seconds.

## Backoff without jitter = scheduled waves

The outage is the synchronizer: everyone fails at the same instant, so everyone's deterministic backoff clock starts together — as Ray put it, *"they're all using exponential backoff, so everyone retries at 2s, 4s, 8s... after the first failure."* Polite per-client, mob in aggregate: quiet–spike–quiet–spike. Backoff doesn't reduce peak load; it schedules it.

**Fix: full jitter** — `sleep(random(0, min(cap, base × 2^attempt)))`. Randomness decorrelates the clocks; waves smear into a flat trickle. Synchronized clients are [[correlated-failure]] in time — jitter is [[time-diversity]] applied to retries.

Plus: **retry budget** — when ~100% of calls fail, retrying is pure harm; cap retries as a fraction of traffic (e.g. +10%), fail fast beyond it. (Grows into the circuit breaker, Module 6.)
