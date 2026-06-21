---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [operations, observability, monitoring, alerting, slo, arc4, module16]
related: ["[[sli-slo-sla]]", "[[percentiles-vs-averages]]", "[[gray-failure]]", "[[error-budget]]", "[[rep1-the-on-sale]]", "[[kubernetes-as-architecture]]"]
---

# Observability

## Monitoring vs observability
- **Monitoring** = watch for **known** failure modes (alert if CPU>80%). Answers questions you knew to ask. Known unknowns.
- **Observability** = ask **arbitrary new** questions of system behavior *after the fact*, no new code. Debug **unknown unknowns** — the failure you didn't predict.

Your worst outages are the unpredicted ones (predicted → prevented). Test: novel incident ("slow only for Brazil + Android + PIX") — can you answer it from existing telemetry, or must you ship logging and wait for recurrence? → need **high-cardinality, high-dimensionality** data (break down by user/region/device/build, combined). Pre-aggregated metrics throw away the dimensions you'd need (Honeycomb philosophy).

## Three pillars (different questions; a narrowing debug flow)
- **Metrics** — aggregated numbers (rate, p99, error rate). Cheap, for dashboards+alerting. "Is something wrong / what's the trend?" (aggregated → hides specifics, the M2 lesson).
- **Logs** — discrete events. "What exactly happened in this event?" High-volume, hard to correlate.
- **Traces** — one request's path across ALL services + per-hop timing. "Where did this request spend time / fail?" Non-negotiable for distributed systems (needs a correlation ID threaded through every hop).

Flow: **metrics = smoke alarm (something's wrong) → traces = which room (which service) → logs = close-up (what exactly).** Microservices cost you the single stack trace; distributed tracing buys it back.

## Alerting (determines if on-call is survivable)
Alert fatigue (paged 15×/night for noise → numb → miss the 1 real alert) is itself a reliability hazard.

**Alert on SYMPTOMS (what users feel), not CAUSES (internal states).** CPU>80% isn't a problem if users are fine (page = noise → fatigue), and low CPU doesn't mean fine (contention failures → missed). Right alert = on the **SLI** / error-budget burn rate → fires exactly when users are hurt → every page actionable. SLO-based alerting (M2) gives a *principled* threshold. Cause-based = dashboards for debugging; symptom-based = waking a human.

### Three tiers (Ray Q1b — fix for alert fatigue)
1. **Page a human** — ONLY symptoms needing judgment (SLO breach, checkout broken, business metric dropped). Every page **actionable**.
2. **Auto-remediate, no page** — pod restart (k8s loop, M15), CPU spike (autoscaler). Logged, not paged. ("If a machine can handle it, don't wake a human.")
3. **Dashboard/ticket** — "mem 75%", "disk 60%": review by day.

**Fewer alerts = better detection**: symptom alerts catch novel failures regardless of cause; removing noise means on-call responds to the 1 real page. Noise IS a detection failure. Test every alert: "if this fires at 3am, must a human DO something now?" No → dashboard/auto-remediate, not a page.

## Error attribution (Rep 1 debt)
Count **server-attributable (5xx, timeouts)**, not **client (4xx)**. A 4xx flood = client sending garbage / attacker probing — not your system failing; alerting on it wakes you for someone else's bug. 4xx vs 5xx = "is someone sending garbage?" vs "are WE broken?"

## Instrument the business question (Rep 1 debt, PAID)
**Technical metrics = is the system RUNNING; business metrics = is it WORKING.** A silent-2xx bug (broken logic, happy 200s — Ray Q1a) leaves every technical dashboard green while the company bleeds. The catch: **orders/min drops to zero** — absence of successful business outcomes, no error spike needed. Business SLIs (orders/min, checkout-completion, funnel drop-off) are engineering deliverables you must deliberately emit; often the FASTEST incident detector + deploy-agnostic (catch any cause). The gap between "running" and "working" is where silent bugs live, and only business metrics span it.
