# ADR-017 — 12-Field Audit Receipt Schema with Ed25519 Dual Signatures

**Status:** Accepted
**Topic:** #2 Proof of Storage (receipt schema)
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 07, 11, 18

---

## Context

The audit receipt is an architectural decision that cannot be altered once providers begin storing receipts locally and the microservice accumulates millions of rows. Every field must be justified before the schema is locked.

The schema is designed to be identical in structure to AWS CloudTrail and Google Certificate Transparency logs — both proven production-scale audit systems.

## Decision

```sql
audit_receipts (
  receipt_id              UUIDv7       PRIMARY KEY,
  schema_version          SMALLINT     NOT NULL DEFAULT 1,
  chunk_id                BYTEA(32)    NOT NULL,  -- SHA256 of chunk content (content address)
  file_id                 UUID         NOT NULL,  -- pseudonymous file handle (not plaintext CID)
  provider_id             UUID         NOT NULL,  -- FK → providers, soft delete only
  challenge_nonce         BYTEA(32)    NOT NULL,  -- HMAC(server_secret, chunk_id + server_ts)
  server_challenge_ts     TIMESTAMPTZ  NOT NULL,  -- set by server, not provider
  response_hash           BYTEA(32)    NOT NULL,  -- SHA256(chunk_data || challenge_nonce)
  response_latency_ms     INT          NOT NULL,  -- JIT detector: time between challenge sent and response received
  audit_result            ENUM(PASS, FAIL, TIMEOUT) NOT NULL,
  provider_sig            BYTEA(64)    NOT NULL,  -- Ed25519 over all fields above
  service_sig             BYTEA(64)    NOT NULL,  -- Ed25519 over (provider_sig + service_ts)
  service_countersign_ts  TIMESTAMPTZ  NOT NULL
)
-- CONSTRAINTS:
-- INSERT only. No UPDATE. No DELETE. Enforced via Postgres row security policy.
-- provider_id must reference a non-deleted provider (soft FK, checked at insert).
-- receipt_id is UUIDv7 (time-ordered, no coordinator needed).
```

**Field-by-field rationale:**

| Field | Attack mitigated |
|---|---|
| `challenge_nonce = HMAC(server_secret, chunk_id + server_ts)` | Clock skew attack — provider cannot predict nonce before challenge is issued |
| `server_challenge_ts` (server-generated) | Prevents provider from backdating challenges |
| `response_hash = SHA256(chunk_data \|\| challenge_nonce)` | Without the chunk, provider cannot compute a valid response |
| `response_latency_ms` | JIT retrieval detector — anomalously high latency flags just-in-time retrieval |
| `provider_sig` (Ed25519 over all fields) | Receipt replay — provider cannot reuse a receipt for a different chunk |
| `service_sig` (Ed25519 over provider_sig + service_ts) | Microservice cannot silently drop a receipt after countersigning |
| `file_id` (pseudonymous, not plaintext CID) | DHT privacy — chunk identity is not exposed in the audit log |

## Consequences

**Positive:**

- Schema is append-only — tamper-evident at the DB level
- Dual Ed25519 signatures (Edwards-curve Digital Signature Algorithm (EdDSA)) make the receipt verifiable by both parties independently
- `response_latency_ms` doubles as a JIT retrieval detector without extra infrastructure
- `schema_version` field allows backward-compatible migration when the schema must evolve

**Negative / trade-offs:**
- Schema cannot be altered without a migration — forward-plan any new fields carefully
- Ed25519 signing must be implemented correctly on the provider daemon — cryptographic bugs here are critical

## References

- [Paper 07 — SoK](../research/paper-07-sok-dsn.md): JIT retrieval attack; privacy leaks through DHT
- [Paper 11 — Bailis](../research/paper-11-bailis-coordination.md): INSERT audit receipt is I-confluent; schema design
- [ADR-014](ADR-014-adversarial-defences.md): adversarial attack classes this schema defends against
- [ADR-015](ADR-015-audit-trail.md): signed receipt exchange flow
- [Paper 18 — Tahoe-LAFS](../research/paper-18-tahoe-lafs.md): domain-separated hashing — every hash use must prepend a purpose-specific tag; challenge nonce HMAC construction already follows this principle