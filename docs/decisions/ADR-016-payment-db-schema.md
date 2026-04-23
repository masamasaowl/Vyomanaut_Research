# ADR-016 — Payment DB Schema: PN-Counter CRDT with Idempotency Key

**Status:** Accepted
**Topic:** #13 Escrow & Payment Basis (schema)
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 11, 35

---

## Context

Escrow balance is a CRDT PN-counter (deposits increment it, releases and seizures decrement it). A naive implementation using a balance column with UPDATE statements is not I-confluent (floor≥0 invariant) and is vulnerable to double-payment race conditions. An append-only event log solves both problems.

## Decision

Store every deposit and release as an append-only event row. Never UPDATE an existing row.

```sql
escrow_events (
  event_id         UUIDv7        PRIMARY KEY,
  provider_id      UUID          NOT NULL,
  event_type       ENUM(DEPOSIT, RELEASE, SEIZURE),
  amount_paise     BIGINT        NOT NULL,   -- always integer, never decimal
  audit_period     UUID          REFERENCES audit_periods(id),
  idempotency_key  VARCHAR(64)   UNIQUE,     -- SHA256(provider_id + audit_period)
  created_at       TIMESTAMPTZ   NOT NULL
)
```

**Balance computation:**
```sql
SELECT
  SUM(CASE WHEN event_type = 'DEPOSIT' THEN amount_paise ELSE 0 END)
  - SUM(CASE WHEN event_type IN ('RELEASE', 'SEIZURE') THEN amount_paise ELSE 0 END)
FROM escrow_events
WHERE provider_id = $1
```

**Signed payment receipt:**
```
SHA256(event_id + provider_id + amount_paise + audit_period + created_at)
signed by payment microservice Ed25519 private key
```

**Why integer paise (not decimal rupees):**
Floating-point arithmetic is non-deterministic across systems. All amounts are stored and computed as integers (1 rupee = 100 paise). No floating-point anywhere in the payment path.

**Why UUIDv7:**
Time-ordered; no coordinator needed; consistent with [ADR-013](ADR-013-consistency-model.md) (INSERT provider registration is I-confluent with UUIDv7).

**Throughput note:**
At 10,000 providers with monthly payouts: peak ≈ 3 releases/sec. Single-server Postgres handles this without sharding. Not a bottleneck in V2 (Q11-4).

## Consequences

**Positive:**
- Full audit trail — every paise accounted for, permanently
- Idempotency key prevents double-payment race conditions
- CRDT-compatible — append-only rows can be merged across replicas
- No float arithmetic risk

**Negative / trade-offs:**
- Balance queries require a full-table scan per provider (mitigated by materialised view refreshed per audit period)

## References

- [Paper 11 — Bailis](../research/paper-11-bailis-coordination.md): CRDT PN-counter; I-confluence analysis; DECREMENT is non-I-confluent (requires single payment service)
- [Paper 35 — Razorpay API Docs](../research/paper-35-razorpay-upi-docs.md): append-only internal ledger is correct — Razorpay provides no native partial-hold query API; amount_paise BIGINT confirmed; idempotency_key serves double duty as X-Payout-Idempotency header