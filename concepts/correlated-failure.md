---
type: concept
status: active
created: 2026-06-11
updated: 2026-06-11
tags: [reliability, redundancy, arc1, module1]
related: ["[[facebook-2021-outage]]", "[[swiss-cheese-model]]", "[[time-diversity]]"]
---

# Correlated failure

Redundancy math assumes failures are **independent**. A correlated failure is one event taking out multiple "redundant" components at once — making the availability calculation fiction.

**Redundancy protects against hardware dying, not against software and config being wrong** — and modern outages are overwhelmingly the latter. A change propagates to backups as faithfully as to primaries.

## The correlation walk (ask of any "redundant" setup)

What do the redundant copies still share?

- Same host / rack / switch / power circuit *(Ray's answer #1)*
- Same AZ / region / provider *(Ray's answer #2 — note: doesn't take "all of AWS down", one AZ is enough)*
- Same data stream — replication faithfully copies poison (`DELETE` without `WHERE` reaches the standby in milliseconds). **A replica is not a backup** — replicas answer "machine died?", backups answer "data is wrong?" (decorrelation *in time*)
- Same software version — one bug, both nodes
- Same config/deploy pipeline — one bad push, both nodes
- Same failover automation — split-brain (GitHub 2018)
- Same on-call human, same wrong runbook

Every "yes" is a correlation channel. Physical redundancy is a solved problem; logical redundancy isn't — production horror stories live in the top layers.
