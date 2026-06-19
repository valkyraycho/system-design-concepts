---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [data, failover, split-brain, quorum, locking, sharding, arc2, module9]
related: ["[[replication-rpo-rto]]", "[[correlated-failure]]", "[[gray-failure]]", "[[rep1-the-on-sale]]", "[[delivery-guarantees-and-ordering]]", "[[bulkhead]]"]
---

# Failover, split-brain, locking & sharding

## Failover (promote a replica when primary dies)
- **Manual** — human confirms + promotes. Safe, slow RTO.
- **Automatic** — fast RTO, but risks split-brain.

## Split-brain (the failover that makes TWO primaries)
Auto-failover must answer "is the primary dead, or do I just *think* so?" — the [[gray-failure]] "is it just me?" problem. Network partition → monitor can't reach a primary that's **alive** (still taking writes from its side) → promotes a replica → **two primaries, conflicting writes** → partition heals → divergent histories, some writes MUST be discarded → **corruption** (GitHub 2018). Worse than downtime: corrupts the source of truth.

### Defense: quorum + fencing ("is it just me?" made rigorous)
- **Quorum** — require a *majority* to promote. In a partition only the majority side elects; minority **refuses to serve**. Need **odd** node count (or witness/arbiter) — etcd/ZooKeeper/Raft are quorum machines.
- **Fencing (STONITH)** — forcibly disable the old primary before promoting (cut power/revoke storage lease/**fencing token** = monotonic token storage uses to reject a deposed primary's writes).
- Principle: **never act on a local belief that you're primary — require majority/external confirmation.** A partitioned node assumes *it* might be the problem and steps down.

**2-node auto-failover is a trap:** symmetric split (each sees only itself) → no majority → either fails over (split-brain/corruption) or doesn't (no availability gain). 1 node dying = downtime (recoverable); 2-node split = corruption. **3 is the minimum** where a partition gives a 2-vs-1 asymmetry quorum can resolve. (Ray: connected to his my-k8s etcd implementation — same machinery.)

## Optimistic vs pessimistic locking (Rep's etcd-CAS connection, formalized)
- **Pessimistic** — "conflict likely → lock first." `SELECT FOR UPDATE`; others wait. Holds locks → inflates L → pool exhaustion under contention. Good when contention HIGH.
- **Optimistic** — "conflict rare → check-at-write." Read version, write only if unchanged (`UPDATE ... WHERE version=5`), retry on conflict. No locks held → scales. = etcd `resourceVersion` CAS, Rep's `WHERE status='available'`. Good when contention LOW.
- A **bet on contention rate.** Optimistic under HIGH contention → livelock (everyone retries forever — optimistic cousin of the retry storm). Default optimistic (scales, holds nothing); pessimistic only when contention genuinely high AND optimistic retry cost > lock wait cost.

## Hot partitions / shard key choice (Ray Q2b)
Sharding `events` by **date** → all of today's writes hit one shard (moving hot shard; old shards idle) = hot-partition (write-side cousin of hot key / Rep hot row).

**A good shard key needs BOTH** (date and event_type each fail):
1. **High cardinality** — enough distinct values for many shards + room to scale. (`event_type` low-cardinality → caps shard count.)
2. **Even access distribution** — no value dominates. (`event_type` skewed: clicks ≫ purchases → new hot shard. `date` → only "today" is hot.)
Right key: high-cardinality + evenly-accessed (`user_id`/`event_id`/hash). Need range queries too → **composite key** `(user_id, timestamp)`: user_id spreads writes, timestamp keeps locality. **Beware temporal & low-cardinality keys — hotspot factories.** (Rhymes with [[delivery-guarantees-and-ordering]] partition-key rule.)
