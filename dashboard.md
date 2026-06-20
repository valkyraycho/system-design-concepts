---
type: dashboard
status: active
created: 2026-06-11
updated: 2026-06-13
tags: [system-design, progress]
---

# Dashboard

**Current position:** 🎉 **ARC 3 COMPLETE + REP 3 DONE** ([[rep3-garage-to-planet]]). **Next: Arc 4 — Operations, Module 14 (deployment & release engineering).** FINAL ARC.

> Note: vault rebuilt 2026-06-13 after the original `~/learning-vault` was lost (home path changed `/Users/ray_cho` → `/Users/raycho`). All concept/case-study notes reconstructed from session context. Obsidian will regenerate its own `.obsidian/` config when the folder is opened as a vault.

## Progress

| Arc | Modules | Status |
|-----|---------|--------|
| 1 — Foundations | 1–3 | ✅ COMPLETE (M1, M2, M3) — Rep pending |
| 2 — Resilience | 4–9 | ✅ COMPLETE (M4–M9) — Rep 2 pending |
| 3 — Scaling | 10–13 | ✅ COMPLETE (M10–M13) — Rep 3 pending |
| 4 — Operations | 14–17 | ⬜ not started |

## Recall queue (concepts with earned notes)

- [[correlated-failure]] — 2/2 strong (k8s logical-layer walk unprompted)
- [[swiss-cheese-model]] — strong
- [[time-diversity]] — 2/2 strong (rediscovered via rolling-update answer)
- [[off-serving-path-failures]] — strong (failover-drill retrieval). Re-test ~1 week with new disguise.
- [[retry-storm-and-jitter]] — jitter mechanism earned clean
- [[load-shedding]] — earned, 2 unprompted citations of the front-door fix
- [[metastable-failure]] — earned 2026-06-13 (clean wasted-work answer + asked the right "why 8s?" question → led to queue-wait-not-CPU insight)
- [[gray-failure]] — earned 2026-06-13 quiz (clean differential-observability framing in new costume)
- [[measure-the-work-not-the-worker]] — earned 2026-06-13 quiz (reached outlier detection + bounded ejection; taught the missing "evidence already exists in LB" half). Re-test the "what signal already exists" angle.
- [[utilization-latency-cliff]] — earned 2026-06-13 (M3 U1): nailed variability-as-engine ("all 29 arrive at once"); on the manager scenario, chained queueing→headroom→UX→retries→metastable-survives-reboot UNPROMPTED. Standout cross-module reasoning. Refinements given: ρ/(1−ρ) is queue-wait multiplier not total latency; counter-propose (75-80% + autoscaling) rather than just block.
- [[littles-law]] — earned 2026-06-13 (M3 U2): clean math (50→2000), correctly picked Service A (30s timeout) falls first. λ-vs-L decoupling cleared in quiz.
- [[backpressure-and-capacity]] — earned 2026-06-13 (M3 U3): (a) right OOM mechanism (image-size magnitude off but "items are large" instinct → "queue references not payloads" cure). (b) right UX instinct but proposed denial-not-backpressure ("accept when full + spinner") → taught "must change behavior not presentation" + accept/process split. Capacity: L=40 floor clean, 50≈75% util well-reasoned, reached for autoscaling unprompted (taught reaction-lag caveat).
- [[stateless-services]] — earned 2026-06-13 (M4): clean on sticky-session failures — saw "added servers don't help pinned users + old servers stay hot" (the non-obvious one) and "stateful crash = data loss not reroute". Taught: cattle-not-pets, sticky+autoscaling self-inflicted gray failure.
- [[externalized-state]] — earned 2026-06-13 (M4): JWT vs server-sessions trade-off solid (revoke vs lookup), reached for "use both" unprompted. Taught the sharper microservices win (decentralized verification, no Redis chokepoint) + the access/refresh-token hybrid as the concrete "both". "Can't eliminate state, only move it."
- [[idempotency]] — earned 2026-06-13 (M5): clean classification (caught relative-vs-absolute nuance); **atomicity-vs-idempotency distinction fully resolved** — talked himself through the colleague-is-wrong scenario unprompted (the Rep 1 re-test, CLEARED).
- [[idempotency-keys]] — earned 2026-06-13 (M5): nailed the TOCTOU-in-the-mechanism (SET NX fix, with correct reasoning about Redis per-command vs check-then-act atomicity); reinvented the transactional outbox ("a column indicating email sent") + senior match-rigor-to-stakes instinct. Taught: exactly-once effect = at-least-once + idempotent dedupe, dual-write problem.
- [[circuit-breaker]] — earned 2026-06-13 (M6): full marks on (a) (retries-during-prolonged-outage → metastable, breaker stops it + shields recovering dep). (b) named the fallback menu; waved off "which deps deserve it" → taught the critical-path-vs-enhancement framework (his "defaults vs error" WAS that distinction). "Slow is worse than down."
- [[bulkhead]] — earned 2026-06-13 (M6): (a) clean Little's Law starvation mechanism. (b) jumped straight to OLTP/OLAP separation + named Redshift unprompted ("from experience"). Taught: bulkhead within-system (pool split) vs between-systems (OLTP/OLAP); the 3 directions (breaker=downstream, shed=upstream, bulkhead=internal).
- [[queues-async]] — earned 2026-06-13 (M7): sync/async test solid (3/4; missed photo-resize by lumping accept+process — the M3 split again, re-taught). (b) got the lost-failure-signal problem; taught DLQ + duplicate-delivery. "Adopting a queue is a bundle."
- [[delivery-guarantees-and-ordering]] — earned 2026-06-13 (M7): clean on the ordering-vs-parallelism diagnosis; reached for partition-by-key. **Key choice: instinctively said user_id → corrected to order_id** (narrowest key capturing the constraint; user_id over-constrains + hot-key risk). Taught: exactly-once is a myth at delivery layer.
- [[caching-strategies]] — earned 2026-06-13 (M8): (a) write-through fix right → taught invalidate-vs-update (delete safer). (b) celebrity post = hot-key ✅ (taught content-vs-counters split); **wallet balance = the trap** — walked up to "don't cache" then walked back to "short TTL" → taught read-for-display vs read-for-decision (any TTL>0 on the decision path = double-spend, the Rep oversell in wallet costume).
- [[cache-stampede-and-failure]] — earned 2026-06-13 (M8): (a) stampede mechanism + coalescing fix clean (flagged he didn't know the "how" → taught SET NX lock/leases). (b) cold-cache recovery trap nailed + reached for replication. Taught the 2nd defense (DB load-shedding + gradual warm-up) = reduce-probability AND limit-blast-radius.
- [[replication-rpo-rto]] — earned 2026-06-13 (M9): (a) replication-lag diagnosis clean; "sticky to a replica" → corrected to read-from-primary-after-write (read-your-writes vs monotonic-reads anomaly family). (b) RPO/RTO classification right (payments=RPO, cache=RTO) → upgraded to "regenerable data opts out of durability entirely". Sync/async replication = consistency-vs-latency.
- [[failover-split-brain-sharding]] — earned 2026-06-13 (M9): split-brain (a) fully correct (2-node trap, quorum/fencing, connected to his my-k8s etcd impl). Optimistic-vs-pessimistic formalized (the Rep etcd-CAS connection). Sharding (b): diagnosed hot-shard but **picked event_type → recreates hotspot** (low cardinality + skew); taught high-cardinality + even-distribution + composite keys. (Same trap family as M7 partition-key.)
- [[load-balancing]] — earned 2026-06-13 (M10): (a) round-robin-is-blind + latency-aware = measure-the-work ✅ (taught "symptom is the signal" — slow server holds connections → least-conn diverts; pair w/ health checks for fast-fail). (b) named NGINX router + CDN but led with head-of-line-blocking → taught the primary reasons (edge caching near users + origin shielding); "serve from cheapest place that can correctly serve it."
- [[dns-anycast-service-discovery]] — earned 2026-06-13 (M10): both Q2 clean (DNS-caching-blocks-failover → stable entry point; hardcoded Pod IP → Service DNS FQDN, nailed the format). Asked sharp k8s self-registration question → taught node-self-register vs Pod-platform-discover hybrid. "Address the stable abstraction, not the ephemeral instance."
- [[autoscaling-capacity]] — earned 2026-06-13 (M11): (a) IO-bound/CPU-scaling diagnosis right → sharpened "scale on queue depth = outcome signal downstream of all bottlenecks." (b) reached for scheduled scaling unprompted → added pre-warm-not-just-pre-count + scheduled-is-floor-not-ceiling. "Autoscaling tracks trends not shocks"; reaction-lag debt from M3 paid off.
- [[multi-region]] — earned 2026-06-13 (M12): (a) covered cost/consistency/RTO-RPO/active-active conflicts/partition-by-home → taught "interrogate the requirement (which reason?) not price the request" + "multi-region is a data-model change not infra." (b) read-your-writes-across-regions diagnosis clean (replication lag, no RYW). "CAP/PACELC made physical."
- [[cell-based-architecture]] — earned 2026-06-13 (M12): (a) containment + evacuation right; sharp catch that global/cross-tenant data resists partitioning (taught: replicate→consistency vs shared→shared-fate). (b) **capacity ✅ but missed rehearsal** — evacuation needs headroom AND continuous practice (Chaos Kong = off-serving-path lesson); named "routing" (plumbing) not rehearsal. Re-test the rehearsal half.
- [[scaling-evolution]] — earned 2026-06-13 (M13): (a) **senior answer** on premature-sharding (questioned the why, matched cheaper tools, "one DB does more than juniors think + Aurora") → added the 3 reasons sharding-now is actively risky. (b) skew intuition right ("handle the tiny portion differently") → sharpened the WHY (horizontal scale divides by key, hot key indivisible → machine/key mismatch is structural). Twitter fan-out, YouTube DB-scaling order, Discord engine-match.
- [[setting-timeouts]] — partial 2026-06-13 (timeout scenario). Got Inventory timeout + Little's arithmetic + async instinct. **GAP: derives timeouts per-call instead of top-down from budget** — set BankAPI=3s but parent Order→Payment=200ms (budget violation, 100% failure). Taught the Russian-doll/nested derivation. **Re-test with a fresh call-chain: must derive top-down and leave margin.** This is the #1 habit to seat from M3.
- [[percentiles-vs-averages]] — earned 2026-06-13 (computed the 179ms avg, saw it hid the disaster; reached for stddev → corrected to percentiles)
- [[sli-slo-sla]] — earned 2026-06-13 (video SLI: found quality dimension; rebuffer/start-failure taught)
- [[the-nines-table]] — earned 2026-06-13 (derived 99.9% = 43 min/month unaided)
- [[error-budget]] — earned 2026-06-13 (a: computed 50min > 43min budget → freeze; b: pushed back on "100%" with dependency ceiling + exponential cost + opportunity cost, cited vendor SLA unprompted). 2nd swing needed on (b) — first attempt answered the wrong question (recovery vs target-setting). Re-test the "argue against 100%" framing later.

## Weak spots (open)

*(none)*

## Weak spots (cleared)

- ~~Off-serving-path failures~~ — cleared 2026-06-11 (kill primary forcefully in clone).
- ~~Logical vs physical correlation~~ — cleared 2026-06-11 (all k8s answers logical).
- ~~Jitter~~ — earned 2026-06-11 (shared outage = synchronizer).
- ~~Peer-relative vs absolute thresholds / bounded automation~~ — **cleared 2026-06-13**: given a fresh costume (latency-based reboot bot), unprompted produced shared-dependency → fleet-wide reboot → outage, and reached for "is it just me?" peer evaluation. Transfer succeeded. (Suspenders reminder given: hard-cap action blast radius, e.g. max_ejection_percent, in addition to peer detection.)
- ~~Top-down timeout budgeting~~ — **cleared 2026-06-13**: fresh call-chain (mobile feed, 2s budget), derived all three timeouts correctly nested (150→350→500ms, each contains its children, fits budget) unprompted. Also intuited budget slack = retry headroom. λ-vs-L decoupling also cleared same quiz (flat-traffic pool-exhaustion scenario).
- ~~Percentile-tail + per-customer segmentation~~ — **cleared 2026-06-13** (2nd quiz): green-real-metrics + furious 60%-traffic customer → reached for aggregate-percentile-hides-tail + segmented SLIs, NOT gray-failure. "Aggregation is a form of hiding." Correction from prior quiz transferred.
- ~~Partition-key choice (user_id→order_id)~~ — **confirmed transferred 2026-06-13**: match-events scenario, picked match_id + correctly named too-coarse (player) / too-fine (event) failures.

## Rep 2 edges (re-test in future quizzes)

- **Match named failure to mechanism present** — assumed split-brain without confirming auto-failover; certain risk was async data-loss/RTO. Re-test: give a 2-node setup, ask risks — should ask "auto-failover?" before naming split-brain. (STILL OPEN)
- **async ≠ fire-and-forget (payments)** — money needs exactly-once effect, never "forget" the result. Re-test in a payment-async scenario. (STILL OPEN)
- ~~immediate vs durable = sort by time-to-effect~~ — **CLEARED 2026-06-13** (quiz Q2: deploy-bug startup crash → rollback now / fix durable, correctly killed both distractors incl. "more replicas can't help a startup crash").
- ~~never destroy state to "reset"~~ — **CLEARED 2026-06-13** (quiz Q3: Kafka delete-topic trap — named data+metadata+offset-identity loss; added consumer-side fix). Transferred from Redis costume to Kafka costume.

## Concepts taught but not yet tested (quiz later)

- **Query of death** — LB/retry walks a poison request through every replica (taught 3×; quiz directly)
- **Retry budget** — cap retries as % of traffic (taught; Q3 2026-06-13 he named backoff+jitter+cap unprompted but not "budget as % of traffic" specifically — quiz that phrasing)
- ~~Circuit breaker~~ — **taught & earned 2026-06-13 (M6)**, see recall queue.
- ~~Idempotency vs atomicity~~ — **CLEARED 2026-06-13 (M5)**: resolved the distinction unprompted, including the colleague-is-wrong scenario.
- ~~TOCTOU / race needs atomicity~~ — **CLEARED 2026-06-13 (M5)**: identified TOCTOU inside the idempotency mechanism and reached for SET NX / unique-constraint insert unprompted. (3 costumes seen: oversell, idem-key race, GET-then-SET.)
- ~~Optimistic vs pessimistic locking~~ — **taught & earned 2026-06-13 (M9)**: formalized as a bet on contention rate; livelock risk for optimistic under high contention. See [[failover-split-brain-sharding]].
- ~~RPO/RTO~~ — **taught & earned 2026-06-13 (M9)**, see [[replication-rpo-rto]].
- **Instrument the business question** — Rep 1: undersold custom business SLIs (sell-through, checkout-completion, oversell=0) as "non-technical." Reinforce in Module 16 (observability).
- **Error attribution** — count only server-attributable (5xx/timeout), not 4xx — defangs the malicious-input ejection attack
- **Recovery happens under fire** — recovery path must be load-tested (links to off-serving-path)
- **Evacuation needs rehearsal (not just routing/capacity)** — M12 Q2b: got headroom, missed "continuously practice the evacuation or it rots" (Chaos Kong / off-serving-path). Re-test: "what makes a failover capability trustworthy when you need it?"
- **RPO / RTO** — formally introduced 2026-06-13 in quiz (daily snapshot = 24h RPO; restore time = RTO). Full treatment in Module 9. Re-test the data-loss-vs-availability distinction.
- **Decompose-first / sync-async boundary** — 2026-06-13 (M7 + quiz): twice swept a must-be-sync step in with async ones (photo-resize lumping; "store original" in video upload). Boundary to seat: sync line = "is the source of truth durable yet / did I keep my promise?", NOT just "does user wait?". Improving; re-test once more.

## Session log

- 2026-06-11 · Learn · M1 U1: anatomy of cascading failure — Facebook 2021 + GitLab 2017. Concepts: correlated failure, Swiss cheese, time diversity. Highlight: compatibility pushback on software diversity was the right objection.
- 2026-06-11 · Recall · Both U1 weak spots cleared. Taught query-of-death.
- 2026-06-11 · Recall+Learn · U2 (S3 2017): jitter + load-shedding earned, metastable → weak spot. U3 taught: gray failures, measure-work-not-worker, peer-relative ejection; struggled absolute-vs-peer → weak spot.
- 2026-06-13 · Recall · Peer-relative/bounded-automation weak spot CLEARED via reboot-bot transfer. Metastable then CLEARED too (wasted-work answer + "why 8s?" → queue-wait insight). Vault rebuilt after directory loss. **Module 1 fully complete, zero open weak spots.** Earned: metastable-failure note.
- 2026-06-13 · Learn · M2 COMPLETE: percentiles vs averages (179ms hides the tail), SLI/SLO/SLA triad, nines table (derived 43min unaided), video SLI, error budgets (computed freeze scenario; argued down "100%" with a vendor-SLA receipt). Earned 4 notes. Next: Module 3.
- 2026-06-13 · Recall · 3-Q quiz. Gray failure ✅ earned (clean). "Too reliable is a bug" ✅ earned (added: dependents silently depend on actual reliability). Correlation/blast-radius ✅ re-confirmed (same-region snapshots). Gaps: measure-work-not-worker (had mechanism, not the "evidence already exists" insight — taught); RPO/restore-testing (drifted to availability instead of data-loss — taught, full in M9). Earned notes: gray-failure, measure-the-work-not-the-worker.
- 2026-06-13 · Learn · M3 U1 (cliff) + U2 (Little's Law). Standout session: chained 4 modules unprompted on the manager scenario; clean L=λW math + timeout reasoning. Earned: utilization-latency-cliff, littles-law. Then taught setting-timeouts via checkout scenario (budget-violation trap → top-down derivation).
- 2026-06-13 · Recall · 3-Q quiz. **Top-down timeout budgeting CLEARED** (mobile-feed chain, correctly nested unprompted — the #1 M3 habit). λ-vs-L decoupling CLEARED (flat-traffic exhaustion). Timeout+retry Q3: nailed retry-storm half, taught circuit-breaker half.
- 2026-06-13 · Learn · **M3 U3 (backpressure & capacity) → ARC 1 COMPLETE.** Bounded-vs-unbounded queues, where pressure flows (→client), denial-trap, upload accept/process split, L=λW capacity (40 floor → ~53 at 75%), autoscaling reaction-lag. Earned: backpressure-and-capacity.
- 2026-06-13 · **REP 1 — "The On-Sale"** (ticketing flash-sale, Arc 1 capstone). Strong: baseline-first, contention-not-CPU, self-rejected batch idea, reinvented Redis-inventory, etcd↔SQL optimistic-locking transfer. Edges (→ re-test queue): check-before-write=the race (TOCTOU), idempotency≠atomicity, slow-rearchitect under time pressure, undersold business-metric instrumentation. Wrote [[rep1-the-on-sale]].
- 2026-06-13 · Learn · **M4 COMPLETE (stateless services & externalized state)** → Arc 2 begun. Sticky-session failures, cattle-not-pets, "can't eliminate state only move it", state taxonomy (token/Redis/DB), JWT vs sessions + access/refresh hybrid. Earned: stateless-services, externalized-state.
- 2026-06-13 · Learn · **M5 COMPLETE (timeouts/retries/idempotency).** Standout: resolved idempotency-vs-atomicity (Rep 1 debt) unprompted; caught TOCTOU inside the idem mechanism (SET NX); reinvented transactional outbox; "exactly-once effect = at-least-once + dedupe". Two Rep re-tests CLEARED. Earned: idempotency, idempotency-keys.
- 2026-06-13 · Learn · **M6 COMPLETE (circuit breakers, bulkheads, load shedding).** Circuit breaker state machine + "slow is worse than down" + critical-vs-enhancement framework. Bulkhead → OLTP/OLAP (jumped to it unprompted). 3 directions of failure isolation. Load shedding folded in (already owned from M1). Earned: circuit-breaker, bulkhead.
- 2026-06-13 · Recall · 4-Q mixed Arc 2 quiz. Q1 statelessness ✅ (taught WebSocket connection-vs-state). Q2 idempotency ✅✅ (both mechanisms + added "naturally idempotent"). Q3 circuit-breaker/bulkhead ✅ (sharpened bulkhead = own pool + degrade coda; "slow worse than down" retrieved). Q4 ⚠️ reached for gray-failure instead of percentile-tail/segmentation → new weak spot.
- 2026-06-13 · Learn · **M7 COMPLETE (queues & async).** 4 decoupling benefits, "does caller need result to continue?" test, sync/async per-sub-operation, the bundle (queue+idempotent+DLQ+pending UX), 3 delivery guarantees ("exactly-once achieved not delivered"), partition-key ordering (user_id→order_id correction). Earned: queues-async, delivery-guarantees-and-ordering.
- 2026-06-13 · Recall · 4-Q quiz. Q1 percentile/segmentation weak spot **CLEARED** (reached for tail+segmented SLIs, not gray-failure). Q2 ⚠️ over-async'd "store original" → sharpened sync boundary = "source of truth durable yet?" (decompose-first now a tracked weak spot). Q3 double-charge ✅✅. Q4 partition-key (match_id) ✅ correction transferred.
- 2026-06-13 · Learn · **M8 COMPLETE (caching).** Cache-as-shield, cache-aside, invalidation (TTL/write-through/write-behind; delete>update), read-for-display vs read-for-decision (wallet trap), stampede + leases/SET NX coalescing, cache-death + cold-recovery trap + defense-in-depth. Earned: caching-strategies, cache-stampede-and-failure.
- 2026-06-13 · Learn · **M9 COMPLETE → ARC 2 RESILIENCE COMPLETE.** RPO/RTO, sync/async replication, read-consistency anomaly family, failover/split-brain/quorum/fencing (2-node trap; connected to his my-k8s etcd), optimistic-vs-pessimistic locking, hot-partition shard keys (event_type trap). Earned: replication-rpo-rto, failover-split-brain-sharding.
- 2026-06-13 · **REP 2 — "The Thundering Monday"** (Arc 2 capstone). Strong: found all 3 latent risks + ranked worst; clean slow-payment→pool-exhaustion→browsing-frozen cascade; Redis-LRU eviction dual-failure mechanism; sensed flush danger. Edges (→ re-test): name-failure-to-mechanism (split-brain assumed), async≠fire-and-forget, immediate-vs-durable sorting, never-destroy-state. Wrote [[rep2-thundering-monday]].
- 2026-06-13 · Learn · **M10 COMPLETE (load balancing & traffic mgmt)** → Arc 3 begun. L4/L7, balancing algorithms (round-robin blind vs least-conn "symptom is the signal"), static-vs-dynamic + CDN/edge, DNS/anycast/service-discovery, k8s node-vs-Pod registration hybrid. Earned: load-balancing, dns-anycast-service-discovery.
- 2026-06-13 · Learn · **M11 COMPLETE (autoscaling & capacity).** Reaction-lag (tracks trends not shocks), signal problem (scale on queue-depth/outcome not CPU-proxy), cold-start-deepens-outage, scale-down traps (flapping/hysteresis/scale-to-zero), scheduled+pre-warm. Earned: autoscaling-capacity.
- 2026-06-13 · Recall · 4-Q quiz (strongest yet). Q1 ✅ "utilization≠keeping up, queue depth is the only keep-up signal" (even when CPU is the right resource). Q2 ✅ **Rep2 edge CLEARED** (rollback/fix + killed both distractors). Q3 ✅ **Rep2 edge CLEARED** (Kafka delete-topic trap, transferred from Redis costume). Q4 ✅ round-robin + load-aware; added outlier-ejection/deep-health-check for the gray-failure framing. 2 Rep2 edges remain (match-failure-to-mechanism, async≠fire-and-forget).
- 2026-06-13 · Learn · **M12 COMPLETE (multi-region & HA).** 3 reasons (DR/latency/compliance), speed-of-light wall = CAP/PACELC made physical, active-passive vs active-active vs partition-by-home, eventual-consistency-leaks-into-UX, cell-based architecture (blast radius 1/N, cell-canary, data-that-won't-partition), regional evacuation needs headroom + rehearsal (Chaos Kong). Earned: multi-region, cell-based-architecture.
- 2026-06-13 · Learn · **M13 COMPLETE → ARC 3 SCALING COMPLETE.** Premature-scaling thesis, Twitter fan-out read/write/hybrid (celebrity=hot key), YouTube DB-scaling order (reads easy/writes hard, shard=last resort), Discord engine-match, skew = universal enemy of horizontal scale (key indivisible). Earned: scaling-evolution.
- 2026-06-13 · **REP 3 — "From Garage to Planet"** (Arc 3 capstone, grow-the-system format). Stage 1 (Day 1): velocity-vs-scale line drawn well (taught CDN≠premature, simple≠naive). Stage 2 (200K read bottleneck): correct cheapest-first ordering (vertical→replica→cache), flagged fan-out as premature. Stage 3 (40M): sharding key tension, partition-by-home > active-active, hybrid fan-out — **ASKED "is fan-out fair at this stage?" = the Module 13 judgment, now internalized (vs Rep 1's premature reach).** Wrote [[rep3-garage-to-planet]]. **34 notes + 3 case studies + 3 reps. Next: Arc 4 / M14 (deployment) — FINAL ARC.**
