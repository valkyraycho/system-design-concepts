---
type: case-study
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [case-study, netflix, cdn, chaos-engineering, peering, real-world]
related: ["[[load-balancing]]", "[[circuit-breaker]]", "[[incident-response-postmortems-chaos]]", "[[cell-based-architecture]]", "[[stateless-services]]"]
---

# Netflix: born from an outage, built to break itself

## Origin story
2008: major database corruption → 3-day outage (still primarily DVD-by-mail). Monolith + central relational DB = single point of failure. **The migration to AWS + microservices was driven by resilience, not scale** — never again let one component's failure take down everything (blast radius, M1/M6). Also the root of Netflix's obsession with *proving* resilience empirically rather than hoping for it → chaos engineering.

## Thread 1: Open Connect CDN — why give away free hardware? (Ray: business/marketing instinct, redirected to network economics)

Real driver = **peering economics**, not partnerships/marketing. Netflix ≈ huge fraction of North American peak downstream traffic. Without Open Connect: video crosses the **peering/transit boundary** between Netflix's network and the user's ISP. Peering is traditionally settlement-free when traffic is *balanced* — but Netflix's traffic is **wildly asymmetric** (Netflix sends enormous volume in, ISP sends ~nothing back), breaking that assumption. Real history: public peering disputes with Comcast/Verizon (~2013-14) — congested/degraded streaming until Netflix paid for direct interconnection.

**Open Connect's fix: remove the peering link from the picture entirely** (not negotiate a better price for crossing it) — a physical appliance (OCA), cached with Netflix's library, installed **inside the ISP's own network**. Video never crosses the contentious boundary.

**Win-win:** Netflix avoids transit/peering costs + congestion + gets better QoE (its core product). ISP avoids buying/upgrading expensive transit capacity for Netflix's inbound flood + its own customers get better experience. Free rack space/power is cheap for both sides to make a genuinely expensive, contentious cost disappear.

**= [[load-balancing]] "geography is a resource" (M10) taken to its logical extreme** — not just "a nearby region" but literally *inside* the last-mile network, eliminating the single most expensive/congested hop (the peering boundary itself), not just shortening distance.

## Thread 2: the blind spot in Chaos Monkey → Latency Monkey (Ray diagnosed the gap correctly, vague on the fix)

Chaos Monkey only **kills** instances — tests the *easy* failure case (M1/M6: dead = fast, clean, LB routes around instantly). It never tests **"slow, not down"** — the dangerous mode (holds connections hostage, gray-failure-like, cascades).

**Real tool: Latency Monkey** — injects artificial delay into inter-service calls (instance stays healthy, health checks pass) to simulate a slow dependency, *without killing anything*.

**Chaos Monkey validates [[stateless-services]] (M4: redundancy/self-healing). Latency Monkey validates [[circuit-breaker]] (M6: timeouts/breakers/bulkheads).** Different failure categories — a system can pass one and fail the other completely (beautifully redundant instances, still taken down by an untested timeout that doesn't actually fire). You can *read* code and see a timeout configured; you only *know* it works by making the dependency slow and watching.

**General principle: chaos engineering = a deliberate COVERAGE EXERCISE across distinct failure modes**, not "randomly break things." Simian Army = purpose-built tools per identified-but-unproven risk: Chaos Monkey (instance death), Latency Monkey (slow deps), Chaos Kong (regional evacuation, M12/M17), Conformity Monkey (best-practice violations pre-incident), Doctor Monkey (health-driven removal), Security Monkey (misconfig). Same discipline as a test suite, applied to infrastructure resilience instead of code correctness.
