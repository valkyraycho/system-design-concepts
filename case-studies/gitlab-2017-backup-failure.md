---
type: case-study
status: done
created: 2026-06-11
updated: 2026-06-11
tags: [outage, backups, data-loss, arc1, module1]
related: ["[[facebook-2021-outage]]", "[[correlated-failure]]", "[[off-serving-path-failures]]"]
---

# GitLab 2017 database deletion

Exhausted engineer deleted the production database directory. **All five** backup mechanisms turned out broken or empty — some silently failing for months (pg_dump version mismatch producing nothing, failure emails rejected by the mail server, empty S3 buckets, snapshots not covering DB disks). Recovered by luck: a staging copy ~6 hours old.

## The failure mode

Every mechanism was judged by "the job ran" — not "the data can be brought back." Five green checkmarks, zero recoverable bytes.

**A backup that has never been restored is a hope, not a backup.** The only proof a backup exists is a completed restore (recurring lesson 4). Only real defense: **exercise the restore path regularly, automated** — restore to scratch instance, run integrity queries, alert on *that*.

## The deeper asymmetry

**Failures on the serving path announce themselves; failures off the serving path must be actively hunted.** Anything whose only job is "be there in an emergency" (backups, failover automation, runbooks, audit tools) silently rots because normal operation never exercises it — *safety mechanisms have no users*. Facebook's buggy audit tool is the same failure mode in different clothes. Hence DR drills, game days, chaos engineering: put load on the emergency path before the emergency does. See [[off-serving-path-failures]].
