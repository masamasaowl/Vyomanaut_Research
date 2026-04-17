# ADR-012 — Pay Providers Per Audit Passed, Not Per GB Stored

**Status:** Accepted
**Topic:** #13 Escrow & Payment Basis (payment metric)
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 07

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

**Why per audit passed:**
- Directly incentivises high MTTF — a provider who is online passes more audits and earns more
- Audit results flow through the microservice, so payment releases are naturally coupled to audit receipts
- Payment mechanism and P2P transfer layer remain decoupled

**Safeguards:**
- Idempotency key on every payment release prevents double-payment race conditions ([ADR-016](ADR-016-payment-db-schema.md))
- Minimum escrow floor maintained for unexpected repair cost spikes
- Adopt Neuman's proxy protocol for bandwidth accounting — reads bandwidth usage agnostically of payments; incremental accounting prevents fraud

## Consequences

**Positive:**
- Payment and P2P transfer are decoupled — a microservice outage does not create payment liability
- Directly incentivises availability; high-MTTF providers earn more

**Negative / trade-offs:**
- Providers earn even if their data is never retrieved — cost to data owner is not purely usage-based
- Audit pass rate must be a reliable proxy for actual storage quality (mitigated by PoR design — [ADR-002](ADR-002-proof-of-storage.md))

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): Neuman's proxy protocol for bandwidth accounting
- [Paper 07 — SoK](../research/paper-07-sok-dsn.md): Swarm SWAP failure with per-BW payment
