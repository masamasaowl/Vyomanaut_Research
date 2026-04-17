# Architecture Decision Records

One file per decision. Immutable once accepted — create a new ADR and mark the old one `Superseded` if a decision changes.

Status values: `Accepted` · `Proposed` · `Superseded by ADR-NNN` · `Deprecated`

---

## Accepted decisions

| ADR | Topic | Title | Set by |
|---|---|---|---|
| [ADR-001](ADR-001-coordination-architecture.md) | #1 | Microservices + Kademlia DHT for coordination | Papers 01–05, 07 |
| [ADR-002](ADR-002-proof-of-storage.md) | #2 | PoR Merkle challenge (continuous + transitory) | Papers 05, 07 |
| [ADR-003](ADR-003-erasure-coding.md) | #3 | RS erasure coding: s=16, r=40, r0=8, lf=256 KB | Papers 05, 10 |
| [ADR-004](ADR-004-repair-protocol.md) | #4 | Lazy repair, r0=8, 72 h trigger | Papers 06, 09, 10 |
| [ADR-005](ADR-005-peer-selection.md) | #5 | Storj 4-subsystem reputation pipeline | Papers 01, 07 |
| [ADR-006](ADR-006-polling-interval.md) | #6 | Polling: 24 h; departure threshold: 72 h | Papers 06, 08, 09 |
| [ADR-007](ADR-007-provider-exit-states.md) | #7 | 4 exit states: temp / promised / silent / announced | Papers 08, 09 |
| [ADR-008](ADR-008-reliability-scoring.md) | #8 | 3-window rolling score: 24 h / 7 d / 30 d | Paper 08 |
| [ADR-009](ADR-009-background-execution.md) | #11 | Desktop-only V2; ≤5% CPU for background tasks | Paper 09 |
| [ADR-010](ADR-010-desktop-only-v2.md) | #5 | No mobile providers in V2; deferred to V3 | Papers 05, 06, 08 |
| [ADR-011](ADR-011-escrow-payments.md) | #13 | Fiat escrow; UPI via Razorpay/Cashfree; India-first | Papers 05, 07 |
| [ADR-012](ADR-012-payment-basis.md) | #13 | Payment per audit passed, not per GB stored | Papers 05, 07 |
| [ADR-013](ADR-013-consistency-model.md) | #14 | I-confluence map: 6 coordinated, 14 free | Paper 11 |
| [ADR-014](ADR-014-adversarial-defences.md) | #19 | 4 defence classes: Geppetto / outsourcing / generation / JIT | Paper 07 |
| [ADR-015](ADR-015-audit-trail.md) | #2 | Signed receipt exchange + Transparent Merkle Log | Papers 07, 11 |
| [ADR-016](ADR-016-payment-db-schema.md) | #13 | PN-counter CRDT; paise integer; idempotency key UUID | Paper 11 |
| [ADR-017](ADR-017-audit-receipt-schema.md) | #2 | 12-field audit receipt with Ed25519 dual signatures | Papers 07, 11 |
| [ADR-018](ADR-018-hot-cold-storage-bands.md) | #5 | Explicit hot/cold provider bands | — |

## Proposed (TBD — blocked on future research)

| ADR | Topic | Title | Blocked on |
|---|---|---|---|
| [ADR-019](ADR-019-client-side-encryption.md) | #9 | Client-side encryption scheme | Szabó 2025 (reading-list Phase 1 #10) |
| [ADR-020](ADR-020-key-management.md) | #10 | Key management strategy | Phase 3 |
| [ADR-021](ADR-021-p2p-transfer-protocol.md) | #12 | P2P transfer protocol (libp2p + QUIC candidate) | RFC 9000 + libp2p spec (reading-list Phase 1 #8, #9) |
| [ADR-022](ADR-022-encryption-erasure-order.md) | #15 | Encrypt-then-code vs code-then-encrypt | Szabó 2025 (reading-list Phase 1 #10) |
| [ADR-023](ADR-023-provider-storage-engine.md) | #16 | Provider-side storage engine | Phase 2C |
| [ADR-024](ADR-024-economic-mechanism.md) | #18 | Economic mechanism design | Phase 5 |
