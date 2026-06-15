---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, observability, health-checks, arc1, module1]
related: ["[[measure-the-work-not-the-worker]]", "[[off-serving-path-failures]]", "[[sli-slo-sla]]"]
---

# Gray failure (differential observability)

The system's **detector says healthy while users experience broken.** Neither up nor down — and it looks different depending on who's checking. Microsoft's term; defined by *differential observability*: the system's own observer sees a different reality than its clients.

Ray's framing (earned): *"the health check only tells us the server is not dead, but as far as users are concerned the app is not performing well and is occasionally returning errors."*

## Why it's worse than a clean crash

- **Clean crashes are a gift** — detection fires, LB ejects, traffic reroutes in seconds.
- Gray failures **evade automation** (every recovery mechanism is triggered by detection; an undetected failure disarms the immune system) and **poison instead of vanish** (the sick node keeps eating its share of traffic and ruining it). With fan-out, one slow node out of 100 lands in the critical path of *most* requests ("The Tail at Scale").

## Health is relational, not boolean

A node can be healthy-to-the-LB, dead-to-the-DB, slow-to-users simultaneously. `/healthz` returning 200 asks "can you return 200?"; users ask "can you do your job, fast?" Different questions.

## Defenses

- Deep/dependency-aware health checks (probe the real critical path) — but beware: now a shared-dependency hiccup makes *all* nodes report unhealthy at once (mass suicide; this is why k8s splits liveness from readiness — Module 4)
- **Outlier detection** — compare nodes to peers, eject the one 10× its neighbors (with a hard cap on % ejected)
- Synthetic probes through the full path
- Best of all: [[measure-the-work-not-the-worker]] — the SLI/LB already sees real outcomes

Ties to [[sli-slo-sla]]: a buffering stall is a gray failure — every server 200s, dashboards green, millions watch spinners. The SLI you choose determines which outages you can even see.
