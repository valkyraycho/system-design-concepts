---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [operations, feature-flags, rollback, mttr, arc4, module14]
related: ["[[deployment-strategies]]", "[[off-serving-path-failures]]", "[[idempotency-keys]]", "[[correlated-failure]]"]
---

# Feature flags & rollback as a design goal

## Deploy ≠ release (feature flags decouple them)
- **Deploy** = put new code on servers (feature OFF, behind a flag).
- **Release** = flip the flag ON (a config change, no deploy).
Decouples the risky act (ship code) from the risky act (activate behavior) → do them at different times, audiences, and **undo activation instantly without a deploy.**

```
if flag("new_feed", user): new_feed(user) else old_feed(user)
```

**Flags = canary at the APPLICATION layer** (canary by *user* not server) + more flexible: 1% of users, internal-only (dogfood), per-region/tenant, and **instant OFF without redeploy** (kill switch = config flip in ms, not a rollback deploy in minutes). Knight Capital died partly for lack of this.
Cost: flag accumulation = tech debt (each flag is a code branch; clean up stale ones) + testing combinatorics.
Mantra: **deploy frequently, release carefully.**

## Rollback as a first-class design goal
The question isn't "will this deploy succeed?" but **"when it fails, how fast can I undo it?"** You WILL ship bugs; reliability = fast, boring recovery.
- **Every change must be reversible** → why expand/contract matters (a one-shot rename can't roll back). Backward-compatible = reversible.
- **Rollback must be fast AND practiced** → [[off-serving-path-failures]] / rehearsal: an untested rollback fails when needed. Best teams roll back routinely.
- **Forward-fix vs rollback** — sometimes you can't roll back (migration mangled data) → make changes reversible by default so you're never trapped.

**Optimize for recovery, not perfection. The metric is MTTR, not deploy-success-rate.** (DORA: deploy frequency & stability are POSITIVELY correlated — frequency forces small batches + automation + practiced rollback.)

## Big-batch "careful" deploy is RISKIER (Ray Q2a)
1. **Hard to find what broke** (which of 200 changes?) vs small deploy = obviously the one thing.
2. **Rollback is all-or-nothing** — undo a month + 199 good changes to kill 1 bug.
3. **Correlated emergent bugs** — bundled changes interact; holes line up (Swiss cheese / [[correlated-failure]]).
4. "Needs a weekend + all-hands" is itself the tell: too big, too rare. Fix = smaller/more frequent, individually boring. (Batching to "be careful" = the off-serving-path rot applied to the deploy pipeline.)

## The flag's limit: code is reversible, STATE/side effects are not (Ray Q2b — found it himself)
A flag toggles *behavior* cleanly; it does NOT undo *state already written* or side effects already fired. Flag a new checkout that writes orders in a new format → flip off → code reverts instantly but **the new-format data those users created is stranded** for old code. Same boundary as async≠fire-and-forget and rollback-needs-reversible-changes: **reversibility of code ≠ reversibility of effects.**
→ A flagged feature that writes state must ALSO write it backward-compatibly, OR have a *separate data rollback plan*. Flag handles behavior; you need a separate answer for state.
Also: **the flag system is itself a dependency** — needs a sane default when unreachable + local caching. The kill-switch must be more reliable than what it controls.
