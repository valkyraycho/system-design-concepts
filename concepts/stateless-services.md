---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, statelessness, scaling, arc2, module4]
related: ["[[utilization-latency-cliff]]", "[[gray-failure]]", "[[correlated-failure]]"]
---

# Stateless services (externalized state)

**Stateless ≠ no state.** It means no state stored *locally* — state lives somewhere **shared and external** (DB, Redis, or a token in the request itself). The defining test: **if request #2 lands on a different server than request #1, does it still work?**

## Why it's the foundation of Arc 2

Local state makes a server **special**, and "special" is the enemy of every reliability property:
- Can't **load balance** freely → sticky sessions
- Can't **fail over** cleanly → state dies with the server
- Can't **autoscale** → new servers useless to existing users; removed servers take state with them
- Can't **zero-downtime deploy** → killing a server destroys live state

**Cattle, not pets:** statelessness makes servers interchangeable & disposable. You can only freely kill/replace/multiply things that hold no irreplaceable state. Disposability is the foundation of resilience.

## Sticky sessions seem like a fix but re-introduce every problem (Ray, Q1)

4 stateful servers, carts in local RAM, users pinned:
- **(a) Add 4 more servers** → does nothing for existing users (pinned to original 4); old 4 stay hot/overloaded, new 4 idle → **load imbalance**, the LB can't balance what it can't move. Worst combo: **sticky + autoscaling** → scaler sees idle new servers, reports "healthy" while pinned users hit the latency cliff (a self-inflicted [[gray-failure]]); and removing a server kills its sessions. Statefulness vs elasticity are at war.
- **(b) One server crashes** → its state is *gone, not relocated*. Pinned users log in again, carts empty.

## Same event, opposite outcome

One server dying:
- **Stateless:** LB reroutes, next request reads cart from Redis → **user notices nothing.** Non-event.
- **Stateful:** state died with the server → **data-loss incident.** Failures partitioned by which server held your state (some users destroyed, some fine, dashboards average green → hard to detect).
