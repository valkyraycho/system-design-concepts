---
type: concept
status: active
created: 2026-06-11
updated: 2026-06-11
tags: [reliability, backups, failover, arc1, module1]
related: ["[[gitlab-2017-backup-failure]]", "[[facebook-2021-outage]]", "[[correlated-failure]]"]
---

# Off-serving-path failures (safety mechanisms have no users)

**Failures on the serving path announce themselves; failures off the serving path must be actively hunted.**

Serving-path failures (OOM, pool exhaustion, latency) surface in minutes — users and alerts feel them. But anything whose only job is "be there in an emergency" — backups, failover automation, standby capacity, runbooks, audit tools — silently rots, because normal operation never exercises it. Every dashboard stays green while it decays.

- GitLab 2017: five backup mechanisms, all judged by "the job ran," none by "the data comes back." **A backup that has never been restored is a hope. The only proof is a completed restore.**
- Facebook 2021: the audit tool that should have caught the bad command had an unknown bug — same failure mode.
- "Failover is configured" is a claim about the past; "failover works" is only provable by killing the primary and watching. As Ray put it: *create the same setup and kill the primary forcefully* — forcefully matters, graceful shutdown tests the polite path.

## Defense

Exercise the emergency path, on a schedule, before the emergency does:
- automated restore-to-scratch + integrity checks, alerting on *that*
- failover drills / game days — eventually in production, announced, business hours
- once proves nothing durable; the rot is continuous, so the exercise must recur

(Chaos engineering = this idea industrialized. Module 17.)
