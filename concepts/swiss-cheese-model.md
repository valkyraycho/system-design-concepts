---
type: concept
status: active
created: 2026-06-11
updated: 2026-06-11
tags: [reliability, incident-analysis, arc1, module1]
related: ["[[facebook-2021-outage]]", "[[correlated-failure]]"]
---

# Swiss cheese model (contributing factors > root cause)

Every defense layer has holes; an accident happens only when the holes in **all** layers line up. (Borrowed from aviation safety.)

Facebook 2021: bad command (hole 1) + audit tool bug (hole 2) + DNS self-withdrawal behavior (hole 3) + no out-of-band access (hole 4). As Ray put it: *"the command is the root cause, but the bug in the auditing tool prevented it from doing its job"* — two failures were already needed before anything bad happened. Mature analysis extends this: don't ask "which one was THE cause?", ask **"why did all the holes align, and which holes are cheapest to close?"**

This is why Google/Amazon postmortems list **contributing factors** (plural), not a single root cause.

Corollary: big outages are rarely "something broke." They're **correct components interacting in unforeseen ways** — often safety mechanisms amplifying each other (the DNS servers did exactly what they were designed to do).
