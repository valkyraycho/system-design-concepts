---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [data, replication, durability, availability, rpo-rto, arc2, module9]
related: ["[[off-serving-path-failures]]", "[[correlated-failure]]", "[[caching-strategies]]", "[[the-nines-table]]"]
---

# Replication, RPO & RTO

The data tier is the one thing you **can't treat as cattle** (Module 4 pushed state here). Two distinct goals, conflating them = #1 junior mistake:
- **Availability** — keeps *serving* despite failure (replication + failover)
- **Durability** — data isn't *lost* ever (replication + backups)

## RPO vs RTO (independent, priced separately)

- **RPO (Recovery Point Objective)** — how much data can you afford to lose? (time, about the **past** — how far back the recovery point sits)
- **RTO (Recovery Time Objective)** — how long can you be down recovering? (time, about the **future**)

Daily backups = 24h RPO *and* maybe bad RTO (restoring a huge backup takes hours). A replica = great RTO+RPO for *machine* failure but **is NOT a backup** (replicates `DROP TABLE` faithfully — no protection vs *logical* corruption, [[correlated-failure]]). → need BOTH replicas (machine failure) AND backups (logical/data corruption). Different failure classes, different tools.

**Business decisions priced in engineering.** "Zero RPO, zero RTO" = the five-nines fallacy (exponentially expensive). Ask the business the two numbers; they dictate architecture.

## Replication: sync vs async (the DDIA trade)

- **Synchronous** — primary waits for replica to confirm before acking client → **zero RPO**, but slower writes + fragile (primary's availability coupled to replica health).
- **Asynchronous** — primary acks immediately, ships in background → fast writes, but **non-zero RPO**: primary dies before a write replicates → that acked write is **lost** (replication lag).
- The real knob: **how many replicas must confirm before ack?** 0 (async) / all (sync) / k-of-n (**quorum**, DDIA sweet spot). Each = a point on the RPO-vs-latency curve. Semi-sync (≥1 of N) = bounded RPO without waiting for all.

This is consistency-vs-latency at the storage layer — can't have fast writes AND zero loss AND replica-failure tolerance all at once (CAP/PACELC flavor).

## Read-consistency anomalies (a FAMILY, distinct fixes — Ray Q1a)

Stale reads aren't one bug:
- **Read-your-own-writes** — "see *my own* write." Symptom: save profile, reload, see old. Fix: **read from primary for N sec after a write** (or track write LSN, route to a caught-up replica). [Ray's "sticky to a replica" doesn't guarantee freshness — that replica can still lag.]
- **Monotonic reads** — "don't see data go *backwards*." Fix: pin user to ONE replica. [This is what sticky-replica actually solves — right tool, wrong anomaly.]
- **Consistent prefix** — see writes in causal order.

Naming which anomaly you're fighting is half the fix. Configured at app/driver layer or proxy (ProxySQL/Vitess read-write split), not a DB setting.

## Match durability rigor to stakes (Ray Q1b)

- **orders/payments**: RPO dominates (lost = don't know who paid) → sync/semi-sync replication + frequent backups + PITR (WAL archiving).
- **regenerable cache table**: RPO requirement ≈ **zero** → *opt out of the durability apparatus entirely* (no backup, no durable replication); for RTO, fast *regeneration* or serve degraded (generic recs). **Cheapest data to protect is data you can recompute.** Don't give money-data and cache-data the same treatment even in the same DB.
