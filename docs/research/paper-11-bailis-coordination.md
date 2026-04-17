# Paper 11 — Coordination Avoidance in Database Systems

**Authors:** Peter Bailis, Alan Fekete, Michael J. Franklin, Ali Ghodsi, Joseph M. Hellerstein, Ion Stoica
**Venue / Year:** UC Berkeley + University of Sydney | VLDB 2015
**Topics:** #14, #1

---

## Problem Solved

1. Every DSN implements consistency wherever possible without testing its implications on reduced scalability
2. This paper gives a formal test — invariant confluence (I-confluence) — to check where consistency is precisely needed
3. When database invariants are specified, we get the sufficient condition for coordination-free execution

---

## Key Findings

**Sec 2 — Coordination cost at scale:**
Coordination is catastrophically bad at scale. With 8 servers and two-phase commit: 173 transactions/second maximum. Our audit log alone will exceed this at 10,000 providers.

**Sec 4.2 — I-confluence test:**
If an operation is I-confluent, do not add coordination. If it is not I-confluent, coordination cannot be removed (see Table 2).

**Sec 5.1 — Uniqueness requires coordination:**
If uniqueness must be maintained when assigning unique values (IDs) into the network, coordination must exist.

**Sec 5.1 — Materialised view maintenance is I-confluent:**
The log of all audits in Postgres must be maintained as a materialised view instead of updating scores on every audit receipt insert.

**Sec 5.1 — Insertions under foreign key constraints are I-confluent:**
All deletions are unsafe without cascading. Do not delete a provider while receipts reference it.

**Sec 5.2 — Threshold direction matters:**
A greater-than (>) threshold invariant is I-confluent for counter increment but not decrement. Reverse for (<). We have a > threshold for reliability scores — the audit checker must be designed accordingly.

**Sec 6.1 — Production scale:**
TPC-C can scale to 12.7M transactions/second with coordination-free execution on 200 servers. Our system, being simpler, can handle 95%+ of operations with zero coordination. The 0.19% cycle cost is the one non-I-confluent operation: ID Assignment Synchronisation (spinlock contention).

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Coordination-free execution | Serialisable execution | Scales linearly with hardware; 25× throughput gain. Requires programmer to specify invariants explicitly. Silent incorrectness if invariants are wrong. |
| Eventual consistency for non-critical ops | Strong consistency everywhere | No central coordinator; merges happen asynchronously. Merge operator must be commutative, associative, idempotent (CAI). |
| Merge-based convergence | Consensus-based convergence | Our reputation score: last-write-wins is NOT CAI (order-dependent). Must use a CRDT counter instead. |

---

## Breaks in Our Case

- **Paper assumes transactions are known in advance** ≠ **our audit events are unbounded and unpredictable**
  → Not a fundamental break; apply I-confluence analysis to each new operation as it is designed

- **Paper's merge operator is set-union** ≠ **our escrow balance requires a CRDT PN-counter**
  → PN-counter tracks increments and decrements separately and merges by summing each column

- **Not all invariants are known yet** ≠ **paper requires complete invariant specification**
  → Apply the 5-step protocol to each new operation before adding it to the codebase

---

## Decisions Influenced

- **[ADR-016](../decisions/ADR-016-consistency-model.md) [#14 Consistency Model — FINAL DECISION]:**

| Operation | I-Confluent? | Implementation |
|---|---|---|
| INSERT audit receipt (audit pass) | YES | Append-only, fire-forget |
| INSERT audit receipt (audit fail) | YES (append) | Append-only, fire-forget |
| UPDATE reputation score (increment on pass) | YES | Async materialised view |
| UPDATE reputation score (decrement on fail) | NO (floor) | Single authoritative scorer, check floor |
| INSERT provider registration | YES (UUID) | UUIDv7, no coordinator |
| INSERT file record | YES (UUID) | UUIDv7, no coordinator |
| INSERT chunk record | YES (UUID) | UUIDv7, no coordinator |
| READ provider metadata | YES | Any replica, no coord |
| READ chunk location | YES | Any replica, no coord |
| DECREMENT escrow balance (payment release) | NO (floor≥0) | Single payment service with idempotency key |
| INCREMENT escrow balance (new deposit) | YES | Async, no coordinator |
| DELETE provider (soft delete, status flag) | YES | Append status update |
| DELETE provider (physical row removal) | NO | NEVER — use soft delete |
| ASSIGN chunk to provider (new file upload) | NO (unique placement) | Assignment service, single coordinator |
| TRIGGER repair (redundancy below r0) | YES | Async event, eventual |
| RECORD repair completion | YES | Append-only log |
| ISSUE audit challenge | YES | Async scheduler |
| VALIDATE capability token (retrieval auth) | NO (expiry) | Token service, TTL check |
| RELEASE escrow on announced exit | NO | Payment service, serialised per provider |
| SEIZE escrow on silent departure | NO | Payment service, serialised per provider |

**Out of 20 core operations, only 6 require coordination. The other 14 scale horizontally without limit.**

- **[ADR-016](../decisions/ADR-016-consistency-model.md) [#14 Consistency Model]:** Coordinate on: escrow debit, escrow seizure, reputation score floor enforcement, chunk placement uniqueness, capability token validation. The 6 non-I-confluent operations map exactly to what blockchains provided in other DSNs (payment finality, audit receipt immutability). A single hardened payment microservice with idempotency-keyed Postgres handles all 6.

---

## Disagreements

- **Classic database literature (Gray 1981, Bernstein 1987):** Advocates for serialisable execution everywhere.
- **CALM Theorem (Hellerstein, Alvaro 2011):** Related theoretical foundation for monotone computation.

---

## Engineering Questions

**Q1 — How to implement the PN-counter CRDT for escrow balance without losing auditability?**

Store every deposit and release as an append-only event row:

```sql
escrow_events table:
  event_id         UUIDv7        PRIMARY KEY
  provider_id      UUID          NOT NULL
  event_type       ENUM(DEPOSIT, RELEASE, SEIZURE)
  amount_paise     BIGINT        NOT NULL  -- never decimals
  audit_period     UUID          FK
  idempotency_key  VARCHAR(64)   UNIQUE    -- SHA256(provider+period)
  created_at       TIMESTAMPTZ   NOT NULL

Balance = SUM(amount) WHERE event_type = DEPOSIT
        - SUM(amount) WHERE event_type IN (RELEASE, SEIZURE)
        for a given provider_id
```

Properties:
1. Full audit trail (every paise accounted for)
2. Idempotency (duplicate release = duplicate key error)
3. CRDT-compatible (append-only, mergeable)
4. No float arithmetic (always integer paise)

The signed receipt for payments:
```
SHA256(event_id + provider_id + amount + audit_period + created_at)
signed by the payment microservice private key
```

**Q2 — What is the exact DB schema for the audit receipt table?**

```sql
audit_receipts table:
  receipt_id              UUIDv7       PRIMARY KEY
  schema_version          SMALLINT     NOT NULL DEFAULT 1
  chunk_id                BYTEA(32)    NOT NULL   -- SHA256 of chunk content
  file_id                 UUID         NOT NULL   -- pseudonymous file handle
  provider_id             UUID         NOT NULL   -- FK → providers (soft delete only)
  challenge_nonce         BYTEA(32)    NOT NULL   -- HMAC(server_secret, chunk_id+ts)
  server_challenge_ts     TIMESTAMPTZ  NOT NULL   -- set by server, not provider
  response_hash           BYTEA(32)    NOT NULL   -- SHA256(chunk||nonce)
  response_latency_ms     INT          NOT NULL   -- just-in-time detector
  audit_result            ENUM(PASS,FAIL,TIMEOUT)
  provider_sig            BYTEA(64)    NOT NULL   -- Ed25519 over all above
  service_sig             BYTEA(64)    NOT NULL   -- Ed25519 over provider_sig+ts
  service_countersign_ts  TIMESTAMPTZ  NOT NULL
-- CONSTRAINTS:
-- INSERT only, no UPDATE, no DELETE (enforced via Postgres row security)
-- provider_id must reference an existing provider (soft FK, checked at insert)
-- receipt_id is UUIDv7 (time-ordered, no coordinator needed)
```

**Q3 — How to apply I-confluence analysis to a new operation before adding it to the codebase?**

1. State the invariant explicitly
2. Ask the merge question: "If two replicas independently execute this operation starting from the same valid state, and then merge, is the merged state still valid under the invariant?"
3. Check against Table 2 of this paper
4. If NOT I-confluent, scope coordination to the minimum
5. Document the decision in the PR description

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q11-1 through Q11-3.
