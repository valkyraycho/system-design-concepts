---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [scaling, load-balancing, cdn, arc3, module10]
related: ["[[stateless-services]]", "[[littles-law]]", "[[measure-the-work-not-the-worker]]", "[[gray-failure]]", "[[caching-strategies]]", "[[bulkhead]]"]
---

# Load balancing & traffic management

The LB is **where horizontal scaling happens** — [[stateless-services]] *makes* servers interchangeable; the LB *exploits* that. Two jobs: (a) keep each server off the utilization cliff ([[littles-law]]), (b) detect dead servers and reroute around them.

## L4 vs L7

- **L4 (TCP/UDP)** — sees only IP:port, forwards without inspecting. Fast, protocol-agnostic, **dumb** (no content-based decisions).
- **L7 (HTTP)** — terminates the connection, reads URL/headers/cookies → content-based routing (`/api/*` vs `/static/*`, route by header, per-route rate-limit). Smart, but parses every request. Home of TLS termination.
- Often layered: fast L4 at the edge → fleet of L7 LBs doing smart routing. **"What decision am I making, and what's the cheapest layer with the info to make it?"**

## Balancing algorithms

- **Round-robin** — 1,2,3,1,2,3. Even *count*, but assumes equal request cost + equal server state (often false). Blind to backend health → keeps feeding a slow/sick server (gray failure).
- **Least connections** — route to fewest active connections. Adapts to real load.
- **Least response time** — route to fastest. Reacts to degradation.
- **Weighted / hash-based** — heterogeneous fleets / consistent hashing (sticky, cache/shard routing).

**Round-robin = blind; least-connections/least-response-time = [[measure-the-work-not-the-worker]] applied to routing** (route by observed behavior = passive outlier detection in the LB).

### The symptom is the signal (Ray Q1a)
Slow server → requests stay open longer → its active-connection count climbs → least-connections **naturally stops sending it new traffic.** Detection + remediation folded into one mechanism (no separate health check needed). A server's open-connection count = its in-flight L; routing away from high-L = routing away from the cliff.
- Caveat: a server failing *instantly* shows LOW connections → attracts MORE traffic → pair least-connections (handles "slow") with **health checks** (handle "fails fast"). The slow-vs-down distinction again.

## Static vs dynamic: serve from the cheapest place that can correctly serve it (Ray Q1b)

Routing static assets through backend servers is wasteful — primary reasons:
1. **Static is identical for everyone + rarely changes** → cache it *close to users*, don't burn precious centralized backend compute serving a logo.
2. **A CDN is geographically distributed** — caches at **edge locations** near users (Tokyo user gets the asset in ~5ms from a Tokyo edge, not ~150ms cross-Pacific) AND **shields the origin** (most static traffic never reaches it — [[caching-strategies]] cache-as-shield, applied geographically).
(Secondary: static no longer waits behind dynamic — head-of-line/[[bulkhead]].)

Pieces: **L7 router** (NGINX/ALB) splits `/api/*`→backend, `/static/*`→CDN; **CDN** (CloudFront/Cloudflare/Fastly) caches static at the edge.

**Scaling principle: at scale, geography is a resource you manage.** Static → push to dumb edge caches near users; dynamic → keep on backend. Don't waste scarce centralized compute on what abundant distributed edge does better.
