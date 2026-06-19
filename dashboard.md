---
type: dashboard
status: active
created: 2026-06-11
updated: 2026-06-13
tags: [system-design, progress]
---

# Dashboard

**Current position:** Arc 2 — Resilience. **Module 6 (circuit breakers, bulkheads, load shedding) ✅ COMPLETE.** Next: Module 7 (queues & async as reliability tools).

> Note: vault rebuilt 2026-06-13 after the original `~/learning-vault` was lost (home path changed `/Users/ray_cho` → `/Users/raycho`). All concept/case-study notes reconstructed from session context. Obsidian will regenerate its own `.obsidian/` config when the folder is opened as a vault.

## Progress

| Arc | Modules | Status |
|-----|---------|--------|
| 1 — Foundations | 1–3 | ✅ COMPLETE (M1, M2, M3) — Rep pending |
| 2 — Resilience | 4–9 | 🟡 M4 ✅, M5 ✅, M6 ✅; M7 next |
| 3 — Scaling | 10–13 | ⬜ not started |
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
- [[setting-timeouts]] — partial 2026-06-13 (timeout scenario). Got Inventory timeout + Little's arithmetic + async instinct. **GAP: derives timeouts per-call instead of top-down from budget** — set BankAPI=3s but parent Order→Payment=200ms (budget violation, 100% failure). Taught the Russian-doll/nested derivation. **Re-test with a fresh call-chain: must derive top-down and leave margin.** This is the #1 habit to seat from M3.
- [[percentiles-vs-averages]] — earned 2026-06-13 (computed the 179ms avg, saw it hid the disaster; reached for stddev → corrected to percentiles)
- [[sli-slo-sla]] — earned 2026-06-13 (video SLI: found quality dimension; rebuffer/start-failure taught)
- [[the-nines-table]] — earned 2026-06-13 (derived 99.9% = 43 min/month unaided)
- [[error-budget]] — earned 2026-06-13 (a: computed 50min > 43min budget → freeze; b: pushed back on "100%" with dependency ceiling + exponential cost + opportunity cost, cited vendor SLA unprompted). 2nd swing needed on (b) — first attempt answered the wrong question (recovery vs target-setting). Re-test the "argue against 100%" framing later.

## Weak spots (open)

- **Percentiles/tail + per-customer segmentation vs gray-failure** — 2026-06-13 quiz Q4: given honest green metrics (99.95%, p99 250ms) + furious heavy customer, he reached for "shallow health check / gray failure" (wrong house — metrics were real, not a /healthz ping). Right answer: p99 hides how bad the slow 1% is + the highest-VOLUME customer lives in the tail + global aggregation hides the bad segment. "Right neighborhood, wrong house." Re-test: green real-request metrics + one unhappy segment → must reach for tail/percentile + segmentation, NOT probe representativeness.

## Weak spots (cleared)

- ~~Off-serving-path failures~~ — cleared 2026-06-11 (kill primary forcefully in clone).
- ~~Logical vs physical correlation~~ — cleared 2026-06-11 (all k8s answers logical).
- ~~Jitter~~ — earned 2026-06-11 (shared outage = synchronizer).
- ~~Peer-relative vs absolute thresholds / bounded automation~~ — **cleared 2026-06-13**: given a fresh costume (latency-based reboot bot), unprompted produced shared-dependency → fleet-wide reboot → outage, and reached for "is it just me?" peer evaluation. Transfer succeeded. (Suspenders reminder given: hard-cap action blast radius, e.g. max_ejection_percent, in addition to peer detection.)
- ~~Top-down timeout budgeting~~ — **cleared 2026-06-13**: fresh call-chain (mobile feed, 2s budget), derived all three timeouts correctly nested (150→350→500ms, each contains its children, fits budget) unprompted. Also intuited budget slack = retry headroom. λ-vs-L decoupling also cleared same quiz (flat-traffic pool-exhaustion scenario).

## Concepts taught but not yet tested (quiz later)

- **Query of death** — LB/retry walks a poison request through every replica (taught 3×; quiz directly)
- **Retry budget** — cap retries as % of traffic (taught; Q3 2026-06-13 he named backoff+jitter+cap unprompted but not "budget as % of traffic" specifically — quiz that phrasing)
- ~~Circuit breaker~~ — **taught & earned 2026-06-13 (M6)**, see recall queue.
- ~~Idempotency vs atomicity~~ — **CLEARED 2026-06-13 (M5)**: resolved the distinction unprompted, including the colleague-is-wrong scenario.
- ~~TOCTOU / race needs atomicity~~ — **CLEARED 2026-06-13 (M5)**: identified TOCTOU inside the idempotency mechanism and reached for SET NX / unique-constraint insert unprompted. (3 costumes seen: oversell, idem-key race, GET-then-SET.)
- **Optimistic vs pessimistic locking** — Rep 1: connected etcd CAS unprompted (strong). Conditional UPDATE = optimistic; FOR UPDATE = pessimistic; optimistic better on hot rows (no held lock). Module 9 deepens.
- **Instrument the business question** — Rep 1: undersold custom business SLIs (sell-through, checkout-completion, oversell=0) as "non-technical." Reinforce in Module 16 (observability).
- **Error attribution** — count only server-attributable (5xx/timeout), not 4xx — defangs the malicious-input ejection attack
- **Recovery happens under fire** — recovery path must be load-tested (links to off-serving-path)
- **RPO / RTO** — formally introduced 2026-06-13 in quiz (daily snapshot = 24h RPO; restore time = RTO). Full treatment in Module 9. Re-test the data-loss-vs-availability distinction.

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
- 2026-06-13 · Recall · 4-Q mixed Arc 2 quiz. Q1 statelessness ✅ (taught WebSocket connection-vs-state). Q2 idempotency ✅✅ (both mechanisms + added "naturally idempotent"). Q3 circuit-breaker/bulkhead ✅ (sharpened bulkhead = own pool + degrade coda; "slow worse than down" retrieved). Q4 ⚠️ reached for gray-failure instead of percentile-tail/segmentation → new weak spot. Next: M7 (queues & async).
