---
type: case-study
status: done
created: 2026-06-13
updated: 2026-06-13
tags: [case-study, uber, geospatial, h3, candidate-generation, real-world]
related: ["[[instagram-scaling]]", "[[failover-split-brain-sharding]]", "[[discord-message-storage]]"]
---

# Uber: finding a needle in a moving haystack (geospatial indexing)

## The problem: proximity queries break 1-D indexes

"Find drivers within 2km" is a **2-dimensional proximity query**, not exact-match or single-dimension range. A B-tree index on lat/lng is sorted 1-D — two points can have near-identical latitude and be far apart in longitude (or vice versa), so a range scan on one dimension doesn't answer "nearby."

## Two different questions, easy to conflate (Ray's graph-DB instinct was real, just for stage 2)

Ray correctly identified that real travel distance ≠ straight-line distance — roads, one-ways, traffic — and that a **graph** (intersections=nodes, road segments=weighted edges, Dijkstra/A*-family shortest path) is the right model for **routing/ETA**. Genuinely correct — Uber does this.

**But that's stage 2.** Running shortest-path against millions of drivers per request is computationally impossible at volume. You need **candidate generation** first: cheaply narrow millions of drivers to a small plausible set, BEFORE any expensive computation runs on any of them.

= the exact two-stage shape from [[instagram-scaling]]'s feed (cheap candidate pool + expensive personalized ranking only on candidates). Cheap stage answers "what's plausible?"; expensive stage answers "of the plausible ones, what's best?"

## Stage 1: H3 — turning "nearby" into a key lookup

Uber's real, open-sourced solution: **H3**, a hierarchical **hexagonal** spatial index. Divide Earth's surface into hexagonal cells at multiple resolutions; every cell has a deterministic ID **computable directly from lat/lng** (pure math, no lookup).

"Find nearby drivers" becomes:
1. Compute rider's cell ID (cheap math).
2. Look up drivers in that cell ID + its 6 neighbor cell IDs (fixed, small number of **exact-key lookups**).
3. Done — no geometric scan.

**Why hexagons, not squares:** a hexagon's 6 neighbors are all exactly equidistant from center. A square's 4 diagonal neighbors are farther than its 4 edge neighbors (×√2) — messier, non-uniform adjacency. Hexagons = clean uniform neighbor-lookup, chosen deliberately for this.

**Same trick as [[instagram-scaling]]'s photo IDs and [[discord-message-storage]]'s message buckets** — turn an expensive lookup into a deterministic computation from a key. H3 does it for **physical space**: proximity becomes key-equality instead of geometry.

## Full pipeline
**H3 cell lookup (cheap, candidate gen)** → small candidate set → **road-network graph shortest path (expensive, precise, only on the few candidates)** → best match. Ray's graph-DB instinct is exactly stage 2 — needed stage 1 in front of it to be tractable at Uber's scale.
