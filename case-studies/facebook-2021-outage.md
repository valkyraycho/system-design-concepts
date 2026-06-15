---
type: case-study
status: done
created: 2026-06-11
updated: 2026-06-11
tags: [outage, bgp, dns, config-change, arc1, module1]
related: ["[[swiss-cheese-model]]", "[[correlated-failure]]", "[[gitlab-2017-backup-failure]]"]
---

# Facebook 2021 global outage (Oct 4, ~6 hours)

FB/IG/WhatsApp/Messenger/Oculus all dark globally; internal tools (badges, conference rooms) dead too.

## The chain

1. Routine maintenance: capacity-assessment command on the **backbone** network
2. Bad command disconnected **all** data centers; the audit tool meant to catch it had a bug
3. DNS servers' self-protection: "can't reach data centers → I'm unhealthy → stop advertising myself" — smart for one server, catastrophic for all of them at once
4. **BGP routes withdrawn** → facebook.com didn't go "down", its address stopped existing from the internet's perspective
5. Recovery nightmare: repair tools ran on the dead network; physical access required; people with access ≠ people with knowledge; hardened security slowed repair
6. Comeback: thundering herd of every client reconnecting → gradual restoration required (tens of MW power swings in DCs)

## Lessons extracted

- "Routine" ≠ scripted: runbooks are prose, humans type commands; frequency breeds trust, trust kills scrutiny → **config changes are the #1 outage cause** (recurring lesson 1)
- Mature defense: make dangerous commands *impossible*, not *warned-against* — GitOps/declarative, dry-run by default, **blast-radius limits in the tooling itself** (cannot touch >1 region per invocation)
- A guardrail that fails silently is worse than none — everyone behaves as if it's there (same failure mode as [[gitlab-2017-backup-failure]]: safety mechanisms have no users)
- Defenses when the trigger can't be prevented: cap blast radius (cells) · staged rollout · **fail static** (serve last-known-good instead of self-removing) · **out-of-band management** (repair path sharing nothing with production)
- Redundancy didn't help: a *change* propagates to backups too → [[correlated-failure]]
- **Fail fast vs fail static**: the DNS self-removal was textbook fail-fast and it caused the outage. Refinement — fail fast on errors you *own* locally; fail static on uncertainty you *share* globally (a node can't tell "I'm broken" from "everyone's broken").
