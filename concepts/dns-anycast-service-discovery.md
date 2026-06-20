---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [scaling, dns, anycast, service-discovery, kubernetes, arc3, module10]
related: ["[[load-balancing]]", "[[stateless-services]]", "[[caching-strategies]]", "[[gray-failure]]", "[[failover-split-brain-sharding]]"]
---

# How traffic finds you: DNS, anycast & service discovery

At scale, **everything moves** (autoscaling, crashes, deploys, region evacuation). Traffic finds you through a stack of routing decisions at **descending scope + different speed-of-change**:

```
DNS/anycast → which region/edge   (slow to change — cached)
Load balancer → which server      (fast)
Service discovery → what IS the pool, live  (real-time)
```

## [1] DNS — phone book + coarsest load balancer
Maps name → IP. Also LB: **DNS round-robin** (different IPs to different users), **GeoDNS** (IP by user location → nearest region).
**Catch: DNS is cached aggressively per TTL** (OS/browser/ISP). → terrible for fast failover: changing a record leaves clients hitting the old IP for minutes-hours. **Never rely on DNS for sub-minute failover.**

## [2] Anycast — one IP announced from many locations
BGP routes each user to the *nearest* announcement. Same IP *is* nearest-by-construction (CDNs, 8.8.8.8). Geographic LB baked into the network layer.

## [4] Service discovery — THE scaling problem
Autoscaled instances constantly appear/disappear with new IPs → can't hardcode IPs. Need a **service registry**: live authoritative list of "healthy instances of service X right now."
- **Self-registration** — instance registers itself on startup + heartbeats (Consul/etcd/Eureka).
- **Platform-managed** — orchestrator tracks it. **k8s Services**: control plane knows healthy Pods via **readiness probes** (M4), maintains routable set; call a stable name → resolved live. (= my-k8s.)

### k8s is a hybrid (Ray's question)
- **Nodes self-register** (kubelet registers the node — stable, long-lived, knows its identity).
- **Pods are platform-discovered** (EndpointSlice controller watches selector + readiness; Pods never announce themselves).
- Lesson: **self-registration needs the thing healthy enough to register AND cleanly deregister** — can't trust that for the most ephemeral/failure-prone components → the more ephemeral, the more the *platform* tracks it externally (via probes), not self-report. (Same "can't trust a sick component to report its own death" as [[gray-failure]].)

**Service discovery is the dynamic counterpart to [[stateless-services]]:** disposability is only *usable* if something tracks the churning set. Stop addressing *machines* (IPs churn), address *services* (stable names resolved live). Discovery + health-checking + LB are one integrated system.

## The unifying lesson (Ray Q2): address the stable abstraction, not the ephemeral instance
- (a) DNS failover slow → keep a **stable entry point (LB/anycast IP)**, reroute behind it (seconds) vs DNS TTL (minutes-hours). DNS failover = coarse last resort for whole-region/LB loss.
- (b) Hardcoded Pod IP → ephemeral, detonates on next deploy/scale (latent bug: works in test, breaks later). Use Service DNS `<svc>.<ns>.svc.cluster.local`.
- Both = put indirection between the *name* and *current physical reality* so reality can change without breaking callers. **Everything churns; the names don't.** The core enabling trick of elastic infra.
