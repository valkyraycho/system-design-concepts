---
type: concept
status: active
created: 2026-06-13
updated: 2026-06-13
tags: [resilience, retries, idempotency, arc2, module5]
related: ["[[retry-storm-and-jitter]]", "[[setting-timeouts]]", "[[rep1-the-on-sale]]", "[[idempotency-keys]]"]
---

# Idempotency

**An operation is idempotent if doing it N times has the same effect as doing it once.** The property that makes retries *safe*.

Light switch: "set to ON" = idempotent; "toggle" = not.

## Why retries need it: a timeout tells you NOTHING

When a state-changing call times out, two indistinguishable cases:
(a) request never arrived / failed → safe to retry, or
(b) **request succeeded but the ACK was lost** → retry duplicates the effect.
The network can't tell you which. The **action** and the **acknowledgment of the action** are separate events, either can be lost. → the **double-charge**: card debited, response lost, retry charges again.

## Atomicity vs idempotency (orthogonal — the Rep 1 confusion, resolved)

- **Atomicity** — two *different* requests race for one resource → exactly one wins. *Concurrency* safety. (lock / conditional write / CAS)
- **Idempotency** — the *same* request arrives twice → second is a no-op returning the first result. *Retry* safety. (idempotency key)

The double-charge is ONE intent retried (not a race) → atomicity can't fix it. Ticketing needs **both**: conditional write so two people don't get seat A5; idempotency key so one buyer's retry doesn't double-buy.

**One-liner:** atomicity makes the *world* consistent; idempotency makes the *retry return the same answer as the first try.* A locked-but-not-idempotent system can be oversell-free AND lying to half its users (Ray's (b): buyer's retry sees `status='sold'` → told "failed" though they succeeded & were charged).

## Categories (the practical skill)

- **Naturally idempotent** → retry freely: reads, "set to absolute value", DELETE. (HTTP GET/PUT/DELETE defined idempotent; **POST is not** — by this exact property.)
- **NOT idempotent** → must add a mechanism: charges, INSERT, "increment/`balance = balance + 50`", sending email.
- Idempotency often lives in *how you write it*: `balance = balance + 50` (no) vs `balance = 250` (yes).
- **External side effects are hardest** (sent emails, fired webhooks) — can't UPDATE them away; need dedupe before the effect.
