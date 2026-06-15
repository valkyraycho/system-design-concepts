---
type: case-study
status: done
created: 2026-06-11
updated: 2026-06-11
tags: [outage, retry-storm, config-change, arc1, module1]
related: ["[[facebook-2021-outage]]", "[[retry-storm-and-jitter]]", "[[load-shedding]]", "[[off-serving-path-failures]]"]
---

# AWS S3 2017 outage (Feb 28, ~4 hours, US-EAST-1)

## The chain

1. Debugging slow billing subsystem; engineer follows an established playbook
2. **One typo'd argument** removes far more servers than intended (same first domino as [[facebook-2021-outage]])
3. Removed servers underpinned the **index subsystem** (metadata brain — no GET/PUT/LIST without it) and **placement subsystem**
4. Full restart required — and S3 hadn't fully restarted these subsystems in its largest region **for years**; the region had grown enormously → restart + integrity checks took hours
5. Blast radius: EC2 launches, EBS, Lambda, half the internet — including **the AWS status dashboard icons, hosted on S3**

## AWS's fix

Tool now removes capacity slowly and **refuses to go below a safety floor** — dangerous command made impossible, not warned-against.

## Unit-2 lessons

- The restart path is an [[off-serving-path-failures]] mechanism — "we restart regularly just to prove we can" exists because of this outage. **Recovery happens under fire**, so it must be load-tested like any serving path.
- The hours of downtime bred a planet-scale retry mob → see [[retry-storm-and-jitter]] and [[load-shedding]] for the mechanism and the defenses.
