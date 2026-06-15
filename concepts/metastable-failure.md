---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, overload, queueing, arc1, module1]
related: ["[[retry-storm-and-jitter]]", "[[load-shedding]]", "[[off-serving-path-failures]]"]
---

# Metastable failure

A system has two stable states: **healthy at load N** and **drowning at load ~10N**. Once knocked into the drowning state, it *stays there* even after the original trigger is gone — because the failure now manufactures its own demand.

**The trigger can be fully fixed while the outage continues.** Servers back, network healed, hardware fine — yet still down. The loop no longer needs the cause; it feeds on its own timeouts.

## The engine: wasted work (the amplifier)

Retries alone don't trap a system — there must be an **amplifier**: work that costs capacity but yields nothing.

Litmus test (Ray's earned answer): *"the server wasted the 6 extra seconds past the 2-second mark, computing a result nobody receives."* Client times out at 2s; server grinds the request to completion anyway, delivering an answer to someone who already left.

The loop:
1. queue deep → total time (wait + service) > client timeout
2. client gives up + retries; server doesn't know, keeps the abandoned request in the queue
3. that service capacity produces **zero goodput**, and the retry adds another queue item
4. queue grows → waits longer → more breaches → more wasted work → …

## Where the latency actually comes from

Not CPU-per-request (that's unchanged, ~300ms). It's **queue wait**. "8 second processing" = ~0.3s work + ~7.7s standing in line. **Latency under overload is dominated by waiting, not working** — misdiagnosing this leads to tuning code / adding CPU, which the queue just swallows.

## Fixes (attack the waste, not the volume)

- **Deadline propagation** — client stamps "I'll wait 2s"; server drops anything already past deadline before serving → reclaims capacity from ghosts
- **Load shedding / bounded queues** — cap queue length, reject at the door (see [[load-shedding]])
- **LIFO under overload** — serve newest first; oldest items most likely already timed out → maximizes goodput
