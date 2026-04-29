# Vyomanaut V2 — Repository Structure

**Status:** Authoritative — update this document whenever a directory is added, a
package is renamed, or an ownership boundary changes. Where this document and `mvp.md`
conflict, this document wins.
**Version:** 1.0
**Date:** April 2026
**Author:** Vyomanaut Engineering
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research
**Supersedes:** —
**Companion documents:**
- [`mvp.md`](./mvp.md) — build plan; M0 scope defines the initial directory tree
- [`architecture.md`](./architecture.md) — component descriptions
- [`interface-contracts.md`](./interface-contracts.md) — exported package interfaces
- [`data-model.md`](./data-model.md) — database schema and invariants
- [`requirements.md`](./requirements.md) — functional and non-functional requirements
- [`ADR index`](../decisions/README.md) — all accepted architectural decisions

---

## Table of Contents

1. [Monorepo Decision](#1-monorepo-decision)
   - [1.1 Why One Repository](#11-why-one-repository)
   - [1.2 Go Module Layout](#12-go-module-layout)
   - [1.3 Vendoring Policy](#13-vendoring-policy)
2. [Top-Level Directory Map](#2-top-level-directory-map)
3. [Detailed Package Descriptions](#3-detailed-package-descriptions)
   - [3.1 cmd/](#31-cmd)
   - [3.2 internal/](#32-internal)
4. [Configuration and Infrastructure](#4-configuration-and-infrastructure)
5. [Documentation Layout](#5-documentation-layout)
6. [Naming Conventions](#6-naming-conventions)
7. [Ownership Map](#7-ownership-map)
8. [Code That Must Never Live Here](#8-code-that-must-never-live-here)

---

## 1. Monorepo Decision

### 1.1 Why One Repository

Vyomanaut V2 ships three binaries that share cryptographic, erasure-coding, and payment
logic: the coordination microservice, the provider daemon, and the data owner client.
Splitting these into separate repositories would require either duplicating the shared
`internal/crypto` and `internal/erasure` packages or publishing them as external modules.
Duplication violates the single-source-of-truth principle for security-critical code — a
bug fixed in the microservice's copy of `AONTEncodeSegment` would not automatically be
fixed in the daemon. Publishing them as external modules introduces a versioning surface
that cannot be atomically updated across all three consumers.

The concrete shared packages that justify co-location are:

| Package | Consumers |
|---------|-----------|
| `internal/crypto` | `cmd/client` (AONT encoding, key derivation), `cmd/microservice` (pointer file blind store, cluster audit secret HKDF) |
| `internal/erasure` | `cmd/client` (upload encode), `cmd/microservice` (repair re-encode) |
| `internal/p2p` | `cmd/provider` (daemon P2P host), `cmd/microservice` (challenge dispatch) |
| `internal/audit` | `cmd/provider` (challenge response), `cmd/microservice` (challenge generation, receipt recording) |

Beyond package sharing, a single repository enables atomic cross-package changes. The
`challenge_nonce BYTEA(33)` schema correction ([`ADR-027`](../decisions/ADR-027-cluster-audit-secret.md),
[`requirements.md §9.3`](./requirements.md#93-hard-constraints)) must be applied in the
same commit to the migration file, the `internal/audit` package, and the
`interface-contracts.md` documentation. A split-repo model makes this atomicity
impossible without cross-repository PRs.

CI ownership enforcement (Section 7) relies on a single `CODEOWNERS` file. This is
not possible across multiple repositories without a shared CI platform; the monorepo
keeps ownership rules and code in the same diff.

### 1.2 Go Module Layout

A single `go.mod` at the repository root with module path `github.com/masamasaowl/vyomanaut`.
All three entry points (`cmd/microservice`, `cmd/provider`, `cmd/client`) and all shared
packages (`internal/*`) are part of the same module.

**Rationale for single-module over workspace:** Go workspaces (`go.work`) are designed
for multi-module development where modules will eventually be published independently.
Vyomanaut's internal packages are not published. A workspace would add indirection
(maintaining both `go.mod` per package and a root `go.work`) with no benefit, and would
make it possible for one binary's `go.mod` to drift from another's — reintroducing the
version-skew problem the monorepo is intended to solve.

**Build tags:** The three binaries are built separately:

```bash
go build ./cmd/microservice   # produces bin/microservice
go build ./cmd/provider       # produces bin/provider  (supports --sim-count)
go build ./cmd/client         # produces bin/vyomanaut-client
```

No custom build tags are used to select between packages. Conditional cipher selection
(ChaCha20 vs AES-256-CTR) is a runtime check via CPUID, not a compile-time tag, per
[`ADR-019`](../decisions/ADR-019-client-side-encryption.md).

### 1.3 Vendoring Policy

**Default: module proxy, no vendor directory.**
`go.sum` is committed to the repository and pinned for all dependencies.
`GONOSUMCHECK` and `GONOSUMDB` must be empty in the CI environment.

**Exception — `go-libp2p` and its sub-modules:** The custom DHT key validator
(`/vyomanaut/dht-key/1.0.0`) is a fork-point in go-libp2p's Kademlia configuration.
A `go-libp2p` version upgrade that silently resets the key validator to the default CID
namespace is a critical security failure (DHT privacy, [`ADR-001`](../decisions/ADR-001-coordination-architecture.md),
[`failure-analysis.md FM NET-05`](./failure-analysis.md)). If `TestDHTKeyValidatorPersists`
fails after a `go-libp2p` upgrade, the dependency must be vendored until the upstream fix
is available. The vendoring decision is recorded in a version-controlled change to this
file (Section 1.3) at the time of the decision; it is not made silently.

**Procedure for upgrading `go-libp2p`:**
1. Run `go get github.com/libp2p/go-libp2p@<new-version>` in a feature branch.
2. Run `TestDHTKeyValidatorPersists` and `TestDHTKeyValidator`. Both must pass.
3. If they fail: do not merge. Either fix the custom validator registration for the new
   API or vendor the previous passing version until the upstream fix lands.
4. Record the upgrade in the commit message with the format:
   `go-libp2p: upgrade to v{version}; TestDHTKeyValidatorPersists verified`.

**RocksDB (`linxGnu/grocksdb`):** CGo dependencies require a pre-built shared library.
The CI pipeline uses a Docker image with the correct `librocksdb` version pre-installed.
The exact Docker image tag and RocksDB version must be pinned in `.github/workflows/ci.yml`
and updated together when the Go binding version is changed. If the shared library version
and the Go binding version diverge, the daemon will fail to link at build time — this is
detectable in CI, not at runtime.

---

## 2. Top-Level Directory Map

```
/
├── cmd/
│   ├── microservice/          # Coordination microservice binary entrypoint
│   ├── provider/              # Provider daemon binary; supports --sim-count and --sim-asn-count
│   └── client/                # Data owner CLI binary
│
├── internal/
│   ├── crypto/                # AONT cipher, HKDF, Argon2id, pointer file AEAD, Ed25519 helpers
│   ├── erasure/               # Reed-Solomon RS(16,56) encode/decode via klauspost/reedsolomon
│   ├── storage/               # WiscKey: RocksDB chunk index + append-only vLog
│   ├── p2p/                   # libp2p host, QUIC/TCP transports, DHT, NAT traversal, heartbeat
│   ├── audit/                 # Challenge generation, dispatch, receipt two-phase write, cluster secret
│   ├── scoring/               # Three-window reliability score, consecutive-pass counter, EWMA RTO
│   ├── repair/                # Departure detector, repair job queue, repair executor (RS re-encode)
│   ├── payment/               # PaymentProvider interface, Razorpay implementation, escrow ledger
│   └── client/
│       ├── account/           # Registration, mnemonic, session key derivation, keystore
│       ├── upload/            # Upload orchestrator: encode + provider assignment + parallel transfer
│       ├── retrieve/          # Retrieval orchestrator: pointer file decrypt + shard download + RS+AONT decode
│       └── manage/            # File list, delete, escrow balance view
│
├── migrations/                # Numbered SQL DDL files; one .sql per forward migration, one .down.sql per destructive migration
│
├── deployments/               # Infrastructure-as-code; one subdirectory per environment
│   ├── production/            # Terraform modules or Kubernetes manifests for production
│   ├── staging/               # Staging environment; mirrors production topology at reduced scale
│   └── dev/                   # Local developer environment (Docker Compose)
│
├── scripts/                   # Developer tooling: lint, test, simulation runner, benchmarks
│
├── runbooks/                  # Operational runbooks referenced from architecture.md §23
│
├── docs/
│   ├── decisions/             # Architecture Decision Records (ADR-001 through ADR-029+)
│   ├── research/              # Paper research notes, reading list, open questions, answered questions
│   └── system-design/         # Architecture and specification documents
│       ├── sequence-diagrams/ # Six runtime flow diagrams (01 through 06)
│       └── api/               # openapi.yaml (REST/HTTP contract)
│
├── .github/
│   ├── workflows/             # CI pipeline definitions
│   └── CODEOWNERS             # Ownership boundaries (generated from Section 7 below)
│
├── go.mod                     # Single module: github.com/masamasaowl/vyomanaut
├── go.sum                     # Committed; all dependency checksums pinned
└── .golangci.yml              # Linter configuration; strict mode from M0
```

---

## 3. Detailed Package Descriptions

### 3.1 `cmd/`

Each `cmd/` subdirectory contains exactly one `main.go` file. No business logic lives in
`cmd/` — it is the wiring layer that constructs dependencies and calls into `internal/`.
Command-line flag parsing (`flag` stdlib or `cobra`) is acceptable in `cmd/`; anything
that requires a test belongs in `internal/`.

#### `cmd/microservice/`

Constructs and starts the coordination microservice. Responsibilities at startup:
1. Load `server_secret_vN` from the secrets manager (fail-closed if unavailable per
   [`ADR-027`](../decisions/ADR-027-cluster-audit-secret.md)).
2. Initialise the PostgreSQL connection pool.
3. Start the gossip cluster membership loop ([`ADR-025`](../decisions/ADR-025-microservice-consistency-mechanism.md)).
4. Start the audit challenge scheduler.
5. Start the departure detector and repair job dispatcher.
6. Register HTTP handlers from `internal/audit`, `internal/payment`, `internal/repair`,
   `internal/scoring`.
7. Expose the Prometheus scrape endpoint.
8. Block on signal (SIGTERM / SIGINT) for graceful shutdown.

The `--dev-mode` flag disables secrets manager enforcement and allows
`VYOMANAUT_CLUSTER_MASTER_SEED` as a substitute — **never in production**.

#### `cmd/provider/`

Constructs and starts the provider daemon. Supports two modes:

- **Normal mode:** One provider instance, persistent on-disk key pair, real vLog path,
  real libp2p addresses.
- **Simulation mode (`--sim-count=N`):** N isolated provider instances in a single
  process, each with its own key pair, RocksDB instance, vLog, and synthetic ASN.
  Simulation mode satisfies the network readiness gate (FR-053, FR-056) with
  `--sim-asn-count=5` (default) and does not bypass it
  ([`ADR-029`](../decisions/ADR-029-bootstrap-minimum-viable-network.md)).

Flags defined in `cmd/provider/main.go`:

| Flag | Default | Description |
|------|---------|-------------|
| `--microservice-url` | — | Required. HTTPS base URL of the coordination microservice. |
| `--data-dir` | `$HOME/.vyomanaut` | Path to the daemon's persistent data directory. |
| `--sim-count` | 0 (disabled) | Number of simulated provider instances. 0 = normal mode. |
| `--sim-base-port` | 4001 | Base libp2p listen port for simulation instances. |
| `--sim-data-dir` | `/tmp/vyomanaut-sim` | Root directory for simulation instance data. |
| `--sim-asn-count` | 5 | Number of synthetic ASNs for simulation mode. |
| `--relay-addrs` | — | Comma-separated list of relay node multiaddrs. |
| `--declared-storage-gb` | — | Required in normal mode. Provider's storage allocation. |

#### `cmd/client/`

The data owner CLI. Subcommands map directly to the user stories in
[`requirements.md §5`](./requirements.md#5-user-stories):

| Subcommand | User story | Package called |
|-----------|-----------|---------------|
| `register` | DO-01 onboarding | `internal/client/account` |
| `recover` | DO-05 | `internal/client/account` |
| `upload <path>` | DO-01 | `internal/client/upload` |
| `retrieve <file_id>` | DO-02 | `internal/client/retrieve` |
| `ls` | DO-04 | `internal/client/manage` |
| `rm <file_id>` | DO-03 | `internal/client/manage` |
| `balance` | DO-04 | `internal/client/manage` |
| `deposit` | DO-06 | `internal/client/manage` |

The CLI never prints the master secret, AONT key K, BIP-39 mnemonic words (after initial
setup), or any pointer file decryption key. These are in-memory only.

---

### 3.2 `internal/`

All business logic lives in `internal/`. No package under `internal/` may import from
`cmd/`. Packages within `internal/` may import each other, subject to the dependency
constraints in Section 3.2.10.

#### `internal/crypto/`

Key derivation (HKDF-SHA256, Argon2id), AONT cipher (ChaCha20-256 / AES-256-CTR),
pointer file authenticated encryption (AEAD_CHACHA20_POLY1305), and Ed25519 signing
helpers.

This package exports the interface defined in
[`interface-contracts.md §5.1`](./interface-contracts.md#51-internalcrypto). Every
exported function is pure (no shared mutable state) and goroutine-safe. The AES-NI
detection result is set once in `init()` via `DetectAESNI()` and is a package-level
constant for the lifetime of the process.

**Critical invariants enforced here:**
- AONT key K is never returned by any exported function. `AONTEncodeSegment` embeds K
  in the coded output; `AONTDecodePackage` recovers and immediately discards K after
  decryption. K is never stored in a struct field or passed to a logger.
- `DecryptPointerFile` uses `crypto/subtle.ConstantTimeCompare` for Poly1305 tag
  verification before any decryption is attempted (NFR-019).
- `ErrCanaryMismatch` causes the caller to zero the decoded plaintext buffer before
  returning an error to any upstream caller.

**Files in this package:**
- `hkdf.go` — HKDF-SHA256 derivation functions
- `argon2.go` — Argon2id master secret derivation
- `aoni.go` — AONT encode/decode (ChaCha20 and AES-256-CTR paths)
- `aont_canary.go` — fixed canary constant (a `const [16]byte`; never a variable)
- `chacha20poly1305.go` — pointer file AEAD encrypt/decrypt
- `aesni.go` — CPUID detection (`//go:build amd64` tag; stub for other GOARCH)
- `errors.go` — sentinel errors (`ErrTagMismatch`, `ErrCanaryMismatch`)
- `*_test.go` — table-driven tests including cross-platform vector tests and fuzz targets

#### `internal/erasure/`

Reed-Solomon RS(16, 56) encode and decode over GF(2^8) via `klauspost/reedsolomon`.

Exports `EncodeSegment` and `DecodeSegment` as specified in
[`interface-contracts.md §5.2`](./interface-contracts.md#52-internalerasure). The RS
parameters (s=16, r=40, n=56, lf=262144) are package-level constants. No function
accepts parameters that would allow changing these values — they are fixed by
[`ADR-003`](../decisions/ADR-003-erasure-coding.md) and must never be overridable at
runtime.

**Files:**
- `params.go` — exported constants `DataShards`, `ParityShards`, `TotalShards`, `ShardSize`
- `codec.go` — `EncodeSegment`, `DecodeSegment`
- `errors.go` — `ErrTooFewShards`, `ErrShardSize`
- `codec_test.go` — round-trip, any-k-shards, and shard-size tests

#### `internal/storage/`

WiscKey-style chunk storage engine: RocksDB chunk index + append-only vLog.

Exports the `ChunkStore` interface defined in
[`interface-contracts.md §5.3`](./interface-contracts.md#53-internalstorage).

The single-writer constraint is architectural, not just documented. The `VLog` type
internally owns one goroutine (`runWriter`) and one buffered channel. `AppendChunk` sends
to the channel and blocks on a result channel — callers that violate the single-writer
contract by calling `AppendChunk` from multiple goroutines will deadlock on the result
channel, not produce data corruption. This makes the violation detectable in CI under
the race detector.

SSD vs HDD detection: at `NewChunkStore` construction time, the package reads
`/sys/block/{dev}/queue/rotational` (Linux) or the platform equivalent (macOS IOKit,
Windows DeviceIoControl). The result sets the RocksDB `rate_limiter` default — separate
constants for SSD (`rocksdbRateLimiterSSD`) and HDD (`rocksdbRateLimiterHDD`), both
overridable via `ChunkStoreOptions`.

**Files:**
- `store.go` — `ChunkStore` interface, `NewChunkStore`
- `vlog.go` — vLog append, read, GC, crash recovery tail-scan
- `index.go` — RocksDB wrapper: Bloom filter, lookup, insert, delete
- `rotational.go` — HDD/SSD detection (`//go:build linux`)
- `rotational_other.go` — stub returning `false` (assume SSD) for non-Linux
- `errors.go` — sentinel errors
- `store_test.go` — store/retrieve, corruption detection, crash recovery, GC tests
- `single_writer_test.go` — `TestSingleWriterGoroutine` with 100-goroutine contention

#### `internal/p2p/`

libp2p host construction, QUIC/TCP transport initialisation, three-tier NAT traversal
(AutoNAT → DCUtR → Circuit Relay v2), Kademlia DHT with the custom HMAC key validator,
and the provider heartbeat sender.

Exports the `Host` and `DHT` interfaces defined in
[`interface-contracts.md §5.4`](./interface-contracts.md#54-internalp2p).

**Critical invariant:** The DHT key validator (`/vyomanaut/dht-key/1.0.0`) is registered
in `NewHost()` using the package-level constant `dhtKeyNamespace`. Any `go-libp2p`
upgrade must be followed by running `TestDHTKeyValidatorPersists` before merging. This
test is gated in `.github/workflows/ci.yml` as a required check.

**Files:**
- `host.go` — `NewHost`, `Host` interface implementation, 0-RTT policy enforcement
- `dht.go` — `DHT` interface implementation, custom HMAC key validator
- `dht_namespace.go` — `const dhtKeyNamespace = "/vyomanaut/dht-key/1.0.0"` (sole definition)
- `nat.go` — AutoNAT, DCUtR, Circuit Relay v2 configuration (`maxHolePunchRetries = 1`)
- `heartbeat.go` — 4-hour signed heartbeat to microservice (`POST /api/v1/provider/heartbeat`)
- `identity.go` — Ed25519 key pair generation and persistence in the provider keystore
- `errors.go` — `ErrPeerIDMismatch`, `ErrAllAddrsFailed`, `ErrDHTKeyInvalid`
- `dht_test.go` — `TestDHTKeyValidator`, `TestDHTKeyValidatorPersists` (CI required check)

#### `internal/audit/`

Audit challenge generation (versioned nonce), challenge dispatch, receipt two-phase write,
and cluster audit secret loading from the secrets manager.

Exports the functions and types defined in
[`interface-contracts.md §5.5`](./interface-contracts.md#55-internalaudit).

The secrets manager client is injected as a `SecretsManagerClient` interface at
construction time — no global state. In simulation mode, the `VYOMANAUT_CLUSTER_MASTER_SEED`
environment variable is substituted for the secrets manager by the `cmd/provider` entrypoint.
`internal/audit` itself has no knowledge of simulation mode; it only accepts a
`SecretsManagerClient` implementation.

**Files:**
- `challenge.go` — `ChallengeNonce`, versioned nonce generation (33 bytes)
- `validate.go` — `ValidateResponse`, Ed25519 signature verification
- `receipt.go` — `WriteReceiptPhase1`, `WriteReceiptPhase2`, GC of stale PENDING rows
- `secret.go` — `ClusterSecretCache` (5-minute TTL, fail-closed on expiry)
- `secrets_iface.go` — `SecretsManagerClient` interface
- `jit.go` — JIT detection threshold computation and `jit_flag` evaluation
- `errors.go` — `ErrInvalidSignature`, `ErrNonceLength`, `ErrReceiptAlreadyFinal`
- `audit_test.go` — nonce generation, cross-replica validation, two-phase crash safety, idempotent retry

#### `internal/scoring/`

Three-window reliability score, consecutive-pass counter management, EWMA RTO update,
and the dual-window deterioration flag computation.

Exports the functions defined in
[`interface-contracts.md §5.6`](./interface-contracts.md#56-internalscoring).

Score computation reads from the `mv_provider_scores` materialised view for operational
queries (up to 60 s stale), but the monthly release multiplier must read from the primary
— enforced by `GetScoreFromPrimary` which accepts a primary-directed `*sql.DB` parameter.
See [`failure-analysis.md FM MS-06`](./failure-analysis.md#31-coordination-microservice-cluster).

**Files:**
- `score.go` — `GetScore`, `GetScoreFromPrimary`, `ProviderScore`
- `passes.go` — `IncrementConsecutivePasses`, `ResetConsecutivePasses`
- `rto.go` — EWMA update for `avg_rtt_ms`, `var_rtt_ms`, `p95_throughput_kbps`
- `errors.go` — `ErrProviderNotFound`, `ErrProviderNotVetting`
- `score_test.go` — known-history score, dual-window flag, VETTING→ACTIVE transition race guard

#### `internal/repair/`

Departure detector, repair job queue, and repair executor (RS re-encode + provider
selection + upload to replacement).

Exports the functions and types defined in
[`interface-contracts.md §5.7`](./interface-contracts.md#57-internalrepair).

The departure detector runs on a configurable polling interval (default 60 s) and issues
`SELECT provider_id FROM providers WHERE status='ACTIVE' AND last_heartbeat_ts < NOW() -
INTERVAL '72 hours'`. On detection, it calls into `internal/payment` (escrow seizure) and
`internal/scoring` (freeze) before enqueuing repair jobs — all within the same database
transaction to ensure atomicity.

The repair executor contacts 16 surviving fragment holders via `internal/p2p`, calls
`internal/erasure.DecodeSegment`, re-encodes missing shards, and uploads to replacement
providers selected by `internal/scoring` (Power of Two Choices + 20% ASN cap
re-enforcement per [`ADR-014`](../decisions/ADR-014-adversarial-defences.md)).

**Files:**
- `departure.go` — departure detector loop, 72h threshold, state transitions
- `queue.go` — `EnqueueJob`, `DequeueNextJob`, `MarkJobComplete`
- `executor.go` — repair execution: download 16 shards → RS decode → re-encode → upload to replacements
- `assignment.go` — Power of Two Choices + ASN cap enforcement for replacement provider selection
- `errors.go`
- `repair_test.go` — priority ordering, ASN cap enforcement, emergency floor trigger

#### `internal/payment/`

`PaymentProvider` interface, Razorpay implementation, append-only escrow ledger, monthly
release computation, and escrow seizure.

Exports the interfaces and functions defined in
[`interface-contracts.md §5.8`](./interface-contracts.md#58-internalpayment).

**Critical invariant — no floating-point anywhere in this package.** The CI check
`TestNoFloatArithmetic` runs `go vet` with a custom analyser and a grep over all `.go`
files in this package matching `float64|float32`. All Razorpay webhook `amount` fields
are unmarshalled into the `PaiseAmount` type (Section 6.6), not `interface{}`.

**Files:**
- `provider.go` — `PaymentProvider` interface definition
- `razorpay.go` — Razorpay implementation of `PaymentProvider`
- `ledger.go` — `InsertEscrowEvent`; `EscrowEventType` constants
- `balance.go` — `GetBalance` (queries `mv_provider_escrow_balance` or primary)
- `release.go` — monthly release computation, release multiplier table, dual-window flag check
- `seizure.go` — escrow seizure on departure, Razorpay Route reversal
- `paise.go` — `PaiseAmount int64` type with custom JSON unmarshaller that rejects fractional values
- `rbi_holidays.go` — `LastWorkingDayOfMonth` using the annually-updated RBI bank holiday table
- `errors.go` — `ErrDuplicateIdempotencyKey`
- `payment_test.go` — PN-counter balance, idempotent release, no-float static check, dual-window flag
- `razorpay_test.go` — webhook handler tests (payment.captured, payout.reversed, account.created)

#### `internal/client/account/`

Data owner registration, Argon2id master secret derivation, BIP-39 mnemonic generation
and confirmation gate, keystore management, and new-device recovery.

**Files:**
- `register.go` — registration flow, OTP handling, Ed25519 key pair generation
- `master_secret.go` — `DeriveMasterSecret` wrapper enforcing UI gate before upload
- `mnemonic.go` — BIP-39 mnemonic generation and two-word confirmation gate (FR-003)
- `keystore.go` — encrypted keystore: Ed25519 key, pointer file nonce counter
- `recover.go` — passphrase and mnemonic recovery paths, nonce-counter recovery from `files` table
- `account_test.go` — mnemonic confirmation gate, cross-platform Argon2id vector, nonce recovery

#### `internal/client/upload/`

Upload orchestrator: AONT-RS encoding, assignment request, parallel libp2p shard
transfer, pointer file construction and registration, session state persistence for
crash recovery (FR-060).

Exports `UploadOrchestrator.UploadFile` and `UploadOrchestrator.ResumeUpload` per
[`interface-contracts.md §5.9`](./interface-contracts.md#59-internalclient).

**Files:**
- `orchestrator.go` — `UploadOrchestrator` implementation
- `session.go` — local session state: file_id, chunk_ids, ack_status[56], persisted to disk
- `assign.go` — assignment request to microservice, readiness gate handling (HTTP 503)
- `transfer.go` — parallel libp2p shard upload, receipt collection, progress callback
- `pointer.go` — pointer file construction, HKDF key derivation, AEAD encryption, microservice registration
- `upload_test.go` — full pipeline, progress callback, escrow pre-check, crash recovery via resume

#### `internal/client/retrieve/`

Retrieval orchestrator: pointer file fetch and decryption, parallel shard download (all
56, cancel after first 16 valid), content-address verification per shard, RS decode,
AONT decode, canary verification, padding strip.

Exports `UploadOrchestrator.RetrieveFile` per
[`interface-contracts.md §5.9`](./interface-contracts.md#59-internalclient).

**Files:**
- `orchestrator.go` — `RetrieveFile` implementation
- `pointer.go` — fetch encrypted pointer file, derive key, constant-time tag verify, decrypt
- `download.go` — parallel 56-provider dial, cancel after 16 valid shards, shard content-address verification
- `decode.go` — RS decode, AONT decode, K recovery, canary check, padding strip, buffer zeroing on canary fail
- `retrieve_test.go` — any-16-of-56, canary fail aborts with no plaintext, tag mismatch aborts before decrypt

#### `internal/client/manage/`

File list, file delete (chunk assignment removal + provider notification), escrow balance
view, and escrow deposit UPI Intent flow.

**Files:**
- `files.go` — file list query, availability status from shard counts
- `delete.go` — DELETE request to microservice, pending_deletion tracking
- `escrow.go` — escrow balance view, deposit initiation (UPI Intent, not UPI Collect per NFR-029)

#### Package Import Dependency Constraints

The following import directions are **prohibited**. Any PR that introduces a prohibited
import must be rejected:

| Package | Must NOT import |
|---------|----------------|
| `internal/crypto` | Any other `internal/` package |
| `internal/erasure` | Any other `internal/` package |
| `internal/storage` | `internal/payment`, `internal/scoring`, `internal/repair` |
| `internal/payment` | `internal/repair`, `internal/p2p` |
| `internal/scoring` | `internal/repair`, `internal/payment` |
| `internal/audit` | `internal/scoring`, `internal/repair`, `internal/payment` |
| Any `internal/client/*` | `cmd/` |

The dependency graph flows: `cmd/*` → `internal/client/*` → (`internal/upload`,
`internal/retrieve`, `internal/manage`) → (`internal/crypto`, `internal/erasure`,
`internal/p2p`) → (no `internal/` imports). The microservice and daemon wire up
`internal/audit`, `internal/scoring`, `internal/repair`, and `internal/payment`
from their `cmd/` entrypoints.

---

## 4. Configuration and Infrastructure

#### `migrations/`

SQL migration files applied sequentially to the PostgreSQL schema. Every forward
migration has a corresponding `down` file for any migration that alters existing rows.
New-column-only additions do not require a `down` file.

```
migrations/
  001_initial_schema.sql            # All tables, types, indexes, row security policies (M0)
  001_initial_schema.down.sql       # Drops all tables
  002_add_promised_return_at.sql    # providers.promised_return_at (if added as a separate migration)
  ...
```

The migration runner (`scripts/migrate.sh`) applies migrations in numeric order and
records the last-applied migration number in a `schema_migrations` table. No migration
may be applied out of order. No migration number may be reused.

The M0 migration (`001_initial_schema.sql`) must include, in order:
1. `CREATE EXTENSION IF NOT EXISTS btree_gist;` (required for `audit_periods` EXCLUDE constraint)
2. All `CREATE TYPE` statements
3. All `CREATE TABLE` statements
4. All `CREATE INDEX` statements
5. All `ALTER TABLE ENABLE ROW LEVEL SECURITY` and `CREATE POLICY` statements
6. All `CREATE MATERIALIZED VIEW` statements and their unique indexes

The checklist in [`data-model.md §9`](./data-model.md#9-migration-checklist) must be
verified before `001_initial_schema.sql` is committed.

#### `deployments/`

Infrastructure-as-code. The exact tooling (Terraform vs. Pulumi vs. Kubernetes YAML) is
an infrastructure team decision; the directory structure is fixed.

```
deployments/
  production/
    microservice/              # VM provisioning, managed Postgres, load balancer, relay nodes
    secrets/                   # Secrets manager bootstrap (does NOT contain secrets)
  staging/
    microservice/
    secrets/
  dev/
    docker-compose.yml         # Local: microservice + Postgres + relay nodes + simulation providers
    docker-compose.override.yml # Developer-specific overrides (not committed)
```

`deployments/` must never contain actual secret values, API keys, or the cluster master
seed — only references to secrets manager paths. See Section 8.

#### `scripts/`

Developer tooling. All scripts are POSIX shell or Go programs.

```
scripts/
  migrate.sh                   # Apply pending SQL migrations
  lint.sh                      # Run golangci-lint with .golangci.yml
  test-crash-recovery.sh       # M2 exit criterion: store 1000 chunks, kill, restart, verify all retrievable
  test-p2p-connection.sh       # M3 exit criterion: two providers connect and exchange a test message
  test-audit-cycle.sh          # M4 exit criterion: 56-provider simulated audit cycle
  test-self-healing.sh         # M5 exit criterion: provider departure → repair → fragment count restored
  test-payment-cycle.sh        # M6 exit criterion: payout computation and Razorpay test API interaction
  test-end-to-end.sh           # M7 exit criterion: upload → retrieve → SHA-256 match → delete
  test-live-e2e.sh             # M8 exit criterion: production upload/retrieval
  manual-audit-challenge.sh    # Operator tooling: issue a challenge to a specific provider
  benchmarks/
    aont_encode.sh             # Q16-1: AONT encoding benchmark on minimum-spec hardware
    argon2id.sh                # Q18-1: Argon2id benchmark on minimum-spec hardware
    rocksdb_ssd.sh             # Q27-1: RocksDB audit latency on SSD
    rocksdb_hdd.sh             # Q27-1 HDD variant: audit latency on 7200 RPM HDD
```

#### `runbooks/`

Operational runbooks linked from [`architecture.md §23`](./architecture.md#23-error-handling)
and required by [`mvp.md §M8`](./mvp.md#milestone-8--network-readiness-gate-and-private-beta).
All runbooks must exist before M8 closes.

```
runbooks/
  microservice-failover.md
  postgres-failover.md
  relay-node-replacement.md
  secrets-manager-outage.md
  razorpay-api-outage.md
  provider-mass-departure.md
  rbi-holiday-table-update.md
  audit-secret-rotation.md
```

#### `.github/`

```
.github/
  workflows/
    ci.yml                     # Triggered on every PR: go build, go vet, golangci-lint, go test -race, migration apply/rollback
    release.yml                # Triggered on version tags: cross-platform binary builds + signed installers
  CODEOWNERS                   # Generated from Section 7
```

CI required checks (all must pass before merge):
- `go build ./...` (zero warnings, strict mode)
- `go vet ./...`
- `golangci-lint run`
- `go test ./... -race`
- `TestDHTKeyValidatorPersists` (labelled as a separate required check; blocks merge if failing)
- `TestNoFloatArithmetic` (blocks merge if `internal/payment/` contains any float type)
- Migration apply + rollback against CI Postgres instance
- Mermaid render check (all `.md` files in `docs/system-design/`)
- Hyperlink check (`markdown-link-check`)
- Grep checks:
  - `challenge_nonce BYTEA(32)` in any document context → **fail** (must be BYTEA(33))
  - `float64|float32|FLOAT|DECIMAL|NUMERIC` in payment amount context → **fail**
  - `ADR-039` anywhere → **fail** (non-existent ADR reference)

---

## 5. Documentation Layout

```
docs/
├── decisions/
│   ├── README.md                          # ADR index (existing)
│   ├── ADR-001-coordination-architecture.md
│   ├── ADR-002-proof-of-storage.md
│   ├── ADR-003-erasure-coding.md
│   ├── ADR-004-repair-protocol.md
│   ├── ADR-005-peer-selection.md
│   ├── ADR-006-polling-interval.md
│   ├── ADR-007-provider-exit-states.md
│   ├── ADR-008-reliability-scoring.md
│   ├── ADR-009-background-execution.md
│   ├── ADR-010-desktop-only-v2.md
│   ├── ADR-011-escrow-payments.md
│   ├── ADR-012-payment-basis.md
│   ├── ADR-013-consistency-model.md
│   ├── ADR-014-adversarial-defences.md
│   ├── ADR-015-audit-trail.md
│   ├── ADR-016-payment-db-schema.md
│   ├── ADR-017-audit-receipt-schema.md
│   ├── ADR-018-hot-cold-storage-bands.md
│   ├── ADR-019-client-side-encryption.md
│   ├── ADR-020-key-management.md
│   ├── ADR-021-p2p-transfer-protocol.md
│   ├── ADR-022-encryption-erasure-order.md
│   ├── ADR-023-provider-storage-engine.md
│   ├── ADR-024-economic-mechanism.md
│   ├── ADR-025-microservice-consistency-mechanism.md
│   ├── ADR-026-repair-bw-optimisation.md
│   ├── ADR-027-cluster-audit-secret.md
│   ├── ADR-028-provider-heartbeat.md
│   └── ADR-029-bootstrap-minimum-viable-network.md
│
├── research/
│   ├── reading-list.md                    # Paper reading list with phase annotations
│   ├── open-questions.md                  # Unresolved questions (OQ-001 through OQ-FMEA-006+)
│   ├── answered-questions.md             # Resolved questions with citations
│   ├── benchmarking-protocol.md          # Q16-1, Q18-1, Q27-1 benchmark specifications
│   ├── paper-01-bittorrent.md            # Research notes per paper
│   ├── paper-02-kademlia.md
│   ├── ...
│   └── paper-41-vakilinia-incentive-compatible-dsn.md
│
└── system-design/
    ├── README.md                          # Universal rules (hyperlink discipline, source of truth hierarchy, diagram format)
    ├── architecture.md                    # System architecture document
    ├── requirements.md                    # Product requirements document (PRD)
    ├── data-model.md                      # Canonical database schema
    ├── interface-contracts.md             # Inter-component contracts
    ├── trade-offs.md                      # Accepted trade-offs reference
    ├── capacity.md                        # Capacity estimation
    ├── failure-analysis.md                # FMEA table
    ├── tech-stack.md                      # Technology stack rationale
    ├── mvp.md                             # MVP build plan and milestone roadmap
    ├── repo-structure.md                  # This document
    └── sequence-diagrams/
        ├── 01-file-upload.md
        ├── 02-audit-cycle.md
        ├── 03-repair-flow.md
        ├── 04-payment-release.md
        ├── 05-provider-lifecycle.md
        └── 06-file-retrieval.md
```

**Invariants for the documentation layout:**

- Every reference to another document in `docs/` must be a relative hyperlink.
  `grep -r '"ADR-' docs/` without a `.md` suffix is a CI failure (hyperlink discipline
  per [`docs/system-design/README.md §0.1`](./README.md#01-hyperlink-discipline)).
- No new ADR number may be introduced without a corresponding file. The non-existent
  `ADR-039` grep check (Section 4) prevents stale references.
- Every ADR added must have a row in [`docs/decisions/README.md`](../decisions/README.md).
- Research notes for a new paper must follow the existing `paper-NN-slug.md` naming
  convention. The NN is the sequential paper number from the reading list.

---

## 6. Naming Conventions

### Migration files

```
NNN_short_description.sql          # Forward migration (NNN zero-padded to 3 digits)
NNN_short_description.down.sql     # Rollback (required for any migration altering existing rows)
```

Examples:
- `001_initial_schema.sql`
- `002_add_providers_promised_return_at.sql`
- `002_add_providers_promised_return_at.down.sql`

Rules:
- NNN is sequential with no gaps. If migration 003 is abandoned, 004 is still named 004.
- `short_description` uses underscores, lowercase, no special characters. Describes the
  change, not the ticket number.
- A migration that only adds columns (NOT NULL with DEFAULT) does not require a down file.
- A migration that changes a column type, removes a column, or alters a constraint requires
  a down file.
- Never edit a committed migration. Write a new one.

### Go package names

- Lowercase, single word where possible. No abbreviations except those in the
  [`mvp.md §Terminology and Glossary`](./mvp.md#terminology-and-glossary):
  `aont`, `vlog`, `p2p`, `rto`, `hkdf`, `dht`.
- Package names must match the directory name exactly (Go convention).
- Test files: `foo_test.go` in the same package (`package foo`) for white-box tests;
  `foo_external_test.go` with `package foo_test` for black-box tests of the exported surface.

### Go exported symbol names

Exported function and type names must match exactly the names specified in
[`interface-contracts.md §5`](./interface-contracts.md#5-internal-go-package-contracts).
Renaming an exported symbol requires updating both the implementation and
`interface-contracts.md` in the same PR. No silent renames.

Key names to never change without an `interface-contracts.md` update:
- `AONTEncodeSegment`, `AONTDecodePackage` — used in cross-package references
- `ChunkStore`, `AppendChunk`, `LookupChunk` — storage engine interface
- `InsertEscrowEvent`, `EscrowEventType` — ledger functions
- `ChallengeNonce` — must always produce a 33-byte slice
- `PaiseAmount` — monetary amount type; renaming breaks the float-prevention static analysis

### Simulation mode directories

The simulation instance data path is fixed across all documents and CI scripts:
`/tmp/vyomanaut-sim/{instance_id}/` where `instance_id` is the zero-padded instance
index (e.g. `0000`, `0001`). Sub-directories:
- `/tmp/vyomanaut-sim/{instance_id}/keys/` — Ed25519 key pair
- `/tmp/vyomanaut-sim/{instance_id}/db/` — RocksDB instance
- `/tmp/vyomanaut-sim/{instance_id}/vlog/chunks.vlog` — value log file

This path is used verbatim in [`mvp.md §M0`](./mvp.md#milestone-0--foundation),
[`ADR-029`](../decisions/ADR-029-bootstrap-minimum-viable-network.md), and the
`test-*.sh` scripts in `scripts/`. Changing it requires updating all four locations
atomically.

### Runbook filenames

Must use the exact topic names from [`architecture.md §23`](./architecture.md#23-error-handling):
`microservice-failover`, `postgres-failover`, `relay-node-replacement`,
`secrets-manager-outage`, `razorpay-api-outage`, `provider-mass-departure`,
`rbi-holiday-table-update`, `audit-secret-rotation`. Format: `{topic-name}.md`.

### Prometheus metric names

Follow the pattern `vyomanaut_{subsystem}_{name}_{unit}` where applicable. Examples:
- `vyomanaut_audit_challenges_issued_total`
- `vyomanaut_repair_queue_depth`
- `vyomanaut_provider_score` (histogram)
- `vyomanaut_content_hash_failures_total`

All metrics defined in [`requirements.md NFR-025 and NFR-026`](./requirements.md#76-observability-and-operability)
must use these exact names — Grafana dashboard JSON and alert rules reference them by name.

---

## 7. Ownership Map

This table is authoritative for code review requirements. The `.github/CODEOWNERS` file
below is generated from it and must be kept in sync. When a new directory is added, a
new row must be added to this table and `CODEOWNERS` updated in the same PR.

### Ownership table

| Directory / File | Primary owner | Required reviewers | Notes |
|-----------------|---------------|-------------------|-------|
| `internal/crypto/` | Eng-A | Eng-B or Eng-C (any one) | Any change to `ErrCanaryMismatch`, `AONTEncodeSegment`, `DecryptPointerFile` also requires a note in the PR description explaining the security impact |
| `internal/erasure/` | Eng-A | Eng-B or Eng-C (any one) | RS parameter constants must never change without a new ADR |
| `internal/storage/` | Eng-B | Eng-A or Eng-C (any one) | Changes to the vLog entry format require a migration checklist review |
| `internal/p2p/` | Eng-C | Eng-A or Eng-B (any one) | `dht_namespace.go` requires both Eng-A and Eng-B |
| `internal/p2p/dht_namespace.go` | Eng-C | Eng-A AND Eng-B (both required) | Critical security constant; single-engineer change prohibited |
| `internal/audit/` | Eng-B (lead), Eng-C | Eng-A AND Eng-C for any change to `challenge.go` or `secret.go` | Challenge nonce and secret loading are security-critical paths |
| `internal/scoring/` | Eng-B | Eng-A or Eng-C (any one) | |
| `internal/repair/` | Eng-B | Eng-A or Eng-C (any one) | |
| `internal/payment/` | Eng-C (lead) | Eng-B (required); `paise.go` additionally requires Eng-A | `TestNoFloatArithmetic` must pass; no PR to `internal/payment/` may be merged without it |
| `internal/client/` | Eng-A (lead) | Eng-B or Eng-C (any one) | |
| `migrations/` | Any | Two of three engineers required | Single-engineer migration prohibited |
| `docs/decisions/` | Any | Any one reviewer | New ADR also requires a row in `docs/decisions/README.md` |
| `docs/system-design/` | Any | Any one reviewer | Changes to `interface-contracts.md` require the owner of the affected package |
| `deployments/` | Eng-C | Eng-B | Production deployments require explicit approval from both |
| `runbooks/` | Any | None required (best-effort review) | Corrections welcome from any engineer |
| `.github/workflows/` | Eng-C | Eng-B | CI changes must not remove any required check |
| `.github/CODEOWNERS` | Eng-C | Eng-A AND Eng-B (both required) | Must reflect this table; both sign off |

### `.github/CODEOWNERS`

```
# Vyomanaut V2 — CODEOWNERS
# Generated from docs/system-design/repo-structure.md §7
# Update both files in the same PR when ownership changes.

# Default: any one reviewer required
*                                @eng-a @eng-b @eng-c

# internal/crypto — security-critical; any one of B or C
/internal/crypto/                @eng-a @eng-b @eng-c

# internal/erasure
/internal/erasure/               @eng-a @eng-b @eng-c

# internal/storage
/internal/storage/               @eng-b @eng-a @eng-c

# internal/p2p — DHT namespace requires B AND C
/internal/p2p/                   @eng-c @eng-a @eng-b
/internal/p2p/dht_namespace.go   @eng-c @eng-a @eng-b

# internal/audit — challenge.go and secret.go require A AND C
/internal/audit/                 @eng-b @eng-c
/internal/audit/challenge.go     @eng-b @eng-c @eng-a
/internal/audit/secret.go        @eng-b @eng-c @eng-a

# internal/scoring + internal/repair
/internal/scoring/               @eng-b @eng-a @eng-c
/internal/repair/                @eng-b @eng-a @eng-c

# internal/payment — float prohibition enforced here
/internal/payment/               @eng-c @eng-b
/internal/payment/paise.go       @eng-c @eng-b @eng-a

# internal/client
/internal/client/                @eng-a @eng-b @eng-c

# migrations — two reviewers required
/migrations/                     @eng-a @eng-b @eng-c

# deployments — production requires both B and C
/deployments/production/         @eng-c @eng-b

# .github — CI changes require B and C
/.github/workflows/              @eng-c @eng-b
/.github/CODEOWNERS              @eng-c @eng-a @eng-b
```

---

## 8. Code That Must Never Live Here

The following categories of content are prohibited from the repository. A PR introducing
any of the below must be rejected at review and must not be merged:

### Secrets and credentials

- **Private keys of any kind** — Ed25519 private keys, TLS private keys, SSH keys —
  even in encrypted or base64-encoded form. Private keys belong in the provider's
  encrypted keystore on their machine or in the secrets manager.
- **The `VYOMANAUT_CLUSTER_MASTER_SEED` value** — must live in the secrets manager only
  per [`ADR-027`](../decisions/ADR-027-cluster-audit-secret.md). The environment variable
  name may appear in code and documentation; the value may never appear.
- **Razorpay live API keys** (`rzp_live_*`) — must live in the secrets manager.
  Test keys (`rzp_test_*`) may appear in CI environment variables via GitHub Actions
  secrets, never as literals in source files.
- **Razorpay webhook secret** — must live in the secrets manager.
- **BIP-39 mnemonic words for any real account** — test vectors using fixed entropy
  values from RFC documents are acceptable; real user mnemonic phrases are not.

### Hardcoded values that require annual updates

- **RBI bank holiday data outside `internal/payment/rbi_holidays.go`** — this file is
  the single authoritative location for holiday data. Hard-coding holiday dates in test
  files, migration scripts, or deployment configuration bypasses the update procedure in
  `runbooks/rbi-holiday-table-update.md`.

### Vendored dependencies (unless explicitly decided)

- **A vendor directory for `go-libp2p`** unless the vendoring decision has been made
  due to `TestDHTKeyValidatorPersists` failing after an upgrade. If that decision is
  made, it must be documented in Section 1.3 of this file as a version-controlled change.
  Silent vendoring — adding `vendor/github.com/libp2p/` without updating this document —
  is prohibited.

### Code architecture violations

- **Business logic in `cmd/`** — `cmd/` is wiring only. Any function that has testable
  behaviour belongs in `internal/`.
- **Imports from `cmd/` in `internal/`** — `internal/` packages must never import from
  `cmd/`. This would create a circular dependency and is caught by `go build`.
- **Prohibited cross-package imports** — see Section 3.2.10. `internal/crypto` importing
  any other `internal/` package is a hard prohibition; any such import must be rejected
  at review regardless of the stated justification.
- **Float arithmetic in `internal/payment/`** — `float64`, `float32`, `FLOAT`, `DECIMAL`,
  `NUMERIC` in any form. The `TestNoFloatArithmetic` CI check enforces this. A PR that
  disables or weakens this test must be rejected.

### Deprecated patterns

- **UPI Collect endpoint calls** — deprecated by NPCI as of 28 February 2026.
  Any call to Razorpay's Collect API must not be introduced. All deposit flows
  must use UPI Intent (per [`NFR-029`](./requirements.md#77-compliance-and-payments)).
- **`challenge_nonce` as a 32-byte field** — the `challenge_nonce` column is always
  `BYTEA(33)` (1 version byte + 32 HMAC bytes). A migration or schema change introducing
  `BYTEA(32)` for this field must be rejected. The CI grep check enforces this.
- **RC4-128 or MD5 in any cryptographic context** — RC4 is cryptographically broken
  (RFC 7465, 2015) and is listed under rejected technologies in
  [`tech-stack.md §17`](./tech-stack.md#17-technologies-explicitly-rejected).
- **Convergent encryption or per-file deduplication** — each AONT key K is fresh random
  per segment by design. Any code path that reuses K across files or segments violates
  the zero-knowledge property.
- **ADR-039 or any non-existent ADR number** — references to ADRs that do not exist
  create false confidence and are caught by the CI grep check.