# Vyomanaut V2 — MVP Roadmap

---

| Field | Value |
|---|---|
| **Document ID** | `VYOM-ROAD-001` |
| **Version** | 2.0 |
| **Status** | Build-ready |
| **Classification** | Internal Engineering |
| **Date** | April 2026 |
| **Author** | Vyomanaut Engineering |
| **Approved by** | — (pending) |
| **Repository** | [masamasaowl/Vyomanaut_Research](https://github.com/masamasaowl/Vyomanaut_Research) |

**Companion documents** (authoritative where this document conflicts):

| Document | Path | Relationship |
|---|---|---|
| System Architecture | `docs/system-design/architecture.md` | Describes the finished system; this document describes how to build it |
| Product Requirements | `docs/system-design/requirements.md` | Defines what must be built; this document defines in what order |
| Accepted Trade-offs | `docs/system-design/trade-offs.md` | Explains what was chosen and why; this document assumes all trade-offs are settled |
| Capacity Estimation | `docs/system-design/capacity.md` | Quantifies scaling limits; this document references them per milestone |
| ADR Index | `docs/decisions/README.md` | One file per architectural decision; milestones reference specific ADRs |
| Open Questions | `docs/research/open-questions.md` | Questions whose answers affect implementation; resolved per milestone |
| Benchmarking Protocol | `docs/research/benchmarking-protocol.md` | Defines how to run the benchmarks required by M1 and M2 |

**Change log:**

| Version | Date | Change | Author |
|---|---|---|---|
| 1.0 | April 2026 | Initial draft | Vyomanaut Engineering |
| 2.0 | April 2026 | Full rewrite: added executive summary, glossary, requirement traceability, risk register, capacity implications, dependency graph, observability rollout, security checklist, appendices | Vyomanaut Engineering |

---

## Executive Summary

This document is the build plan for Vyomanaut V2 — a paid, zero-knowledge, distributed cold storage network for the Indian market. It transforms the system described in `architecture.md` into nine sequential milestones (M0 through M8) that an engineering team of three can execute in six sprints.

**What is being built.** A system where data owners pay to store files, and storage providers — people running a daemon on home desktops or NAS devices — earn money by keeping those files available and passing daily cryptographic audits. The service never sees the data. Files are split into 56 encrypted fragments across 56 independent providers. Any 16 of those 56 are enough to reconstruct the original. Payment is tied to proof, not trust.

**What makes this an MVP.** The MVP is the smallest version of the system that exercises all four critical paths end-to-end under real conditions, is safe to run with real users, and is honest about what it does not yet do. The four critical paths are:

| # | Critical Path | Failure if broken |
|---|---|---|
| 1 | Encode > upload > store | Corrupted data that passes audits and is unrecoverable |
| 2 | Challenge > respond > score | Providers earning without holding data |
| 3 | Departure > repair | Files silently falling below the reconstruction floor |
| 4 | Audit pass > escrow release | Providers working for free, or paid twice |

Every milestone exists to bring one or more of these paths to correctness. Nothing is deferred if it sits on a critical path.

**Timeline.** Six sprints, three engineers. M0, M1, and M2 execute in parallel (Sprint 1). M3 and M4 in Sprint 2. M5 in Sprint 3. M6 in Sprint 4. M7 in Sprint 5. M8 (deployment, beta launch) in Sprint 6. Sprint duration is a team decision; the dependency graph is fixed.

**Key constraints from capacity analysis** (see `capacity.md`):

- At the 56-provider floor, only 1 file segment can upload at a time (all 56 slots occupied)
- Repair window exceeds 12 hours below 500 providers — manual oversight required at launch scale
- Relay infrastructure is the first scaling bottleneck, binding at ~570-850 providers
- Per-provider storage must not exceed ~70 GB at MTTF=180d to stay within bandwidth budget
- Postgres audit INSERT rate is the architectural ceiling at ~100,000 providers x 10,000 chunks

---

## Table of Contents

- [Terminology and Glossary](#terminology-and-glossary)
- [Architecture Context](#architecture-context)
- [Critical Path Analysis](#critical-path-analysis)
- [Milestone Dependency Graph](#milestone-dependency-graph)
- [MVP Scope and Exclusions](#mvp-scope-and-exclusions)
- [Reading the Milestones](#reading-the-milestones)
- [Milestone 0 — Foundation](#milestone-0--foundation)
- [Milestone 1 — Encoding Pipeline](#milestone-1--encoding-pipeline)
- [Milestone 2 — Provider Storage Engine](#milestone-2--provider-storage-engine)
- [Milestone 3 — P2P Networking](#milestone-3--p2p-networking)
- [Milestone 4 — Audit System](#milestone-4--audit-system)
- [Milestone 5 — Reliability Scoring and Repair](#milestone-5--reliability-scoring-and-repair)
- [Milestone 6 — Payment System](#milestone-6--payment-system)
- [Milestone 7 — Data Owner Client](#milestone-7--data-owner-client)
- [Milestone 8 — Network Readiness Gate and Private Beta](#milestone-8--network-readiness-gate-and-private-beta)
- [Cumulative Delivery Summary](#cumulative-delivery-summary)
- [Staffing and Parallel Work](#staffing-and-parallel-work)
- [Observability Rollout Plan](#observability-rollout-plan)
- [Security Verification Checklist](#security-verification-checklist)
- [Invariants That Must Never Break](#invariants-that-must-never-break)
- [Risk Register](#risk-register)
- [Open Questions Resolution Schedule](#open-questions-resolution-schedule)
- [The First Five Minutes](#the-first-five-minutes)
- [Appendix A — Requirement Traceability Matrix](#appendix-a--requirement-traceability-matrix)
- [Appendix B — ADR Reference Index](#appendix-b--adr-reference-index)
- [Appendix C — Research Paper Index](#appendix-c--research-paper-index)
- [Appendix D — Capacity Quick-Reference](#appendix-d--capacity-quick-reference)

---

## Terminology and Glossary

| Term | Definition |
|---|---|
| **AONT** | All-or-Nothing Transform. An encryption scheme where the key K is embedded in the ciphertext and can only be recovered by assembling all codewords. Used before erasure coding so that possessing fewer than k=16 fragments reveals nothing. |
| **ASN** | Autonomous System Number. Identifies an ISP or network operator. The 20% ASN cap ensures no correlated provider group holds more than ~11 of 56 fragments of any file. |
| **BWavg** | Steady-state repair bandwidth per provider (Kbps). Computed via Giroire Formula 1. Target: <= 100 Kbps. At MTTF=300d, ~39 Kbps. |
| **Canary** | A fixed 16-byte value appended to plaintext before AONT encoding. Verified on decode to detect corruption. If the canary fails, the segment is corrupt. |
| **Chunk** | A 256 KB encrypted fragment. The atomic unit of storage, audit, and repair. Each file segment produces 56 chunks. |
| **Circuit Relay v2** | A libp2p protocol for routing traffic through an intermediary when direct connections are impossible (symmetric NAT). |
| **DCUtR** | Direct Connection Upgrade through Relay. A libp2p NAT hole-punching protocol with 97.6% first-attempt success rate for cone NAT. |
| **DHT** | Distributed Hash Table. Specifically, Kademlia. Used for chunk-address lookup on the data plane. Provider discovery goes through the microservice, not the DHT. |
| **Ed25519** | A digital signature scheme. Used for audit receipts (both provider and microservice sign), pointer file integrity, and peer identity. |
| **Escrow** | Funds held by the system on behalf of providers. Balance is computed, never stored: `SUM(DEPOSIT) - SUM(RELEASE + SEIZURE)`. Integer paise only. |
| **Giroire Formula** | A family of analytical formulas for computing data loss rate, repair bandwidth, and burst transfer volume in lazy-repair erasure-coded systems. From Paper 10. |
| **HKDF** | HMAC-based Key Derivation Function (SHA-256). Derives file keys, pointer keys, and keystore keys from the master secret. |
| **JIT flag** | Just-In-Time retrieval detection. Set when a provider responds faster than `0.3x` the expected transfer time, suggesting they did not read from local disk. |
| **K (AONT key)** | A fresh 256-bit random key generated per segment. Embedded in the erasure-coded data via `c_{s+1} = K XOR SHA-256(all codewords)`. Never stored or transmitted separately. |
| **lf** | Fragment (chunk) size. Fixed at 256 KB (262,144 bytes) in V2. |
| **Master secret** | `Argon2id(passphrase, owner_id)`. The root of the data owner's key hierarchy. Never written to disk or transmitted. |
| **MTTF** | Mean Time To Failure. For V2 desktop providers: target 300 days, minimum acceptable 180 days. |
| **Pointer file** | A per-file metadata structure containing provider IDs, chunk content addresses, and erasure parameters. Encrypted with AEAD_CHACHA20_POLY1305. Stored as ciphertext by the microservice (which cannot decrypt it). |
| **PN-counter CRDT** | A conflict-free replicated data type for counters that support both increment and decrement. The escrow ledger uses this pattern. |
| **Qpeek** | Burst repair bandwidth: total network transfer required when one provider fails. At N=1,000, 50 GB/provider: ~793 GB. |
| **r** | Number of parity fragments. Fixed at 40 in V2. Analytically optimal per Giroire Formula 4. |
| **r0** | Lazy repair trigger buffer. Fixed at 8. Repair fires when available fragments drop to s+r0=24, not at every loss. Reduces bandwidth by ~38x vs eager repair. |
| **RS(s, n)** | Reed-Solomon erasure code with s data shards and n total shards. V2: RS(16, 56). Systematic form: first 16 output shards are identity-mapped from the AONT package. |
| **RTO** | Retransmission Timeout. Per-provider: `AVG + 4 x VAR` of recent audit response latencies. New providers use pool median. |
| **s** | Number of data (reconstruction threshold) shards. Fixed at 16. |
| **Segment** | A 14 MB unit of file data (56 x 256 KB). Files larger than 14 MB are split into multiple segments, each processed independently. |
| **Silent departure** | A provider absent >= 72 hours without announcement. Triggers escrow seizure and immediate repair. |
| **vLog** | The append-only Value Log in the WiscKey storage engine. Fixed-size entries of 262,212 bytes. All reads verify `SHA-256(chunk_data) == content_hash`. |
| **Vetting period** | 4-6 months after provider registration. 60-day escrow hold, 50% release cap. Ends after 80 consecutive audit passes. |
| **WiscKey** | Key-value separation architecture: small index in RocksDB, large values in an append-only log. Reduces write amplification from 10-14x to ~1.0 at 256 KB values. |

---

## Architecture Context

This section provides enough context for the roadmap to be read standalone. The full architecture is in `architecture.md`.

### System overview

```
  DATA OWNER                                              PROVIDER (x56+)
  (desktop app)                                       (daemon on desktop/NAS)
      |                                                       |
      | HTTPS: metadata, pointer files                        | libp2p/QUIC: chunks
      | QUIC: direct P2P chunk upload                         | HTTPS: heartbeat, audit
      |                                                       |
      +------------- COORDINATION MICROSERVICE ---------------+
                     (control plane only)
                            |
                  +---------+----------+
                  |                    |
              RAZORPAY          SECRETS MANAGER
          (payment rails)    (cluster audit secret)
```

**The microservice never touches file data.** It knows which chunk is on which provider, who passed their last audit, and who should be paid. It stores encrypted pointer file ciphertext it cannot decrypt. All file data moves directly between data owners and providers via libp2p.

### Technology stack

| Layer | Technology |
|---|---|
| Language | Go (microservice, daemon, client) |
| Database | PostgreSQL (INSERT-only audit log, CRDT escrow ledger) |
| Provider storage | RocksDB (index) + append-only vLog (values) — WiscKey separation |
| P2P networking | libp2p (QUIC v1 primary, TCP+Noise XX fallback) |
| Encryption | ChaCha20-256 (no AES-NI) / AES-256-CTR (AES-NI); AEAD_CHACHA20_POLY1305 for pointer files |
| Erasure coding | Reed-Solomon RS(16, 56) via `klauspost/reedsolomon` |
| Key derivation | HKDF-SHA256 (operational), Argon2id (master secret) |
| Signatures | Ed25519 |
| Payment | Razorpay (Route + Smart Collect 2.0 + RazorpayX Payouts) |

### Erasure parameters (V2 — fixed, do not change)

| Parameter | Value | Meaning |
|---|---|---|
| s | 16 | Data shards (reconstruction threshold) |
| r | 40 | Parity shards |
| n = s + r | 56 | Total shards per segment |
| r0 | 8 | Lazy repair buffer above floor |
| s + r0 | 24 | Repair trigger threshold |
| lf | 256 KB | Fragment size |
| lb = s x lf | 4 MB | Block size (one full segment of user data) |
| Max segment on wire | 14 MB | n x lf = 56 x 256 KB |
| Storage efficiency | 28.57% | s/n = 16/56 |

### Key hierarchy (data owner side)

```
passphrase + owner_id
    |
    v  Argon2id (t=3, m=64 MB, p=4)
    master_secret ──────────────────── BIP-39 mnemonic (offline backup)
    |
    +-- HKDF(\"vyomanaut-file-v1\" || file_id)     --> file_key
    +-- HKDF(\"vyomanaut-pointer-v1\" || file_id)   --> pointer file encryption key
    +-- HKDF(\"vyomanaut-keystore-v1\")              --> keystore encryption key
```

The AONT key K is **not** in this hierarchy. It is embedded in the erasure-coded fragments and recovered automatically when k=16 shards are assembled.

---

## Critical Path Analysis

The four critical paths define the MVP boundary. Each path is a sequence of steps where a bug anywhere in the chain produces a wrong outcome that the user **cannot detect** and **cannot recover from**.

### Path 1: Encode > Upload > Store

**Data integrity from data owner to provider disk.**

```
Plaintext
  --> [Pad if < 4 MB]              FR-008
  --> [Segment at 14 MB]           architecture.md $10
  --> [AONT transform]             ADR-019, ADR-022 | Milestone 1
  --> [RS(16,56) dispersal]        ADR-003          | Milestone 1
  --> [56 chunks of 256 KB]
  --> [P2P upload to 56 providers] ADR-021          | Milestone 3
  --> [Store in vLog + RocksDB]    ADR-023          | Milestone 2
  --> [Pointer file encrypted]     ADR-020          | Milestone 1
  --> [Pointer ciphertext stored]                   | Milestone 7
```

**Milestones involved:** M1 (encoding), M2 (storage engine), M3 (P2P), M7 (client orchestration).

**Failure modes this path prevents:**
- Corrupted data that passes audits → canary verification, content_hash on every read
- Data readable by the service → AONT key K never leaves the client; microservice holds only encrypted pointer ciphertext
- Single-provider failure causes data loss → 40 parity fragments; survives loss of any 40 of 56

### Path 2: Challenge > Respond > Score

**Proof that providers actually hold the data they claim.**

```
Microservice
  --> [Generate challenge_nonce]       ADR-027      | Milestone 4
  --> [Dispatch via heartbeat address] ADR-028      | Milestone 3, 4
  --> [Provider reads chunk from vLog] ADR-023      | Milestone 2
  --> [Verify content_hash]            ADR-023      | Milestone 2
  --> [Compute response_hash]          ADR-002      | Milestone 4
  --> [Sign receipt (Ed25519)]         ADR-017      | Milestone 4
  --> [Microservice verifies + records INSERT-only]  | Milestone 4
  --> [Update reliability score]       ADR-008      | Milestone 5
```

**Milestones involved:** M2 (storage read), M3 (network delivery), M4 (audit system), M5 (scoring).

**Failure modes this path prevents:**
- Provider fakes a response without holding data → SHA-256(chunk_data || nonce) is uncomputable without the chunk
- Provider outsources retrieval just-in-time → Timing deadline: `(chunk_size / p95_throughput) x 1.5`
- Audit results are altered after the fact → INSERT-only table enforced by Postgres row security policy

### Path 3: Departure > Repair

**Self-healing when providers leave the network.**

```
Provider silent >= 72h
  --> [Detect via heartbeat timeout]    ADR-006, ADR-007 | Milestone 5
  --> [Set status = DEPARTED]           ADR-007           | Milestone 5
  --> [Seize escrow]                    ADR-024           | Milestone 6
  --> [Enqueue repair jobs]             ADR-004           | Milestone 5
  --> [Download 16 fragments]           ADR-003           | Milestone 1, 3
  --> [RS decode + re-encode missing]   ADR-003           | Milestone 1
  --> [Select replacements (ASN cap)]   ADR-005, ADR-014  | Milestone 5
  --> [Upload to new providers]         ADR-021           | Milestone 3
  --> [Update chunk_assignments]                          | Milestone 5
```

**Milestones involved:** M1 (RS decode/encode), M3 (P2P transfer), M5 (repair scheduler), M6 (escrow seizure).

**Capacity constraint:** At N < 500 providers, repair window exceeds 12 hours. Manual oversight required at launch scale (`capacity.md` Section 3.2).

### Path 4: Audit Pass > Escrow Release

**Providers get paid for verified work.**

```
Audit PASS recorded
  --> [Accumulate per-audit earnings]       ADR-012      | Milestone 6
  --> [Monthly: compute 30d reliability]    ADR-008      | Milestone 5
  --> [Apply release multiplier]            ADR-024      | Milestone 6
  --> [Check dual-window deterioration]     ADR-024      | Milestone 6
  --> [Razorpay PATCH on_hold_until]        Paper 35     | Milestone 6
  --> [INSERT RELEASE event]                ADR-016      | Milestone 6
  --> [Provider receives INR to bank]                    | Milestone 6
```

**Milestones involved:** M5 (scoring), M6 (payment system).

**Failure modes this path prevents:**
- Provider paid without holding data → Only PASS receipts generate DEPOSIT events
- Provider paid twice → Idempotency key = `SHA-256(provider_id + audit_period)` on every payout
- Float rounding errors in payment → All amounts are integer paise; float arithmetic is a correctness violation

---

## Milestone Dependency Graph

```
                    +-------+
                    |  M0   |  Foundation
                    | repo, |  (CI, schema, sim mode)
                    | CI,   |
                    | schema|
                    +---+---+
                        |
          +-------------+-------------+
          |             |             |
      +---v---+     +--v----+     +--v----+
      |  M1   |     |  M2   |     |  M3   |
      |Encoding|    |Storage |    |  P2P   |
      |Pipeline|    | Engine |    |Network |
      +---+---+     +---+---+     +---+---+
          |             |             |
          |             +------+------+
          |                    |
          |               +----v----+
          |               |   M4    |
          |               |  Audit  |
          |               |  System |
          |               +----+----+
          |                    |
          |               +----v----+
          +-------------->|   M5    |
                          |Scoring &|
                          | Repair  |
                          +----+----+
                               |
                          +----v----+
                          |   M6    |
                          | Payment |
                          +----+----+
                               |
          +--------------------+
          |
      +---v---+
      |  M7   |  (also depends on M1, M3, M4, M6)
      | Data  |
      | Owner |
      | Client|
      +---+---+
          |
      +---v---+
      |  M8   |
      | Beta  |
      | Launch|
      +-------+

Sprint 1:  M0 + M1 + M2  (parallel, independent code paths)
Sprint 2:  M3 + M4        (M4 depends on M2 and M3)
Sprint 3:  M5             (depends on M4)
Sprint 4:  M6             (depends on M5)
Sprint 5:  M7             (depends on M1, M3, M4, M6)
Sprint 6:  M8             (all milestones complete)
```

### Parallelisation rules

| Sprint | Parallel tracks | Gate condition |
|---|---|---|
| 1 | M0, M1, M2 run independently | M0 provides repo layout; M1 and M2 have no code dependencies on each other |
| 2 | M3 can start independently; M4 requires M2 (storage read) + M3 (network dispatch) | M4 cannot begin integration testing until M2 and M3 pass their exit criteria |
| 3 | M5 is full-team; no parallelisation | M4 exit criteria met (audit receipts flowing) |
| 4 | M6 can be single-engineer | M5 exit criteria met (scores computed, repair functional) |
| 5 | M7 is 2 engineers | M1 + M3 + M4 + M6 all passing |
| 6 | M8 is full-team deployment | All milestones passing CI |

---

## MVP Scope and Exclusions

### What the MVP is

The MVP is the smallest system that:

1. Exercises every critical path end-to-end under real conditions
2. Is safe to run with real users (not a prototype — correctness first)
3. Is honest about what it does not yet do

### What the MVP is not

These are deliberately excluded. They are recorded here so no one builds them accidentally before the critical paths work.

| Excluded | Deferred to | Rationale |
|---|---|---|
| Provider local dashboard (tray app / web UI) | V2 post-MVP | CLI-only is sufficient for private beta (PR-02 is P1, not P0) |
| Hot Storage Band | V3 | Separate erasure parameters, separate payment model; not researched |
| Transparent Merkle Log | V3 | Signed receipts are sufficient for V2 trust; public verifiability requires infrastructure not ready |
| Hitchhiker repair bandwidth optimisation | V3 | BWavg is ~39 Kbps at launch scale, well under the 100 Kbps budget |
| Mobile providers | V3 | BWavg at MTTF=90d is ~130 Kbps, exceeding the budget; iOS/Android background limits unresolved |
| International payments (Stripe Connect) | Post-India-launch | `PaymentProvider` interface is ready; implementation requires multi-jurisdiction compliance |
| File versioning | Never in V2 | Files are immutable by design (Trade-off 15) |
| Convergent encryption / deduplication | Never | Privacy is the product (Trade-off 15, ADR-022) |
| Geographic proximity routing (Coral DSHT) | V3 | Inter-city latency is homogeneous in India at launch scale |
| Upload optimality threshold (cancel slow shards) | V3 | Requires cancel signals to slow providers; complexity not justified at launch |

---

## Reading the Milestones

Each milestone is a **sprint epic**. A sprint epic ends when:

1. Every component in scope passes its **exit criterion** — a concrete test you can run
2. The system is **runnable** — not just compilable, but executable in a state a reviewer can observe
3. Nothing from a previous milestone has regressed

The milestones are strictly sequential. Milestone N depends on Milestone N-1 being correct. Parallelising them is possible within a team only when the dependency graph allows it — the dependency notes in each milestone make this explicit.

### Per-milestone structure

Each milestone contains:

| Section | Purpose |
|---|---|
| **Goal** | One sentence: what is true after this milestone that was not true before |
| **Dependencies** | Which prior milestones must be complete |
| **Requirement traceability** | Which FR and NFR IDs from `requirements.md` are satisfied |
| **Scope** | Technical specification of what is built |
| **Tests** | Table-driven unit tests and integration tests |
| **Benchmarks** | Performance targets that must pass on minimum-spec hardware |
| **Observability** | Metrics wired in this milestone (cumulative plan in [Observability Rollout](#observability-rollout-plan)) |
| **Risk assessment** | What could go wrong in this milestone, and what to do about it |
| **Capacity notes** | Implications from `capacity.md` relevant to this milestone |
| **Exit criterion** | Concrete commands that prove the milestone is done |

---

## Milestone 0 — Foundation

**Goal:** A monorepo with tooling, CI, and a 56-node simulated provider network running on a single developer laptop. No business logic yet. This milestone establishes the substrate everything else builds on.

**Dependencies:** None.
**Parallelisable with:** M1, M2 (no code overlap).

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-055 | `--sim-count=N` launches N simulated providers | Simulation mode skeleton implemented here |
| FR-056 | Simulation must not bypass readiness gate | Gate wired (all conditions FAIL in M0) |
| NFR-021 | `audit_receipts` INSERT-only row security policy | Postgres schema created with policy enforced |
| NFR-022 | `escrow_events` INSERT-only | Schema created with policy enforced |

### Scope

**Repository layout:**

```
/
  cmd/
    microservice/     <-- coordination microservice entrypoint
    provider/         <-- provider daemon entrypoint (supports --sim-count flag)
    client/           <-- data owner CLI entrypoint
  internal/
    erasure/          <-- (empty, filled in M1)
    crypto/           <-- (empty, filled in M1)
    storage/          <-- (empty, filled in M2)
    p2p/              <-- (empty, filled in M3)
    audit/            <-- (empty, filled in M4)
    scoring/          <-- (empty, filled in M5)
    repair/           <-- (empty, filled in M5)
    payment/          <-- (empty, filled in M6)
  migrations/         <-- SQL schema migrations (Postgres)
  docs/               <-- existing research and system-design (read-only during build)
  scripts/            <-- dev tooling (lint, test, sim)
```

**Postgres schema — all tables, created now, populated later.**

Create every table defined in the ADRs in a single `migrations/001_initial_schema.sql`. Tables that cannot be filled yet get their rows inserted by later milestones. The schema is the contract — get it right once, here.

Key constraints to enforce from day one:

| Constraint | Source | Why |
|---|---|---|
| `audit_receipts`: row security policy blocking UPDATE and DELETE | FR-039, NFR-021 | Tamper-evident audit log |
| `escrow_events`: row security policy blocking UPDATE and DELETE | NFR-022 | CRDT-safe ledger |
| `audit_receipts.challenge_nonce`: BYTEA(33) not BYTEA(32) | ADR-027, requirements.md Section 9.3 | 1-byte version prefix + 32-byte HMAC |
| `audit_receipts.audit_result`: accepts NULL | ADR-015 | In-flight PENDING state |
| `audit_receipts.challenge_nonce`: UNIQUE index | ADR-015 | Idempotent retry |
| `providers.last_known_multiaddrs`: JSONB column | ADR-028 | Heartbeat-maintained addresses |

> **Cross-document correction:** `architecture.md` Section 14 shows `challenge_nonce BYTEA(32)` — this is incorrect. The schema must use `BYTEA(33)` (32-byte HMAC + 1-byte version prefix per ADR-027). Fix `architecture.md` before or during M0.

**Provider daemon simulation mode (FR-055, FR-056):**

The `--sim-count=N` flag launches N independent provider instances in one process. Each gets its own:

| Resource | Location | Notes |
|---|---|---|
| Ed25519 key pair | `/tmp/vyomanaut-sim/{id}/keys/` | Generated fresh, persisted |
| RocksDB instance | `/tmp/vyomanaut-sim/{id}/db/` | Isolated per instance |
| vLog file | `/tmp/vyomanaut-sim/{id}/vlog/chunks.vlog` | Isolated per instance |
| libp2p listen address | `127.0.0.1:{base_port + i}` | No port conflicts |
| Synthetic ASN | `SIM-AS{(i % sim_asn_count) + 1}` | Default `--sim-asn-count=5` |
| Synthetic region | `[\"Delhi\", \"Mumbai\", \"Bangalore\"][i % 3]` | Covers the 3-region minimum |

Simulation mode must not bypass the network readiness gate. The simulation with `--sim-count=56 --sim-asn-count=5` must satisfy all seven readiness conditions (FR-056) before any upload can proceed. The gate will trivially fail in M0 (most conditions not yet implemented) — that is correct.

**CI pipeline:**

| Check | Requirement |
|---|---|
| `go build ./...` | Zero warnings (strict mode from day one) |
| `go vet ./...` | Pass |
| `go test ./...` | Pass (no tests yet — harness wired, tests come per milestone) |
| `golangci-lint run` | Checked-in `.golangci.yml` config |
| Postgres migration | Applied and rolled back cleanly against CI Postgres instance |
| All checks | Run on every PR; nothing merges red |

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Schema design error discovered in later milestone | Medium | High (migration required) | Review schema against all ADRs before M0 closes; use explicit schema version in migration filename |
| Simulation mode port conflicts on CI runners | Low | Medium | Use `--sim-base-port` flag with CI-specific range |
| RocksDB CGo build fails on CI | Low | High | Pin RocksDB version; cache CGo builds in CI |

### Exit criterion

```bash
# Simulation starts and produces structured output (all gates show FAIL — expected in M0)
go run ./cmd/provider --sim-count=56 --sim-asn-count=5 \
  --microservice-url=http://localhost:8080 \
  --sim-data-dir=/tmp/vyomanaut-sim

# Readiness gate returns JSON with all conditions FAIL
curl http://localhost:8080/api/v1/admin/readiness | jq .

# CI passes green
```

---

## Milestone 1 — Encoding Pipeline

**Goal:** A correct, benchmarked, and tested encoding pipeline that transforms plaintext into 56 encrypted RS fragments and back. This is the zero-knowledge property in code.

**Dependencies:** M0 (repository layout, Go toolchain).
**Parallelisable with:** M2 (independent code paths), but not with anything downstream — every later milestone depends on this being correct.

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-002 | Argon2id master secret derivation | `internal/crypto` implements `Argon2id(passphrase, owner_id, t=3, m=65536, p=4)` |
| FR-007 | All encryption on data owner's device | Encoding pipeline is a local library, not a service |
| FR-008 | AONT-RS encoding with padding | Full pipeline: pad > segment > AONT > RS(16,56) |
| FR-011 | Pointer file AEAD encryption | `AEAD_CHACHA20_POLY1305` with HKDF-derived key |
| FR-017 | SHA-256 content address per fragment | Each shard's chunk_id = SHA-256(shard_data) |
| FR-018 | Canary verification on decode | Fixed 16-byte canary verified after AONT decode |
| NFR-009 | AONT encode p50 <= 200 ms (no AES-NI) | Benchmarked on minimum-spec hardware |
| NFR-010 | Argon2id p50 <= 500 ms | Benchmarked on minimum-spec hardware |
| NFR-014 | Service never holds decryption key | K is embedded in AONT, never transmitted |
| NFR-019 | Constant-time Poly1305 tag comparison | Pointer file decryption uses `crypto/subtle.ConstantTimeCompare` |

### Scope

**Package: `internal/crypto`**

Key hierarchy (ADR-019, ADR-020):

| Derivation | Function | Output |
|---|---|---|
| Master secret | `Argon2id(passphrase, owner_id, t=3, m=65536, p=4)` | `[32]byte` |
| File key | `HKDF-SHA256(master_secret, salt=owner_id, info=\"vyomanaut-file-v1\"+file_id)` | `[32]byte` |
| Pointer enc key | `HKDF-SHA256(master_secret, salt=owner_id, info=\"vyomanaut-pointer-v1\"+file_id)` | `[32]byte` |
| Keystore enc key | `HKDF-SHA256(master_secret, salt=owner_id, info=\"vyomanaut-keystore-v1\")` | `[32]byte` |

AES-NI detection at package init via CPUID (x86) — sets a package-level constant.

**ChaCha20 AONT pass** (no AES-NI path, ADR-019):

```
K = SecureRandom(256 bits)
for i, word in enumerate(16-byte words of segment + canary):
    c_i = word XOR ChaCha20(key=K, counter=i/4, nonce=zeros)[word_offset]
h = SHA-256(c_0 || ... || c_s)
c_{s+1} = K XOR h[:32]
```

**AES-256-CTR AONT pass** (AES-NI path, ADR-019):

```
K = SecureRandom(256 bits)
for i, word in enumerate(words):
    c_i = word XOR AES-256-ECB(K, i+1)    // counter mode
h = SHA-256(c_0 || ... || c_s)
c_{s+1} = K XOR h[:32]
```

**Canary:** 16-byte fixed value appended before AONT. Verified during decode.

**Pointer file** schema and AEAD encryption (`AEAD_CHACHA20_POLY1305`):
- Plaintext schema exactly as defined in ADR-022 and `architecture.md` Section 10
- 96-bit nonce: counter from keystore, never random
- AAD: `owner_id || file_id || schema_version`
- Ed25519 signature by the data owner over all fields

**Package: `internal/erasure`**

Reed-Solomon RS(s=16, r=40):
- Use `klauspost/reedsolomon` (production-quality GF(2^8) library)
- Systematic form: first 16 output shards are identity-mapped from the AONT package
- Shards are exactly lf=256 KB (262,144 bytes) each
- Segment padding: files < 4 MB padded to minimum segment size; original size recorded in pointer file

**Full encode pipeline** (ADR-022):

```
plaintext --> [pad if < 4 MB] --> [segment at 14 MB boundaries]
          --> [AONT transform] --> [RS(16,56) dispersal]
          --> [56 shards of 256 KB]
```

**Full decode pipeline:**

```
[any 16 shards] --> [RS decode --> AONT package]
                --> [recover K: h=SHA256(all codewords), K=c_{s+1} XOR h]
                --> [decrypt each word] --> [verify canary] --> [strip padding]
                --> plaintext
```

### Tests

All tests use table-driven subtests. Fuzzing is strongly encouraged for the AONT round-trip.

| Test | Verifies |
|---|---|
| `TestAONTRoundTrip/ChaCha20` | Encode then decode, canary passes, plaintext identical |
| `TestAONTRoundTrip/AES` | Same with AES path |
| `TestAONTKeyRecovery` | K must not be computable with < s+1 codewords |
| `TestAONTCorruptionDetection` | Flip any bit in any codeword, canary fails |
| `TestRSRoundTrip` | Encode then decode with exactly 16 shards |
| `TestRSAnyKShards` | 100 random subsets of size 16 each reconstruct correctly |
| `TestRSShardSize` | Each output shard is exactly 262,144 bytes |
| `TestFullPipeline` | plaintext > 56 shards > reconstruct from any 16 > plaintext |
| `TestPaddingRoundTrip` | File < 4 MB pads and strips correctly |
| `TestKeyHierarchy` | HKDF derivation is deterministic given same inputs |
| `TestArgon2idDeterminism` | Same passphrase + owner_id always produces same master_secret |
| `TestPointerFileEncryption` | Encrypt then decrypt, AAD verified, tampered ciphertext fails |

### Benchmarks (must pass before milestone closes)

Run on minimum-spec hardware (dual-core, no AES-NI, 2 GB RAM, HDD) per `docs/research/benchmarking-protocol.md`:

| Benchmark | Target | Source |
|---|---|---|
| `BenchmarkAONTEncode14MB/ChaCha20` | p50 <= 200 ms, p99 <= 400 ms | Q16-1, NFR-009 |
| `BenchmarkArgon2id` | p50 <= 500 ms at t=3, m=64 MB | Q18-1, NFR-010 |

Record hardware spec alongside benchmark output. If Argon2id fails on minimum-spec, apply the fallback protocol in `docs/research/benchmarking-protocol.md` Q18-1 and update ADR-020 with the confirmed parameters.

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Argon2id exceeds 500 ms on minimum-spec | Medium | High | Fallback protocol defined in benchmarking-protocol.md; reduce t or m per documented procedure |
| `klauspost/reedsolomon` API breaking change | Low | Medium | Pin dependency version; vendor if necessary |
| AONT round-trip fuzzing finds edge case | Low | Critical | Fuzzing is mandatory, not optional. Fix before closing M1. |

### Capacity notes

- AONT encoding throughput directly determines upload latency. At p50=200 ms per 14 MB segment and 100 MB file (8 segments): encoding phase = ~1.6 seconds. Well within the NFR-033 target of 3 minutes total for 100 MB.
- The encoding pipeline is CPU-bound. At 5% background CPU budget (NFR-011), a provider assisting with repair (which requires RS decode + re-encode) can process approximately 1 repair segment every 4 seconds on minimum-spec hardware.

### Exit criterion

```bash
go test ./internal/crypto/... ./internal/erasure/... -v -race
# All tests pass, zero data races

go test ./internal/crypto/... -bench=. -benchtime=100x
# Benchmark output recorded, p50 targets met on minimum-spec machine
```

---

## Milestone 2 — Provider Storage Engine

**Goal:** A provider daemon that can store and retrieve 256 KB chunks from local disk correctly and efficiently, with corruption detection, write-safety, and crash recovery.

**Dependencies:** M0 (repository layout).
**Parallelisable with:** M1 (independent code paths).

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-028 | Provider reads chunk and verifies content_hash | vLog read + SHA-256 verification on every audit lookup |
| FR-041 | Corruption detected = FAIL with corruption code | content_hash mismatch returns FAIL immediately |
| NFR-008 | Audit lookup p99 <= 100 ms SSD, 200 ms HDD | WiscKey design: 1 random read per lookup |
| NFR-013 | Write amplification <= 1.1x at 256 KB | WiscKey separation: compaction moves only 44-byte index entries |
| NFR-023 | Single writer goroutine for vLog | Serialised appends via buffered channel |
| NFR-024 | Crash recovery: scan vLog tail on restart | Re-insert missing index entries from valid vLog entries |

### Scope

**Package: `internal/storage`**

**WiscKey key-value separation** (ADR-023):

*The chunk index (RocksDB):*

| Property | Value |
|---|---|
| Key | `chunk_id` (32 bytes, SHA-256 of chunk content) |
| Value | `vlog_offset` (uint64) + `chunk_size` (uint32) = 12 bytes |
| Total entry size | 44 bytes |
| Bloom filter | 10 bits per key (~1% false-positive rate) |
| `max_background_compactions` | 1 |
| `max_background_flushes` | 1 |
| `rate_limiter` | 10 MB/s starting value (calibrate via Q27-1 benchmark) |

SSD vs HDD detection at daemon startup via `/sys/block/{dev}/queue/rotational` (Linux) or platform equivalent; sets rate_limiter from two separate constants.

*The value log (vLog):*

Single append-only file, fixed-size entries of exactly 262,212 bytes:

```
chunk_id     (32 bytes)    -- content address
chunk_size   (4 bytes)     -- always 262,144 in V2
chunk_data   (262,144 bytes) -- raw 256 KB fragment
content_hash (32 bytes)    -- SHA-256(chunk_data), verified on every read
```

Critical properties:
- `fsync()` the vLog entry before the RocksDB insert (write ordering)
- Single writer goroutine with a buffered channel (buffer size: 32); all other goroutines submit write requests and block on a result channel (ADR-023)

*Crash recovery:*

1. On startup, read `last_vlog_head_offset` from RocksDB
2. Scan vLog from that offset to EOF
3. For each complete entry: verify content_hash; if valid and chunk_id absent from RocksDB, re-insert the index entry
4. Incomplete (truncated) tail entries are discarded

*Garbage collection:*

- Triggered only by explicit chunk deletion events (announced departure, file owner delete, repair reassignment)
- Read vLog from tail, check each chunk_id against RocksDB; absent = invalid
- Append valid entries to vLog head; update `last_vlog_tail_offset` in RocksDB
- `fallocate(FALLOC_FL_PUNCH_HOLE)` to release freed regions

*Audit lookup path* (must complete within NFR-008 thresholds):

```
receive (chunk_id, challenge_nonce)
  --> Bloom filter check: absent --> FAIL immediately (no disk I/O)
  --> RocksDB get chunk_id --> (vlog_offset, chunk_size) (block cache hit = no disk I/O)
  --> read 262,212 bytes from vLog at vlog_offset (one random read)
  --> verify SHA-256(chunk_data) == content_hash --> FAIL with corruption code if mismatch
  --> response_hash = SHA-256(chunk_data || challenge_nonce)
  --> return response_hash
```

### Tests

| Test | Verifies |
|---|---|
| `TestStoreAndRetrieve` | Store a chunk, retrieve by chunk_id, data identical |
| `TestBloomFilterNegative` | Lookup of un-stored chunk_id never touches disk |
| `TestContentHashVerification` | Corrupt one byte in vLog entry, lookup returns corruption FAIL |
| `TestSingleWriterGoroutine` | 100 concurrent goroutines store chunks; no vLog overlap |
| `TestCrashRecovery` | Write 100 chunks, kill without RocksDB flush, restart, all 100 retrievable |
| `TestGarbageCollection` | Delete 50 of 100 chunks, GC runs, deleted absent, rest intact |
| `TestRateLimiter` | Concurrent reads and compaction, p99 read latency within threshold |

### Benchmarks (must pass before milestone closes)

Q27-1 from `docs/research/benchmarking-protocol.md`:

| Benchmark | Target | Source |
|---|---|---|
| `BenchmarkAuditLookup/SSD` | p99 <= 100 ms under concurrent writes | NFR-008 |
| `BenchmarkAuditLookup/HDD` | p99 <= 200 ms under concurrent compaction | NFR-008 |
| `BenchmarkVLogAppend` | Report throughput in MB/s; must match declared upload speed | ADR-023 |

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| RocksDB rate limiter too aggressive on HDD | Medium | High (audit TIMEOUT) | Q27-1 benchmark protocol defines calibration procedure; run on actual HDD hardware |
| `fallocate` not available on macOS | Certain | Low | Implement platform-specific GC; macOS uses `fcntl(F_PUNCHHOLE)` or file rewrite |
| vLog corruption during power loss | Low | High | fsync() before index insert; crash recovery scan is the safety net |

### Capacity notes (from `capacity.md` Section 2.2)

| Provider storage | Chunks stored | RocksDB index size | vLog on disk | Bloom filter memory |
|---|---|---|---|---|
| 50 GB | ~200,700 | 8.8 MB | 52.7 GB | 2.5 MB |
| 500 GB | ~2,007,000 | 88.3 MB | 527 GB | 25.1 MB |

The RocksDB index at any realistic allocation stays fully cached in memory. Audit lookup is a memory operation + one vLog read.

### Exit criterion

```bash
go test ./internal/storage/... -v -race
# All tests pass, zero data races

# Demonstrate crash recovery:
scripts/test-crash-recovery.sh
# Stores 1000 chunks, kills process mid-write, restarts, verifies all 1000 retrievable
```

---

## Milestone 3 — P2P Networking

**Goal:** Provider instances can discover each other, establish authenticated libp2p connections with NAT traversal, maintain the Kademlia DHT with HMAC-pseudonymised chunk keys, and send 4-hourly signed heartbeats to the microservice.

**Dependencies:** M0 (repository layout).
**Parallelisable with:** M1, M2.

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-010 | Upload via direct libp2p/QUIC P2P | Transport stack established here |
| FR-023 | Ed25519 key pair at first launch | Peer identity generation |
| FR-027 | Signed heartbeat every 4 hours | Heartbeat protocol implemented |
| NFR-006 | Symmetric NAT providers auditable via relay | Three-tier NAT traversal |
| NFR-016 | Transport-layer authentication (TLS 1.3 / Noise XX) | All connections authenticated |
| NFR-017 | HMAC-pseudonymised DHT keys | Custom DHT key validator |

### Scope

**Package: `internal/p2p`**

*Peer identity (ADR-021):*
- Generate Ed25519 key pair at first daemon start; persist encrypted under keystore_enc_key
- Derive libp2p Peer ID as `multihash(public_key)` (standard libp2p convention)
- Register public key with the microservice at registration

*Transport stack (ADR-021):*

| Transport | Protocol | When used |
|---|---|---|
| Primary | QUIC v1 (RFC 9000) via go-libp2p | Default for all connections |
| Fallback | TCP + Noise XX handshake + yamux multiplexer | When UDP is blocked by middlebox |
| Session resumption | 0-RTT disabled for audit/receipt flows; permitted for pure data transfer | Security: prevents replay attacks on payment-relevant interactions |

*NAT traversal — three tiers (ADR-021):*

```
AutoNAT --> classify provider as: public / cone NAT / symmetric NAT
  --> cone NAT:      attempt DCUtR hole-punch (max_hole_punch_retries = 1)
  --> symmetric NAT: fall back to Circuit Relay v2
```

Relay nodes are hardcoded in the daemon configuration at launch (`--relay-addrs`). Three Vyomanaut-operated relay nodes.

*DHT configuration (ADR-001):*

| Parameter | Value | Rationale |
|---|---|---|
| k-bucket size | k=16 | S/Kademlia disjoint-path: k = 2 x d where d=8 |
| Parallel lookups | alpha=3 | O(log n / 3) round trips; sub-second median |
| DHT mode | Server (all V2 desktop providers) | Must be reachable for challenge dispatch |
| Key validator | Custom: accepts only `HMAC-SHA256(chunk_hash, file_owner_key)` | Rejects plain CIDs; DHT privacy |
| Republication | Disabled internally; driven by availability microservice | 12-hour interval, 24-hour expiry |

*Heartbeat protocol (ADR-028):*

Every 4 hours, the daemon POSTs a signed heartbeat to `/api/v1/provider/heartbeat`:
- `provider_id`, `current_multiaddrs` (JSON array), `timestamp`, `daemon_version`
- `provider_sig`: Ed25519 over all fields
- Microservice updates `providers.last_known_multiaddrs` and `providers.last_heartbeat_ts`
- Timer resets after each successful heartbeat; missed heartbeats do not accumulate

### Tests

| Test | Verifies |
|---|---|
| `TestPeerIdentityPersistence` | Key pair survives daemon restart; same Peer ID |
| `TestLibp2pConnection` | Two instances on same machine establish QUIC connection |
| `TestNoiseHandshakeFallback` | With QUIC blocked (iptables DROP udp), falls back to TCP+Noise |
| `TestDHTKeyValidator` | Plain CID rejected; HMAC-keyed lookup accepted |
| `TestHeartbeatSigned` | Heartbeat payload verifies against registered public key |
| `TestHeartbeatInterval` | Heartbeat fires within 4h +/- 5 minutes |
| `TestRelayFallback` | With AutoNAT classifying as symmetric NAT, relay path used |

### Integration test (requires simulation mode from M0)

```bash
# Launch 56 simulated providers
go run ./cmd/provider --sim-count=56 --sim-asn-count=5 \
  --microservice-url=http://localhost:8080 \
  --sim-data-dir=/tmp/vyomanaut-sim &

# Verify all 56 have sent at least one heartbeat
curl http://localhost:8080/api/v1/admin/providers | jq '. | length'
# Expected: 56
```

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Indian ISPs blocking UDP (QUIC) for > 30% of providers | Medium | High | TCP+Noise fallback has identical hole-punch success rate (Paper 30); track transport type via Q14-1 telemetry |
| libp2p API breaking change across versions | Medium | Medium | Pin go-libp2p version; custom DHT validator pinned explicitly |
| Custom DHT key validator silently dropped by upgrade | Low | Critical | Integration test `TestDHTKeyValidator` catches this; must be in CI |

### Capacity notes (from `capacity.md` Section 5)

- DHT lookup latency at 56 providers: ~2 hops, ~1.2s. At 10,000: ~5 hops, ~3.1s. Logarithmic scaling — not a bottleneck.
- Relay infrastructure at 3 nodes: 384 slots. Sufficient until ~570-850 providers depending on CGNAT fraction.
- Heartbeat ingress at 10,000 providers: 12 MB/day. Negligible.
- DHT record memory per provider: ~40 MB at 50 GB storage, ~400 MB at 500 GB. Must be documented in daemon minimum spec.

### Exit criterion

```bash
go test ./internal/p2p/... -v -race
# All tests pass

# Two provider instances connect and exchange a test message
scripts/test-p2p-connection.sh
```

## Milestone 4 — Audit System

**Goal:** The microservice issues audit challenges to providers, providers respond with signed receipts, and the receipts are durably recorded in the INSERT-only audit log. The cluster audit secret is correctly shared across microservice replicas. The 12-field receipt schema is complete and correct.

**Dependencies:** M0 (Postgres schema), M2 (storage engine — provider needs to read chunks), M3 (P2P — microservice needs to reach providers).

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-037 | One challenge per chunk per 24h, randomised timing | Challenge scheduler with jittered dispatch |
| FR-038 | HMAC-SHA256 nonce with server_ts | Challenge generation with version-prefixed nonce |
| FR-039 | INSERT-only 12-field receipt, dual-signed | Full receipt recording pipeline |
| FR-040 | Per-provider TCP-style RTO | `AVG + 4 x VAR`; new providers use pool median |
| FR-041 | Content hash fail = accelerated re-audit | Corruption code triggers re-audit of all provider chunks |
| NFR-018 | Cluster audit secret via HKDF from secrets manager | `server_secret_vN` derivation; fail-closed on load failure |
| NFR-020 | 1-byte version prefix on challenge nonce | Version byte is `N mod 256` |
| NFR-021 | INSERT-only audit_receipts enforced at DB level | Row security policy verified in test suite |

### Scope

**Package: `internal/audit`**

*Cluster audit secret (ADR-027):*

```
server_secret_vN = HKDF-SHA256(
    ikm  = cluster_master_seed,
    salt = cluster_id,
    info = \"vyomanaut-audit-secret-v\" + N,
    len  = 32
)
```

- `cluster_master_seed` from secrets manager (or `VYOMANAUT_CLUSTER_MASTER_SEED` env var in dev/sim — never in production)
- In-memory cache with 5-minute TTL; refreshed from secrets manager on expiry
- On startup failure to load secret: daemon does not start (fail-closed)

*Challenge generation (ADR-017):*

```
version_byte (1 byte) || HMAC-SHA256(server_secret_vN, chunk_id + server_ts)
```

Total: 33 bytes. Version byte is `N mod 256`. Stored in `challenge_nonce BYTEA(33)`.

*Challenge dispatch (ADR-006, ADR-028):*
- Source for provider address: `providers.last_known_multiaddrs` (heartbeat-maintained)
- Fallback (if `multiaddr_stale = true`): DHT FIND_NODE lookup
- Per-provider RTO: `AVG + 4 x VAR` of recent audit response latencies (stored as `(avg_rtt_ms FLOAT, var_rtt_ms FLOAT, sample_count INT)` per provider)
- New providers: initialise RTO from pool median; switch to per-provider formula after 5 samples
- Schedule: exactly one challenge per assigned chunk per 24-hour window, with randomised jitter within the window

*Receipt recording — crash-safe two-phase write (ADR-015):*

| Phase | Action | Durability |
|---|---|---|
| 1 | INSERT PENDING row (audit_result = NULL, provider_sig populated) | Durable before Phase 2 |
| 2 | Validate response_hash; UPDATE row: SET audit_result = PASS/FAIL, service_sig, service_countersign_ts | — |
| 3 | Return countersignature to provider | — |

If microservice crashes between Phase 1 and Phase 2: orphaned PENDING row is garbage-collected after 48 hours (set `abandoned_at`). Provider retries within 48h: microservice detects duplicate `challenge_nonce` (UNIQUE index) and returns the existing receipt.

*JIT flag (ADR-014 Defence 3):*
- If `response_latency_ms < (chunk_size / p95_throughput_kbps) x 0.3` then `jit_flag = true`
- 3+ jit_flags from same provider in 7-day window: apply 0.5x weight to audit passes for 30 days

*Audit receipt schema — complete 12-field form (ADR-017):*

| Field | Type | Notes |
|---|---|---|
| `receipt_id` | UUIDv7 PK | Time-ordered; no coordinator needed |
| `schema_version` | SMALLINT NOT NULL DEFAULT 1 | Forward compatibility |
| `chunk_id` | BYTEA(32) NOT NULL | Content address |
| `file_id` | UUID NOT NULL | Pseudonymous file handle |
| `provider_id` | UUID NOT NULL | Which provider responded |
| `challenge_nonce` | BYTEA(33) NOT NULL | 1 version byte + 32 HMAC bytes |
| `server_challenge_ts` | TIMESTAMPTZ NOT NULL | Set by server; prevents backdating |
| `response_hash` | BYTEA(32) NOT NULL | SHA-256(chunk_data \|\| nonce) |
| `response_latency_ms` | INT NOT NULL | JIT retrieval detector |
| `audit_result` | TEXT CHECK (IN 'PASS','FAIL','TIMEOUT') | NULL = in-flight PENDING |
| `provider_sig` | BYTEA(64) NOT NULL | Ed25519 over all fields |
| `service_sig` | BYTEA(64) NOT NULL | Ed25519 countersignature |

Additional columns: `service_countersign_ts TIMESTAMPTZ NOT NULL`, `jit_flag BOOLEAN NOT NULL DEFAULT false`, `abandoned_at TIMESTAMPTZ`.

Row security policy: INSERT only. No UPDATE. No DELETE. Enforced at DB level.

### Tests

| Test | Verifies |
|---|---|
| `TestChallengeNonceVersion` | Nonce prefix encodes correct version byte |
| `TestChallengeNonceDeterminism` | Same inputs produce same nonce |
| `TestCrossReplicaValidation` | Nonce generated by replica A is validated by replica B |
| `TestReceiptPhase1Durability` | Kill microservice after Phase 1; on restart, PENDING row present |
| `TestReceiptPhase2Idempotency` | Provider retries same challenge; second returns existing receipt |
| `TestReceiptInsertOnly` | UPDATE and DELETE return permission denied |
| `TestAuditPassRoundTrip` | Challenge > provider responds > receipt countersigned > PASS |
| `TestAuditFailOnMissingChunk` | Provider without the chunk returns FAIL |
| `TestAuditCorruptionFail` | content_hash mismatch on storage read results in FAIL with corruption code |
| `TestJITFlagThreshold` | Response faster than 0.3x deadline sets jit_flag=true |
| `TestAbandonedRowGC` | PENDING row older than 48h is marked ABANDONED |
| `TestRTOUpdated` | After 5 audit responses, RTO switches from pool-median to per-provider formula |

### Observability (wired in this milestone)

| Metric | Type |
|---|---|
| `audit_challenges_issued_total` | Counter |
| `audit_results_total{result=\"PASS|FAIL|TIMEOUT\"}` | Counter |
| `audit_receipt_latency_ms` | Histogram |
| `content_hash_failures_total` | Counter |

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Secrets manager unavailable at dev time | Medium | Blocks development | `VYOMANAUT_CLUSTER_MASTER_SEED` env var for dev/sim mode |
| Postgres INSERT ceiling lower than expected for this schema | Low at V2 scale | N/A at launch | Benchmark actual schema at M8; plan for partitioning |
| Clock skew between replicas invalidates nonce | Low | High | Use `server_ts` from issuing replica; version byte allows any replica to validate |

### Capacity notes (from `capacity.md` Section 4)

- At 56 providers x 200,000 chunks: 11.2M challenges/day = 130/sec. Well below Postgres ceiling.
- At 1,000 providers x 200,000 chunks: 200M/day = 2,315/sec. Still below planning ceiling of 5,000-10,000/sec.
- Audit receipt table: ~1.8 TB/year at 56 providers x 50 GB. **Monthly partitioning mandatory from day one.**

### Exit criterion

```bash
go test ./internal/audit/... -v -race

# Full audit cycle against 56 simulated providers
scripts/test-audit-cycle.sh
# Expected:
# 56 challenges issued
# 56 PASS receipts countersigned
# 56 rows in audit_receipts with audit_result='PASS'
# 0 PENDING rows older than 48h
```

---

## Milestone 5 — Reliability Scoring and Repair

**Goal:** The system computes per-provider reliability scores from audit history, detects provider departures, and automatically repairs affected chunks through the lazy repair protocol. The self-healing loop is complete.

**Dependencies:** M4 (audit system must generate receipts before scoring can compute anything).

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-035 | Silent departure at 72h = seizure + repair + block | Departure detection + repair trigger |
| FR-042 | Repair at s+r0=24 fragments | Lazy repair threshold implemented |
| FR-043 | Permanent departure jobs drain before pre-warning | Priority queue ordering |
| FR-044 | Emergency floor: immediate repair at s=16 | Emergency floor implemented |
| FR-045 | Replacement providers respect 20% ASN cap | Power of Two Choices + ASN cap |
| NFR-001 | Annual file loss rate < 10^-15 | Repair keeps redundancy above r0 |
| NFR-002 | 20% ASN cap is co-requisite for NFR-001 | Enforced at replacement selection |
| NFR-012 | BWavg <= 100 Kbps/provider | Lazy repair produces ~39 Kbps at target MTTF |

### Scope

**Package: `internal/scoring`**

Three-window rolling score (ADR-008):

| Window | Weight | Query |
|---|---|---|
| Last 24h | 0.5 | `SELECT COUNT(PASS), COUNT(FAIL), COUNT(TIMEOUT) FROM audit_receipts WHERE provider_id=? AND server_challenge_ts > NOW()-24h` |
| Last 7d | 0.3 | Same, 7-day window |
| Last 30d | 0.2 | Same, 30-day window |

- Score per window = `PASS / (PASS + FAIL + TIMEOUT)`
- Final score = weighted combination (0.5 / 0.3 / 0.2)
- Score decrement on FAIL/TIMEOUT is non-I-confluent (ADR-013): single authoritative scorer
- Materialised view refreshed asynchronously after each receipt batch

Failure clustering signal (ADR-008, Paper 32):
- If provider has > 1 audit FAIL in rolling 7-day window: set `accelerated_reaudit = true` on all their chunks

**Package: `internal/repair`**

*Departure detection (ADR-007, ADR-006):*

```
Query: providers WHERE last_heartbeat_ts < NOW() - 72h AND status = 'ACTIVE'
On match:
  1. SET providers.status = 'DEPARTED'
  2. DELETE chunk_assignments for this provider_id
  3. Seize escrow (no-op until M6 is complete)
  4. Enqueue repair jobs for all affected chunks
```

*Repair scheduler — lazy trigger (ADR-004):*

| Fragment count | Action | Priority |
|---|---|---|
| Drops to s+r0=24 | Enqueue pre-warning repair job | Lower |
| Drops to s=16 | Enqueue **immediately** (emergency floor) | Infinite |

*Priority ordering (ADR-004, Paper 39):*
- Two queues: `PERMANENT_DEPARTURE` and `PRE_WARNING`
- Drain all `PERMANENT_DEPARTURE` before any `PRE_WARNING`
- Within each queue: FIFO

*Repair execution:*
1. Contact k=16 surviving fragment holders via libp2p (M3)
2. Download 16 fragments, RS decode (M1), re-encode missing fragments
3. Select replacement providers using Power of Two Choices + 20% ASN cap (ADR-005, ADR-014)
4. Upload to replacements, collect signed upload receipts
5. Update `chunk_assignments` in Postgres

### Tests

| Test | Verifies |
|---|---|
| `TestThreeWindowScore` | Known audit history produces correct per-window scores |
| `TestScoreWeighting` | Weighted combination is correctly computed |
| `TestAcceleratedReaudit` | 2 FAILs in 7 days sets accelerated_reaudit on all provider's chunks |
| `TestDepartureDetection` | Provider with last_heartbeat_ts 73h ago is detected and DEPARTED |
| `TestRepairPriorityOrdering` | PERMANENT_DEPARTURE jobs drain before PRE_WARNING |
| `TestRepairASNCap` | Replacement provider selection respects 20% ASN cap |
| `TestEmergencyFloor` | Fragment count at s=16 triggers immediate repair |
| `TestLazyRepairThreshold` | Fragment count at s+r0=24 triggers pre-warning, not immediate |
| `TestRepairBandwidthEstimate` | At N=1,000, 50GB/provider: BWavg ~ 39 Kbps |

### Integration test

```bash
scripts/test-self-healing.sh
# 1. Start 56 simulated providers
# 2. Store a test file (1 segment = 56 shards)
# 3. Kill 8 simulated providers (count drops to 48 > 24 — no repair)
# 4. Verify: no repair triggered
# 5. Kill 24 more providers (count drops to 24 = s+r0)
# 6. Verify: repair job enqueued within 60 seconds
# 7. Verify: after repair, 56 providers hold the chunk again
```

### Observability (wired in this milestone)

| Metric | Type |
|---|---|
| `provider_score_histogram` | Histogram |
| `repair_queue_depth` | Gauge |
| `repair_jobs_completed_total` | Counter |

Alert: `repair_queue_depth > 1000` (NFR-027).

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| False-positive departure at 72h (weekend absence) | Medium | High (unnecessary repair + seizure) | Q06-3: track providers declared departed who reconnect within 72h; if > 5%, adjust threshold |
| Repair window > 12h at launch (N < 500) | Certain at launch | High | Manual oversight of `repair_queue_depth`; no concurrent departures without verification |
| ASN cap cannot be satisfied during replacement selection | Medium at small N | High (repair blocked) | Assignment service returns best-effort with warning if exact cap is temporarily unsatisfiable |

### Capacity notes (from `capacity.md` Section 3)

- At N=56, Qpeek = 44 GB, repair window = ~22 hours. **Exceeds 12h safety target.** Manual monitoring required.
- At N=500, repair window = ~18 hours. Still borderline.
- At N=1,000, repair window = ~8 hours. Within budget.
- Per-provider storage ceiling at MTTF=180d: ~70 GB. At MTTF=300d: ~130 GB. **Must be enforced in onboarding.**

### Exit criterion

```bash
go test ./internal/scoring/... ./internal/repair/... -v -race

scripts/test-self-healing.sh
# Output: repair completes, fragment count restored to 56
```

---

## Milestone 6 — Payment System

**Goal:** Providers accumulate earnings per audit pass, receive monthly partial releases based on reliability score, and have escrow seized on silent departure. Razorpay integration is real (test mode at this stage, live mode at M8).

**Dependencies:** M5 (scoring must compute release multipliers; repair must trigger seizure).

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-046 | All amounts in integer paise | No float anywhere in payment package |
| FR-047 | X-Payout-Idempotency header on every payout | Idempotency key = `SHA-256(provider_id + audit_period)` |
| FR-048 | Monthly release on 23rd; on_hold_until = last working day | RBI holiday table embedded in Go |
| FR-049 | Release multiplier per 30d score | Four-band table (1.00 / 0.75 / 0.50 / 0.00) |
| FR-050 | Dual-window deterioration flag | 7d vs 30d score drop > 0.20 triggers lower multiplier |
| FR-051 | Vetting period: 60d hold, 50% cap | Status-dependent release logic |
| FR-052 | Escrow non-transferable | Refused unconditionally |
| NFR-029 | UPI Intent only (no UPI Collect) | Collect endpoint never called |
| NFR-030 | X-Payout-Idempotency header mandatory | Enforced on every Razorpay API call |

### Scope

**Package: `internal/payment`**

*Escrow ledger — PN-counter CRDT (ADR-016):*
- `escrow_events` table: INSERT-only, ENUM(DEPOSIT, RELEASE, SEIZURE), `amount_paise BIGINT`
- `idempotency_key VARCHAR(64) UNIQUE`
- Balance = `SUM(DEPOSIT) - SUM(RELEASE + SEIZURE)` per provider_id
- Materialised view for balance, refreshed asynchronously
- **No floating-point anywhere in this package (NFR-046)**

*Per-audit-pass earnings (ADR-012, ADR-024):*

```
payout_per_audit_pass = (storage_rate_paise x chunk_size_GB) / audits_per_month
```

Earn on PASS only; FAIL and TIMEOUT earn nothing; INSERT a DEPOSIT event per PASS.

*Monthly release computation — runs on the 23rd of each month (ADR-024):*

```
For each active provider:
    score_30d = query scoring package
    score_7d  = query scoring package
    dual_window_flag = (score_30d - score_7d) > 0.20
    effective_multiplier = min(
        release_multiplier(score_30d),
        dual_window_flag ? release_multiplier(score_7d) : 1.0
    )
    releasable_amount_paise = escrow_window_balance x effective_multiplier
    on_hold_until = last_working_day_of_month()    // RBI holiday table
    call Razorpay PATCH /transfers/:id with on_hold_until
    INSERT RELEASE event to escrow_events
```

*Release multiplier table (ADR-024):*

| score_30d | Multiplier | Provider outcome |
|---|---|---|
| >= 0.95 | 1.00 | Full release |
| 0.80-0.94 | 0.75 | Partial release; withheld rolls forward |
| 0.65-0.79 | 0.50 | Partial release; withheld rolls forward |
| < 0.65 | 0.00 | Hold in full; withheld rolls forward (not seized) |

*RBI holiday table:*
- Embedded as a Go constant: `internal/payment/rbi_holidays.go`
- Updated each December deployment
- `last_working_day_of_month(year, month) time.Time` — not Saturday, Sunday, or RBI holiday

*Escrow seizure on silent departure (ADR-024, M5 hooks):*
1. Freeze provider escrow (set `frozen` flag; refuse new DEPOSIT events)
2. INSERT SEIZURE event for full 30-day rolling window balance
3. Call Razorpay Route reversal API if any transfer has not yet settled

*Razorpay integration (ADR-011, Paper 35):*

| Product | Purpose |
|---|---|
| Smart Collect 2.0 | Virtual UPI ID per data owner contract |
| Route with `on_hold/on_hold_until` | Monthly release hold |
| RazorpayX Payouts | Monthly transfer to provider's bank account |

**X-Payout-Idempotency header mandatory** on every payout call.
`Payout.reversed` webhook: append REVERSED event to escrow_events.
**Razorpay Escrow+ is never called** — requires NBFC registration.

*PaymentProvider interface (ADR-011):*

```go
type PaymentProvider interface {
    InitiateEscrow(ownerID, amount, contractID string) error
    ReleaseEscrow(providerID string, amount int64, auditPeriod string) error
    Penalise(providerID string, amount int64, reason string) error
    GetBalance(entityID string) (int64, error)
}
```

### Tests

| Test | Verifies |
|---|---|
| `TestEscrowLedgerPNCounter` | Deposits and releases compute correct balance |
| `TestIdempotentRelease` | Duplicate idempotency_key returns existing event |
| `TestNoFloatArithmetic` | Static analysis: no float64/float32 in payment package |
| `TestReleaseMultiplier` | All four score bands return correct multiplier |
| `TestDualWindowFlag` | 0.21 drop sets flag; 0.19 does not |
| `TestVettingPeriodCap` | VETTING provider release capped at 50% |
| `TestSeizureOnDeparture` | DEPARTED provider's 30-day balance moved to SEIZURE |
| `TestLastWorkingDay` | Correct day accounting for weekends and RBI holidays |
| `TestRazorpayIdempotency` | Payout API called twice; second call is no-op |
| `TestRazorpayReversedWebhook` | Webhook triggers REVERSED event insertion |

### Observability (wired in this milestone)

| Metric | Type |
|---|---|
| `escrow_events_total{type=\"DEPOSIT|RELEASE|SEIZURE\"}` | Counter |
| `payment_payout_total_paise` | Counter |
| `payment_seizure_total_paise` | Counter |

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Razorpay Route API changes `on_hold_until` semantics | Low | High | Abstract behind `PaymentProvider` interface; monitor API changelog |
| Storage rate (OQ-001) not decided before M6 | High | Blocks FR-013, FR-014 | Use placeholder rate; make configurable at runtime |
| Float arithmetic accidentally introduced | Medium | Critical (correctness) | Static analysis test `TestNoFloatArithmetic` in CI |

### Capacity notes (from `capacity.md` Section 6)

- Monthly payout at N=1,000: 75,000 INR gross, 4,000 INR Razorpay fees (5.3% at floor rate).
- Razorpay rate limit: 500 req/min = 8.3/sec. Sufficient for N up to ~120,000 providers.
- `escrow_events` table: ~10,000 events/month at N=10,000. Negligible storage concern.

### Exit criterion

```bash
go test ./internal/payment/... -v -race

scripts/test-payment-cycle.sh
# Provider accumulates 30 days of simulated audit passes
# Monthly release computation runs
# Razorpay test API receives PATCH with on_hold_until
# Provider account shows releasable balance
# Simulate departure: seizure event recorded, Razorpay reversal called
```

---

## Milestone 7 — Data Owner Client

**Goal:** A data owner can register an account, upload a file that is encrypted and distributed across 56 providers, retrieve it on any device using their passphrase, and view their file list and escrow balance. All four critical paths are exercised end-to-end with real users for the first time.

**Dependencies:** M1 (encoding pipeline), M3 (P2P transfer), M4 (audit — microservice endpoints must exist), M6 (payment — escrow deposit and balance queries).
**Parallelisable with:** Nothing at this stage. M7 requires all upstream milestones to be correct before integration is possible.

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-001 | Phone number registration, OTP-verified | Registration endpoint; Argon2id derivation on first login |
| FR-002 | Argon2id master secret, never written to disk | `internal/crypto` (M1); invoked here |
| FR-003 | BIP-39 mnemonic displayed once, two words confirmed | Mnemonic generation + confirmation gate in registration flow |
| FR-004 | Restore on new device from passphrase or mnemonic | Recovery flow implemented |
| FR-005 | Data loss warning disclosed at onboarding | Warning displayed before mnemonic confirmation |
| FR-006 | UPI Intent deposit flow (not UPI Collect) | Escrow deposit triggers Smart Collect 2.0 virtual UPI ID |
| FR-007 | All encryption on data owner's device | Encoding pipeline (M1) called locally; no data sent to microservice |
| FR-008 | AONT-RS encoding with padding | Full pipeline invoked from upload orchestrator |
| FR-009 | 56 providers, 20% ASN cap, HTTP 503 if unsatisfiable | Assignment request to microservice enforces cap |
| FR-010 | Upload via direct libp2p/QUIC P2P | libp2p (M3) called per fragment |
| FR-011 | Pointer file encrypted, ciphertext stored by microservice | HKDF key + AEAD + POST to microservice |
| FR-012 | Upload progress shown per fragment | Progress callback at chunk upload completion |
| FR-013 | Monthly cost displayed before upload | Pricing query from microservice; computed from storage rate |
| FR-014 | Refuse upload if escrow balance insufficient | Balance check before assignment request |
| FR-015 | Retrieve via decrypted pointer file, P2P download | Pointer file fetch → decrypt → P2P download from 16 providers |
| FR-016 | Retrieve from 16 fastest-responding providers | Parallel dial; cancel on first 16 |
| FR-017 | SHA-256 verification per fragment on retrieval | chunk_id = SHA-256(shard_data) verified before decode |
| FR-018 | Canary verification; do not return corrupted plaintext | Canary check after AONT decode |
| FR-019 | File list: name, size, upload date, cost, availability | Microservice query; availability derived from chunk_assignment count |
| FR-020 | File delete removes all 56 chunk assignments | DELETE request to microservice; provider notify to GC vLog |
| FR-021 | Escrow balance view: balance, reserved, available, history | Materialised view query |
| NFR-014 | Service never holds decryption key | AONT key K never leaves client; microservice holds only ciphertext |
| NFR-019 | Constant-time Poly1305 tag comparison on pointer file decrypt | `crypto/subtle.ConstantTimeCompare` in decode path |

### Scope

**Package: `internal/client/account`**

*Registration (FR-001 through FR-005):*

```
1. POST /api/v1/owner/register { phone_number }
2. OTP challenge/response (Razorpay SMS or equivalent)
3. owner_id = UUIDv7 returned from microservice
4. master_secret = Argon2id(passphrase, owner_id, t=3, m=65536, p=4)  -- local, never transmitted
5. Generate BIP-39 mnemonic from master_secret
6. Display mnemonic: single screen, no surrounding UI noise
7. Require confirmation: two randomly selected word positions must be entered correctly
8. Block progression until confirmed
9. Warning: "These words are the ONLY way to recover your files if you forget your passphrase."
10. Derive Ed25519 signing key; store encrypted under HKDF(master_secret, "vyomanaut-keystore-v1")
11. Register public key with microservice
```

*Recovery flow (FR-004):*
- Passphrase path: prompt passphrase + known owner_id → re-derive master_secret → re-derive all keys
- Mnemonic path: parse 24 words → reconstruct master_secret → identical to passphrase path from that point
- Download encrypted pointer files from microservice; decrypt locally

*Escrow deposit (FR-006):*
- POST `/api/v1/owner/deposit` → microservice returns Smart Collect 2.0 virtual UPI VPA
- Display VPA or QR code; poll `/api/v1/owner/balance` until `payout.created` webhook received

**Package: `internal/client/upload`**

*Upload orchestrator (FR-007 through FR-014):*

```
1. Verify escrow balance >= cost_for_30_days(file_size)    -- FR-014
2. POST /api/v1/upload/assign { file_id, num_segments }
   --> receive [ [56 provider assignments per segment] ]    -- FR-009
3. For each segment:
   a. Pad if < 4 MB
   b. Encode: AONT transform + RS(16,56)                   -- FR-008, M1
   c. For each of 56 shards in parallel:
      - Dial provider via libp2p (M3)
      - Stream 256 KB shard
      - Receive signed upload receipt
      - Report progress callback                            -- FR-012
4. Create pointer file (provider_ids, chunk_ids, erasure params)
5. Derive pointer_enc_key via HKDF
6. Encrypt pointer file: AEAD_CHACHA20_POLY1305
7. POST /api/v1/file/register { file_id, pointer_ciphertext, pointer_nonce, pointer_tag }
```

Price display (FR-013):
```
GET /api/v1/pricing/estimate?size_bytes={N}
--> { monthly_cost_paise: int64, storage_rate_paise_per_gb_per_month: int64 }
Display before any upload begins.
```

**Package: `internal/client/retrieve`**

*Retrieval orchestrator (FR-015 through FR-018):*

```
1. GET /api/v1/file/{file_id}/pointer
   --> { pointer_ciphertext, pointer_nonce, pointer_tag }
2. Derive pointer_dec_key = HKDF(master_secret, "vyomanaut-pointer-v1" || file_id)
3. Decrypt: verify tag (constant-time), recover pointer file plaintext          -- NFR-019
4. For each segment:
   a. Dial all 56 providers in parallel
   b. Collect first 16 valid responses; cancel remaining connections             -- FR-016
   c. Verify each shard: SHA-256(shard_data) == chunk_id                        -- FR-017
   d. RS decode (any 16 shards) --> AONT package
   e. Recover K: h = SHA-256(all codewords), K = c_{s+1} XOR h
   f. Decrypt each word
   g. Verify canary -- fail hard if wrong                                        -- FR-018
   h. Strip padding; accumulate segment plaintext
5. Concatenate segments; write to destination file
```

**Package: `internal/client/manage`**

File list (FR-019):
```
GET /api/v1/owner/{owner_id}/files
--> [{ file_id, name, size_bytes, upload_ts, monthly_cost_paise,
       shard_count_available, shard_count_total }]
Display availability status: shard_count_available >= 16 = OK; < 16 = degraded; < s+r0=24 triggers repair banner.
```

File delete (FR-020):
```
DELETE /api/v1/file/{file_id}
--> microservice marks all 56 chunk_assignments as DELETED
--> notifies each provider daemon to remove from vLog GC list
```

Escrow balance (FR-021):
```
GET /api/v1/owner/{owner_id}/escrow
--> { balance_paise, reserved_next_30d_paise, available_paise,
      events: [{event_type, amount_paise, timestamp}] }
```

**Command: `cmd/client`**

Minimal CLI for private beta. No tray app or graphical dashboard in the MVP.

```
vyomanaut register              -- account creation + mnemonic display
vyomanaut upload <path>         -- encode and distribute
vyomanaut retrieve <file_id>    -- reconstruct and write to disk
vyomanaut ls                    -- list files with availability
vyomanaut rm <file_id>          -- delete file
vyomanaut balance               -- show escrow balance
vyomanaut deposit               -- display UPI VPA / QR for deposit
vyomanaut recover               -- restore on new device from passphrase or mnemonic
```

### Tests

| Test | Verifies |
|---|---|
| `TestRegistrationMnemonicConfirmation` | Registration blocks until two correct words are entered |
| `TestMnemonicRecovery` | Mnemonic reconstructs identical master_secret as original passphrase |
| `TestUploadPriceCheck` | Upload refused if escrow balance < 30-day cost |
| `TestUploadFullPipeline` | 56 fragments reach 56 simulated providers; pointer file stored |
| `TestUploadProgressCallback` | Callback fires once per completed shard |
| `TestRetrieveAny16Shards` | Retrieve with 40 simulated providers unavailable; succeeds with any 16 |
| `TestRetrieveCanaryFail` | Flip a bit in any codeword; retrieve returns error, no corrupted plaintext |
| `TestPointerFileConstantTimeCompare` | Tag comparison uses `crypto/subtle`; no timing oracle |
| `TestRetrieveOnNewDevice` | Re-derive all keys from passphrase on fresh key store; pointer file decrypts |
| `TestDeleteRemovesAssignments` | File delete results in all 56 chunk_assignments marked DELETED |
| `TestFileListAvailability` | File list shows correct shard_count_available per file |
| `TestEscrowBalanceView` | Balance matches SUM of escrow_events for this owner |

### Integration test (full end-to-end)

```bash
scripts/test-end-to-end.sh
# 1. Register a data owner account
# 2. Deposit 1,000 paise (test mode)
# 3. Upload a 10 MB test file (3 segments)
# 4. Verify: 56 shards per segment stored across 56 simulated providers
# 5. Verify: pointer file ciphertext stored in Postgres (cannot be decrypted without passphrase)
# 6. Retrieve file: reconstruct plaintext, SHA-256 matches original
# 7. Verify: canary passes on all segments
# 8. Delete file: all chunk_assignments removed
# 9. Verify: retrieve after delete returns 404
```

### Observability (wired in this milestone)

| Metric | Type |
|---|---|
| `file_upload_started_total` | Counter |
| `file_upload_completed_total` | Counter |
| `file_upload_failed_total{reason}` | Counter |
| `file_retrieve_total` | Counter |
| `upload_shard_latency_ms` | Histogram |
| `retrieve_shard_latency_ms` | Histogram |
| `escrow_deposit_initiated_total` | Counter |

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Mnemonic UX causes data owners to skip backup | High | Critical | Gate: cannot proceed without confirming two correct words; no clipboard copy |
| Retrieval fails silently due to expired pointer file nonce counter | Low | High | Nonce counter persisted in keystore with durability guarantee; crash recovery scans counter state |
| libp2p parallel dial produces non-deterministic fastest-16 selection | Medium | Low (correctness) | Cancel on 16th receipt; order does not affect correctness, only latency |
| Escrow deposit webhook delayed by Razorpay | Medium | Medium (UX friction) | Poll balance endpoint as fallback; show pending deposit state to data owner |
| OQ-001 (storage rate) not set before M7 | High | Blocks FR-013, FR-014 | Inject rate as runtime config; display placeholder during private beta |

### Capacity notes (from `capacity.md` Sections 2 and 3)

- At 56-provider floor, only one file segment can be uploading at a time (all 56 slots occupied). Data owners experience sequential segment uploads. This is acceptable at private beta scale.
- Retrieval at 56-provider floor: all 56 providers contacted in parallel; 40 connections cancelled after the first 16 respond. Connection overhead is negligible.
- Maximum encoded wire data per 10 MB file: ~17.5 MB (8 segments × 14 MB × 28.57% efficiency correction). Transfer time at 5 Mbps: ~28 seconds.

### Exit criterion

```bash
go test ./internal/client/... -v -race
# All tests pass, zero data races

scripts/test-end-to-end.sh
# Upload, retrieve, verify SHA-256 match, delete
# Output: "End-to-end test PASSED. SHA-256 match confirmed."
```

---

## Milestone 8 — Network Readiness Gate and Private Beta

**Goal:** All seven readiness conditions are simultaneously true. The network is open to 20 invited data owners and 100 provider volunteers under manual oversight. The system is running on real infrastructure with real money in test mode.

**Dependencies:** M0 through M7 (all milestones complete, all exit criteria passing in CI).
**Parallelisable with:** Nothing. This milestone is the deployment gate.

### Requirement traceability

| FR/NFR | Description | How this milestone addresses it |
|---|---|---|
| FR-022 | Daemon installable on Windows 10+, macOS 12+, Ubuntu 22.04+ | Signed installers built and tested |
| FR-024 | Provider registration: phone number, Ed25519 key, storage allocation, city | Registration flow live |
| FR-025 | Razorpay Linked Account created; 24h cooling before chunk assignment | Onboarding flow enforces cooling |
| FR-026 | Provider lifecycle: PENDING → VETTING → ACTIVE | Status transitions verified in production |
| FR-053 | Network readiness gate returns HTTP 503 until all 7 conditions met | Gate wired (M0); all conditions now satisfiable |
| FR-054 | `GET /api/v1/admin/readiness` returns per-condition status | Monitoring endpoint live |
| NFR-004 | File retrievable with any 16 of 56 fragment holders | Verified in production upload + retrieval |
| NFR-005 | Microservice survives single-replica failure | Manual failover test during M8 |
| NFR-006 | Relay overhead < 50 ms for Indian cloud-hosted nodes | Relay latency measured at launch |
| NFR-025 | Prometheus metrics on microservice | Scraped and dashboarded |
| NFR-026 | Provider daemon local metrics exposed | Verified on test provider machines |
| NFR-027 | Operational alerts defined and firing | Alert rules configured |
| NFR-029 | UPI Intent only; no UPI Collect | Confirmed in Razorpay test-to-live migration |
| NFR-030 | X-Payout-Idempotency header on all payouts | Confirmed in Razorpay live mode |
| NFR-031 | RBI holiday table current for this year | Verified for current calendar year |

### Scope

**Infrastructure provisioning:**

| Component | Specification | Location |
|---|---|---|
| Microservice replicas | 3 × Go binary; 2 vCPU, 4 GB RAM each | Mumbai AZ-1, AZ-2; Chennai |
| Postgres primary | Managed (Cloud SQL / RDS), Multi-AZ | Mumbai |
| Postgres read replicas | 2 × replica | Mumbai AZ-1, AZ-2 |
| Relay nodes | 3 × Go + libp2p Circuit Relay v2; 1 vCPU, 1 GB RAM each | Mumbai AZ-1, AZ-2; Delhi |
| Secrets manager | Configured with cluster master seed under `/vyomanaut/audit-secret/v1` | Same cloud region |
| Load balancer | In front of heartbeat endpoint only; audit challenges use client-driven routing | Cloud LB |
| Monitoring | Prometheus + Grafana; alerting via PagerDuty or equivalent | Ops |

**Readiness gate verification (FR-053, ADR-029):**

The gate checks seven conditions every 60 seconds. M8 does not close until all seven are simultaneously true in production:

| Condition | Minimum | How to reach it |
|---|---|---|
| Active vetted providers | ≥ 56 | Provider recruitment campaign; at least 80 consecutive audit passes each |
| Distinct ASNs | ≥ 5 | Recruit from at least 5 distinct ISPs (BSNL, Airtel, Jio, ACT, Hathway) |
| Distinct metro regions | ≥ 3 | Recruit from Delhi NCR, Mumbai, and one southern metro |
| Microservice quorum | Full (3,2,2) | All 3 replicas running and gossiping |
| Razorpay Linked Accounts (cooling complete) | ≥ 56 | Created during provider onboarding; 24h cooldown elapsed |
| Relay nodes deployed | ≥ 3 | Deployed in provisioning above |
| Cluster audit secret loaded | All replicas | Confirmed via startup health check log |

```bash
# Monitor gate status
watch -n 10 'curl -s http://microservice.internal/api/v1/admin/readiness | jq .'
```

**Razorpay test → live mode migration:**

| Step | Action | Verification |
|---|---|---|
| 1 | Switch `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET` from test to live | Verified in secrets manager |
| 2 | Confirm UPI Intent flow (not Collect) is the only active deposit path | Manual QA with test UPI ID |
| 3 | Confirm `X-Payout-Idempotency` header present in live payout call | Razorpay dashboard shows no rejected payouts |
| 4 | Run a ₹1 end-to-end live deposit + release (test provider, test data owner) | Bank statement confirms transfer |
| 5 | Confirm `payout.reversed` webhook handler is reachable from Razorpay | Razorpay webhook delivery log shows 200 response |

**Provider onboarding (private beta — 100 providers):**

1. Invite providers via registration link with OTP gate.
2. Provider installs daemon via signed installer; registration endpoint live.
3. Razorpay Linked Account created automatically; 24h cooling tracked in `providers` table.
4. Provider enters VETTING status; assigned non-critical chunks under extra redundancy.
5. Dashboard link shared: provider sees their score, chunk count, and pending earnings.
6. Monitor for 80 consecutive audit passes to advance to ACTIVE.

**Data owner onboarding (private beta — 20 data owners):**

1. Manual invitation only; no public registration.
2. Walk each data owner through registration, mnemonic backup, and first deposit via screen share.
3. Verify each data owner confirms mnemonic (two-word gate must be passed).
4. Assist with first upload (minimum 1 MB file) and retrieval verification.

**Manual failover test (NFR-005):**

```bash
# During M8, while the network is live:
# 1. Take replica-2 offline
kubectl scale deployment/microservice --replicas=2

# 2. Confirm reads and writes continue (R=2, W=2 satisfied with 2 healthy replicas)
curl http://microservice.internal/api/v1/admin/readiness | jq '.healthy_replicas'
# Expected: 2

# 3. Issue an audit challenge manually
scripts/manual-audit-challenge.sh --provider-id=<id>
# Expected: challenge issued, receipt recorded despite 2-replica cluster

# 4. Restore replica-2
kubectl scale deployment/microservice --replicas=3

# 5. Confirm replica-2 reconciles via gossip within 30 seconds
curl http://microservice.internal/api/v1/admin/readiness | jq '.healthy_replicas'
# Expected: 3
```

**Relay latency measurement (NFR-006):**

```bash
# From each relay-dependent provider (symmetric NAT):
scripts/measure-relay-latency.sh --relay-addr=<relay-multiaddr>
# Expected: p50 < 50 ms, p99 < 150 ms for Indian cloud-hosted relays
```

**Operational runbooks (must exist before M8 closes):**

| Runbook | Covers |
|---|---|
| `runbooks/microservice-failover.md` | Replica failure and recovery |
| `runbooks/postgres-failover.md` | Primary failure; promote read replica |
| `runbooks/relay-node-replacement.md` | Relay node failure; swap without provider disruption |
| `runbooks/secrets-manager-outage.md` | Unavailability; replicas continue with 5-minute cached secret |
| `runbooks/razorpay-api-outage.md` | Payment pause; audit and storage continue; catch-up on recovery |
| `runbooks/provider-mass-departure.md` | Multiple departures triggering simultaneous repair; manual repair queue oversight |
| `runbooks/rbi-holiday-table-update.md` | Annual December update procedure |
| `runbooks/audit-secret-rotation.md` | Scheduled and emergency rotation; 24-hour version overlap protocol |

### Tests

| Test | Verifies |
|---|---|
| `TestReadinessGateAllConditions` | Gate returns PASS only when all 7 conditions are true |
| `TestReadinessGatePartialFail` | Gate returns 503 if any single condition fails |
| `TestMicroserviceQuorumDegradedMode` | 2-replica cluster continues serving reads and writes |
| `TestAuditSecretCrossReplica` | Nonce from replica-1 validates on replica-2 |
| `TestRelayLatencyThreshold` | Circuit Relay v2 adds < 50 ms RTT from Delhi relay to Mumbai provider |
| `TestRazorpayLiveMode` | ₹1 live deposit + release completes end-to-end |
| `TestProviderOnboardingLifecycle` | Provider: PENDING → VETTING → ACTIVE after 80 passes |
| `TestProviderHeartbeatResumed` | Provider reconnecting after DEPARTED is refused HTTP 403 |

### Observability (all metrics active by M8 close)

By the end of M8, the full observability stack specified in NFR-025 through NFR-028 must be live:

| Dashboard Panel | Metric | Alert |
|---|---|---|
| Network readiness | Per-condition status | Alert if any condition falls below threshold |
| Audit throughput | `audit_challenges_issued_total`, `audit_results_total` | Alert if TIMEOUT rate > 5% in 1h |
| Provider health | `provider_score_histogram` | Alert if > 10% of providers have score < 0.65 |
| Repair health | `repair_queue_depth` | Alert if depth > 1,000 |
| Disk corruption | `content_hash_failures_total` | Alert if > 0 in 7d for any provider |
| Payment health | `escrow_events_total`, `payment_seizure_total_paise` | Alert on unexpected seizure spikes |
| Relay health | Relay slot utilisation | Alert if > 80% utilisation on any relay node |
| Infrastructure | Replica count, Postgres replica lag | Alert if healthy_replicas < 3 |

### Risk assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Cannot recruit 56 vetted providers before launch | High | Launch blocker | Begin provider recruitment 6 weeks before M8; track funnel weekly; target 100 recruits for 56 vetted |
| Indian CGNAT fraction > 45%, relay slots exhausted | Medium | High | Pre-provision 4 relay nodes rather than 3; cheaper to over-provision than to scramble |
| RocksDB CGo binary fails on provider Windows machine | Medium | High | Pre-built static binaries in signed installer; test on Windows 10 and 11 before M8 |
| Secrets manager unavailable during microservice restart | Low | Critical (fail-closed = no audit challenges) | Verify secrets manager HA independently; test restart-under-outage in staging before M8 |
| Razorpay live mode approval delayed | Medium | High | Apply for live mode access in Sprint 4 (not Sprint 6); use test mode for M8 if delayed, switch before first payout |
| OQ-001 (storage rate) not decided by M8 | High | Blocks FR-013, FR-014 | Enforce pricing decision as M8 gate; use ₹4/GB/month as placeholder with explicit flag |

### Capacity notes (from `capacity.md` Sections 3 and 5)

- At 56-provider launch, repair window is ~22 hours — exceeds the 12-hour safety budget. **Manual oversight of `repair_queue_depth` is mandatory during private beta.** Do not allow simultaneous multi-provider departures without verification.
- At 100 providers (full private beta), relay slots needed: 30–45 providers × 1.5 slots = 45–68 slots. 384 available — more than 5× headroom.
- First relay upgrade trigger: ~570 providers at 30% CGNAT, ~400 providers at 45% CGNAT. Plan upgrade before reaching 300 providers.
- Audit receipt table at 56 providers × 50 GB: ~1.8 TB/year. Monthly Postgres partitioning must be operational from day one (wired in M0 schema).

### Exit criterion

```bash
# All prior milestones passing CI
go test ./... -v -race

# Network readiness gate green in production
curl https://api.vyomanaut.com/api/v1/admin/readiness | jq '.all_conditions_met'
# Expected: true

# First end-to-end live upload and retrieval
scripts/test-live-e2e.sh --data-owner-id=<invited-id> --file=test-10mb.bin
# Expected: upload completes, retrieval SHA-256 matches

# Relay latency in production
scripts/measure-relay-latency.sh --relay-addr=<production-relay>
# Expected: p50 < 50 ms

# Manual failover test passes
scripts/test-microservice-failover.sh
# Expected: 2-replica cluster continues; replica-3 recovers within 30 seconds
```

**Private beta is open when this criterion is met.** Do not open to data owners before the readiness gate is green in production.

---

## Cumulative Delivery Summary

What each milestone delivers, and what capability is unlocked at each stage.

| Milestone | Sprint | Unlocks |
|---|---|---|
| **M0 — Foundation** | 1 | Monorepo, CI, Postgres schema, 56-node simulation on a laptop. No business logic; all gates FAIL. |
| **M1 — Encoding Pipeline** | 1 | Correct AONT-RS encode/decode. Plaintext becomes 56 encrypted 256 KB shards and back. Zero-knowledge property in code. |
| **M2 — Storage Engine** | 1 | Provider daemon stores and retrieves 256 KB chunks from local disk. Write amplification ~1.0. Corruption detected on every read. |
| **M3 — P2P Networking** | 2 | Provider instances connect via libp2p/QUIC, traverse NAT (three tiers), maintain DHT with privacy-preserving keys, send heartbeats. |
| **M4 — Audit System** | 2 | Microservice issues daily challenges; providers respond with signed receipts; receipts durably recorded in INSERT-only Postgres table. Cluster audit secret shared across replicas. |
| **M5 — Scoring and Repair** | 3 | Three-window reliability scores computed per provider. Departure detected at 72h. Lazy repair triggers at r0=8; emergency floor at s=16. Self-healing loop complete. |
| **M6 — Payment System** | 4 | Providers accumulate paise per audit pass. Monthly release with dual-window multiplier. Escrow seized on departure. Razorpay Route integration in test mode. |
| **M7 — Data Owner Client** | 5 | Data owner can register (BIP-39 backup), upload (encode + distribute), retrieve (reconstruct + verify), and manage files via CLI. All four critical paths exercised end-to-end. |
| **M8 — Beta Launch** | 6 | Production infrastructure live. 56+ vetted providers across 5+ ASNs. Readiness gate green. 20 invited data owners. Manual oversight of repair queue. Real money flowing (Razorpay live mode). |

---

## Staffing and Parallel Work

The build assumes a team of three engineers. This is a minimum-viable team for the described scope. Adding engineers beyond three is possible but requires careful ownership boundaries.

### Engineer assignment

| Engineer | Primary ownership | Sprint 1 | Sprint 2 | Sprint 3 | Sprint 4 | Sprint 5 | Sprint 6 |
|---|---|---|---|---|---|---|---|
| **Eng-A** | Crypto, Encoding, Client | M0 (shared) + M1 | M3 (partial) | M5 (partial) | M6 (partial) | M7 (lead) | M8 (partial) |
| **Eng-B** | Storage, Audit, Repair | M0 (shared) + M2 | M4 (lead) | M5 (lead) | M6 (partial) | M7 (partial) | M8 (lead infra) |
| **Eng-C** | P2P, Payment, Infra | M0 (shared) + M3 | M4 (partial) | M5 (partial) | M6 (lead) | M7 (partial) | M8 (lead ops) |

### Parallelisation rules (enforced by dependency graph)

| Sprint | What runs in parallel | Blocker |
|---|---|---|
| **1** | M0, M1, M2 are fully independent code paths. All three engineers run in parallel. | M0 must complete first (repo layout); M1 and M2 can start immediately. |
| **2** | M3 begins independently (no code dependency on M1 or M2). M4 cannot begin integration testing until M2 and M3 pass exit criteria. | M4 unit tests can be written without M2 complete; integration tests cannot run. |
| **3** | M5 is full-team. No parallelisation. M4 exit criteria must be met before M5 starts. | Scoring cannot compute without audit receipts. Repair cannot execute without scoring. |
| **4** | M6 is single-engineer lead. M5 exit criteria must be met (scores computed, seizure hooks wired). | Release multiplier depends on scoring package. |
| **5** | M7 is two engineers. M1 + M3 + M4 + M6 must all be passing. | Upload orchestrator needs encoding (M1), P2P (M3), microservice endpoints (M4), balance check (M6). |
| **6** | M8 is full-team deployment. All milestones passing CI. | Infrastructure requires all code to be correct first. |

### Code ownership boundaries

| Package | Owner | No-merge-without-review-from |
|---|---|---|
| `internal/crypto`, `internal/erasure` | Eng-A | Eng-B or Eng-C (any) |
| `internal/storage` | Eng-B | Eng-A or Eng-C (any) |
| `internal/p2p` | Eng-C | Eng-A or Eng-B (any) |
| `internal/audit` | Eng-B (lead), Eng-C | Both Eng-A and Eng-C for audit secret changes |
| `internal/scoring`, `internal/repair` | Eng-B (lead) | Eng-A or Eng-C (any) |
| `internal/payment` | Eng-C (lead) | Eng-B |
| `internal/client` | Eng-A (lead) | Eng-B or Eng-C (any) |
| `migrations/` | Any | Two of three engineers |

---

## Observability Rollout Plan

Metrics are wired in the milestone where the underlying system is built. This table shows the cumulative state of the observability stack at the end of each milestone. Nothing is added retrospectively.

| Milestone | New metrics added |
|---|---|
| **M0** | Infra: `microservice_replica_count{state="healthy"}`, `db_read_p99_latency_ms`. CI: no metrics. |
| **M1** | `encoding_duration_ms` histogram (AONT + RS encode; ChaCha20 vs AES-NI). |
| **M2** | `storage_chunks_stored_total`, `storage_audit_lookup_latency_ms` histogram, `content_hash_failures_total`, `vlog_append_latency_ms` histogram. |
| **M3** | `p2p_connections_active{transport="quic|tcp"}`, `p2p_nat_class{class="public|cone|symmetric"}`, `p2p_heartbeat_sent_total`, `p2p_heartbeat_failures_total`. |
| **M4** | `audit_challenges_issued_total`, `audit_results_total{result="PASS|FAIL|TIMEOUT"}`, `audit_receipt_latency_ms` histogram, `content_hash_failures_total` (provider-side), `jit_flags_total`. |
| **M5** | `provider_score_histogram`, `repair_queue_depth` gauge, `repair_jobs_completed_total`, `provider_departed_total`. |
| **M6** | `escrow_events_total{type="DEPOSIT|RELEASE|SEIZURE"}`, `payment_payout_total_paise`, `payment_seizure_total_paise`, `payment_release_multiplier_histogram`. |
| **M7** | `file_upload_started_total`, `file_upload_completed_total`, `file_upload_failed_total{reason}`, `file_retrieve_total`, `upload_shard_latency_ms` histogram, `retrieve_shard_latency_ms` histogram, `escrow_deposit_initiated_total`. |
| **M8** | Dashboards live. Alert rules active. `relay_slot_utilisation` gauge, `readiness_gate_condition{condition}` gauge. All metrics scraped by Prometheus. |

### Alert thresholds (operational from M8)

| Alert | Condition | Severity | Action |
|---|---|---|---|
| Audit TIMEOUT spike | `rate(audit_results_total{result="TIMEOUT"}[1h]) / rate(audit_challenges_issued_total[1h]) > 0.05` | P1 | Check relay nodes; check heartbeat address freshness |
| Disk corruption detected | `increase(content_hash_failures_total[7d]) > 0` | P1 | Accelerated re-audit already triggered; investigate provider hardware |
| Repair queue backlog | `repair_queue_depth > 1000` | P1 | Check departure rate; check bandwidth budget; manual oversight |
| Microservice replica loss | `microservice_replica_count{state="healthy"} < 3` | P1 | Follow `runbooks/microservice-failover.md` |
| Relay slot pressure | `relay_slot_utilisation > 0.80` | P2 | Provision additional relay node |
| Provider score degradation | `histogram_quantile(0.10, provider_score_histogram) < 0.65` | P2 | Investigate cohort departure or hardware failure pattern |
| DB read latency | `db_read_p99_latency_ms > 40` | P2 | Background task throttle triggers at 50 ms; investigate query plan |

---

## Security Verification Checklist

This checklist must be run against the codebase before M8 closes. Each item has a verifiable outcome.

### Cryptographic correctness

- [ ] **AONT key K is never logged, returned in error responses, or written to any persistent store.** Verify by `grep -r "AONT_key\|aont_key\|\.K\b" internal/` — zero results outside `internal/crypto` test files.
- [ ] **Poly1305 tag comparison uses `crypto/subtle.ConstantTimeCompare`, not `bytes.Equal`.** Verify: `grep -r "bytes.Equal" internal/crypto/` — zero results in decode paths.
- [ ] **Argon2id parameters are exactly (t=3, m=65536, p=4) in production.** Verify: `grep -r "Argon2id\|argon2id" internal/crypto/` — parameters hard-coded, not configurable except via benchmarking fallback protocol.
- [ ] **ChaCha20 nonce is 0x000...0 for the AONT pass.** This is safe because K is fresh per segment. Verify: nonce zero is explicitly set, not left as default.
- [ ] **Pointer file nonce counter is monotone and persisted before use.** Verify: counter is incremented and written to keystore before the AEAD encryption call.
- [ ] **BIP-39 mnemonic is never stored in the application or transmitted to the microservice.** Verify: `grep -r "mnemonic\|BIP39\|bip39" internal/` — only appears in `internal/client/account` display/confirmation paths.

### Audit log integrity

- [ ] **`audit_receipts` row security policy blocks UPDATE and DELETE.** Verify: `TestReceiptInsertOnly` passes; attempt UPDATE from application-level role returns `permission denied`.
- [ ] **`escrow_events` row security policy blocks UPDATE and DELETE.** Same pattern.
- [ ] **All `audit_receipts` rows have non-null `provider_sig` and `service_sig`.** Verify: Postgres `CHECK` constraint or NOT NULL constraint enforced in schema.
- [ ] **`challenge_nonce` is `BYTEA(33)`, not `BYTEA(32)`.** Verify: `\d audit_receipts` in Postgres shows correct type.
- [ ] **The PENDING row phase (audit_result = NULL) is durable before Phase 2 validation.** Verify: `TestReceiptPhase1Durability` passes.

### Payment correctness

- [ ] **No `float64` or `float32` in `internal/payment`.** Verify: `grep -rn "float64\|float32" internal/payment/` — zero results. `TestNoFloatArithmetic` in CI.
- [ ] **Every Razorpay payout call includes `X-Payout-Idempotency` header.** Verify: mock Razorpay client in tests asserts header presence; live mode integration test confirms no rejected payouts.
- [ ] **Escrow transfer to a different `provider_id` is refused unconditionally.** Verify: `TestEscrowNonTransfer` — any transfer request returns 403.
- [ ] **`VYOMANAUT_CLUSTER_MASTER_SEED` env var is absent in production config.** Verify: production secrets are injected from secrets manager only; env var path is blocked by CI environment check.

### Network and transport

- [ ] **0-RTT is disabled for all connections carrying audit receipts or signed receipts.** Verify: `TestZeroRTTDisabledForAudit` — libp2p config explicitly sets `DisableEarlyData: true` for audit protocol streams.
- [ ] **DHT key validator rejects plain CIDs.** Verify: `TestDHTKeyValidator` in CI.
- [ ] **Custom DHT key validator is pinned in libp2p configuration.** Verify: configuration is not overridden by go-libp2p defaults on version upgrade; integration test in CI.
- [ ] **Secrets manager is unreachable → microservice does not start (fail-closed).** Verify: `TestSecretManagerUnavailableAtStartup` — startup returns non-zero exit code.

### Data plane isolation

- [ ] **Microservice API endpoints never return file plaintext or chunk data.** Verify: code review of all microservice handler functions; none read from vLog or invoke RS decode.
- [ ] **Pointer file ciphertext stored in microservice cannot be decrypted without master_secret.** Verify: `TestPointerFileMicrosserviceCantDecrypt` — microservice stores ciphertext; decryption requires key derived from master_secret, which is never transmitted.

---

## Invariants That Must Never Break

These are properties that, if violated, make the system unsafe to operate regardless of any other quality consideration. A PR that breaks any of these must not merge under any circumstances.

**1. The encoding invariant — data owner's device is the only site of plaintext.**
The AONT key K is generated, used, and discarded on the data owner's device. It is never transmitted to the microservice or providers. Providers store only encrypted fragments. The microservice stores only encrypted pointer file ciphertext. Any code that transmits K, plaintext segments, or pointer file decryption keys to the microservice or providers breaks this invariant.

**2. The audit log invariant — receipts are permanent.**
`audit_receipts` is INSERT-only. No UPDATE or DELETE. Ever. This is enforced at the Postgres row security policy level, not just in application code. A migration that removes this policy, or a code path that calls UPDATE or DELETE on this table, breaks this invariant. The PENDING → PASS/FAIL transition is an UPDATE in the crash-safe two-phase write — this is the single permissible exception, bounded to `audit_result`, `service_sig`, and `service_countersign_ts` only.

**3. The payment invariant — all amounts are integer paise.**
No floating-point arithmetic may appear anywhere in the payment path from audit pass to bank transfer. This includes intermediate calculations, paise-to-rupee display conversions, and Razorpay API payloads. Display conversions are the caller's responsibility and happen in the UI layer, not in `internal/payment`. Static analysis test `TestNoFloatArithmetic` enforces this in CI.

**4. The ASN cap invariant — no correlated group holds > 20% of any file's fragments.**
This is a co-requisite for both the durability guarantee (LossRate < 10⁻¹⁵, ADR-003) and the adversarial defence (Honest Geppetto, ADR-014). The assignment service must enforce this cap at write time for original uploads and repair replacements. The cap is never deactivated after the readiness gate opens. Any assignment path that bypasses the cap — including repair under time pressure — breaks this invariant.

**5. The escrow integrity invariant — balance is always computable from events.**
The escrow balance is never stored as a mutable column. It is always: `SUM(DEPOSIT) - SUM(RELEASE + SEIZURE)` for a given provider_id in `escrow_events`. Any code that stores a cached balance in a mutable column, or that attempts to compute balance by any other method, breaks this invariant. The materialised view is a read optimisation, not the source of truth.

**6. The cluster secret invariant — audit challenges are verifiable by any replica.**
All microservice replicas must share `server_secret_vN` derived from the same `cluster_master_seed`. A challenge nonce generated by replica-A must be validatable by replica-B. Any change to the HKDF derivation inputs (`ikm`, `salt`, `info`, `len`) or the version byte scheme without a corresponding rotation protocol breaks this invariant and causes replicas to disagree on audit validity.

**7. The idempotency invariant — duplicate payout calls are safe.**
Every Razorpay payout call includes `X-Payout-Idempotency: SHA-256(provider_id + audit_period)`. A retry of a failed payout call with the same key must return the existing payout state, not create a second transfer. Any payout call path that omits this header, or that constructs the key inconsistently, breaks this invariant.

---

## Risk Register

Global risks that span multiple milestones or the entire project. Per-milestone risks are documented in each milestone's risk section.

| # | Risk | Probability | Impact | Milestone | Mitigation | Owner |
|---|---|---|---|---|---|---|
| R-01 | Provider recruitment below 56 before M8 | High | Launch blocker | M8 | Begin recruitment in Sprint 4 (parallel with M6). Target 120 recruits for 56 vetted. | Eng-C |
| R-02 | Indian ISP CGNAT fraction > 45%, relay slots exhausted earlier than modelled | Medium | High (audit TIMEOUT) | M3, M8 | Pre-provision 4 relay nodes rather than 3. Track NAT class via telemetry from M3. | Eng-C |
| R-03 | OQ-001 (storage rate) not decided before M7 | High | Blocks FR-013, FR-014 | M7 | Use ₹4/GB/month as runtime-configurable placeholder. Pricing decision must be locked before M8 closes. | Product |
| R-04 | Razorpay live mode approval delayed past Sprint 5 | Medium | M8 delayed | M8 | Apply in Sprint 4. Use test mode for M8 if delayed; block real data owner deposits until live. | Eng-C |
| R-05 | RocksDB CGo binary fails on Windows provider machines | Medium | Provider daemon unusable on Windows | M2, M8 | Pre-built static binary via CGo; tested on Windows 10/11 in M2. Signed installer bundles all dependencies. | Eng-B |
| R-06 | Minimum-spec hardware benchmark fails (AONT, Argon2id) | Medium | Cipher path or Argon2id parameters must change | M1 | Fallback protocol in `benchmarking-protocol.md`; run on actual minimum-spec hardware in M1, not CI. | Eng-A |
| R-07 | Repair window > 12h at private beta scale (N < 500) | Certain at launch | High (manual oversight required) | M5, M8 | Documented in `capacity.md`; manual `repair_queue_depth` monitoring mandatory during private beta. | Eng-B |
| R-08 | `klauspost/reedsolomon` API breaking change | Low | M1 regression | M1 | Pin dependency version; vendor if necessary. | Eng-A |
| R-09 | libp2p API breaking change drops custom DHT key validator | Low | Critical (DHT privacy broken silently) | M3 | `TestDHTKeyValidator` in CI; pin go-libp2p version. | Eng-C |
| R-10 | Secrets manager unavailable during production incident | Low | Critical (fail-closed; no audit challenges) | M8 | Verify secrets manager SLA. Pre-test restart-under-outage in staging before M8. | Eng-B |
| R-11 | Data owner loses passphrase AND mnemonic | Low per user; certain at scale | High (data loss; no support path) | M7 | Onboarding warning is mandatory (FR-005). Cannot be engineered away. Disclosed at registration. | Product |
| R-12 | Float arithmetic introduced in payment package in a later PR | Medium | Critical (correctness) | M6+ | `TestNoFloatArithmetic` static analysis check in CI. Block merge if it fails. | Eng-C |

---

## Open Questions Resolution Schedule

Open questions whose answers affect implementation. Each entry states what is blocked and when it must be resolved.

| Question | Blocks | Must resolve by | Method |
|---|---|---|---|
| **OQ-001** — What is the storage rate (paise/GB/month) that makes provider participation viable while keeping data owner costs below cloud alternatives? | FR-013, FR-014 (upload price display and escrow check) | Sprint 5 start (before M7 begins) | Product decision; use ₹4/GB/month as placeholder in earlier milestones |
| **OQ-002** — What fraction of Indian home routers are behind symmetric NAT? | Relay infrastructure sizing (confirm 3 vs 4 nodes at launch) | M8 start (Sprint 6) | Q20-1: measure at private beta via `p2p_nat_class{class="symmetric"}` telemetry from M3 |
| **OQ-003** — Does the dual-window partial hold trigger (0.20 drop) correctly catch degrading providers without penalising weekend absences? | ADR-024 threshold tuning | 30 days after private beta opens | Q31-1: measure provider score trajectories before silent departures |
| **OQ-004** — What RocksDB rate limiter value satisfies p99 ≤ 200 ms under concurrent compaction on a 7200 RPM HDD? | Default rate_limiter constant in `internal/storage` | M2 close | Q27-1: HDD benchmark protocol in `docs/research/benchmarking-protocol.md` |
| **OQ-005** — At what provider count N does observed BWavg exceed 60 Kbps/peer, triggering Hitchhiker code evaluation? | ADR-026 V3 adoption gate | 6 months post-launch | Q39-1: telemetry from production; compare to Giroire prediction |

---

## The First Five Minutes

What to observe, and what to do, in the first five minutes after the readiness gate turns green and the first data owner completes an upload.

**Observe these metrics immediately:**

```bash
# Replica health
curl https://api.vyomanaut.com/api/v1/admin/readiness | jq .

# First audit challenge issued
curl https://api.vyomanaut.com/api/v1/admin/audit/stats | jq '.challenges_issued_last_1h'

# Repair queue (must be zero at launch)
curl https://api.vyomanaut.com/api/v1/admin/repair/queue | jq '.depth'

# Relay utilisation
curl https://relay-1.vyomanaut.com/metrics | grep relay_slot_utilisation

# First upload in Postgres
psql -c "SELECT COUNT(*) FROM chunk_assignments WHERE created_at > NOW() - INTERVAL '10 minutes';"
```

**These numbers confirm the system is live:**

| Signal | Expected value | If wrong |
|---|---|---|
| Readiness gate | `all_conditions_met: true` | Stop; do not proceed until green |
| Healthy replicas | 3 | Follow microservice-failover runbook |
| Repair queue depth | 0 | Investigate immediately; no departures should have occurred |
| Relay utilisation | < 10% at launch | Healthy headroom |
| Challenges issued in first hour | equals number of chunks stored × expected audit frequency | Scheduler running |

**The first upload test:**

After the readiness gate turns green, the engineering team should perform one internal upload before inviting any data owners:

```bash
# Engineer-1 uploads a 10 MB test file from a development machine
vyomanaut upload test-10mb.bin

# Verify: 56 shards placed across 56 providers in the Postgres chunk_assignments table
psql -c "SELECT COUNT(DISTINCT provider_id) FROM chunk_assignments WHERE file_id='<file_id>';"
# Expected: 56

# Engineer-2 retrieves the file from a different machine (no shared keystore)
# This requires recovery from mnemonic to prove the recovery path works in production
vyomanaut recover  # enter mnemonic
vyomanaut retrieve <file_id>

# Verify SHA-256 match
sha256sum test-10mb.bin retrieved-file.bin
# Expected: identical hashes
```

**Only proceed to data owner invitations after this test passes.**

---

## Appendix A — Requirement Traceability Matrix

Every P0 functional and non-functional requirement from `requirements.md`, mapped to the milestone where it is delivered.

| ID | Requirement summary | Milestone |
|---|---|---|
| FR-001 | Phone number registration, OTP | M7 |
| FR-002 | Argon2id master secret | M1 (library), M7 (invocation) |
| FR-003 | BIP-39 mnemonic, two-word confirmation | M7 |
| FR-004 | Restore on new device | M7 |
| FR-005 | Data loss warning at onboarding | M7 |
| FR-006 | UPI Intent deposit flow | M7 |
| FR-007 | All encryption on data owner's device | M1 (encoding library) |
| FR-008 | AONT-RS with padding | M1 |
| FR-009 | 56 providers, 20% ASN cap, HTTP 503 | M5 (cap) + M8 (gate) |
| FR-010 | Upload via libp2p/QUIC P2P | M3 |
| FR-011 | Pointer file encrypted, ciphertext at microservice | M1 (encryption) + M7 (orchestration) |
| FR-012 | Upload progress per fragment | M7 |
| FR-013 | Monthly cost before upload | M7 |
| FR-014 | Refuse upload if balance insufficient | M7 |
| FR-015 | Retrieve via decrypted pointer file | M7 |
| FR-016 | Retrieve from 16 fastest providers | M7 |
| FR-017 | SHA-256 verification per fragment | M7 |
| FR-018 | Canary verification; no corrupted plaintext | M1 (canary logic) + M7 (client check) |
| FR-019 | File list | M7 |
| FR-020 | File delete | M7 |
| FR-021 | Escrow balance view | M7 |
| FR-022 | Signed installer on Windows/macOS/Linux | M8 |
| FR-023 | Ed25519 key pair at first launch | M3 |
| FR-024 | Provider registration | M8 (live), M3 (skeleton) |
| FR-025 | Razorpay Linked Account, 24h cooling | M6 (integration), M8 (live) |
| FR-026 | Provider lifecycle: PENDING → VETTING → ACTIVE | M5 + M8 |
| FR-027 | 4-hour signed heartbeat | M3 |
| FR-028 | Audit challenge response within deadline | M4 |
| FR-029 | Local daemon status (CLI) | M7 (minimal CLI) |
| FR-030 | Storage cap per provider | M2 (storage limit) + M8 (enforcement) |
| FR-031 | Auto-start on OS boot | M8 |
| FR-032 | Promised offline period | M5 (departure states) |
| FR-033 | Overrun of promised period reclassifies | M5 |
| FR-034 | Announced departure: repair + proportional release | M5 (repair) + M6 (release) |
| FR-035 | Silent departure at 72h: seizure + repair + block | M5 (departure + repair) + M6 (seizure) |
| FR-036 | HTTP 403 on reconnect after departure | M5 |
| FR-037 | One challenge per chunk per 24h, randomised | M4 |
| FR-038 | HMAC-SHA256 nonce with server_ts | M4 |
| FR-039 | INSERT-only 12-field receipt, dual-signed | M4 |
| FR-040 | Per-provider TCP-style RTO | M4 |
| FR-041 | Corruption detected = FAIL with code | M2 (detection) + M4 (reporting) |
| FR-042 | Repair at s+r0=24 | M5 |
| FR-043 | Permanent departure jobs drain before pre-warning | M5 |
| FR-044 | Emergency floor at s=16 | M5 |
| FR-045 | Replacement providers respect 20% ASN cap | M5 |
| FR-046 | All amounts in integer paise | M6 |
| FR-047 | X-Payout-Idempotency header on every payout | M6 |
| FR-048 | Monthly release on 23rd; on_hold_until last working day | M6 |
| FR-049 | Release multiplier per 30d score | M6 |
| FR-050 | Dual-window deterioration flag | M6 |
| FR-051 | Vetting period: 60d hold, 50% cap | M6 |
| FR-052 | Escrow non-transferable | M6 |
| FR-053 | Network readiness gate HTTP 503 | M0 (wired) + M8 (conditions satisfied) |
| FR-054 | `GET /api/v1/admin/readiness` | M0 |
| FR-055 | `--sim-count=N` simulation mode | M0 |
| FR-056 | Simulation does not bypass gate | M0 |
| NFR-001 | LossRate < 10⁻¹⁵/year | M1 (RS) + M5 (repair) + M8 (validated) |
| NFR-002 | 20% ASN cap co-requisite | M5 (enforcement) + M8 (gate) |
| NFR-003 | 40-fault tolerance | M1 (RS parameters) |
| NFR-004 | File retrievable with any 16 of 56 | M1 + M7 |
| NFR-005 | Microservice survives single-replica failure | M0 (quorum schema) + M8 (failover test) |
| NFR-006 | Relay overhead < 50 ms | M3 (relay) + M8 (measured) |
| NFR-007 | Audit deadline per provider | M4 |
| NFR-008 | Audit lookup p99 SSD/HDD | M2 |
| NFR-009 | AONT encode p50 ≤ 200 ms | M1 |
| NFR-010 | Argon2id p50 ≤ 500 ms | M1 |
| NFR-011 | ≤ 5% CPU background | M2 (rate limiter) + M8 (measured) |
| NFR-012 | BWavg ≤ 100 Kbps/provider | M5 (lazy repair) |
| NFR-013 | Write amplification ≤ 1.1× | M2 |
| NFR-014 | Service never holds decryption key | M1 + M7 |
| NFR-015 | PoR response unforgeable without chunk | M4 |
| NFR-016 | Transport-layer authentication | M3 |
| NFR-017 | HMAC-pseudonymised DHT keys | M3 |
| NFR-018 | Cluster audit secret via secrets manager | M4 |
| NFR-019 | Constant-time tag comparison | M1 + M7 |
| NFR-020 | 1-byte version prefix on nonce | M4 |
| NFR-021 | INSERT-only audit_receipts at DB level | M0 |
| NFR-022 | INSERT-only escrow_events at DB level | M0 |
| NFR-023 | Single writer goroutine for vLog | M2 |
| NFR-024 | Crash recovery: vLog tail scan | M2 |
| NFR-025 | Prometheus metrics on microservice | M4 (partial) + M8 (complete) |
| NFR-026 | Provider daemon local metrics | M2 (partial) + M8 (complete) |
| NFR-027 | Operational alerts | M8 |
| NFR-028 | Background task throttling at 50 ms | M0 (schema) + M4 (implementation) |
| NFR-029 | UPI Intent only | M7 (client) + M8 (live confirmed) |
| NFR-030 | X-Payout-Idempotency mandatory | M6 |
| NFR-031 | RBI holiday table current | M6 (embedded) + M8 (verified) |

---

## Appendix B — ADR Reference Index

Every ADR cited in this roadmap, with the milestone where the decision is implemented.

| ADR | Title | Milestone |
|---|---|---|
| ADR-001 | Microservices + Kademlia DHT | M3 (DHT), M0 (schema) |
| ADR-002 | PoR Merkle Challenge | M4 |
| ADR-003 | Reed-Solomon RS(16,56) | M1 |
| ADR-004 | Lazy Repair, 72h threshold | M5 |
| ADR-005 | Storj 4-subsystem peer selection | M5 (repair) + M8 (gate) |
| ADR-006 | Polling interval 24h, departure 72h | M4 (RTO) + M5 (departure) |
| ADR-007 | Four provider exit states | M5 |
| ADR-008 | Three-window reliability score | M5 |
| ADR-009 | Background CPU ≤ 5% | M2 (rate limiter) |
| ADR-010 | No mobile providers in V2 | Design constraint; no implementation |
| ADR-011 | Fiat escrow via Razorpay/UPI | M6 |
| ADR-012 | Payment per audit passed | M6 |
| ADR-013 | Consistency model (14 free, 6 coordinated) | M0 (schema), M5 (scorer), M6 (payment) |
| ADR-014 | Four adversarial defence classes | M4 (JIT) + M5 (ASN cap) |
| ADR-015 | Signed receipt exchange | M4 |
| ADR-016 | PN-counter CRDT escrow ledger | M0 (schema) + M6 (logic) |
| ADR-017 | 12-field audit receipt schema | M0 (schema) + M4 (logic) |
| ADR-018 | Hot/cold storage bands | Design decision; Cold band only in M2 |
| ADR-019 | Client-side cipher: ChaCha20/AES | M1 |
| ADR-020 | Key management: HKDF hierarchy | M1 |
| ADR-021 | libp2p + QUIC v1 | M3 |
| ADR-022 | AONT-RS encoding pipeline | M1 |
| ADR-023 | WiscKey storage engine | M2 |
| ADR-024 | Economic mechanism | M6 |
| ADR-025 | (3,2,2) microservice quorum + gossip | M0 (schema) + M8 (live) |
| ADR-026 | V3 repair BW (Hitchhiker) | Not in MVP scope |
| ADR-027 | Cluster audit secret HKDF + rotation | M4 |
| ADR-028 | Provider heartbeat every 4h | M3 |
| ADR-029 | Bootstrap minimum viable network | M0 (gate wired) + M8 (conditions met) |

---

## Appendix C — Research Paper Index

Every research paper that directly informs an implementation decision in this roadmap.

| Paper | Title | Informs |
|---|---|---|
| Paper 06 | Blake & Rodrigues — Pick Two | BWavg formula; 100 Kbps background budget; lazy repair bandwidth savings |
| Paper 09 | Bolosky et al. — Feasibility | Bimodal absence distribution; 72h departure threshold; 5% CPU budget |
| Paper 10 | Giroire et al. — Be Lazy | All four erasure formulas (BWavg, Qpeek, LossRate, optimal r); r=40 derivation |
| Paper 11 | Bailis et al. — Coordination Avoidance | I-confluence map; 6 coordinated operations; CRDT escrow ledger |
| Paper 12 | DeCandia et al. — Dynamo | (3,2,2) quorum parameters; gossip at 1 peer/sec; client-driven routing |
| Paper 16 | Resch & Plank — AONT-RS | AONT transform mechanics; canary; production validation |
| Paper 17 | RFC 8439 — ChaCha20-Poly1305 | Cipher specification; nonce management; AEAD construction |
| Paper 18 | Wilcox-O'Hearn — Tahoe-LAFS | Capability model; HKDF key hierarchy; pointer file as read-cap |
| Paper 20 | Trautwein et al. — IPFS Measurement | DHT republication 12h/24h gap; AS concentration validates ASN cap |
| Paper 22 | Goparaju et al. — MSR Codes | Clay codes infeasible at n=56; only RS is viable |
| Paper 27 | Lu et al. — WiscKey | Key-value separation; write amplification ~1.0 at 256 KB; audit lookup |
| Paper 28 | Rhea et al. — Handling Churn | Per-provider TCP-style RTO = AVG + 4×VAR |
| Paper 29 | Protocol Labs — Filecoin | Graduated penalty template; Manage.RepairOrders → ADR-007 exit states |
| Paper 30 | Trautwein et al. — DCUtR NAT | 70% hole-punch success; relay-independence; 97.6% first-attempt; max_retries=1 |
| Paper 32 | Schroeder et al. — Flash Reliability | Prior errors 30× predictive; reactive scrubbing only; sequential vLog acceptable |
| Paper 35 | Razorpay API Docs | Route vs Escrow+; UPI Intent; idempotency key mandatory; on_hold_until timing |
| Paper 36 | Dalle et al. — Failure Correlation | Real repair σ is 22× higher than independent model; ASN cap is structural mitigation |
| Paper 37 | Crystal et al. — SHELBY | Proposition 1: peer audits collapse without trusted backstop; Theorem 1 Nash conditions |
| Paper 38 | Nath et al. — Correlated Failures | Large-m RS schemes require ASN cap as co-requisite for durability guarantee |
| Paper 39 | Silberstein et al. — Lazy Means Smart | Permanent-departure jobs before pre-warning; lazy repair validated |

---

## Appendix D — Capacity Quick-Reference

Key numbers from `capacity.md` that affect implementation decisions. All derived under the V2 parameter set (RS(16,56), lf=256 KB, MTTF=300d planning target).

### Storage geometry

| Value | Number |
|---|---|
| Storage efficiency (usable / raw) | 28.57% (16/56) |
| Chunk size on disk | 262,212 bytes per vLog entry |
| RocksDB index entry | 44 bytes per chunk |
| Bloom filter memory at 200k chunks (50 GB) | 2.5 MB |
| Bloom filter memory at 2M chunks (500 GB) | 25.1 MB |

### Bandwidth

| Scenario | BWavg |
|---|---|
| MTTF = 180d (desktop minimum) | ~65 Kbps/peer |
| MTTF = 300d (planning target) | ~39 Kbps/peer |
| MTTF = 90d (mobile; excluded) | ~130 Kbps/peer — exceeds budget |
| Per-provider storage ceiling at MTTF=180d | ~70 GB |
| Per-provider storage ceiling at MTTF=300d | ~130 GB |
| Qpeek at N=1,000, 50 GB/provider | ~793 GB per failure event |
| Repair window at N=1,000 | ~8 hours (within 12h budget) |
| Repair window at N=56 (launch) | ~22 hours — **exceeds budget; manual oversight required** |

### Audit throughput

| Scale | Challenges/day | Challenges/sec | Postgres ceiling? |
|---|---|---|---|
| 56 providers × 50 GB | ~11.2M | ~130/sec | Far below (5,000–10,000/sec) |
| 1,000 × 50 GB | ~200M | ~2,315/sec | Below ceiling |
| 5,000 × 50 GB | ~1B | ~11,574/sec | **At ceiling** |
| 100,000 × 10 GB | ~1B | ~11,574/sec | **At ceiling** |

### Relay capacity

| Relay nodes | Total slots | Max providers (30% CGNAT) | Max providers (45% CGNAT) |
|---|---|---|---|
| 3 (launch) | 384 | ~850 | ~570 |
| 4 | 512 | ~1,130 | ~760 |
| 6 | 768 | ~1,700 | ~1,140 |

### Audit receipt table growth

| Scale | Rows/day | Storage/year (at 450 B/row) |
|---|---|---|
| 56 × 50 GB | 11.2M | 1.8 TB |
| 1,000 × 50 GB | 200M | 32.9 TB |
| 10,000 × 10 GB | 100M | 16.4 TB |

**Monthly partitioning is mandatory from M0. Archive partitions older than 30 days to cold object storage.**

### First relay upgrade trigger

Add a fourth relay node when `relay_slot_utilisation > 0.80` on any relay node, or when active provider count approaches:
- **300 providers** under pessimistic 45% CGNAT assumption
- **570 providers** under optimistic 30% CGNAT assumption

Measure actual CGNAT fraction via `p2p_nat_class{class="symmetric"}` telemetry from M3 and update the trigger accordingly.