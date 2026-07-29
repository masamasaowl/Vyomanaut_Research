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

## Addendum — Gross Amount and Release Multiplier on RELEASE Events

*Appended following a UX-surface review of the M13 provider-daemon status interface (FR-029). Filed here, against the schema ADR, rather than against [ADR-024](./ADR-024-economic-mechanism.md) (which decides the release-multiplier policy and is unchanged by this addendum) — the fix is a column addition to the table this ADR defines, not a change to when or how much is withheld.*

**Context.** ADR-024 §3 ("Payout release multiplier") already decided that a RELEASE can be for less than the full 30-day accrual — `released = escrow_held × release_multiplier(score)` — and that *"the withheld portion on a partial release is not seized. It is rolled forward into the next [window]."* The `escrow_events` schema above stores only the final `amount_paise` actually transferred on a RELEASE row. Neither the pre-multiplier gross figure nor the multiplier applied is persisted anywhere. `internal/api/provider.go`'s status handler (FR-029, `GET /api/v1/provider/{id}/status`) confirms the practical effect in its own code comment: `held_earnings_paise` is hardcoded to `0`, because the ledger has no way to distinguish a rolled-forward withheld remainder from ordinary not-yet-processed earnings — both would have to be read off the same figure. A provider whose reliability score drops therefore has no way, even in principle, to be told *why* a specific month's payout came in lower than expected.

**Decision.** Extend the `RELEASE` row shape with two additional columns, populated only when `event_type = 'RELEASE'` (NULL otherwise):

```sql
ALTER TABLE escrow_events
  ADD COLUMN gross_amount_paise      BIGINT,  -- pre-multiplier accrual for this release's window; RELEASE rows only
  ADD COLUMN release_multiplier_bps  SMALLINT -- multiplier actually applied, in basis points (9500 = 0.95); RELEASE rows only
    CHECK (release_multiplier_bps IS NULL OR release_multiplier_bps BETWEEN 0 AND 10000);
```

Basis points, not a decimal, for the same reason `amount_paise` is an integer (this ADR's own "why integer paise" note) — no floating point anywhere in the payment path, including the multiplier itself.

`amount_paise` continues to record what was actually transferred (`gross_amount_paise × release_multiplier_bps / 10000`, subject to integer rounding rules already governing paise arithmetic elsewhere). The withheld amount for that release is then simply `gross_amount_paise − amount_paise` — derived at query time, not separately stored, since it isn't a separate ledger fact; it's already reflected in the escrow balance not having been decremented by the full accrual.

`internal/payment`'s release computation (already implementing ADR-024 §3) sets both new columns at the point it currently sets `amount_paise` — no change to *when* or *how much* is released, only to what gets recorded about it.

`GET /api/v1/provider/{id}/status` can now compute a genuine `held_earnings_paise` (sum of `gross_amount_paise − amount_paise` across recent RELEASE rows where `release_multiplier_bps < 10000`) instead of the current hardcoded `0`, and IC §14.1 can gain a row (or the provider status view can render directly) explaining the withholding: *"₹X was withheld this month because your reliability score dropped below 95%. It will roll into next month's payout."*

## Consequences (addendum)

**Positive:** closes the gap `internal/api/provider.go`'s own comment already flags; the provider-facing "why was my payout smaller" question (§6.1's trust-in-earnings promise, and a plausible contributor to early provider churn) finally has a data source; no change to release timing or amounts, only to what's recorded.

**Negative / trade-offs:** two new nullable columns on an append-only, already-populated table (additive migration, no backfill needed — historical RELEASE rows simply have NULL gross/multiplier and fall back to the current `held_earnings_paise = 0` behavior for those older rows).

**Affected:** `escrow_events` schema (migration), `internal/payment` (release computation — sets two more fields), `internal/api/provider.go` (`ProviderStatusHandler`, replaces the hardcoded `0`), IC §14.1 (new copy for the withholding explanation once M13 renders it).

## References

- [Paper 11 — Bailis](../research/paper-11-bailis-coordination.md): CRDT PN-counter; I-confluence analysis; DECREMENT is non-I-confluent (requires single payment service)
- [Paper 35 — Razorpay API Docs](../research/paper-35-razorpay-upi-docs.md): append-only internal ledger is correct — Razorpay provides no native partial-hold query API; amount_paise BIGINT confirmed; idempotency_key serves double duty as X-Payout-Idempotency header
- [ADR-024](./ADR-024-economic-mechanism.md): release-multiplier policy and rolling-hold-window design that this addendum makes queryable
