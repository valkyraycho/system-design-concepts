---
type: concept
status: active
created: 2026-06-11
updated: 2026-06-11
tags: [reliability, change-management, arc1, module1]
related: ["[[correlated-failure]]", "[[facebook-2021-outage]]"]
---

# Time diversity (vs implementation diversity)

Two ways to break a correlation between redundant components:

1. **Implementation diversity** — different software/hardware. Expensive, usually impractical; reserved for extremes (aviation's independently-programmed flight computers, DNS operators running two implementations).
2. **Time diversity** — same software, changed at **different moments**. The industry workhorse: canary, soak, stagger.

Came out of Ray's pushback: *"if they're not the same software and config, wouldn't this itself be a compatibility problem?"* — correct. A replication pair **must** share software; that correlation is accepted, not eliminated. The danger is concentrated at **transition time** (upgrade, config push, cert rotation), so the defense is staggering: upgrade standby first → soak under real load → then primary. Replication protocols deliberately tolerate N and N+1 versions so you never change both nodes in one motion.

Same principle, three costumes:
- staged rollout (Facebook's missing defense)
- rolling upgrade with soak (database pairs)
- backups (your data, decorrelated in time)

Correlation you can neither remove nor stagger (config that must match both nodes): minimize and accept. Engineering, not magic.
