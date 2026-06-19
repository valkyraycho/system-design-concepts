---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, failure-isolation, arc2, module6]
related: ["[[littles-law]]", "[[retry-storm-and-jitter]]", "[[setting-timeouts]]", "[[bulkhead]]", "[[load-shedding]]"]
---

# Circuit breaker

Failure spreads **upstream through waiting**: slow dep → caller's threads pile up waiting → caller's pool exhausts (Little's Law) → caller dies → *its* callers pile up… One slow leaf can take down the whole system.

**Slow is worse than down.** A dep that fails *fast* (connection refused) is nearly harmless (instant error, no resources held). A dep that's *slow* (30s hang) is lethal (every caller holds a thread/L-slot waiting). Speed of failure is a mercy.

## The state machine

Wraps calls to a dependency:
- **CLOSED** (normal): requests flow, count failures. Cross threshold (e.g. 50% fail in a window) → trip → OPEN.
- **OPEN** (tripped): requests **fail instantly, no call attempted** → frees resources (vs every request waiting 30s to time out → pool exhaustion). After cooldown → HALF-OPEN.
- **HALF-OPEN** (testing): allow ONE trial request. Success → CLOSED; fail → OPEN again.

It **converts a slow failure into a fast one** — turning the lethal case into the harmless one. It's the retry budget (Module 1) grown into a state machine + fail-fast applied deliberately.

Protects **both sides**: caller's resources AND the dep's recovery (shields a reviving dep from the retry mob — recovery-under-fire).

## When OPEN, what do you serve? (grace order)

1. **Graceful degradation** — fallback (recs down → generic "popular items")
2. **Serve stale** — older cached value
3. **Fail fast with a clear error** — if no sensible fallback

## Which deps deserve one? (the framework Ray was missing)

Classify on: **critical-path vs enhancement.**

| Type | Example | If down |
|---|---|---|
| **Critical/hard** | payment, inventory | Can't complete — fail *fast & honest* (no fake fallback; can't pretend to charge). Breaker buys resource-preserving fast failure. |
| **Enhancement/soft** | recommendations, reviews, "also bought" | **Degrade gracefully** — hide/default/stale. Core action still works. Breaker buys graceful degradation. |

Both deserve a breaker, for *different wins*. The breaker's value comes from the fallback; whether a good fallback exists depends on core-vs-enhancement.

**Principle:** a failure in an enhancement dep must NEVER fail the core action. If "recommendations" down breaks the whole product page, you've coupled a non-critical feature to a critical path. Breaker + fallback enforces the decoupling. (Ray's "defaults vs error modal" answer WAS this distinction in disguise.)
