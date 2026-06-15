---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, measurement, availability, arc1, module2]
related: ["[[sli-slo-sla]]", "[[error-budget]]"]
---

# The nines (availability math)

Month ≈ 2,592,000 s. Downtime = (1 − availability) × that.

| Availability | Downtime/month | Downtime/year |
|---|---|---|
| 99% (two nines) | ~7.2 hours | ~3.65 days |
| 99.9% (three nines) | ~43 min | ~8.76 hours |
| 99.99% (four nines) | ~4.3 min | ~52 min |
| 99.999% (five nines) | ~26 sec | ~5.25 min |

(Ray derived 99.9% = 2,592 s ≈ 43 min unaided.)

## The non-linear cost

Each nine is ~10× harder for a shrinking benefit. The architectural cliff is **three → four nines**: 43 min is enough for *a human to get paged and fix it*; 4 min means **no human can respond in time → recovery must be fully automated** (often multi-region, different on-call, different architecture).

So "what's your SLO?" is a budget question in disguise — every nine multiplies cost. Senior move: ask *"do we actually need this nine, or is this gold-plating?"* Five-nines on a feature nobody uses at 3am is wasted money.
