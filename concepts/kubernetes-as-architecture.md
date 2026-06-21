---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [operations, kubernetes, control-plane, reliability, arc4, module15]
related: ["[[stateless-services]]", "[[deployment-strategies]]", "[[dns-anycast-service-discovery]]", "[[bulkhead]]", "[[failover-split-brain-sharding]]", "[[multi-region]]", "[[autoscaling-capacity]]"]
---

# Kubernetes as architecture

**K8s is a reliability machine, not a deploy tool: a control loop that continuously drives actual state → declared desired state.** Almost every primitive = a reliability pattern from this curriculum, automated. (Ray's my-k8s built the etcd piece = the control plane's memory.)

## The reconciliation loop (the heart)
Declare desired ("5 healthy replicas behind a stable address"); controllers **observe → compare → act**, forever. Pod dies → loop replaces it. Node vanishes → reschedule. = **self-healing = "cattle not pets" (M4) automated.** Declarative not imperative: you state desired state the loop maintains, not one-shot commands. Self-correcting by construction → most manual reliability (restart-on-crash, maintain-count) is free. (Same desired-vs-actual / consensus machinery as etcd/M9.)

## Primitives re-seen as patterns
| primitive | really is | concept |
|---|---|---|
| Pod | disposable instance | — |
| ReplicaSet | keep N copies | statelessness/disposability (M4) |
| Deployment | rolling updates + rollback | M14 |
| Service | stable VIP/DNS → healthy pods | service discovery (M10) |
| liveness probe | "broken? restart it" | failure detection (M1) |
| readiness probe | "ready for traffic?" | "safe to send?" (M4/M10) |
| HPA | scale on a metric | autoscaling (M11) |
| resource requests/limits | per-pod CPU/mem caps | bulkhead (M6) |
| PodDisruptionBudget | "don't kill >X at once" | blast-radius limit (M1) |
| namespaces/quotas | partition cluster | failure-domain isolation (M6/M12) |

## The #1 anti-pattern: liveness checking a shared dependency (Ray Q1a)
Liveness must check **"is MY OWN process wedged?"** — never a shared dependency. If liveness checks DB/Redis/payment: a dependency blip → **every pod's liveness fails simultaneously** (shared signal) → k8s restarts the WHOLE FLEET at once → total outage + restart-storm hammers the recovering dep (recovery-under-fire). = **Facebook DNS mass-suicide (M1) via a probe.**
Caveat: a shared-dependency check in *readiness* has the milder version (all pods pulled from rotation together → Service has zero endpoints). Safest readiness is also mostly self-focused.
**Unifying rule: automated life-or-death keyed off a CORRELATED signal remediates the whole fleet = self-inflicted outage.** Seen 4×: FB DNS, reboot-bot, absolute-threshold ejection, liveness-on-dependency. Defense always: key automated remediation off *local/independent* signals ("am I broken?"), never *shared* ones.

## Control plane is a SPOF (Roblox 2021, 73h)
All self-healing depends on the control plane (API server, scheduler, controllers, **etcd**). Roblox: Consul (coordination layer) degraded under load → cascaded → recovery itself depended on the broken layer → 73h. = recurring lesson 3 (control plane = SPOF) + vicious-cycle recovery trap.
Two design rules:
1. Control plane must be robust → etcd is a **quorum cluster (3/5 nodes, M9)**.
2. **Data plane must survive control plane down** — running pods keep serving even if API server unreachable (data plane decoupled from control plane). Roblox's coordination layer was *in the recovery path* — fatal.

## etcd losing quorum = CAP choice, deliberate (Ray Q1b)
Single node = total loss if it dies; **even number = no majority on a split** (worse). Odd because consensus needs a majority to safely agree each change.
Lose quorum → etcd goes **read-only / refuses writes** — picks **Consistency over Availability** (would rather stop changing than risk split-brain of cluster desired-state). Effect: **desired state can't change** (no deploys/scaling/rescheduling/endpoint updates) BUT **running pods keep serving** (data-plane decoupling = freeze, not death; recoverable). Disaster only if something *also* needs to change during the freeze (node dies → can't reschedule). Getting etcd quorum right = choosing correctness over availability for the one state everything trusts.
