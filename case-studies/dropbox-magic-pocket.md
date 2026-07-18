---
type: case-study
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [case-study, dropbox, build-vs-buy, correlated-failure, real-world]
related: ["[[scaling-evolution]]", "[[correlated-failure]]", "[[incident-response-postmortems-chaos]]", "[[bulkhead]]"]
---

# Dropbox: when do you stop renting and start building?

Different KIND of question than prior case studies: not a technical mechanism, a **build-vs-buy economics judgment.** Founded 2007 on AWS S3 (sensible early choice); by ~2015-16, exabyte scale, AWS storage cost a huge/fastest-growing line item → built **Magic Pocket** (custom storage hardware+software, own data centers), migrated most data off S3.

## The flip conditions (generalizing M13's "don't scale prematurely" from DB sharding to infrastructure)

Ray correctly diagnosed the "don't build early" half (low effort/battle-tested/risk-averse at small scale = correct M13 discipline) but initially stopped at "bills piled up" — missing the *positive* flip conditions:

1. **Scale changes what a % means in absolute $.** Cloud margin is a rounding error small; at exabyte scale the same % becomes enormous absolute dollars — enough to fund a whole custom system and still win.
2. **Predictability, not just size, is the real precondition.** Custom hardware = large upfront capital investment, only safe if usage is forecastable for years. Volatile/uncertain growth → building is RISKIER (overbuild = wasted capital, underbuild = no cloud elasticity to fall back on). Cloud elasticity is worth paying for when usage is unpredictable; not worth it once it isn't. Steady, forecastable growth is what de-risks the multi-year hardware bet.
3. **Organizational capability as precondition** — echoes M17 "chaos engineering is the LAST maturity step": building your own infra needs existing observability/incident-response maturity already solid, else you're flying blind on something critical.

**Deeper economic insight: a generic multi-tenant cloud service is priced/engineered for the AVERAGE customer's variable needs — can't hyper-optimize for any one customer's exact pattern.** Dropbox's workload at their scale is singular, known, predictable → could build hardware/erasure-coding precisely tuned to their exact durability target and cost-per-byte. Same principle as OLTP/OLAP (M6) and Discord's LSM-tree engine match (M13) — "match the tool to your known access pattern" — generalized from choosing the right *software* to building the right *physical infrastructure*.

## The closing thread: why keep data on public cloud even after building your own (Ray derived it unprompted)

Magic Pocket is ONE system, one team's design assumptions, untested-at-this-scale until they ran it. Any systemic flaw (bug in custom erasure coding, a wrong assumption at scale, an operational mistake in their own DCs) is **correlated across every byte stored there.** Keeping a slice on an independent, externally-battle-tested provider (S3/GCS) means Dropbox's OWN infrastructure isn't the sole failure domain the whole business depends on.

**Full-circle callback: [[correlated-failure]] (Module 1's very first lesson) applied to your OWN newest, least-battle-tested system.** "What do the redundant copies still share?" doesn't stop mattering once you've built something impressive — it applies ESPECIALLY there. Magic Pocket removed dependency on AWS pricing but introduced a brand-new correlated-failure domain: Dropbox's own design, now the single thing every byte depends on. Same instinct as backups on a different medium, or active-passive across two cloud regions: decorrelate failure domains even from your own best work. **A system that's never failed yet isn't proven safe — it's unproven risky in a way you haven't discovered yet.**
