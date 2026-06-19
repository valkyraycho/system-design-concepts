---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, queues, ordering, delivery, kafka, arc2, module7]
related: ["[[queues-async]]", "[[idempotency-keys]]", "[[externalized-state]]", "[[bulkhead]]"]
---

# Delivery guarantees & ordering

## Three delivery guarantees

1. **At-most-once** — may lose, never duplicates. For data where loss is OK but dupes aren't (metrics samples). Rare in business logic.
2. **At-least-once** — may duplicate, never loses. **The default** (SQS, RabbitMQ, Kafka). Cause: consumer processes → must *ack* so queue deletes; crash after-process-before-ack (or lost ack) → **redeliver**. Better twice than lost.
3. **Exactly-once** — mostly a **myth at the delivery layer** (same two-generals impossibility as the double-charge). What's marketed as exactly-once = **at-least-once + idempotent processing** + transactional dedupe under the covers.

**"Exactly-once is achieved, never delivered."** Assume at-least-once; manufacture exactly-once *effect* via an idempotent consumer. "We use exactly-once queues so we don't need idempotency" = the same bug as the locked-but-not-idempotent colleague. Idempotency is load-bearing structure for async, not a safety net.

## Ordering vs scaling

Adding consumers (scaling benefit) can **lose ordering**: consumer B finishes msg 2 before consumer A finishes msg 1. Fine for order-independent work (welcome emails); a correctness bug for causally dependent events (deposit/withdraw; order lifecycle).

Resolution: **you don't need *global* order, only order within a related group.** Partition by a **key** → same key → same partition → consumed in-order by one consumer; different keys → parallel. = **per-key ordering + cross-key parallelism** (Kafka). Same "find the axis where both properties hold" move as access/refresh tokens, OLTP/OLAP.

## Choosing the partition key (the consequential decision)

**Narrowest identifier that still captures every ordering constraint — no narrower, no wider.**
- **Too wide** (`user_id` for order events): correct but over-constrained — a user's 50 orders forced serial through one partition; **hot-key bottleneck** for whales (Rep hot-row shape).
- **Too narrow** (`event_type`): breaks needed ordering (placed/delivered split apart).
- **Just right** (`order_id` for order-lifecycle events): each order's events ordered; different orders (even same user) parallel.

The cost of ordering is **not fixed — it's set by the key.** Fine-grained + evenly distributed → ordering nearly free; coarse/skewed → hot-partition bottleneck. Ask "what's the actual unit of ordering?" — usually finer than the obvious top-level entity. (Ray's instinct `user_id` → corrected to `order_id`: a user owns many independently-ordered orders.)

## Case studies
Slack queue redesign → Kafka for durable buffering + partitioned ordered processing + independent scaling. WhatsApp → ~900M users/small team on Erlang's native mailbox-per-process model ("queues all the way down").
