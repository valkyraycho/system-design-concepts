---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, state, auth, jwt, arc2, module4]
related: ["[[stateless-services]]", "[[correlated-failure]]", "[[percentiles-vs-averages]]"]
---

# Externalized state — where does the state go?

Making app servers stateless doesn't eliminate state; it **moves** it. Wherever it lands becomes the thing you must make reliable.

## You can't eliminate state, only move it to where it's managed best

If 8 stateless servers all read from **one Redis**, Redis is now the special/irreplaceable thing — SPOF moved, not removed; plus a network hop per request. But this is usually a *good* trade: you turn *N* hard problems (make every app server reliable) into *one* (make the data tier reliable), concentrated into purpose-built components (DB/cache/consensus store) that are designed for it (replication, persistence, failover — Module 9). The skill is **being deliberate about where state lives and making that one place excellent.**

## Taxonomy of state homes (match the kind of state to the home)

1. **In the request (client-side, e.g. JWT)** — signed token, server verifies signature, *no lookup*. Maximally stateless. ✅ identity. ✗ things that change / need revoking.
2. **Shared in-memory store (Redis/Memcached)** — fast, purpose-built for ephemeral session/cart. Cost: must be made HA; network hop. Default for session state.
3. **Database** — durable, transactional. Cost: slower; don't hammer the primary with high-churn data. Right for state that must survive (orders/money).

Identity → token · session/cart → Redis · orders → DB.

## Auth trade-off: server-side sessions vs JWT (Ray Q2)

- **A. Server-side sessions** (Redis lookup per request): revocable ✅, but every microservice hits Redis on every request → **shared auth chokepoint** + latency on every fan-out hop.
- **B. JWT** (local signature verification): **decentralized verification, zero network calls** — the real microservices win (not just "rarely revoke"). Cost: **can't revoke** — token valid until expiry (fired employee / stolen token / password change all keep working). Severity ∝ token lifetime.

## The hybrid that dissolves it: access + refresh tokens

Split the thing into two parts optimized for each property:
- **Access token (JWT, ~5–15 min)** — every request, local verify, no lookup → fast path stays fast.
- **Refresh token (long-lived, stored server-side, revocable)** — used only when access token expires, to mint a new one.

Result: 99% of requests are lookup-free; revoke the *refresh* token → stolen/fired user locked out after ≤15 min. Revocation window shrinks from "forever" to "minutes" = good enough for most threat models. Cost: more moving parts (auth service, refresh logic).

**General move:** when two properties seem mutually exclusive (fast+stateless vs revocable), split into two parts each optimized for one.
