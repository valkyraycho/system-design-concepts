---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [reliability, observability, health-checks, arc1, module1]
related: ["[[gray-failure]]", "[[off-serving-path-failures]]"]
---

# Measure the work, not the worker

To detect a sick node, prefer **the outcomes of real requests** over **the node's self-report.**

- **Active probe (asking the worker):** synthetic `/healthz` every Ns. Cheap → fits inside a GC alive-burst → passes while real 500ms requests fail. Only catches failure modes its author predicted.
- **Passive detection (measuring the work):** the load balancer **already forwards every real request and sees every outcome** (latency, status) per backend. That data exists *for free* before you build anything. Slow/erroring real traffic = the sick node, by definition — including failure modes nobody imagined.

**Testimony vs evidence:** a self-report can lie (200 on a ping while drowning on real work); served traffic can't. Outlier detection is just the *consumer* of this evidence — the insight is the evidence was already in the LB's hands. (Ray reached for outlier detection; the missing half was "the signal already exists.")

## The honest limit → pairing

Passive detection needs *traffic* — a just-booted node has no evidence yet. So real systems pair them:
- active probe answers **"safe to START sending you traffic?"** (k8s *readiness*)
- passive measurement answers **"should you KEEP getting it?"** (mesh *outlier ejection*)

Envoy outlier detection / nginx `max_fails` are the productized versions.
