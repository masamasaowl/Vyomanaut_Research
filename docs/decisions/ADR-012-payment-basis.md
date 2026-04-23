# ADR-012 — Pay Providers Per Audit Passed, Not Per GB Stored

**Status:** Accepted
**Topic:** #13 Escrow & Payment Basis (payment metric)
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 07, 21, 33, 35

---

## Context

Three payment models were evaluated: per GB stored per month, per GB transferred, and per audit passed. The choice determines what providers are incentivised to do and how the payment microservice interacts with the P2P transfer layer.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Per GB stored per month | Simple to understand | Incentivises hoarding storage; doesn't reward continuous availability |
| Per GB transferred | Rewards active serving | Creates coupling between payment microservice and P2P transfer layer — if microservice is down, credit liability accrues |
| **Per audit passed** | Rewards continuous availability; payment is tied to proven storage | Providers who are always available but rarely retrieved still earn |

## Decision

Providers are paid for passing audits, not for successful retrievals or GB stored.

**Why not per GB transferred:**
The payment mechanism runs via the microservice, whereas storage is maintained by pure P2P transfers. These two must not collide. If payment depended on successful transfers and the microservice went down, a credit liability would accumulate for every transfer that occurred during the downtime.

**Why not per GB ingress:** 
Paying for incoming data creates the delete-and-restore attack. Storj's previous version paid for ingress, which led nodes to delete data to be paid for re-storing it ([Paper 05](../research/paper-05-storj.md), Section 4.3). Vyomanaut does not pay for upload receipts — only for continuous presence proven by ongoing audit passes.

**Why per audit passed:**
- Directly incentivises high MTTF — a provider who is online passes more audits and earns more
- Audit results flow through the microservice, so payment releases are naturally coupled to audit receipts
- Payment mechanism and P2P transfer layer remain decoupled

**Why bandwidth-as-currency fails structurally:**
 Swarm SWAP (the bandwidth-exchange payment model used by Ethereum's Swarm DSN) assumes symmetric peer relationships where upload/download ratios balance over time. Vyomanaut's model is unidirectional — a data owner uploads once and never stores for others; providers store but never consume. A provider in Vyomanaut earns zero in a SWAP model because no one is downloading from them to create a balanced ledger. SWAP's failure in production ([Paper 07](../research/paper-07-sok-dsn.md), Lakhani et al.) confirms this is a structural incompatibility, not a parameter calibration issue.

**Safeguards:**
- Idempotency key on every payment release prevents double-payment race conditions ([ADR-016](ADR-016-payment-db-schema.md))
- Minimum escrow floor maintained for unexpected repair cost spikes
- Adopt Neuman's proxy protocol ([paper-05](../research/paper-05-storj.md)) for bandwidth accounting — reads bandwidth usage agnostically of payments; incremental accounting prevents fraud

## Consequences

**Positive:**

- Payment and P2P transfer are decoupled — a microservice outage does not create payment liability
- Directly incentivises availability; high-MTTF providers earn more

**Negative / trade-offs:**
- Providers earn even if their data is never retrieved — cost to data owner is not purely usage-based
- Audit pass rate must be a reliable proxy for actual storage quality (mitigated by PoR design — [ADR-002](ADR-002-proof-of-storage.md))

**Open Constraints:**

 - Razorpay payout idempotency: as of 15 March 2025, the X-Payout-Idempotency header is mandatory for all Razorpay payout API calls. The escrow_events.idempotency_key column (ADR-016) must be used as this header value. Format: SHA256(provider_id + audit_period) — 64 hex characters, satisfies the 10-character alphanumeric minimum.

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): delete-and-restore attack from ingress payment; ingress explicitly excluded in Storj v3
- [Paper 07 — SoK DSN](../research/paper-07-sok-dsn.md): Swarm SWAP structural failure; bandwidth-as-currency is incompatible with asymmetric data-owner / provider model
- [Paper 33 — Ihle et al.](../research/paper-33-ihle-incentive-mechanisms.md): per-audit-passed model is in the selfish-attack-resistant category; free-riding impossible when payment is contingent on non-forgeable proof
- [Paper 35 — Razorpay API Docs](../research/paper-35-razorpay-upi-docs.md): X-Payout-Idempotency header mandatory from 15 Mar 2025; idempotency_key from [ADR-016](./ADR-016-payment-db-schema.md) serves as the header value directly
- [Paper 21 — Saroiu et al.](../research/paper-21-saroiu-p2p-measurement.md): 25% of unincentivised Gnutella peers share nothing; confirms free-riding is eliminated by storage-contingent payment