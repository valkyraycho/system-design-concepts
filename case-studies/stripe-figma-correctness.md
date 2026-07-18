---
type: case-study
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [case-study, stripe, figma, correctness, ledger, crdt, real-world]
related: ["[[idempotency-keys]]", "[[idempotency]]", "[[observability]]", "[[failover-split-brain-sharding]]", "[[multi-region]]", "[[discord-message-storage]]"]
---

# Correctness at scale: Stripe's ledger & Figma's multiplayer

Two different flavors of "wrong is unacceptable" — money conservation (mutual-exclusion-adjacent) vs collaborative merge (the opposite of mutual exclusion).

## Thread 1: Stripe — double-entry bookkeeping as a structural invariant

Idempotency keys (M5) solve ONE problem: don't double-charge on retry. They don't catch a **logical bug** (race condition, partial failure) silently creating/destroying money across hundreds of internal services.

**Ray's instinct: transaction log / WAL** — right family (append-only, same DDIA idea as [[discord-message-storage]]'s LSM-tree), wrong tool for this gap. A log records **what happened**, not **whether it was correct** — it faithfully replays a bug's mistake; replaying doesn't catch it. WAL = crash recovery; this problem needs to catch **logical** corruption with no crash at all.

**Real mechanism: double-entry bookkeeping as a technical (not just accounting) pattern.** Every financial event = ≥2 balanced entries (debit + credit) that **sum to exactly zero**, appended immutably. Structural consequence: money can only *move* between accounts, never be created/destroyed silently. → **continuously-checkable, system-wide invariant**: sum every ledger entry across the whole system; must always = 0. Mismatch → instant, often precise ("which account") alarm — corruption caught **whatever caused it**, no need to have predicted the specific bug.

**Direct callback to Rep 1 / [[observability]] silent-2xx debt**: every API call can succeed, every dashboard green, books still don't balance — the bug isn't in any single request, it's in the aggregate. Continuous reconciliation = the "orders per minute" business metric, but airtight by mathematical construction rather than merely correlated with reality. Fintech's most sensitive "is the business actually correct" signal.

## Thread 2: Figma — the OPPOSITE correctness problem

Everything so far (Rep 1 seat lock, row locks, CAS) = **mutual exclusion**: ensure exactly ONE of N competitors wins. Multiplayer editing needs the opposite: multiple simultaneous edits should **both survive and merge**, not pick a winner and discard the loser.

**Ray's instinct: local-first + background sync** — correct, exactly the real architecture (optimistic local edit for responsiveness, reconcile with server/other clients). Correctly recalled from DDIA's multi-writer/Google-Docs-style discussion.

**Ray's key insight, stated as uncertainty but actually correct: for a true conflict (same object, same property, same instant) there IS no "correct" merged answer** (red vs blue — no meaningful blend). **The resolution: redefine the goal from CORRECTNESS to CONVERGENCE.** CRDTs (Conflict-free Replicated Data Types) don't find the objectively right answer — they guarantee every replica computes the **SAME** final state regardless of delivery order, with zero coordination.

**Mechanism — 3 properties on every edit operation:**
- Commutative (order doesn't matter)
- Associative (grouping doesn't matter)
- **Idempotent** (apply twice = apply once — [[idempotency]], M5, now protecting against duplicate delivery of a collaborative edit over a flaky real-time connection, not a retried request)

Together: however scrambled the delivery order, once two clients have seen the same *set* of edits, they land on the identical state. Most edits (different objects/properties) have **zero real conflict** — trivial commutative merge. True conflicts get a **deterministic tie-break** (logical clock + actor-ID fallback) — every replica computes the *identical* winner independently, no negotiation, no merge-conflict dialog ever shown to the user (sharp contrast with Git, which surfaces conflicts to humans because it can't auto-resolve).

Figma's real system: CRDT-*inspired* custom design (not textbook, adapted to a nested-object tree not flat text) + a central sequencing server + optimistic local client state. Design discipline, not a specific library: make operations commutative + idempotent so correctness doesn't depend on message ordering/delivery guarantees you can't fully control — same Arc 2 instinct (design for failure modes you can't prevent), applied to concurrent editing instead of server failure.
