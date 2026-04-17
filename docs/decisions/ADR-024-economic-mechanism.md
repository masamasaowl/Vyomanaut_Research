# ADR-024 — Economic Mechanism Design

**Status:** Proposed — deferred to Phase 5
**Topic:** #18 Economic Mechanism Design
**Supersedes:** —
**Superseded by:** —
**Research source:** TBD

---

## Context

The system's long-term viability depends on providers being economically incentivised to stay. Specific open questions that this ADR must resolve: minimum vetting period length, held-earnings percentage, per-segment pricing, and the breakeven MTTF for the cold storage business model.

## Blocked on

Phase 5 research. Reading-list Phase 2B #18 (TrustDSN) and Filecoin whitepaper (reading-list Phase 1 #11) are prerequisites.

## Known constraints and open questions (from existing papers)

From Q05-4: What is the minimum vetting period and held-earnings percentage that deters exit while keeping providers engaged? Storj held earnings for 6 months — too taxing for average users.

From Q05-8: Should per-segment pricing be adopted to cover metadata overhead (5+ KB per segment at n=80)?

From Q01-6: Should providers with higher speed, uptime, and storage be proportionally rewarded?

From Q06-2: Low-MTTF provider entry is unprofitable for existing nodes — admission control must enforce a minimum MTTF at join time (or during vetting).

## Existing decisions that constrain this ADR

- Payment per audit passed ([ADR-012](ADR-012-payment-basis.md)) — the unit of payment is fixed
- UPI/Razorpay fiat escrow ([ADR-011](ADR-011-escrow-payments.md)) — the payment mechanism is fixed
- PN-counter CRDT payment DB ([ADR-016](ADR-016-payment-db-schema.md)) — the accounting model is fixed

This ADR covers: pricing tiers, holding periods, penalty structures, and SLA enforcement — everything above the payment primitive.

## Decision

**Not yet made.** Fill this ADR after Phase 5 research.
