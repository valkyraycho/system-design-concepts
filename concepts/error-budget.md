---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, slo, sre, arc1, module2]
related: ["[[sli-slo-sla]]", "[[the-nines-table]]", "[[percentiles-vs-averages]]"]
---

# Error budget

Turns reliability into a **currency** that resolves the dev-vs-ops war with arithmetic instead of whoever argues loudest.

- Devs want to ship (change = risk). Ops wants stability (safest system never changes, but never shipping = dead).
- SLO of 99.9% means **0.1% of requests are *allowed* to fail** — that's not shame, it's a **budget to spend.** ~2.6M req/month → ~2,600 failures / ~43 min downtime to spend.

## The policy (mechanical, no politics)

- **Budget remaining** → ship freely, take risks, run migrations. You earned the room.
- **Budget exhausted** → automatic, pre-agreed **feature freeze**; all hands to reliability until back under SLO. The number decided, not the meeting.

Aligns incentives: devs who want to keep shipping are now motivated to make deploys safe (canary, flags, tests) — sloppy deploys burn the budget that funds their *next* feature.

## 100% is the WRONG target

1. **Dependency ceiling** — you can't exceed the product of everything you depend on. DB, AWS, the internet aren't 100% (even S3 doesn't promise it). (Ray's receipt: cite a real vendor SLA — devastating to "let's do 100%".)
2. **Exponential cost** — each nine ~10× harder (see [[the-nines-table]]) for benefit users may not perceive.
3. **Opportunity cost** — effort on nines = effort not shipping features.

**Too reliable is also a bug:** consistently delivering 99.999% on a 99.9% SLO = over-spent on reliability, under-shipped, and users now expect five-nines. The budget is meant to be *spent*, not hoarded. (Google sometimes takes services down to stop dependents assuming a stricter SLO.)

## How to push back on a VP wanting 100%

Don't say "impossible" — reframe as a business decision and hand them the trade: *"100% means we stop shipping and pay exponentially more for reliability users can't perceive. The real question is: what target lets us ship fast AND keep users happy?"* — that question's answer **is** the SLO.

**Thesis of the module:** Reliability is a feature with a cost that competes for resources — so we **budget** it, we don't **maximize** it.
