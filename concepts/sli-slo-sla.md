---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, measurement, slo, arc1, module2]
related: ["[[percentiles-vs-averages]]", "[[the-nines-table]]", "[[gray-failure]]", "[[error-budget]]"]
---

# SLI / SLO / SLA

Reliability defined from the **user's perspective**, replacing naive "uptime."

- **SLI** (Indicator) — the measurement, as *good events / total events*. E.g. "fraction of requests returning 200 **in under 300ms**." Bakes in correctness AND latency.
- **SLO** (Objective) — internal target for the SLI. E.g. "99.9% over rolling 28 days." The line you stay above.
- **SLA** (Agreement) — the customer *contract* with financial penalties. Always set **looser** than the SLO, so you get paged and fix it before you owe refunds.

## Why "uptime" is the wrong measure (two holes)

1. **"Up" lies about success** — the metastable server *responds*… in 8 seconds. Uptime says "up"; the user says "outage." Uptime measures "did the box answer?", users care about "correct answer, fast enough?" These diverge in exactly the Module 1 failure modes (gray failures, slow responses, box-up-DB-down).
2. **Averaging over time hides concentrated pain** — 43 min/month scattered = nobody notices; 43 min all at once at noon on Black Friday = company incident. Same 99.9%, wildly different harm.

**A good SLI counts "good experiences," not "server was awake."** Then gray failures and metastable slowness automatically count against you.

## The SLI determines what outages you can even see

Ray's video example: a buffering stall is a [[gray-failure]] — every server returns 200, dashboards green, millions watch spinners. SLI = "200 responses" → blind. SLI = "rebuffer ratio" → seen instantly.

**Method:** before writing an SLI, ask *"what does THIS user define as a good experience?"* and measure that. Video → start-success, rebuffer ratio, % time above 720p (not HTTP codes). Payments → correctness over speed. One service often needs several SLIs; "200 in under 300ms" is the default for when you haven't thought hard enough.
