# Vyomanaut V2 Research

Decentralised cold-storage network for India. Data owners pay to store encrypted files; home desktop operators earn money by holding shards and passing daily cryptographic storage proofs.

The Version 1 of the project is archived at [masamasaowl/Vyomanaut](https://github.com/masamasaowl/Vyomanaut).
V1 failed due to:

1. Lack of research in architecture
2. Structural compromises made during build
3. Inefficient transfer speed, storage, and peer discovery

The version 2 is a research-first rebuild. This repository contains the system design and the full research log behind it.

It can be found at: [masamasaowl/Vyomanaut_V2](https://github.com/masamasaowl/Vyomanaut_V2)

**Three core properties:**

- **Zero-knowledge** — encryption happens entirely on the data owner's device. The network never sees plaintext. The microservice stores only AEAD ciphertext it cannot decrypt.
- **Erasure-tolerant** — each file is split across 56 independent providers using RS(16, 56). Any 16 of 56 fragments reconstruct the file; 40 providers can simultaneously fail without data loss.
- **Proof-gated payments** — providers earn only by passing a daily challenge: `SHA-256(chunk_data ‖ nonce)`. The challenge cannot be answered without the actual stored data.

---

## Documentation

| Document | What it covers |
|---|---|
| [`docs/system-design/architecture.md`](docs/system-design/architecture.md) | System overview, component descriptions, capacity planning, accepted trade-offs |
| [`docs/system-design/requirements.md`](docs/system-design/requirements.md) | Functional and non-functional requirements (FR-001 … FR-069, NFR-001 … NFR-046) |
| [`docs/system-design/data-model.md`](docs/system-design/data-model.md) | Canonical PostgreSQL schema, design invariants, row security policies |
| [`docs/system-design/interface-contracts.md`](docs/system-design/interface-contracts.md) | libp2p wire formats, Go package interfaces, Razorpay webhook contracts |
| [`docs/system-design/mvp.md`](docs/system-design/mvp.md) | Demo vs production mode, `NetworkProfile` struct, build order |
| [`docs/system-design/api/openapi.yaml`](docs/system-design/api/openapi.yaml) | REST API specification (single source of truth for all HTTP contracts) |
| [`docs/decisions/`](docs/decisions/) | ADR-001 → ADR-031 — every architectural decision with rationale |

---

## Build

**Start here: [`docs/system-design/mvp.md §9`](docs/system-design/mvp.md#9-build-order-recommendation)**

Build `internal/config` (`NetworkProfile`) first. Every subsystem takes a profile as a constructor argument. No subsystem reads `VYOMANAUT_MODE` directly.

Demo mode runs the full end-to-end lifecycle (register → vet → upload → audit → repair → pay → depart) in ~35 minutes on one laptop:

```bash
VYOMANAUT_MODE=demo go run ./cmd/microservice \
  --sim-count=5 --sim-asn-count=5
```

Before writing a single line of application code, apply the specification patches in [`docs/system-design/pre-build-errata.md`](docs/system-design/pre-build-errata.md). Five implementation-blocking gaps are resolved there.

---

## Repository Layout

```
cmd/
  microservice/    # Coordination microservice — wiring only, no business logic
  provider/        # Provider daemon; --sim-count=N for local testing
  client/          # Data owner CLI
internal/
  config/          # NetworkProfile struct; ProductionProfile and DemoProfile vars
  crypto/          # AONT, HKDF, Argon2id, Ed25519, BIP-39
  erasure/         # Reed-Solomon RS(16,56) via klauspost/reedsolomon
  storage/         # WiscKey: RocksDB chunk index + append-only vLog
  p2p/             # libp2p host, QUIC/TCP transports, DHT, NAT traversal
  audit/           # Challenge generation, two-phase crash-safe receipt write
  scoring/         # Three-window reliability score, EWMA RTO
  repair/          # Departure detector, lazy repair scheduler, vetting GC
  payment/         # PaymentProvider interface, Razorpay impl, escrow ledger
  client/          # Upload and retrieval orchestrators, session crash recovery
migrations/        # Numbered SQL DDL — never edited after commit
docs/
  decisions/       # ADR-001 … ADR-031
  research/        # Paper notes, answered questions, benchmarking protocols
  system-design/   # The six canonical documents above
```

---

## Seven Invariants

These must never be broken by any migration or code path. They are enforced at the Postgres row-security-policy layer and as pre-conditions in `internal/repair.EnqueueJob`.

1. **Append-only audit log.** `audit_receipts` is INSERT-only. The sole permitted UPDATE promotes a `NULL` (PENDING) row to `PASS`, `FAIL`, or `TIMEOUT`.
2. **Append-only escrow ledger.** `escrow_events` is INSERT-only. Balance is always `SUM(DEPOSIT) − SUM(RELEASE + SEIZURE)`. Never a mutable column.
3. **No physical provider deletion.** `providers` rows are never deleted. The only exit is `status = 'DEPARTED'`.
4. **Integer paise everywhere.** All monetary amounts are `int64` paise (₹1 = 100 paise). No `float64`, `DECIMAL`, or `NUMERIC` in the payment path — ever.
5. **33-byte challenge nonces.** `challenge_nonce` is always `BYTEA(33)` (1 version byte + 32-byte HMAC). Never 32. A 32-byte nonce silently breaks cross-replica failover.
6. **Vetting isolation.** No real file shard is ever assigned to a `VETTING` provider. No repair job is ever enqueued for a synthetic vetting chunk (`is_vetting_chunk = TRUE`).
7. **Constant shard size.** `ShardSize = 262,144` bytes is a compile-time constant in both demo and production. It is not a `NetworkProfile` field. Changing it breaks the vLog entry format, audit challenge framing, and RocksDB index assumptions simultaneously.

---

## CI Gates (must never be disabled)

```
TestDHTKeyValidatorPersists     — catches silent go-libp2p namespace resets
TestNoFloatArithmetic           — rejects float64/float32 in internal/payment/
TestProfileShardSizeIsConstant  — enforces invariant 7 above
Grep: challenge_nonce BYTEA(32) — enforces invariant 5 above
Grep: UPI Collect endpoint      — NPCI compliance (deprecated 2026-02-28)
Grep: non-existent ADR refs     — keeps design doc references honest
```

---

## Status

Research complete. ADR-001 through ADR-031 are locked. **Do not introduce a new architectural approach without opening a new ADR first.** The build follows the phase sequence in `mvp.md §9`: scaffold NetworkProfile → demo pipeline → provider lifecycle → production infrastructure hardening.

---

*Repository: https://github.com/masamasaowl/Vyomanaut_Research*