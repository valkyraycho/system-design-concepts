---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [operations, incident-response, postmortem, chaos-engineering, arc4, module17]
related: ["[[off-serving-path-failures]]", "[[swiss-cheese-model]]", "[[deployment-strategies]]", "[[cell-based-architecture]]", "[[observability]]", "[[feature-flags-and-rollback]]"]
---

# Incident response, postmortems & chaos engineering (the finale)

**Failure isn't preventable → build the practice of *handling* it.** A triad around inevitable failure:
1. **Incident response** — handle fast & calm (minimize MTTR).
2. **Postmortems** — learn so it doesn't recur.
3. **Chaos engineering** — deliberately cause failure to prove defenses work before reality tests them.

## Incident response — structure beats heroics
Roles: **Incident Commander** (coordinates, decides, owns — does NOT debug, just orchestrates), **Comms** (stakeholders/status page), **Operators** (investigate/fix). Separation stops "everyone codes, nobody steers."

**MITIGATE FIRST, DIAGNOSE LATER.** First job = stop the bleeding (restore service), THEN understand. Bad deploy → **roll back immediately** (M14), don't first find which line broke. = regional-evacuation philosophy (M12): route around the sick thing, don't fix under fire. Senior reflex: "fastest action that restores service?" (rollback/failover/evacuate/shed/flag-off) before "root cause?". MTTR is the metric; mitigation drives it, not diagnosis. (Ray Q1a: nailed it — "leave the logs, roll back, verify healthy, then dig in." Verify-recovery is part of rollback.)

## Postmortems — blameless, and why
What happened / timeline / impact / contributing factors / action items. **Blameless = focus on systemic causes, not individual fault.** "Bob ran the wrong command" isn't a root cause; "the system *allowed* it — no confirmation/canary/rollback" is. Question is never "who screwed up?" but "**what about our system made this possible, and how do we make it impossible?**"

Blamelessness isn't niceness — it's how you get **the truth** (punishment → hiding info, no near-miss reports → lose improvement data). **Humans are part of the system; "human error" is a symptom, not a cause** (Swiss cheese / Knight Capital). Fix the system (guardrails), not the human, so the next person can't repeat it. (Recurring lesson 5: humans + unclear procedures complete most outage chains → fix procedures.)

## Chaos engineering — the ultimate rehearsal (loop-closer)
**Deliberately inject failure into production to prove survival.** Chaos Monkey randomly kills prod instances in business hours, on purpose. Why: **untested resilience = hope, not resilience** (off-serving-path rot / GitLab backups / evacuation-needs-rehearsal). Discover your "resilient" system falls over at 2pm Tuesday with engineers watching, not 3am in a real outage.

= **experimental validation of Arcs 1–3** (designed for instance death M4 / dep failure M6 / region loss M12 → prove it). Maturity ladder: avoid failure (fragile) → tolerate (resilient) → *cause it to verify* (antifragile). Netflix regional evacuation is trustworthy *because* Chaos Kong rehearses it.

### Precondition: chaos is the LAST maturity step, not the first (Ray Q1b)
Irresponsible until you have: (1) **observability to DETECT** the injected impact (M16 — can't validate what you can't see), (2) **incident response + fast recovery** to restore (Ray got this half), (3) **bounded experiment blast radius + abort switch** (staging → small prod scope, one instance/cell M12). Chaos doesn't *replace* the other practices — it *validates* them, so it's only safe once they exist. A team without observability adopting Chaos Monkey has confused a validation tool for a starting tool (the skeptical manager's fear is correct *for an immature team*).

## The maturity ladder (whole curriculum, one line)
prevent what you can → **detect fast** (observability) → **respond calmly** (incident structure, mitigate-first) → **learn** (blameless postmortems) → **rehearse failure** (chaos) so the machine is *proven, not hoped*.
