---
type: dashboard
status: active
created: 2026-06-11
updated: 2026-06-13
tags: [system-design, progress]
---

# Dashboard

**Current position:** 🎉 **ARC 1 — FOUNDATIONS COMPLETE** (Modules 1, 2, 3 all done). **Next: the first REP** — a case-study-grounded, evolving-incident design drill covering Arc 1. Then Arc 2 (Resilience), starting Module 4 (stateless services & externalized state).

> Note: vault rebuilt 2026-06-13 after the original `~/learning-vault` was lost (home path changed `/Users/ray_cho` → `/Users/raycho`). All concept/case-study notes reconstructed from session context. Obsidian will regenerate its own `.obsidian/` config when the folder is opened as a vault.

## Progress

| Arc | Modules | Status |
|-----|---------|--------|
| 1 — Foundations | 1–3 | ✅ COMPLETE (M1, M2, M3) — Rep pending |
| 2 — Resilience | 4–9 | ⬜ not started |
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

## Concepts taught but not yet tested (quiz later)

- **Query of death** — LB/retry walks a poison request through every replica (taught 3×; quiz directly)
- **Retry budget** — cap retries as % of traffic (taught; Q3 2026-06-13 he named backoff+jitter+cap unprompted but not "budget as % of traffic" specifically — quiz that phrasing)
- **Circuit breaker** — previewed 2026-06-13 (Q3): timeout+retry without a breaker keeps every request waiting on a dead dep, burning L. Full treatment Module 6.
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
- 2026-06-13 · Learn · **M3 U3 (backpressure & capacity) → ARC 1 COMPLETE.** Bounded-vs-unbounded queues, where pressure flows (→client), denial-trap, upload accept/process split, L=λW capacity (40 floor → ~53 at 75%), autoscaling reaction-lag. Earned: backpressure-and-capacity. **14 concept notes + 3 case studies. Next: first REP drill, then Arc 2 / Module 4.**
