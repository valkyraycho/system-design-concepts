---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [operations, deployment, release, migrations, arc4, module14]
related: ["[[time-diversity]]", "[[swiss-cheese-model]]", "[[database-migrations]]", "[[sli-slo-sla]]", "[[correlated-failure]]"]
---

# Deployment & release engineering

**A deploy is a controlled risk** — introducing change into a *working* system, and change is the most dangerous thing you do to production (recurring lesson 1: config changes = #1 outage cause). Release engineering = applying resilience principles (stage, limit blast radius, make reversible) to the act of deploying.

## Knight Capital 2012 ($440M in 45 min, company dead in a week)
Manual deploy to 8 servers, **1 didn't update** (stale code) + reused a feature flag (old code path) → millions of bad trades. Three lessons you already own:
1. Manual/inconsistent deploy → not atomic/verified (split-brain state).
2. No fast rollback → 45 min bleeding.
3. 100% instant blast radius → no canary. (Swiss cheese: every layer failed.)

## Strategy ladder (increasing safety)
1. **Recreate** — stop all old, start all new. Downtime + 100% blast. Only if downtime OK.
2. **Rolling** — update in batches, no downtime (k8s default). But both versions run at once; rollback = roll back through batches.
3. **Blue-green** — two full envs; deploy+test green, flip all traffic at once. **Instant rollback** (blue still alive). Cost: 2× infra.
4. **Canary** — deploy to 1% traffic, watch metrics, gradually 1→5→25→100% if healthy. **Smallest blast radius** (bad version hits 1%, caught on metrics). Needs good SLIs/observability to gate. = time diversity + blast-radius limit (what Facebook M1 lacked).

A deploy strategy is a **risk posture, not a tool choice.**

## Both versions run at once (the subtlety behind most deploy bugs)
Rolling + canary → v1 and v2 serve simultaneously → **new code must be compatible with old code, INCLUDING the DB schema.**

### Expand/contract (parallel change) for schema (Ray Q1a)
Never rename-column-and-deploy-in-one-shot (during rollout, half the fleet queries a column that doesn't exist / not yet). A rename = a **4-step migration across multiple deploys** with a **dual-write window**:
1. **Expand**: add `email_address`; deploy code that **writes BOTH**, still **reads `email`**.
2. **Backfill** old rows into the new column.
3. **Switch reads** to `email_address` (still writing both, for rollback).
4. **Contract**: stop writing `email`, drop it.
General rule: **you can't atomically change something two code versions depend on simultaneously — change it in backward-compatible steps where both versions always have something valid to use.** The dual-write window is what makes each intermediate state roll-forward AND roll-back safe. ("Just rename the column" = a multi-day dance.)

## Blue-green vs canary: what RISK are you defending against? (Ray Q1b)
The deciding axis = **can the problem be detected before real users hit it?**
- **Catastrophic + staging-detectable** (payments) → **blue-green**: validate in isolated green, flip atomically, minimize any real-user exposure. (Can't canary payments — even 1% bad = real money.)
- **Tolerable + only-detectable-in-prod** (consumer app, subtle UX/perf regression that staging's synthetic/small traffic won't surface) → **canary**: deliberately expose a few real users at real scale + measure.
Not just budget/severity — *what kind of risk* drives the choice.
