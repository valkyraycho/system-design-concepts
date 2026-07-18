---
type: case-study
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [case-study, discord, cassandra, scylladb, lsm-tree, gray-failure, real-world]
related: ["[[scaling-evolution]]", "[[delivery-guarantees-and-ordering]]", "[[failover-split-brain-sharding]]", "[[gray-failure]]", "[[measure-the-work-not-the-worker]]", "[[instagram-scaling]]"]
---

# Discord: MongoDB → Cassandra → ScyllaDB, trillions of messages

Two migrations of their most critical dataset (public posts: "How Discord Stores Billions of Messages" 2017, "...Trillions of Messages" 2022).

## Stage 1: storage engine match (Ray: append-only-log instinct, corrected SQLite→LSM-tree)

Message shape: enormous write volume, append-mostly, recency-biased reads, naturally grouped by channel. Ray's instinct: "mostly append only" = DDIA's opening "world's simplest database" (append-only log). Right ancestor, wrong modern example (SQLite is actually **B-tree** based, not append-log).

**Real mechanism = LSM-tree** (Cassandra/ScyllaDB): writes → in-memory sorted **memtable** → flushed as immutable sorted **SSTable** (still sequential = cheap) → background **compaction** merges/cleans. Reads use sorted structure for range scans. Contrast **B-tree** (SQLite/Postgres/MySQL default): in-place updates → random seeks even for pure inserts — fine at moderate volume, bottleneck at Discord's firehose. LSM turns every write into an append; sorted SSTables make "last N messages in this channel" a cheap sequential range scan, not a random lookup.

## Stage 2: hot partition → composite key (Ray independently derived it)

Pure `channel_id` partitioning → a busy channel's partition **grows unbounded forever** (operational nightmare: compaction, uneven load). Ray's fix, self-derived: add a **time-bucket** to the key + bonus insight "older messages less accessed."

**Real fix: composite key `(channel_id, time_bucket)`.** Bounded-size partitions per time window instead of one ever-growing partition. **Third costume of the same key-design principle** (M7 Kafka: `order_id` not `user_id`; M9 sharding: `event_type` too coarse) — key must be neither too coarse (loses the grouping) nor too fine (unbounded/breaks locality). Bonus: aligns with the read pattern too — recent bucket = small, hot, cacheable; old buckets = cold, cheap storage tier (hot/cold tiering, same "match resources to actual access frequency" instinct as decision-vs-display caching, applied to storage location over time).

**Full-circle callback:** bucket is computed directly from the message's **Snowflake-style time-sortable ID** — the exact same ID-scheme idea Ray independently derived for [[instagram-scaling]], now reused to make bucketing free (no extra column/index).

## Stage 3: Cassandra → ScyllaDB (Ray recalled the gray-failure callback correctly)

Cassandra (JVM) → occasional unpredictable latency spikes, worse as cluster/data grew. Ray connected it: **stop-the-world GC pause** = the exact GC-death-spiral gray-failure example from Module 1/[[measure-the-work-not-the-worker]] — during a pause the process executes NO code; a cheap health check landing in an "alive window" passes while a real query straddling the pause takes an enormous unpredictable hit = **differential observability** (gray failure), and it corrupts tail latency specifically (median fine, unlucky p99.9 requests hit hard).

**ScyllaDB fix = C++, no garbage collector** (+ shard-per-core, message-passing not shared locks — bulkhead-adjacent). Key distinction: **managing a hazard (GC tuning: G1/ZGC, heap sizing) vs eliminating its category entirely** (no GC process to ever pause). Same spectrum as tuning a timeout down (reduce probability) vs removing a shared pool (bulkhead — make it structurally impossible). Structural fixes cost more (rewrite a whole DB!) but buy a guarantee instead of a probability — worth it when "rare" pauses still happen often enough in absolute terms at extreme scale.

Confirmed industry pattern: Rust/C++/Go's low-pause collector increasingly chosen for latency-sensitive infra (DBs, proxies, CDNs, LBs) for exactly this reason — predictable GC-pause-free tail latency is a competitive property at scale.
