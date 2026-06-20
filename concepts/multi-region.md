---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [scaling, multi-region, consistency, cap, ha, arc3, module12]
related: ["[[replication-rpo-rto]]", "[[failover-split-brain-sharding]]", "[[dns-anycast-service-discovery]]", "[[error-budget]]", "[[cell-based-architecture]]"]
---

# Multi-region & high availability

## Three reasons (don't conflate — they pull toward different designs)
1. **Disaster recovery** — survive a whole region dying (availability/blast-radius).
2. **Latency** — serve users near them, for *dynamic* data (speed-of-light: Sydney↔Virginia ≥ ~200ms RTT, physics).
3. **Data residency/compliance** — GDPR etc. require data to stay in a region.

## The wall: speed of light makes cross-region consistency expensive
Inter-region latency is 100ms+ and that's **physics, not engineering.** Every write across regions:
- **Sync replicate** → +200ms/write, can't write if other region unreachable. Strong consistency, awful latency+availability.
- **Async replicate** → fast local writes, but regions temporarily inconsistent + region failure loses un-replicated writes (RPO>0). Eventual consistency.

**CAP/PACELC made physical.** Single DC: replication sub-ms, so strong consistency + low latency both possible (trade barely bites). Cross-region: latency so large the trade is unavoidable+painful. PACELC: during Partition choose A or C; **Else** (normal!) choose Latency or Consistency. Multi-region forces the "Else" on *every write* even with no failure. The defining hard problem of distributed data (DDIA).

## Topologies

**Active-passive** — region A active, B warm standby (async replicated). Simpler (single writer, no conflicts). Cost: B mostly idle, failover RTO (+ DNS/anycast lag M10), async → RPO.

**Active-active** — all regions take reads+writes. Better latency + resource use; region failure shifts traffic. **Hard part: write conflicts** (multi-master — last-write-wins/CRDT/app-merge, DDIA).

**The escape hatch (find the axis for both):** **partition by home region** — each datum owned by one region (all its writes go there) → single writer per datum (no conflicts) while the *system* is active-active (both regions write, different users). Cost: traveling users / truly-global data still hit the wall.
- active-passive: efficiency/latency ↔ simplicity. active-active: simplicity ↔ efficiency/latency. partition-by-home: most of both.

## Don't price the request — interrogate the requirement (Ray Q1a)
"Make us multi-region for reliability" conflates goal (survive region failure) with solution (multi-region). **First question: for WHICH reason — DR / latency / compliance?** Pure DR may be met far cheaper by cross-region backups + DR runbook or active-passive — not full active-active. (Same "trigger vs root cause" / "argue against 100%" discipline.) Then the killer cost the CEO can't see: **multi-region is a data-model + application-logic change, not an infra change** — every "read my write instantly / one true value" assumption must be re-examined. Not just the 2nd region's bill.

## Eventual consistency leaks into UX (Ray Q1b)
Active-active US/EU: user updates setting, refresh shows old value, "fixes itself" in seconds = **read-your-writes anomaly** ([[replication-rpo-rto]]) at cross-region scale — worse because lag is the 100ms+ light tax (human-perceptible), not sub-ms.
Fixes (multi-region flavor): **sticky routing to the region that took the write** (partition-by-home helps), or **session token carrying last-write version** → read waits for local replica to catch up.
**The consistency model becomes a UX you must design** (read-your-writes locally, or optimistically show the user their own change). Decide the eventual-consistency UX *before* choosing active-active.
