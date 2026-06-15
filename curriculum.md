---
type: curriculum
status: active
created: 2026-06-11
updated: 2026-06-13
tags: [system-design, reliability, scalability, devops]
---

# System Design & Reliability Curriculum

**Goal:** deep, embedded understanding of how to make applications reliable, stable, and scalable — concepts and architectures, not building projects (those happen separately: my-shell, my-redis, my-docker, my-k8s).

**Method:** Socratic teaching + active recall + real-world case studies. Claude teaches and questions; concept notes are written only after understanding is demonstrated. Background assumed: DDIA (read once), AWS SAA.

**Ordering principle:** resilience before scale — scaling a fragile system just builds a larger blast radius.

---

## Arc 1 — Foundations: How Systems Fail and How Load Behaves

### Module 1: How systems fail
Cascading failures, retry storms, correlated failures, gray failures; why most outages are self-inflicted.
📎 Facebook 2021 global outage (BGP/DNS) · AWS S3 2017 outage (one command typo)

### Module 2: Measuring reliability
SLI/SLO/SLA, error budgets, availability math (what each "nine" costs), latency percentiles and why averages lie.
📎 Google SRE book SLO framework · Cloudflare public availability reporting

### Module 3: The mechanics of load
Little's law, queueing intuition (latency explosion near saturation), utilization vs headroom, backpressure, capacity planning.
📎 Robinhood 2020 outage (thundering herd at market open) · "Reddit hug of death" pattern

## Arc 2 — Designing Applications for Resilience

### Module 4: Stateless services & externalized state
Horizontal scaling prerequisites, graceful shutdown, liveness vs readiness, connection draining.
📎 Twitter monorail-to-services (the end of the Fail Whale era)

### Module 5: Timeouts, retries, idempotency
Exponential backoff with jitter, retry budgets, idempotency keys; how naive retries take down systems.
📎 Stripe idempotency keys · AWS backoff + jitter blog

### Module 6: Circuit breakers, bulkheads, load shedding
Failure isolation, graceful degradation instead of total collapse.
📎 Netflix Hystrix · Google "The Tail at Scale"

### Module 7: Queues & async as reliability tools
Decoupling, buffering bursts, dead-letter queues, backpressure propagation, the consistency cost.
📎 WhatsApp (~50 engineers, 900M users, Erlang) · Slack job queue redesign (Redis → Kafka)

### Module 8: Caching for stability and scale
Cache layers, invalidation strategies, stampede protection, what happens when the cache dies. (Ties to my-redis.)
📎 Facebook memcache paper (leases vs thundering herds) · Instagram cache warmup

### Module 9: Data layer reliability
Replication, failover, backups vs replicas, RTO/RPO, connection pooling, hot partitions. (DDIA, reframed operationally.)
📎 GitHub 2018 split-brain MySQL incident · GitLab 2017 database deletion (5 backup mechanisms, none working)

## Arc 3 — Scaling Architectures

### Module 10: Load balancing & traffic management
L4 vs L7, DNS/anycast/CDN, service discovery, sticky sessions and why to avoid them.
📎 Cloudflare edge architecture · Netflix Open Connect (servers inside ISPs)

### Module 11: Autoscaling & capacity
Scaling signals, policies, cold starts, overprovisioning trade-offs; when autoscaling makes things worse.
📎 Shopify flash-sale engineering (Black Friday as planned DDoS) · Coinbase during crypto spikes

### Module 12: Multi-region & high availability
Active-active vs active-passive, data gravity, cell-based architecture, blast radius reduction.
📎 Netflix regional evacuation · AWS cell-based architecture · Slack 2021 cross-region outage

### Module 13: Scaling evolution case studies
Growth stages from one box to planet scale; which steps matter at which order of magnitude.
📎 YouTube (MySQL sharding → Vitess) · Twitter timeline fan-out (the celebrity problem) · Discord (MongoDB → Cassandra → ScyllaDB)

## Arc 4 — Operating in Production

### Module 14: Deployment & release engineering
CI/CD, blue-green, canary, feature flags, zero-downtime schema migrations, rollback as a first-class design goal.
📎 Knight Capital ($440M in 45 minutes) · Google canarying practice

### Module 15: Kubernetes as architecture
Each primitive (Deployment, Service, HPA, PDB, probes) as an answer to a reliability problem. (Ties to my-k8s.)
📎 Roblox 73-hour outage (Consul control-plane death) · why Google built Borg → K8s

### Module 16: Observability
Metrics/logs/traces as three different questions, SLO-based alerting, alert fatigue, debugging unknown-unknowns.
📎 Honeycomb observability philosophy · Cloudflare 2019 regex outage (global CPU exhaustion)

### Module 17: Incident response & learning from failure
On-call design, blameless postmortems, chaos engineering, game days.
📎 Google postmortem culture (SRE book) · Netflix Chaos Monkey

---

## Session mechanics

Three session types:
- **Learn** (~30–45 min/unit) — Socratic dialogue on the next unit; Claude teaches in layers, stops to ask predictions, explain-backs, and challenges. Case studies woven in as the payoff.
- **Recall** (~10–15 min) — quiz on previous modules, prioritized by least-recently-tested and weakest. Auto-triggered as warm-up when 3+ days since last session.
- **Rep** (~45–60 min, after each arc) — case-study-grounded scenario drills. Ray drives; Claude plays skeptical staff engineer + injects what actually happened.

Note flow: Claude teaches → questions → when Ray's answers show understanding, Claude writes the concept note (quoting Ray's framing where good). Shaky answers → weak-spot list, no note yet. Notes are earned, not given. Answers in any language/format — retrieval is about the concept, not the prose.

Entry points any session: "continue" · "quiz me" · "I have 15 minutes" · "explain X again". Sequence is a default, not a cage.

## The 5 recurring lessons (watch for them everywhere)

1. Config changes are the #1 outage cause
2. Retries amplify failures
3. Control planes are single points of failure
4. Backups are worthless until restored
5. Humans + unclear procedures complete most outage chains
