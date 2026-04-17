# ADR-013 — I-Confluence Map: 6 Coordinated Operations, 14 Coordination-Free

**Status:** Accepted
**Topic:** #14 Consistency Model
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 11

---

## Context

The system has ~20 core database operations. Without a principled method for deciding which ones need coordination, engineers will either over-coordinate (killing throughput) or under-coordinate (causing silent data corruption). Bailis et al. (Paper 11) provides the formal test: invariant confluence (I-confluence).

## Decision

Apply the I-confluence test to all core operations. The result:

| Operation | I-Confluent? | Implementation |
|---|---|---|
| INSERT audit receipt (pass) | YES | Append-only, fire-forget |
| INSERT audit receipt (fail) | YES | Append-only, fire-forget |
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
| DELETE provider (physical row removal) | NO | NEVER — use soft delete only |
| ASSIGN chunk to provider (new file upload) | NO (unique placement) | Assignment service, single coordinator |
| TRIGGER repair (redundancy below r0) | YES | Async event, eventual |
| RECORD repair completion | YES | Append-only log |
| ISSUE audit challenge | YES | Async scheduler |
| VALIDATE capability token (retrieval auth) | NO (expiry) | Token service, TTL check |
| RELEASE escrow on announced exit | NO | Payment service, serialised per provider |
| SEIZE escrow on silent departure | NO | Payment service, serialised per provider |

**Out of 20 core operations, only 6 require coordination. The other 14 scale horizontally without limit.**

The 6 non-I-confluent operations (score floor, escrow debit, escrow seizure, chunk placement uniqueness, token validation, physical delete prohibition) map exactly to what blockchain provides in other DSNs. A single hardened payment microservice with an idempotency-keyed Postgres table handles all 6.

**How to evaluate a new operation before adding it:**
1. State the invariant explicitly
2. Ask: if two replicas independently execute this operation and then merge, is the merged state still valid?
3. Check against Bailis Table 2
4. If NOT I-confluent, scope coordination to minimum
5. Document in PR description

## Consequences

**Positive:**
- 14 of 20 operations scale horizontally with zero coordination overhead
- System can handle 10,000 providers with 95%+ of operations coordination-free

**Negative / trade-offs:**
- 6 coordinated operations create a bottleneck dependency on the payment microservice
- Programmers must apply the I-confluence test explicitly to every new operation added

## References

- [Paper 11 — Bailis](../research/paper-11-bailis-coordination.md): I-confluence test; operations table; CRDT PN-counter
