# Vyomanaut V2 — Build Procedure Part 2

**Status:** Authoritative build specification  
**Version:** 1.0  
**Date:** May 2026  
**Repository:** <https://github.com/masamasaowl/Vyomanaut_Research>  
**Derived from:**

1. `docs/system-design/requirements.md` (REQ) — FR/NFR completeness gates, capacity calculations
2. `docs/system-design/architecture.md` (ARCH) — system overview, deployment topology, relay infrastructure
3. `docs/system-design/data-model.md` (DM) — PostgreSQL schema, invariants, indexes, row security policies
4. `docs/system-design/interface-contracts.md` (IC) — wire contracts, Go package interfaces, forbidden patterns
5. `docs/system-design/mvp.md` (MVP) — NetworkProfile, demo/prod mode, repository layout, CI pipeline
6. `docs/system-design/api/openapi.yaml` (OAS) — authoritative REST/HTTP surface; all endpoint schemas

> **Reading convention.** Each session carries a `[REF]` tag pointing to the exact document section that governs it. Every session is atomic: it produces a passing `go test` or a passing migration apply before the next session begins. Logical errors discovered during implementation are fixed in the current session with a note appended to the session log — do not silently carry forward broken invariants.

---

## Table of Contents

- [Build Dependency Graph](#build-dependency-graph)
- [Milestone 7 — Audit System (`internal/audit`)](#milestone-7--audit-system-internalaudit)
  - [Phase 7.1 — Challenge Nonce Generation](#phase-71--challenge-nonce-generation)
  - [Phase 7.2 — Response Validation](#phase-72--response-validation)
  - [Phase 7.3 — Two-Phase Receipt Write](#phase-73--two-phase-receipt-write)
  - [Phase 7.4 — Cluster Secret Cache](#phase-74--cluster-secret-cache)
  - [Phase 7.5 — JIT Detection](#phase-75--jit-detection)
- [Milestone 8 — Scoring System (`internal/scoring`)](#milestone-8--scoring-system-internalscoring)
  - [Phase 8.1 — Score Retrieval](#phase-81--score-retrieval)
  - [Phase 8.2 — Consecutive Pass Counter](#phase-82--consecutive-pass-counter)
  - [Phase 8.3 — EWMA RTO Tracking](#phase-83--ewma-rto-tracking)
- [Milestone 9 — Repair System (`internal/repair`)](#milestone-9--repair-system-internalrepair)
  - [Phase 9.1 — Repair Job Queue](#phase-91--repair-job-queue)
  - [Phase 9.2 — Repair Executor](#phase-92--repair-executor)
  - [Phase 9.3 — Departure Detector](#phase-93--departure-detector)
- [Milestone 10 — Payment System (`internal/payment`)](#milestone-10--payment-system-internalpayment)
  - [Phase 10.1 — PaymentProvider Interface & Mock](#phase-101--paymentprovider-interface--mock)
  - [Phase 10.2 — Escrow Ledger](#phase-102--escrow-ledger)
  - [Phase 10.3 — Razorpay Implementation](#phase-103--razorpay-implementation)
  - [Phase 10.4 — Release Computation & Seizure](#phase-104--release-computation--seizure)
- [Milestone 11 — REST API Layer](#milestone-11--rest-api-layer)
  - [Phase 11.1 — Error Envelope](#phase-111--error-envelope)
  - [Phase 11.2 — Readiness Gate Endpoint](#phase-112--readiness-gate-endpoint)
  - [Phase 11.3 — Endpoint Routing Stubs](#phase-113--endpoint-routing-stubs)
  - [Phase 11.4 — Authentication Endpoints](#phase-114--authentication-endpoints)
  - [Phase 11.5 — Owner Endpoints](#phase-115--owner-endpoints)
  - [Phase 11.6 — Provider REST Endpoints](#phase-116--provider-rest-endpoints)
  - [Phase 11.7 — Upload and File Endpoints](#phase-117--upload-and-file-endpoints)
  - [Phase 11.8 — Pricing Endpoints](#phase-118--pricing-endpoints)
  - [Phase 11.9 — Audit Admin Endpoints](#phase-119--audit-admin-endpoints)
  - [Phase 11.10 — Admin Endpoints](#phase-1110--admin-endpoints)
  - [Phase 11.11 — Per-Provider Chunk Count Ceiling](#phase-1111--per-provider-chunk-count-ceiling)
- [Milestone 12 — Coordination Microservice (`cmd/microservice`)](#milestone-12--coordination-microservice-cmdmicroservice)
  - [Phase 12.1 — Microservice Startup](#phase-121--microservice-startup)
- [Milestone 13 — Provider Daemon Core (`cmd/provider`)](#milestone-13--provider-daemon-core-cmdprovider)
  - [Phase 13.1 — Provider Startup](#phase-131--provider-startup)
  - [Phase 13.2 — Chunk Upload Stream Handler](#phase-132--chunk-upload-stream-handler)
  - [Phase 13.3 — Audit Challenge Stream Handler](#phase-133--audit-challenge-stream-handler)
  - [Phase 13.4 — Repair Download Stream Handler](#phase-134--repair-download-stream-handler)
  - [Phase 13.5 — Vetting GC Stream Handler](#phase-135--vetting-gc-stream-handler)
  - [Phase 13.6 — Provider RAM Check at Installation](#phase-136--provider-ram-check-at-installation)
- [Milestone 14 — Vetting & Synthetic Chunks (`internal/vettingchunk`)](#milestone-14--vetting--synthetic-chunks-internalvettingchunk)
  - [Phase 14.1 — Synthetic Chunk Generator](#phase-141--synthetic-chunk-generator)
  - [Phase 14.2 — GC Delivery](#phase-142--gc-delivery)
- [Milestone 15 — Client SDK (`internal/client`)](#milestone-15--client-sdk-internalclient)
  - [Phase 15.1 — Account Management](#phase-151--account-management)
  - [Phase 15.2 — Upload Orchestrator](#phase-152--upload-orchestrator)
  - [Phase 15.3 — Retrieval Orchestrator](#phase-153--retrieval-orchestrator)
- [Milestone 16 — Demo Mode Validation](#milestone-16--demo-mode-validation)
  - [Phase 16.1 — End-to-End Demo Test](#phase-161--end-to-end-demo-test)
  - [Phase 16.2 — Simulation Mode](#phase-162--simulation-mode)
- [Milestone 17 — Production Hardening](#milestone-17--production-hardening)
  - [Phase 17.1 — Secrets Manager Adapters](#phase-171--secrets-manager-adapters)
  - [Phase 17.2 — HA Microservice & Relay Nodes](#phase-172--ha-microservice--relay-nodes)
- [Milestone M-OBS — Observability & Metrics](#milestone-m-obs--observability--metrics)
  - [Phase OBS.1 — Microservice Metrics](#phase-obs1--microservice-metrics)
  - [Phase OBS.2 — Provider Daemon Metrics](#phase-obs2--provider-daemon-metrics)
  - [Phase OBS.3 — Operational Alerts](#phase-obs3--operational-alerts)
  - [Phase OBS.4 — Prometheus Metric Naming CI Gate](#phase-obs4--prometheus-metric-naming-ci-gate)
- [Milestone 18 — Launch Readiness](#milestone-18--launch-readiness)
  - [Phase 18.1 — Runbooks](#phase-181--runbooks)
  - [Phase 18.2 — Benchmark Scripts](#phase-182--benchmark-scripts)
  - [Phase 18.3 — Security Verification Checklist](#phase-183--security-verification-checklist)
  - [Phase 18.4 — Final CI Gate](#phase-184--final-ci-gate)

### Previous Milestones

- [Milestone 0 — Project Setup & Repository Foundation](./build.md#milestone-0--project-setup--repository-foundation)
  - [Phase 0.1 — Repository Layout](./build.md#phase-01--repository-layout)
  - [Phase 0.2 — Toolchain & Linter Configuration](./build.md#phase-02--toolchain--linter-configuration)
  - [Phase 0.3 — CI Pipeline Skeleton](./build.md#phase-03--ci-pipeline-skeleton)
  - [Phase 0.4 — Development Docker Compose](./build.md#phase-04--development-docker-compose)
- [Milestone 1 — Configuration & NetworkProfile](./build.md#milestone-1--configuration--networkprofile)
  - [Phase 1.1 — NetworkProfile Struct](./build.md#phase-11--networkprofile-struct)
  - [Phase 1.2 — Profile Tests](./build.md#phase-12--profile-tests)
  - [Phase 1.3 — Profile Selection & Guard Rails](./build.md#phase-13--profile-selection--guard-rails)
- [Milestone 2 — Core Cryptography (`internal/crypto`)](./build.md#milestone-2--core-cryptography-internalcrypto)
  - [Phase 2.1 — AES-NI Detection](./build.md#phase-21--aes-ni-detection)
  - [Phase 2.2 — HKDF Key Derivation](./build.md#phase-22--hkdf-key-derivation)
  - [Phase 2.3 — Argon2id Master Secret Derivation](./build.md#phase-23--argon2id-master-secret-derivation)
  - [Phase 2.4 — AONT Cipher](./build.md#phase-24--aont-cipher)
  - [Phase 2.5 — Pointer File AEAD](./build.md#phase-25--pointer-file-aead)
  - [Phase 2.6 — BIP-39 Mnemonic System](./build.md#phase-26--bip-39-mnemonic-system)
  - [Phase 2.7 — Ed25519 Signing Conventions](./build.md#phase-27--ed25519-signing-conventions)
- [Milestone 3 — Erasure Coding Engine (`internal/erasure`)](./build.md#milestone-3--erasure-coding-engine-internalerasure)
  - [Phase 3.1 — Engine Construction](./build.md#phase-31--engine-construction)
  - [Phase 3.2 — Encode & Decode](./build.md#phase-32--encode--decode)
- [Milestone 4 — Database Schema & Migrations](./build.md#milestone-4--database-schema--migrations)
  - [Phase 4.1 — Migration Generator](./build.md#phase-41--migration-generator)
  - [Phase 4.2 — Core Type Definitions](./build.md#phase-42--core-type-definitions)
  - [Phase 4.3 — Core Tables](./build.md#phase-43--core-tables)
  - [Phase 4.4 — Audit, Escrow, and Repair Tables](./build.md#phase-44--audit-escrow-and-repair-tables)
  - [Phase 4.5 — Index Catalogue](./build.md#phase-45--index-catalogue)
  - [Phase 4.6 — Row Security Policies](./build.md#phase-46--row-security-policies)
  - [Phase 4.7 — Materialised Views](./build.md#phase-47--materialised-views)
  - [Phase 4.8 — Migration Checklist Verification](./build.md#phase-48--migration-checklist-verification)
- [Milestone 5 — Provider Storage Engine (`internal/storage`)](./build.md#milestone-5--provider-storage-engine-internalstorage)
  - [Phase 5.1 — ChunkStore Interface & vLog](./build.md#phase-51--chunkstore-interface--vlog)
  - [Phase 5.2 — Storage Tests](./build.md#phase-52--storage-tests)
- [Milestone 6 — P2P Network Layer (`internal/p2p`)](./build.md#milestone-6--p2p-network-layer-internalp2p)
  - [Phase 6.1 — libp2p Host](./build.md#phase-61--libp2p-host)
  - [Phase 6.2 — Kademlia DHT with Custom HMAC Validator](./build.md#phase-62--kademlia-dht-with-custom-hmac-validator)
  - [Phase 6.3 — Heartbeat Goroutine](./build.md#phase-63--heartbeat-goroutine)

---

## Build Dependency Graph

```tree
M0 (Setup)
 └─ M1 (Config / NetworkProfile)
     ├─ M2 (Cryptography)
     │   └─ M3 (Erasure Coding)
     ├─ M4 (Database Schema)
     └─ M5 (Storage Engine)
         └─ M6 (P2P Network Layer)
             ├─ M7 (Audit System)
             ├─ M8 (Scoring System)
             │   ├─ M9 (Repair System)
             │   └─ M10 (Payment System)
             ├─ M11 (REST API Layer)
             └─ M12 (Coordination Microservice)
                 ├─ M-OBS (Observability & Metrics)
                 └─ M13 (Provider Daemon Core)
                     └─ M14 (Vetting & Synthetic Chunks)
                         └─ M15 (Client SDK)
                             └─ M16 (Demo Mode Validation)
                                 ├─ M17 (Production Hardening)
                                    └─ M18 (Launch Readiness)
```

The import constraint DAG (IC §9) enforces that `crypto` and `erasure` have zero internal
dependencies; all data-layer packages flow upward only to the microservice entrypoint. M-OBS depends on M12 and M13 for metric name definitions; it does not export to any `internal/` package.

For a Milestone review use

```bash
Milestone [N] — [Title]

Review the milestone's deliverable holistically against what is currently on disk in Vyomanaut_V2.

Flag:
- Cross-session dependencies within this milestone (which sessions must complete before others)
- Interface mismatches: does what's already built diverge from what this milestone assumes?
- Import constraint risks across the sessions
- Anything the session sequence doesn't address that could block compilation later

Do not write any code. Analysis only.
```

---

## Milestone 7 — Audit System (`internal/audit`)

**Deliverable:** Challenge generation, response validation, two-phase crash-safe receipt write, cluster secret cache, and JIT detection.

**Package import constraints (IC §9):** `internal/audit` must **NOT** import `internal/scoring`, `internal/repair`, or `internal/payment` — score updates and repair triggers are the caller's responsibility; the microservice entrypoint (Milestone 12) orchestrates those after the audit result is written. `internal/audit` **MAY** import `internal/config` (for `NetworkProfile.RequireSecretsManager`, consumed in Phase 7.4) and `internal/crypto` (for `VerifyBytes`, consumed in Phase 7.2, per IC §3.2's canonical verification procedure).

**Reference:** IC §5.5 (full package contract), IC §3.2 (Ed25519 signing conventions), IC §4.2 (audit challenge protocol wire format — this is what the microservice sends; the provider-side stream handler is **Milestone 13, Phase 13.3**, not this milestone), DM §4.7 (`audit_receipts` schema), DM §3 Invariants 1 and 5, IC §8 (secrets manager contract), IC §6 (`audit_receipts` row-level DML contract), FR-037–FR-041 (audit system functional requirements), MVP §8.2 (file inventory)

> **Corrected:** the original top-level reference read *"the provider-side handler is in M8 provider daemon."* Milestone 8 is the Scoring System and contains no `cmd/provider` code. The handler for `/vyomanaut/audit-challenge/1.0.0` is built in **Milestone 13 → Phase 13.3 → Session 13.3.1**.

**Milestone review notes** (cross-session dependencies internal to M7):

- Session 7.2.1 (`ValidateResponse`, secret-version check) has a soft dependency on Session 7.4.1 (`ClusterSecretCache.IsVersionValid`) — see the flagged gap in Phase 7.2.
- Session 7.3.1 declares the `AuditResult` type; it also closes the `.golangci.yml` `exhaustive`-linter placeholder that Session 0.2.1 left as `# M7: internal/audit.AuditResult`.
- Session 7.4.1 must exist and pass before Session 7.6.1 (the consolidated test session) can test rotation-overlap behavior end-to-end.

---

### Phase 7.1 — Challenge Nonce Generation

**Reference:** IC §5.5 (`ChallengeNonce`), DM §3 Invariant 5 (33 bytes always), FR-038 (nonce composition and unpredictability), IC §11 (forbidden: `challenge_nonce` as a 32-byte field)

#### Session 7.1.1 — Implement `ChallengeNonce()`; initialise `internal/audit/errors.go`

**PRECONDITIONS:**

- `internal/audit/doc.go` exists (stub from Session 0.1.3)
- No external dependency required — this session uses only `crypto/hmac` and `crypto/sha256` from the standard library

**TASK:**

1. Create `internal/audit/errors.go`. This is the single accumulating home for every sentinel error this package exports across all of Milestone 7 (mirrors how `internal/crypto/errors.go` accumulates across Milestone 2) — do not declare sentinel errors in any other file. Seed it now with the three sentinels IC §5.5 declares for this package:

```go
package audit

import "errors"

var (
 // ErrInvalidSignature is returned by ValidateResponse when the provider's
 // Ed25519 signature does not verify (IC §5.5, IC §3.2).
 ErrInvalidSignature = errors.New("audit: invalid Ed25519 signature")

 // ErrNonceLength is returned by ValidateResponse when challengeNonce is not
 // exactly 33 bytes (DM §3 Invariant 5, FR-038).
 ErrNonceLength = errors.New("audit: challenge nonce must be exactly 33 bytes")

 // ErrReceiptAlreadyFinal is returned by WriteReceiptPhase2 when the target row
 // already has a non-NULL audit_result — idempotent retry (IC §5.5, ADR-015).
 ErrReceiptAlreadyFinal = errors.New("audit: receipt already has a terminal result")
)
```

   Sessions 7.4.1 and 7.6.1 append further sentinels to this same file (`ErrSecretExpired` and friends). Do not create a second errors file.

1. Create `internal/audit/challenge.go`:

```go
// ChallengeNonce generates a 33-byte versioned challenge nonce (IC §5.5, FR-038).
//   nonce = version_byte || HMAC-SHA256(serverSecretVN, chunkID || serverTsMs)
//
// Pre-conditions (panic in debug builds if violated):
//   len(serverSecretVN) == 32
//   versionByte must match the version of serverSecretVN
// Post-conditions:
//   returns a 33-byte nonce; nonce[0] == versionByte
// Goroutine-safe: yes (pure function).
func ChallengeNonce(serverSecretVN []byte, versionByte uint8, chunkID [32]byte, serverTsMs int64) [33]byte
```

   Signing input for the HMAC: `chunkID (32 bytes) || big-endian int64(serverTsMs) (8 bytes)`. The return type is a fixed `[33]byte` array — not a slice — precisely so that any later code path attempting to treat it as `[32]byte` fails to compile (IC §5.5, IC §11).

1. Add a supplementary grep-fail check to `scripts/ci/grep_checks.sh` (created in Session 0.2.2): flag any call site that assigns `ChallengeNonce(...)`'s result to a `[32]byte`-typed variable. This is additional hardening on top of — not a replacement for — CI check 8 (`challenge_nonce BYTEA(32)` in SQL), since this one catches Go-level truncation rather than schema drift:

```bash
# Supplementary check (not one of the 16 numbered CI gates — add alongside them)
check "NONCE_TRUNCATION_GO" \
  "\[32\]byte.*ChallengeNonce\(|ChallengeNonce\([^)]*\).*\[32\]byte" \
  "internal"
```

1. **Defer, do not skip:** the `.golangci.yml` `exhaustive` linter placeholder `# M7: internal/audit.AuditResult` (set up in Session 0.2.1) is closed in **Session 7.3.1**, where `AuditResult` is actually declared. Note it here so it isn't forgotten.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/audit/errors.go    && echo PASS || echo FAIL
  $ test -f internal/audit/challenge.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/audit/
  EXPECT: exit 0

FUNCTION_SIGNATURE:
  $ grep -c "^func ChallengeNonce(serverSecretVN \[\]byte, versionByte uint8, chunkID \[32\]byte, serverTsMs int64) \[33\]byte" \
      internal/audit/challenge.go
  EXPECT: 1

RETURN_TYPE_IS_FIXED_ARRAY:
  # Must return [33]byte, never []byte — this is the compile-time truncation guard
  $ grep -c "\[33\]byte {" internal/audit/challenge.go
  EXPECT: >= 1
  $ grep -n "ChallengeNonce" internal/audit/challenge.go | grep -c ") \[\]byte"
  EXPECT: 0

SENTINEL_ERRORS_SEEDED:
  $ grep -c "ErrInvalidSignature\|ErrNonceLength\|ErrReceiptAlreadyFinal" internal/audit/errors.go
  EXPECT: 3

HMAC_CONSTRUCTION:
  $ grep -c "hmac\.New\|sha256\.New" internal/audit/challenge.go
  EXPECT: >= 1

VERSION_BYTE_PREFIX:
  $ grep -c "\[0\] = versionByte" internal/audit/challenge.go
  EXPECT: 1

GREP_CHECK_ADDED:
  $ grep -c "NONCE_TRUNCATION_GO" scripts/ci/grep_checks.sh
  EXPECT: 1

UNIT_TESTS:
  $ go test -v -run TestChallengeNonce ./internal/audit/
  EXPECT: exit 0; tests include:
    TestChallengeNonceLength         (always exactly 33 bytes for any input)
    TestChallengeNonceVersionByte    (nonce[0] == versionByte, tried for 0, 1, 255)
    TestChallengeNonceDeterministic  (identical inputs → identical nonce)
    TestChallengeNonceVariesWithTs   (different serverTsMs → different nonce)
    TestChallengeNonceVariesWithChunk (different chunkID → different nonce)

NEGATIVE_CHECKS:
  # IC §9: must not import the three prohibited packages
  $ grep -n "Vyomanaut_V2/internal/scoring\|Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" \
      internal/audit/challenge.go internal/audit/errors.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/audit/
  EXPECT: exit 0; zero output
```

---

### Phase 7.2 — Response Validation

**Reference:** IC §5.5 (`ValidateResponse` and its explicit limitation note), IC §4.2 (Frame 2 signing-input formula), IC §3.2 (canonical verification procedure), NFR-015 (unforgeability requirement)

> **Flagged — signature gap between IC §5.5 and IC §4.2.** IC §5.5 declares
> `func ValidateResponse(challengeNonce [33]byte, responseHash [32]byte, providerSig [64]byte, providerPubKey [32]byte) error`.
> But IC §4.2 states the actual signing input the provider signed is
> `SHA-256(response_hash ‖ challenge_nonce ‖ server_challenge_ts_ms ‖ provider_id)` — four components. The IC §5.5 signature has no parameter for `server_challenge_ts_ms` or `provider_id`, so as declared it cannot reconstruct the input it needs to verify the signature against. Separately, IC §5.5's own item 2 ("`challengeNonce[0]` identifies a currently-valid secret version") requires access to `ClusterSecretCache` (Phase 7.4), which this signature also doesn't carry.
>
> **Resolution used below** (needs reconciling with `interface-contracts.md` in a follow-up PR): extend the signature with `serverChallengeTsMs int64` and `providerID [16]byte`, matching IC §4.2's wire fields exactly. Treat the secret-version check (item 2) as the **caller's** responsibility via `ClusterSecretCache.IsVersionValid`, performed by the microservice audit-dispatch loop (Milestone 12, Session 12.1.2) around the call to `ValidateResponse` — not inside it.

#### Session 7.2.1 — Implement `ValidateResponse()`

**PRECONDITIONS:**

- Session 7.1.1 complete (`errors.go` exists with `ErrInvalidSignature`, `ErrNonceLength`)
- `internal/crypto` complete (Milestone 2), specifically `VerifyBytes` (Session 2.7.1)
- Phase 7.4 need not be complete to compile this session, but its `ClusterSecretCache.IsVersionValid` is required to exercise item 2 of the contract in the consolidated test session (7.6.1) — see the milestone review note above

**TASK:**

In `internal/audit/validate.go`, implement:

```go
// ValidateResponse verifies the structural and cryptographic properties of a
// provider's audit response that the microservice CAN verify without holding
// chunk_data (IC §5.5, IC §4.2):
//   1. len(challengeNonce) == 33                                -> ErrNonceLength
//   2. providerSig is a valid Ed25519 signature by providerPubKey
//      over SHA-256(responseHash || challengeNonce || serverChallengeTsMs || providerID)
//      (IC §4.2 Frame 2, IC §3.2)                                -> ErrInvalidSignature
//
// NOTE — secret-version currency (IC §5.5 item 2, "challengeNonce[0] identifies a
// currently-valid secret version") is NOT checked here. This function has no
// reference to ClusterSecretCache. The caller MUST additionally call
// ClusterSecretCache.IsVersionValid(challengeNonce[0]) (Phase 7.4) — a nil error
// from this function alone is not sufficient to accept the response.
//
// LIMITATION (IC §5.5, NFR-015): the microservice cannot verify that
// responseHash == SHA-256(chunkData || challengeNonce) because it never holds
// chunkData. Correctness depends on economic deterrence and JIT detection
// (ADR-014 Defence 3). This is a stated design property, not a gap to close.
//
// Goroutine-safe: yes.
func ValidateResponse(
 challengeNonce [33]byte,
 responseHash [32]byte,
 serverChallengeTsMs int64,
 providerID [16]byte, // UUID bytes, matching the IC §4.1 capability-token convention
 providerSig [64]byte,
 providerPubKey [32]byte,
) error
```

Build the signing input as a fixed-layout byte sequence per IC §3.2 (never JSON): `SHA-256(responseHash[:] ‖ challengeNonce[:] ‖ bigEndianInt64(serverChallengeTsMs) ‖ providerID[:])`, then call `crypto.VerifyBytes(providerPubKey, signingInput, providerSig)`.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/audit/validate.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/audit/
  EXPECT: exit 0

FUNCTION_SIGNATURE:
  $ grep -c "^func ValidateResponse(" internal/audit/validate.go
  EXPECT: 1

  # Confirms the corrected 6-parameter signature (adds serverChallengeTsMs, providerID)
  $ grep -c "serverChallengeTsMs int64" internal/audit/validate.go
  EXPECT: 1
  $ grep -c "providerID \[16\]byte" internal/audit/validate.go
  EXPECT: 1

NONCE_LENGTH_CHECK_PRESENT:
  $ grep -c "ErrNonceLength" internal/audit/validate.go
  EXPECT: >= 1

USES_CRYPTO_VERIFYBYTES:
  # Must delegate to internal/crypto, not re-implement Ed25519 verification inline
  $ grep -c "crypto\.VerifyBytes\|VerifyBytes(" internal/audit/validate.go
  EXPECT: >= 1

LIMITATION_COMMENT_PRESENT:
  $ grep -c "LIMITATION" internal/audit/validate.go
  EXPECT: 1

VERSION_CHECK_NOTE_PRESENT:
  # Confirms the caller-responsibility note is documented, not silently dropped
  $ grep -c "ClusterSecretCache\|IsVersionValid" internal/audit/validate.go
  EXPECT: >= 1

NO_JSON_IN_SIGNING_INPUT:
  $ grep -n "json\.Marshal" internal/audit/validate.go \
      && echo "FAIL: signing input must be fixed-layout bytes, not JSON (IC §3.2)" || echo "PASS"

UNIT_TESTS:
  $ go test -v -run TestValidateResponse ./internal/audit/
  EXPECT: exit 0; tests include:
    TestValidateResponseAccepts         (correct nonce + correct signature -> nil)
    TestValidateResponseRejectsBadNonceLength (32-byte nonce -> ErrNonceLength)
    TestValidateResponseRejectsBadSignature   (tampered signature -> ErrInvalidSignature)
    TestValidateResponseRejectsWrongPubKey    (signature from a different key -> ErrInvalidSignature)
    TestValidateResponseSigningInputIsFixedLayout (changing field order changes the digest)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/scoring\|Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" \
      internal/audit/validate.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/audit/
  EXPECT: exit 0; zero output
```

---

### Phase 7.3 — Two-Phase Receipt Write

**Reference:** IC §5.5 (`WriteReceiptPhase1`, `WriteReceiptPhase2`, `AuditResult`, `ErrReceiptAlreadyFinal`), DM §4.7 (`audit_receipts` schema, two-phase write semantics), DM §6 (row security policy enforcing the Phase-2 update), DM §3 Invariant 1 (append-only audit log), FR-039 (schema + two-phase requirement), ADR-015

> **Flagged — undefined type.** IC §5.5 declares `WriteReceiptPhase1(ctx, db, fields ReceiptFields)` but `ReceiptFields` is never defined anywhere in `interface-contracts.md`. A concrete definition is proposed below, derived directly from the `audit_receipts` column list in DM §4.7 (excluding DB-generated columns `receipt_id`, `schema_version` and Phase-2-only columns `audit_result`, `service_sig`, `service_countersign_ts`).

#### Session 7.3.1 — Implement `WriteReceiptPhase1()`; declare `AuditResult`; close the `.golangci.yml` placeholder

**PRECONDITIONS:**

- Session 7.1.1 complete
- A UUIDv7-capable UUID library is added to `go.mod` (e.g. `github.com/google/uuid` v1.6.0+, which provides `uuid.NewV7()`) — `receipt_id` is generated at the application layer per IC §5.5, not by Postgres's `gen_random_uuid()` default (that default exists in the schema for defense-in-depth only; DM §4.7)

**TASK:**

1. In `internal/audit/receipt.go`, declare the shared type and the Phase-1 function:

```go
// AuditResult is the terminal state of an audit receipt (IC §5.5).
type AuditResult int

const (
 AuditPass AuditResult = iota
 AuditFail
 AuditTimeout
)

// ReceiptFields carries every audit_receipts column that Phase 1 (the initial
// INSERT) populates. Derived from DM §4.7 — excludes receipt_id and
// schema_version (DB-generated) and audit_result/service_sig/
// service_countersign_ts (Phase-2-only). PROPOSED definition — IC §5.5 uses
// this type name without defining its fields; reconcile with interface-contracts.md.
type ReceiptFields struct {
 ChunkID           [32]byte
 FileID            *uuid.UUID // nil for a synthetic vetting-chunk audit (DM §8.20)
 ProviderID        uuid.UUID
 ChallengeNonce    [33]byte
 ServerChallengeTs time.Time
 AddressWasStale   bool // DM §4.7 — true if dispatched via DHT fallback
}

// WriteReceiptPhase1 performs the crash-safe Phase 1 INSERT to audit_receipts.
// Inserts a PENDING row (audit_result = NULL) with fields.ProviderSig left
// unset — the provider's signature is not yet known before dispatch completes;
// see DM §8.9 for the PENDING-state field semantics. (ADR-015)
//
// Pre-conditions:
//   - fields.ChunkID, fields.ProviderID, fields.ChallengeNonce, fields.ServerChallengeTs
//     are all populated
//   - the database connection is open
// Post-conditions (on nil error):
//   - a row with audit_result = NULL exists in audit_receipts
//   - the row is durable before this function returns: a plain `INSERT ... COMMIT`
//     under PostgreSQL's default synchronous_commit = on already guarantees the
//     WAL is flushed before COMMIT returns — no special flush call is needed.
//     (There is no pg_wal_flush() function in PostgreSQL; do not reference one.)
// Error semantics: database errors are returned; caller must not proceed to
//   Phase 2 if Phase 1 fails.
// Goroutine-safe: yes (uses connection pool).
func WriteReceiptPhase1(ctx context.Context, db *sql.DB, fields ReceiptFields) (receiptID uuid.UUID, err error)
```

1. **Close the `.golangci.yml` placeholder from Session 0.2.1:** in the `exhaustive` linter settings comment block, replace `# M7: internal/audit.AuditResult` with an active list entry so any future `switch result AuditResult { ... }` (e.g. in Milestone 12's dispatch loop) is checked for exhaustiveness.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/audit/receipt.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/audit/
  EXPECT: exit 0

AUDITRESULT_TYPE:
  $ grep -c "^type AuditResult int" internal/audit/receipt.go
  EXPECT: 1
  $ grep -c "AuditPass\|AuditFail\|AuditTimeout" internal/audit/receipt.go
  EXPECT: >= 3

RECEIPTFIELDS_STRUCT:
  $ grep -c "^type ReceiptFields struct" internal/audit/receipt.go
  EXPECT: 1
  $ grep -c "ChunkID\|ProviderID\|ChallengeNonce\|ServerChallengeTs\|AddressWasStale" internal/audit/receipt.go
  EXPECT: >= 5

FUNCTION_SIGNATURE:
  $ grep -c "^func WriteReceiptPhase1(ctx context.Context, db \*sql.DB, fields ReceiptFields) (receiptID uuid.UUID, err error)" \
      internal/audit/receipt.go
  EXPECT: 1

NO_FICTIONAL_PG_FUNCTION:
  # pg_wal_flush() does not exist in PostgreSQL — must not appear anywhere
  $ grep -rn "pg_wal_flush" internal/audit/ migrations/ 2>/dev/null \
      && echo "FAIL: pg_wal_flush() is not a real PostgreSQL function" || echo "PASS"

PENDING_ROW_SEMANTICS:
  # audit_result must be left NULL on INSERT — never defaulted (DM §9 checklist)
  $ grep -c "audit_result.*NULL\|audit_result = NULL" internal/audit/receipt.go
  EXPECT: >= 1

GOLANGCI_PLACEHOLDER_CLOSED:
  $ grep -c "# M7: internal/audit.AuditResult" .golangci.yml
  EXPECT: 0
  $ grep -c "internal/audit\.AuditResult" .golangci.yml
  EXPECT: 1

UNIT_TESTS:
  $ go test -v -run TestWriteReceiptPhase1 ./internal/audit/
  EXPECT: exit 0; tests include:
    TestWriteReceiptPhase1InsertsPending    (audit_result IS NULL after insert)
    TestWriteReceiptPhase1ReturnsUUIDv7     (receipt_id is a valid, time-ordered UUIDv7)
    TestWriteReceiptPhase1RejectsBadFields  (missing required field -> error, no row created)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/scoring\|Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" \
      internal/audit/receipt.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"
```

#### Session 7.3.2 — Implement `WriteReceiptPhase2()`

**PRECONDITIONS:**

- Session 7.3.1 complete (`AuditResult` type and `WriteReceiptPhase1` exist in `receipt.go`)
- Migration from Session 4.6.1 applied to the test database (the `audit_receipts_phase2_update` row security policy must exist for the `WHERE`-clause behavior to be testable end-to-end)

**TASK:**

In `internal/audit/receipt.go`, implement:

```go
// WriteReceiptPhase2 performs the crash-safe Phase 2 UPDATE on audit_receipts.
// Sets audit_result, service_sig, and service_countersign_ts atomically.
// (ADR-015, DM §3 Invariant 1, DM §6 row security policy)
//
// Pre-conditions:
//   - receiptID identifies an existing row
//   - result is AuditPass, AuditFail, or AuditTimeout
//   - len(serviceSig) == 64
// Post-conditions (on nil error):
//   - the row is updated; audit_result is no longer NULL
// Error semantics:
//   - ErrReceiptAlreadyFinal: the row already has a non-NULL audit_result.
//     Idempotent — caller should treat this as success (IC §5.5 idempotent-retry
//     protocol) and return the existing service_sig rather than surfacing an error.
//   - other database errors: returned to caller.
// Goroutine-safe: yes.
func WriteReceiptPhase2(ctx context.Context, db *sql.DB, receiptID uuid.UUID, result AuditResult, serviceSig [64]byte, serviceTS time.Time) error
```

The `WHERE` clause — `WHERE receipt_id = $1 AND audit_result IS NULL AND abandoned_at IS NULL` — must match the `USING` clause of the `audit_receipts_phase2_update` policy from DM §6 exactly; a mismatch here would make legitimate Phase-2 updates silently fail under row-level security rather than surface a clear application error.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/audit/
  EXPECT: exit 0

FUNCTION_SIGNATURE:
  $ grep -c "^func WriteReceiptPhase2(ctx context.Context, db \*sql.DB, receiptID uuid.UUID, result AuditResult, serviceSig \[64\]byte, serviceTS time.Time) error" \
      internal/audit/receipt.go
  EXPECT: 1

WHERE_CLAUSE_MATCHES_RSP:
  $ grep -c "audit_result IS NULL AND abandoned_at IS NULL" internal/audit/receipt.go
  EXPECT: 1

IDEMPOTENT_RETRY_HANDLING:
  $ grep -c "ErrReceiptAlreadyFinal" internal/audit/receipt.go
  EXPECT: >= 2   # sentinel declaration (7.1.1) + return site (7.3.2)

INTEGRATION_TEST_AGAINST_LIVE_RSP (requires CI Postgres with M4 migration applied):
  $ go test -v -run TestWriteReceiptPhase2 ./internal/audit/
  EXPECT: exit 0; tests include:
    TestWriteReceiptPhase2PromotesToTerminal   (PENDING -> PASS/FAIL/TIMEOUT succeeds)
    TestWriteReceiptPhase2Idempotent           (second call with same receiptID -> ErrReceiptAlreadyFinal, no second row)
    TestWriteReceiptPhase2RejectsAbandonedRow  (abandoned_at IS NOT NULL -> update affects 0 rows -> error, not silent success)
    TestWriteReceiptPhase2OnlyTouchesAllowedColumns (all other columns unchanged after update)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/scoring\|Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" \
      internal/audit/receipt.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

  # Invariant 1: no DELETE path may exist anywhere near this file
  $ grep -n "DELETE FROM audit_receipts" internal/audit/receipt.go \
      && echo "FAIL: audit_receipts is append-only (DM §3 Invariant 1)" || echo "PASS"

VET:
  $ go vet ./internal/audit/
  EXPECT: exit 0; zero output
```

**Three real findings from live-RLS testing:**

1. DM §6 defines no SELECT policy for vyomanaut_app on audit_receipts (only
INSERT and two UPDATE policies). Confirmed live: vyomanaut_app cannot
SELECT a row back via plain SELECT, even one it just inserted itself —
RLS returns zero rows silently. Production functions are correctly
designed around this (neither ever needs to SELECT), but the tests
needed a second, privileged connection (openVerifyDB) purely to assert
results, since the actor connection can't read anything back.
2. Independently, and more seriously: PostgreSQL's RLS requires an
applicable SELECT policy for UPDATE to even locate candidate rows — an
UPDATE-only policy's own USING clause is NOT sufficient by itself. This
means WriteReceiptPhase2, run as a correctly-restricted vyomanaut_app
against DM §6 exactly as specified, affects ZERO rows on every call,
for every result — the entire Phase 2 write path is non-functional as
currently specified. I verified this with an isolated raw-SQL
reproduction outside any Go code before tracing it back here. This is a
migrations/-owned fix (needs a SELECT policy added, which needs the
3-reviewer CODEOWNERS process) — I patched it locally, sandbox-only, to
finish verifying the rest of the logic, and did not touch migrations/.
receipt_test.go will correctly FAIL (not skip) against an unpatched
live database — that failure is the intended signal.
3. A third, separate gap, found only after #2 was locally patched enough
to test further: audit_receipts_response_consistency requires
response_hash/provider_sig for PASS/FAIL, and neither
WriteReceiptPhase1 nor WriteReceiptPhase2's given signatures have a
parameter for either. Only AuditTimeout is reachable through these two
functions as specified. Documented in receipt.go; tests rewritten to
assert this as intended behavior (TestWriteReceiptPhase2PromotesToTerminal
now has explicit "PASS/FAIL fails the check constraint" subtests) rather
than silently avoiding AuditPass in the other three tests (which I
switched to AuditTimeout).
4. Added github.com/lib/pq (not requested by name, only google/uuid was) —
without any driver, the live-DB tests could only ever skip, never
actually run, which felt like it would undercut the whole point of the
four named INTEGRATION_TEST_AGAINST_LIVE_RSP tests. Flagging this as a
deliberate scope addition, same as internal/crypto's depguard entry last
session.

TODOs deferred (all pre-existing gaps surfaced, none newly introduced):

- DM §6: add a SELECT policy for vyomanaut_app (and vyomanaut_gc) on
audit_receipts — migrations/ change, needs 3 CODEOWNERS reviewers.
- WriteReceiptPhase2's signature likely needs responseHash [32]byte and
providerSig [64]byte parameters (same pattern as Session 7.2.1's
ValidateResponse fix) before PASS/FAIL can ever be written.
- ci.yml's postgres service uses POSTGRES_USER: vyomanaut_app, which the
official image bootstraps as a superuser; migrations/generator.go's role
creation is CREATE ROLE ... IF NOT EXISTS, so it never downgrades that
superuser bit. Net effect: RLS on audit_receipts is not actually
exercised in CI today, regardless of #1 above. Out of scope here; noted
in receipt_test.go and flagging it loudly since it's a real gap.
- interface-contracts.md still doesn't define ReceiptFields — carried over
from this session's own precondition note.

---

### Phase 7.4 — Cluster Secret Cache

**Reference:** IC §8 (full secrets manager contract — `SecretsManagerClient` interface, path convention `/vyomanaut/audit-secret/v{N}`, 24-hour rotation overlap, fail-closed startup rule, `ErrSecretNotFound`/`ErrSecretManagerUnavailable`/`ErrSecretExpired`), MVP §8.2 (`secret.go`, `secrets_iface.go`), NFR-018 (key management requirement), MVP §5.4 (`RequireSecretsManager` toggle)

> **Corrected reference.** The original Phase 7.4 header cited "IC §5.5 (`SecretsManagerClient` interface, `ErrSecretExpired`)." Neither symbol is declared in IC §5.5 (the `internal/audit` package contract) — both live in **IC §8**. The session body already (correctly) said "from IC §8" for the interface; only the Phase-level header line was wrong. Fixed above.

> **Flagged — no method set specified.** `mvp.md §8.2` names `ClusterSecretCache` but no document gives it a method set. The proposal below is shaped to satisfy: (a) Phase 7.2's need for `IsVersionValid`, (b) IC §8's fail-closed-at-startup and TTL-expiry-during-operation requirements, and (c) the 24-hour rotation overlap (both `v{N}` and `v{N+1}` must be queryable simultaneously).

#### Session 7.4.1 — Implement `SecretsManagerClient` interface and `ClusterSecretCache`

**PRECONDITIONS:**

- Session 7.1.1 complete (`errors.go` exists to receive the additional local sentinels this session adds)
- `internal/config` complete (Milestone 1) — `NetworkProfile.RequireSecretsManager` is read here
- M1 Session 1.3.2's guard rail (`PROD_MODE_ENV_SECRET`) already exists and is what makes it safe for this session to read `VYOMANAUT_CLUSTER_MASTER_SEED` in demo mode — do not duplicate that guard here

**TASK:**

1. Create `internal/audit/secrets_iface.go` with a package-local interface matching IC §8's shape (this package cannot import the not-yet-built `internal/secrets`, created in Milestone 17; the adapters built there satisfy this interface implicitly via Go's structural typing — no import needed either direction):

```go
package audit

import "context"

// SecretsManagerClient abstracts over Vault, AWS SSM, and GCP Secret Manager
// (IC §8). Concrete adapters are implemented in internal/secrets (Milestone 17)
// and satisfy this interface structurally — internal/audit never imports
// internal/secrets.
type SecretsManagerClient interface {
 // GetSecret retrieves the decoded (not base64) secret at path, e.g.
 // "/vyomanaut/audit-secret/v3" (IC §8 path convention).
 GetSecret(ctx context.Context, path string) ([]byte, error)
}
```

1. Append to `internal/audit/errors.go` (do not create a second errors file):

```go
var (
 // ErrSecretNotFound mirrors IC §8: the requested path does not exist.
 ErrSecretNotFound = errors.New("audit: secret path not found")

 // ErrSecretManagerUnavailable mirrors IC §8: the secrets manager is unreachable.
 ErrSecretManagerUnavailable = errors.New("audit: secrets manager unreachable")

 // ErrSecretExpired is returned once the 5-minute cached-secret TTL has elapsed
 // and the secrets manager remains unreachable (IC §8). The caller must back
 // off and must not issue further challenges while this is returned.
 ErrSecretExpired = errors.New("audit: cached secret TTL expired and manager unavailable")
)
```

1. Create `internal/audit/secret.go` with `ClusterSecretCache`:

```go
// ClusterSecretCache holds the cluster audit secret(s) with a 5-minute TTL
// (IC §8). During the 24-hour rotation overlap window it holds both
// server_secret_vN and server_secret_v{N+1} simultaneously.
type ClusterSecretCache struct {
 client SecretsManagerClient
 ttl    time.Duration // always 5 minutes (IC §8) — not profile-variable
 // unexported: mutex-guarded map[versionByte][]byte, loadedAt timestamp
}

// NewClusterSecretCache constructs a cache. In demo mode (profile.RequireSecretsManager
// == false), client should be an env-var-backed adapter that reads
// VYOMANAUT_CLUSTER_MASTER_SEED (IC §8) instead of a real Vault/SSM/GCP client.
// The PROD_MODE_ENV_SECRET guard (M1 Session 1.3.2) is what prevents this env
// var from being used in production — this constructor does not re-check that.
func NewClusterSecretCache(client SecretsManagerClient) *ClusterSecretCache

// Load performs the initial fail-closed fetch. Must be called once at
// microservice startup, before any challenge is issued. On error, the caller
// (Milestone 12 Session 12.1.1) must refuse to start the replica (IC §8:
// "fail-closed" — better zero challenges than unvalidatable ones).
func (c *ClusterSecretCache) Load(ctx context.Context) error

// CurrentSecret returns the highest currently-valid server_secret and its
// version byte, for use when issuing new challenges (ChallengeNonce). Returns
// ErrSecretExpired if the cache TTL has elapsed and the manager is unreachable.
func (c *ClusterSecretCache) CurrentSecret() (secret []byte, versionByte uint8, err error)

// IsVersionValid reports whether versionByte corresponds to a
// currently-accepted secret version — vN or vN+1 during the 24-hour rotation
// overlap window (IC §8). Session 7.2.1's ValidateResponse caller uses this.
func (c *ClusterSecretCache) IsVersionValid(versionByte uint8) bool

// SecretForVersion returns the raw secret for a specific (possibly older, but
// still-valid) version byte — needed to validate a challenge nonce issued
// under a version other than the current one.
func (c *ClusterSecretCache) SecretForVersion(versionByte uint8) (secret []byte, err error)
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/audit/secrets_iface.go && echo PASS || echo FAIL
  $ test -f internal/audit/secret.go        && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/audit/
  EXPECT: exit 0

INTERFACE_SHAPE_MATCHES_IC8:
  $ grep -c "GetSecret(ctx context.Context, path string) (\[\]byte, error)" internal/audit/secrets_iface.go
  EXPECT: 1

NEW_SENTINELS_APPENDED_NOT_DUPLICATED:
  $ grep -c "ErrSecretNotFound\|ErrSecretManagerUnavailable\|ErrSecretExpired" internal/audit/errors.go
  EXPECT: 3
  $ find internal/audit -name "*errors*.go" | wc -l
  EXPECT: 1   # exactly one errors file for the whole package

CACHE_METHODS_PRESENT:
  $ grep -c "func (c \*ClusterSecretCache) Load\|func (c \*ClusterSecretCache) CurrentSecret\|func (c \*ClusterSecretCache) IsVersionValid\|func (c \*ClusterSecretCache) SecretForVersion" \
      internal/audit/secret.go
  EXPECT: 4

TTL_IS_FIVE_MINUTES:
  $ grep -c "5 \* time.Minute\|5\*time.Minute" internal/audit/secret.go
  EXPECT: >= 1

DEMO_MODE_ENV_VAR_PATH:
  $ grep -c "VYOMANAUT_CLUSTER_MASTER_SEED" internal/audit/secret.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestClusterSecretCache ./internal/audit/
  EXPECT: exit 0; tests include:
    TestClusterSecretCacheFailsClosedOnLoadError      (Load returns error when client unreachable; no panic)
    TestClusterSecretCacheServesFromCacheDuringOutage  (client goes unreachable mid-TTL -> CurrentSecret still succeeds)
    TestClusterSecretCacheExpiresAfterTTL              (TTL elapsed + client unreachable -> ErrSecretExpired)
    TestClusterSecretCacheRotationOverlap               (both vN and vN+1 loaded -> IsVersionValid true for both)
    TestClusterSecretCacheRejectsRetiredVersion         (version outside the 24h overlap -> IsVersionValid false)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/scoring\|Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" \
      internal/audit/secret*.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

  # PROD_MODE_ENV_SECRET is M1's job — this session must not re-implement that guard
  $ grep -c "profile.Mode == \"prod\"" internal/audit/secret.go
  EXPECT: 0

VET:
  $ go vet ./internal/audit/
  EXPECT: exit 0; zero output
```

---

### Phase 7.5 — JIT Detection

**Reference:** ARCH §14 (audit receipt schema table, `jit_flag` row — authoritative source of the `×0.3` factor), ARCH §20 §Outsourcing (adversarial-defence rationale for the threshold), DM §4.2 (`p95_throughput_kbps` NULL semantics), DM §4.7 (`jit_flag` column), MVP §8.2 (`jit.go`)

> **Corrected — internal document inconsistency.** IC §4.2's own aside describes "the JIT detection deadline `(256 / p95_throughput_kbps) × 1.5` ... the floor below which a response is flagged as anomalously fast." That `×1.5` is wrong: `×1.5` is the **audit response deadline / timeout** factor (FR-028, NFR-007, DM §4.2's `deadline_ms` comment), not the JIT floor. The JIT anomaly threshold is `×0.3`, per architecture.md §14's audit-receipt-schema table (`jit_flag` row) and §20's Outsourcing defence description — and that `×0.3` figure is what the original session task correctly used. The reference below cites the unambiguous ARCH sections instead of the IC §4.2 aside so this doesn't get miscopied again.

#### Session 7.5.1 — Implement JIT threshold evaluation

**PRECONDITIONS:**

- Session 7.1.1 complete (package skeleton, `errors.go` exists)
- No dependency on Milestone 8 (`internal/scoring`) — this package must not import it (IC §9); `p95_throughput_kbps` is read directly from the `providers` table via SQL, not via a scoring-package call

**TASK:**

Create `internal/audit/jit.go`:

```go
// EvaluateJIT computes whether an audit response is anomalously fast — a
// signal of just-in-time retrieval from a co-located source rather than a
// genuine local-disk read (ARCH §20 §Outsourcing, ARCH §14 jit_flag row).
//
//   jit_flag = responseLatencyMs < (chunkSizeKB / p95ThroughputKbps) * 0.3
//
// chunkSizeKB is always 256, since ShardSize = 262,144 bytes = 256 KB is a
// compile-time constant in both demo and production modes (DM §3 Invariant 7).
//
// p95ThroughputKbps may be nil for a new provider (DM §4.2: NULL until vetting
// audits accumulate samples). When nil, this function returns false (no flag) —
// deliberately, and asymmetrically from how the *audit deadline* handles the
// same NULL case (DM §4.2: deadline computation substitutes the pool median).
// JIT detection is a best-effort anti-fraud heuristic, not a required gate;
// flagging a new, fast, honest provider before its real throughput is known
// would be a false positive with real consequences (score/escrow impact), so
// this function prefers a false negative over that risk.
//
// NOTE: this threshold is distinct from the per-provider RTO / audit deadline
// (IC §4.2, (chunk_size / p95_throughput) * 1.5) — that is a maximum wait time
// before TIMEOUT; this is a minimum plausible time below which a PASS is
// flagged as suspicious. Do not conflate the two multipliers.
func EvaluateJIT(responseLatencyMs int, p95ThroughputKbps *float64) bool
```

> **Flagged — no signature specified in IC.** IC §5.5 does not declare any JIT-related function at all, despite `mvp.md §8.2` naming `jit.go` explicitly. The signature above is proposed to make this testable and to match how the rest of the audit pipeline consumes `p95_throughput_kbps` (a nullable `*float64`, mirroring DM §4.2's column). Reconcile with `interface-contracts.md §5.5` in a follow-up PR.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/audit/jit.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/audit/
  EXPECT: exit 0

FUNCTION_SIGNATURE:
  $ grep -c "^func EvaluateJIT(responseLatencyMs int, p95ThroughputKbps \*float64) bool" internal/audit/jit.go
  EXPECT: 1

CORRECT_MULTIPLIER:
  # Must be 0.3 (JIT floor), never 1.5 (that is the unrelated audit deadline)
  $ grep -c "0\.3" internal/audit/jit.go
  EXPECT: >= 1
  $ grep -n "1\.5" internal/audit/jit.go \
      && echo "FAIL: 1.5 is the audit-deadline multiplier, not the JIT floor" || echo "PASS"

CHUNK_SIZE_CONSTANT:
  $ grep -c "256" internal/audit/jit.go
  EXPECT: >= 1

NULL_HANDLING:
  $ grep -c "p95ThroughputKbps == nil\|p95ThroughputKbps != nil" internal/audit/jit.go
  EXPECT: >= 1

DISTINCT_FROM_RTO_COMMENT:
  $ grep -c "distinct from the per-provider RTO\|distinct from the RTO" internal/audit/jit.go
  EXPECT: 1

UNIT_TESTS:
  $ go test -v -run TestEvaluateJIT ./internal/audit/
  EXPECT: exit 0; tests include:
    TestEvaluateJITFlagsAnomalouslyFastResponse   (latency well below 0.3x expected -> true)
    TestEvaluateJITDoesNotFlagNormalResponse       (latency at or above 0.3x expected -> false)
    TestEvaluateJITSkipsOnNilThroughput            (p95ThroughputKbps == nil -> false, no panic)
    TestEvaluateJITBoundaryExactlyAtThreshold      (latency == 0.3x expected -> false; strict less-than)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/scoring" internal/audit/jit.go \
      && echo "FAIL: must read providers.p95_throughput_kbps via SQL, not import scoring (IC §9)" \
      || echo "PASS"

VET:
  $ go vet ./internal/audit/
  EXPECT: exit 0; zero output
```

---

### Phase 7.6 — Audit Package Tests *(new — closes an MVP §8.2 gap)*

**Reference:** MVP §8.2 (`audit_test.go` — "two-phase crash safety, idempotent retry, cross-replica nonce validation"), DM §3 Invariant 1, IC §8 (rotation overlap)

> No session in the original plan owned `audit_test.go`, though `mvp.md §8.2` names it explicitly with a specific coverage description. This phase adds it, consolidating what Sessions 7.1–7.5's individual `UNIT_TESTS` blocks already exercise in isolation, plus the cross-session integration behaviors that only make sense once every prior M7 session exists together.

#### Session 7.6.1 — Implement `audit_test.go`

**PRECONDITIONS:**

- Sessions 7.1.1 through 7.5.1 all complete
- CI Postgres instance available with the Milestone 4 migration applied (`audit_receipts` table, its row security policies, and the `audit_periods`/`providers` tables it references)

**TASK:**

Create `internal/audit/audit_test.go` covering the three areas `mvp.md §8.2` names for this file, each exercising the full pipeline (nonce → response → two-phase write) rather than a single function in isolation:

| Test name | What it validates |
|---|---|
| `TestTwoPhaseCrashSafety` | Phase 1 succeeds, then simulate a "crash" (no Phase 2 call); confirm the row is queryable as PENDING (`audit_result IS NULL`) and not silently lost |
| `TestTwoPhaseIdempotentRetry` | Call `WriteReceiptPhase2` twice with the same `receiptID` and the same result; second call returns `ErrReceiptAlreadyFinal`; exactly one row exists; no duplicate insert |
| `TestCrossReplicaNonceValidation` | A nonce generated with `server_secret_vN` validates successfully against a `ClusterSecretCache` that has independently loaded `vN` (simulating a second replica) — this is the property IC §27's failover story depends on |
| `TestFullChallengeResponseCycle` | End-to-end: `ChallengeNonce` → construct a signed response → `ValidateResponse` → `WriteReceiptPhase1` → `WriteReceiptPhase2`, asserting the final row's `challenge_nonce` is exactly 33 bytes and `audit_result` is the expected terminal value |
| `TestJITAndValidationAreIndependent` | A response can fail `EvaluateJIT` (flagged as suspiciously fast) while still passing `ValidateResponse` (signature is valid) — confirms the two checks are orthogonal, matching how `jit_flag` and `audit_result` are independent columns in DM §4.7 |

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/audit/audit_test.go && echo PASS || echo FAIL

TEST_NAMES_PRESENT:
  $ grep -c "^func TestTwoPhaseCrashSafety\|^func TestTwoPhaseIdempotentRetry\|^func TestCrossReplicaNonceValidation\|^func TestFullChallengeResponseCycle\|^func TestJITAndValidationAreIndependent" \
      internal/audit/audit_test.go
  EXPECT: 5

ALL_TESTS_PASS:
  $ go test -race -count=1 -v ./internal/audit/
  EXPECT: exit 0; every test in this file and every earlier M7 session's UNIT_TESTS block shows "--- PASS:"

IDEMPOTENT_RETRY_NO_DUPLICATE_ROW:
  $ grep -A15 "^func TestTwoPhaseIdempotentRetry" internal/audit/audit_test.go | grep -c "COUNT(\*)\|len(rows)"
  EXPECT: >= 1

CI_RACE_GATE (M0 CI check 4 must stay green with this package included):
  $ go test -race -count=1 ./...
  EXPECT: exit 0

FULL_PACKAGE_BUILD:
  $ go build ./internal/audit/
  $ go vet ./internal/audit/
  EXPECT: both exit 0
```

---

## Milestone 8 — Scoring System (`internal/scoring`)

**Deliverable:** Three-window reliability score, consecutive-pass counter with the VETTING→ACTIVE transition, EWMA RTO tracking. Read-only against the database.

**Package import constraints (IC §9):** `internal/scoring` must **NOT** import `internal/repair` or `internal/payment`. Score computation is read-only against the audit-receipt history; it does not trigger repairs or move money. Nothing prohibits `internal/scoring` from importing `internal/audit`, but there is also no need to — both packages query the `providers`/`audit_receipts` tables directly via SQL, independently. Keep `internal/scoring`'s only `internal/` import as `internal/config` (for `NetworkProfile`).

**Reference:** IC §5.6 (full package contract), DM §7 (`mv_provider_scores` materialised view, including the `scores_as_of` staleness guard), DM §4.2 (`providers` scoring/RTO columns), DM §9 checklist (`p95_throughput_kbps`/`avg_rtt_ms` default to NULL, not 0/2000), ADR-008 (reliability scoring), FR-050 (dual-window trigger), FR-026 (VETTING→ACTIVE transition), FR-040 (RTO formula), MVP §5.4 (toggle map), MVP §3.4 (demo/prod vetting values), MVP §8.2 (file inventory)

**Milestone review notes:**

- Session 8.2.1's `IncrementConsecutivePasses` needs the active `NetworkProfile` to read `VettingMinPasses`/`VettingMinDuration` — IC §5.6's declared signature doesn't carry one. See the flagged fix in Phase 8.2.
- Session 8.1.1's `GetScoreFromPrimary` is the function DM §7's "scores must be < 60 minutes old before use in payment decisions" rule depends on — Milestone 10 (`internal/payment`, release computation) is the actual caller and must use this variant, not plain `GetScore`.

---

### Phase 8.1 — Score Retrieval

**Reference:** IC §5.6 (`GetScore`, `ProviderScore` struct), DM §7 (`mv_provider_scores` view and its `scores_as_of` column), ADR-008, ADR-024, FR-050 (dual-window trigger)

> **Flagged — missing staleness field.** DM §7 requires consumers to "check the age before using scores for payment decisions" and has the view emit `NOW() AS scores_as_of` specifically so they can. IC §5.6's `ProviderScore` struct has no field to carry that value. Added below as `ScoresAsOf time.Time`.

#### Session 8.1.1 — Implement `GetScore()` and `GetScoreFromPrimary()`; initialise `internal/scoring/errors.go`

**PRECONDITIONS:**

- `internal/scoring/doc.go` exists (stub from Session 0.1.3)
- Migration from Session 4.7.1 applied (`mv_provider_scores` view, including its `scores_as_of` column, exists in the test database)

**TASK:**

1. Create `internal/scoring/errors.go` (the single accumulating home for this package's sentinels, mirroring the `internal/audit/errors.go` pattern from Session 7.1.1):

```go
package scoring

import "errors"

var (
 // ErrProviderNotFound is returned by GetScore when the provider has no rows
 // in mv_provider_scores yet (no audit history) (IC §5.6).
 ErrProviderNotFound = errors.New("scoring: provider not found in score view")

 // ErrProviderNotVetting is returned by IncrementConsecutivePasses when the
 // provider is not in VETTING status (IC §5.6).
 ErrProviderNotVetting = errors.New("scoring: provider is not in VETTING status")
)
```

1. Create `internal/scoring/score.go`:

```go
// ProviderScore holds the three-window scores and the weighted composite
// (ADR-008). ScoresAsOf carries the view's own NOW()-at-refresh-time column
// (DM §7) so callers — especially payment release computation (Milestone 10)
// — can enforce the "< 60 minutes old" staleness rule before using a score
// for a payment decision (DM §7 CRITICAL note, ADR-024).
type ProviderScore struct {
 Score24h       float64   // window weight 0.50 in the composite
 Score7d        float64   // window weight 0.30
 Score30d       float64   // window weight 0.20
 Composite      float64   // 0.50*24h + 0.30*7d + 0.20*30d
 DualWindowFlag bool      // true when score30d - score7d > 0.20 (FR-050, ADR-024 §3)
 ScoresAsOf     time.Time // from mv_provider_scores.scores_as_of (DM §7)
}

// GetScore queries mv_provider_scores for providerID. The view may be up to
// 60 seconds stale — acceptable for general scoring queries (IC §5.6). Reads
// via the connection pool, which may route to a read replica (ARCH §24).
//
// DualWindowFlag is computed in application code, not SQL, per IC §5.6 (easier
// to unit-test in isolation from the database).
//
// Error semantics:
//   - ErrProviderNotFound: no row for providerID in the view yet.
// Goroutine-safe: yes.
func GetScore(ctx context.Context, db *sql.DB, providerID uuid.UUID) (ProviderScore, error)

// GetScoreFromPrimary is identical to GetScore except it forces the read to
// the primary replica (pass a *sql.DB handle pointing at the primary, not the
// pooled/replica-routed one used elsewhere). Required whenever a score is
// about to be used for a payment decision — the monthly release multiplier
// computation (Milestone 10) MUST call this, not GetScore, per DM §7's
// staleness requirement.
func GetScoreFromPrimary(ctx context.Context, primaryDB *sql.DB, providerID uuid.UUID) (ProviderScore, error)
```

> **Flagged — `GetScoreFromPrimary` has no declared signature in IC.** It is named only in `mvp.md §8.2`'s file-inventory parenthetical, with no signature given anywhere. The signature above mirrors `GetScore` exactly except for which `*sql.DB` handle it's given — this keeps the "primary vs. replica" distinction a caller-side wiring concern rather than adding branching logic inside the function.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/scoring/errors.go && echo PASS || echo FAIL
  $ test -f internal/scoring/score.go  && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/scoring/
  EXPECT: exit 0

SENTINEL_ERRORS:
  $ grep -c "ErrProviderNotFound\|ErrProviderNotVetting" internal/scoring/errors.go
  EXPECT: 2

PROVIDERSCORE_STRUCT:
  $ grep -c "^type ProviderScore struct" internal/scoring/score.go
  EXPECT: 1
  $ grep -c "Score24h\|Score7d\|Score30d\|Composite\|DualWindowFlag\|ScoresAsOf" internal/scoring/score.go
  EXPECT: >= 6

SCORESASOF_FIELD_PRESENT:
  # The gap-fix: DM §7 requires staleness checking; the struct must expose it
  $ grep -c "ScoresAsOf time.Time" internal/scoring/score.go
  EXPECT: 1

FUNCTION_SIGNATURES:
  $ grep -c "^func GetScore(ctx context.Context, db \*sql.DB, providerID uuid.UUID) (ProviderScore, error)" \
      internal/scoring/score.go
  EXPECT: 1
  $ grep -c "^func GetScoreFromPrimary(ctx context.Context, primaryDB \*sql.DB, providerID uuid.UUID) (ProviderScore, error)" \
      internal/scoring/score.go
  EXPECT: 1

DUALWINDOWFLAG_COMPUTED_IN_GO_NOT_SQL:
  $ grep -c "DualWindowFlag = \|score30d - score7d > 0.20\|Score30d - .*Score7d.* > 0.20" internal/scoring/score.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestGetScore ./internal/scoring/
  EXPECT: exit 0; tests include:
    TestGetScoreReturnsAllThreeWindows       (Score24h, Score7d, Score30d all populated)
    TestGetScoreDualWindowFlagTrue            (score30d - score7d > 0.20 -> DualWindowFlag == true)
    TestGetScoreDualWindowFlagFalse           (difference <= 0.20 -> DualWindowFlag == false)
    TestGetScoreNotFound                      (unknown providerID -> ErrProviderNotFound)
    TestGetScoreFromPrimaryUsesGivenHandle    (queries route through the supplied *sql.DB, not a pooled default)
    TestGetScoreExposesScoresAsOf             (returned ProviderScore.ScoresAsOf is non-zero and matches the view's NOW())

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" internal/scoring/*.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/scoring/
  EXPECT: exit 0; zero output
```

---

### Phase 8.2 — Consecutive Pass Counter

**Reference:** IC §5.6 (`IncrementConsecutivePasses`, `ResetConsecutivePasses`), DM §4.2 (`consecutive_audit_passes`, `first_chunk_assignment_at`), DM §4.7 (`address_was_stale` — stale-timeout exemption), FR-026 (VETTING→ACTIVE transition conditions), MVP §3.4 (demo/prod vetting values table), MVP §5.4 (exact toggle-to-code mapping — names this function directly)

> **Flagged — signature cannot fulfill its own documented contract.** IC §5.6 declares `func IncrementConsecutivePasses(ctx context.Context, db *sql.DB, providerID uuid.UUID) error` and its doc comment says *"If the new value equals 80, also sets status = 'ACTIVE'."* Two problems: (1) "80" is the **production** value only — `mvp.md §5.4` explicitly assigns this function the job of checking `NetworkProfile.VettingMinPasses` (80 in prod, 5 in demo), and demo mode breaks entirely if 80 is hardcoded; (2) the declared signature has no `profile` parameter at all, so it has no way to read `VettingMinPasses` or `VettingMinDuration` even if it wanted to. Corrected signature below adds `profile config.NetworkProfile`, matching the pattern already used for `erasure.NewEngine(profile)` and `repair.RepairPromotionTimeout(profile)` elsewhere in this codebase.

#### Session 8.2.1 — Implement `IncrementConsecutivePasses()` with the VETTING→ACTIVE transition; implement `ResetConsecutivePasses()`

**PRECONDITIONS:**

- Session 8.1.1 complete (`errors.go` exists with `ErrProviderNotVetting`)
- `internal/config` complete (Milestone 1) — this session is the first in `internal/scoring` to import it

**TASK:**

In `internal/scoring/passes.go`:

```go
// IncrementConsecutivePasses atomically increments consecutive_audit_passes
// for a provider and, if both VETTING→ACTIVE conditions are now satisfied,
// transitions status to 'ACTIVE' in the same transaction (FR-026, ADR-005).
//
// The threshold is profile.VettingMinPasses (80 in production, 5 in demo) —
// NEVER a hardcoded 80; see MVP §5.4, which names this exact function as the
// enforcement point for that field. The duration condition is also checked:
// first_chunk_assignment_at + profile.VettingMinDuration <= NOW() (FR-026,
// MVP §5.4).
//
// Pre-conditions:
//   - providerID identifies a provider with status = 'VETTING'
// Post-conditions (on nil error):
//   - consecutive_audit_passes is incremented by 1
//   - if the new value >= profile.VettingMinPasses AND the duration condition
//     holds, status is set to 'ACTIVE' in the same transaction
// Error semantics:
//   - ErrProviderNotVetting: provider is not in VETTING status; no-op, no error
//     side effects on consecutive_audit_passes
// Goroutine-safe: yes — uses SELECT ... FOR UPDATE within a transaction so two
// concurrent audit-PASS events for the same provider cannot both observe the
// pre-increment count and both trigger the transition redundantly.
func IncrementConsecutivePasses(ctx context.Context, db *sql.DB, providerID uuid.UUID, profile config.NetworkProfile) error

// ResetConsecutivePasses resets consecutive_audit_passes to 0 on any non-PASS
// audit result (ADR-005).
//
// CALLER PRE-CONDITION (enforced by the caller, not inside this function — see
// note below): the caller MUST NOT invoke this when the just-recorded audit
// row has audit_result = 'TIMEOUT' AND address_was_stale = TRUE (DM §4.7). A
// TIMEOUT against a known-stale heartbeat address is evidence the DHT
// fallback didn't work, not evidence the provider failed, and must not reset
// vetting progress. This function's signature carries no addressWasStale
// parameter (matching IC §5.6 as declared), so the gate lives in the audit
// dispatch loop (Milestone 12, Session 12.1.2) which already has the receipt
// row in hand when deciding whether to call this function at all.
//
// Goroutine-safe: yes.
func ResetConsecutivePasses(ctx context.Context, db *sql.DB, providerID uuid.UUID) error
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/scoring/passes.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/scoring/
  EXPECT: exit 0

FUNCTION_SIGNATURES:
  $ grep -c "^func IncrementConsecutivePasses(ctx context.Context, db \*sql.DB, providerID uuid.UUID, profile config.NetworkProfile) error" \
      internal/scoring/passes.go
  EXPECT: 1
  $ grep -c "^func ResetConsecutivePasses(ctx context.Context, db \*sql.DB, providerID uuid.UUID) error" \
      internal/scoring/passes.go
  EXPECT: 1

PROFILE_DRIVEN_NOT_HARDCODED:
  $ grep -c "profile.VettingMinPasses" internal/scoring/passes.go
  EXPECT: >= 1
  $ grep -c "profile.VettingMinDuration" internal/scoring/passes.go
  EXPECT: >= 1
  # The literal 80 must not appear as a comparison threshold
  $ grep -n ">= 80\|== 80\|> 79" internal/scoring/passes.go \
      && echo "FAIL: hardcoded vetting-pass threshold — must read profile.VettingMinPasses" \
      || echo "PASS"

DURATION_CHECK_PRESENT:
  $ grep -c "first_chunk_assignment_at" internal/scoring/passes.go
  EXPECT: >= 1

CONCURRENCY_GUARD:
  $ grep -c "FOR UPDATE" internal/scoring/passes.go
  EXPECT: >= 1

STALE_TIMEOUT_NOTE_DOCUMENTED:
  # The self-contradiction in the original task text ("scoring package must check
  # this flag before calling ResetConsecutivePasses", when ResetConsecutivePasses
  # IS the scoring-package function) is resolved: the note must say the CALLER
  # (M12) performs the check, not this function internally.
  $ grep -c "address_was_stale\|addressWasStale" internal/scoring/passes.go
  EXPECT: >= 1
  $ grep -c "Milestone 12\|Session 12.1.2" internal/scoring/passes.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestIncrementConsecutivePasses ./internal/scoring/
  EXPECT: exit 0; tests include:
    TestIncrementConsecutivePassesUsesDemoProfile        (with config.DemoProfile, transitions to ACTIVE at 5 passes, not 80)
    TestIncrementConsecutivePassesUsesProductionProfile   (with config.ProductionProfile, does NOT transition before 80 passes)
    TestIncrementConsecutivePassesRequiresDurationToo     (VettingMinPasses met but VettingMinDuration not yet elapsed -> stays VETTING)
    TestIncrementConsecutivePassesRejectsNonVettingProvider (status=ACTIVE provider -> ErrProviderNotVetting)
    TestIncrementConsecutivePassesConcurrentSafe          (N concurrent goroutines incrementing the same provider -> exactly one ACTIVE transition, race-detector clean)

  $ go test -v -run TestResetConsecutivePasses ./internal/scoring/
  EXPECT: exit 0; tests include:
    TestResetConsecutivePassesZeroesCounter   (any non-PASS -> counter back to 0)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" internal/scoring/passes.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

RACE_DETECTOR:
  $ go test -race -count=1 -run TestIncrementConsecutivePassesConcurrentSafe ./internal/scoring/
  EXPECT: exit 0; zero data-race reports

VET:
  $ go vet ./internal/scoring/
  EXPECT: exit 0; zero output
```

---

### Phase 8.3 — EWMA RTO Tracking

**Reference:** IC §4.2 (RTO formula, `rto_sample_count >= 5` pool-median cutover), DM §4.2 (`avg_rtt_ms`, `var_rtt_ms`, `rto_sample_count`, `p95_throughput_kbps` — all NULL-until-established), DM §9 checklist ("default to NULL, not 0/2000"), FR-040 (ADR-006, Paper 28 — "TCP-style RTO"), MVP §8.2 (`rto.go`)

> **Corrected — "pool-median" computed as a mean.** DM §4.2, IC §4.2, and FR-040 all consistently name this the **pool-median** RTO. The original task's SQL was `AVG(avg_rtt_ms + 4 * var_rtt_ms)`, which computes the arithmetic mean, not a median — a mean is more sensitive to a handful of very slow (or very fast) providers than a true median would be, which is presumably why "median" was chosen as the term throughout three separate documents. Fixed below to use `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...)`, PostgreSQL's standard true-median construction, so the implementation matches its own name.
>
> **Note on EWMA constants.** No document in scope specifies the smoothing constants (α for the mean, β for the variance). "TCP-style RTO" (FR-040's own phrase, citing ADR-006 and Paper 28) is the standard name for the Jacobson/Karels algorithm, whose conventional constants are α = 1/8 for the smoothed mean and β = 1/4 for the mean deviation, combined as `RTO = SRTT + 4×RTTVAR` — which is exactly the `AVG + 4×VAR` formula used throughout this design. The session below uses those conventional values as the best available inference; confirm against Paper 28 directly before treating them as final.

#### Session 8.3.1 — Implement EWMA update functions

**PRECONDITIONS:**

- Session 8.1.1 complete (package skeleton, `errors.go` exists)
- Session 8.2.1 complete (establishes the `internal/config` import already used for other profile-driven values in this package, though this specific session's constants are not profile-variable)

**TASK:**

In `internal/scoring/rto.go`:

```go
// UpdateRTO updates a provider's EWMA response-time and throughput statistics
// after one audit response is recorded, and increments rto_sample_count.
// Must be called once per PASS or FAIL response (a TIMEOUT has no
// response_latency_ms to sample, per DM §4.7 §8.10, and must not call this).
//
// EWMA constants follow the conventional TCP RTO estimation (Jacobson/Karels),
// consistent with FR-040's "TCP-style RTO" framing (ADR-006, Paper 28):
//   avg_rtt_ms' = avg_rtt_ms + alpha * (sample_ms - avg_rtt_ms),        alpha = 1/8
//   var_rtt_ms' = var_rtt_ms + beta  * (|sample_ms - avg_rtt_ms| - var_rtt_ms), beta = 1/4
// On the FIRST sample for a provider (avg_rtt_ms/var_rtt_ms are NULL — DM
// §4.2), initialise avg_rtt_ms = sample_ms and var_rtt_ms = 0 directly, rather
// than blending against a NULL. NEVER write 0 or 2000 as a placeholder default
// outside of this first-sample initialisation (DM §9 checklist).
//
// p95_throughput_kbps is updated the same way, from the measured upload
// throughput of this response.
func UpdateRTO(ctx context.Context, db *sql.DB, providerID uuid.UUID, responseLatencyMs int, throughputKbps float64) error

// PoolMedianRTO returns the network-wide median RTO across ACTIVE providers,
// for use by any provider with rto_sample_count < 5 (IC §4.2, FR-040). This is
// a TRUE median (PERCENTILE_CONT(0.5)), matching the "pool-median" name used
// throughout DM §4.2 / IC §4.2 / FR-040 — NOT an AVG()/mean, which several
// providers with unusually slow or fast connections would skew.
//
//   SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY avg_rtt_ms + 4 * var_rtt_ms)
//   FROM providers WHERE status = 'ACTIVE' AND rto_sample_count >= 5
//
// The result is cached in-process for 5 minutes to avoid recomputing this
// aggregate on every audit dispatch (an implementation optimisation, not a
// documented requirement — revisit the cache duration if measured DB load
// warrants it).
func PoolMedianRTO(ctx context.Context, db *sql.DB) (float64, error)
```

> **Flagged — `UpdateRTO`/`PoolMedianRTO` have no signature in IC §5.6.** Both are referenced by name only: `UpdateRTO` is called from Milestone 12, Session 12.1.2 ("Update EWMA metrics via `scoring.UpdateRTO()`"), and the pool-median fallback is described narratively in IC §4.2 and DM §4.2 without ever being given a Go signature. The two signatures above are proposed to close that gap; reconcile with `interface-contracts.md §5.6` in a follow-up PR.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/scoring/rto.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/scoring/
  EXPECT: exit 0

FUNCTION_SIGNATURES:
  $ grep -c "^func UpdateRTO(ctx context.Context, db \*sql.DB, providerID uuid.UUID, responseLatencyMs int, throughputKbps float64) error" \
      internal/scoring/rto.go
  EXPECT: 1
  $ grep -c "^func PoolMedianRTO(ctx context.Context, db \*sql.DB) (float64, error)" internal/scoring/rto.go
  EXPECT: 1

TRUE_MEDIAN_NOT_MEAN:
  # The corrected fix: must use PERCENTILE_CONT, not AVG, for the "pool-median"
  $ grep -c "PERCENTILE_CONT(0.5)" internal/scoring/rto.go
  EXPECT: >= 1
  $ grep -n "AVG(avg_rtt_ms" internal/scoring/rto.go \
      && echo "FAIL: pool-median must be a true median (PERCENTILE_CONT), not AVG" \
      || echo "PASS"

SAMPLE_COUNT_THRESHOLD:
  $ grep -c "rto_sample_count >= 5\|rto_sample_count < 5" internal/scoring/rto.go
  EXPECT: >= 1

NULL_INITIALISATION_NOT_ZERO_OR_2000:
  # DM §9 checklist: p95_throughput_kbps / avg_rtt_ms must default to NULL, not 0/2000
  $ grep -n "= 0\.0\b" internal/scoring/rto.go | grep -v "beta\|alpha\|var_rtt_ms.*=.*0" \
      && echo "WARN: verify this is the deliberate first-sample var_rtt_ms=0 case, not a hidden default" \
      || echo "PASS"
  $ grep -c "2000" internal/scoring/rto.go
  EXPECT: 0

EWMA_CONSTANTS_DOCUMENTED:
  $ grep -c "alpha = 1/8\|1 / 8\|0.125" internal/scoring/rto.go
  EXPECT: >= 1
  $ grep -c "beta = 1/4\|1 / 4\|0.25" internal/scoring/rto.go
  EXPECT: >= 1

CACHE_TTL_NOTED_AS_IMPLEMENTATION_DETAIL:
  $ grep -c "5 \* time.Minute\|5\*time.Minute" internal/scoring/rto.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestUpdateRTO ./internal/scoring/
  EXPECT: exit 0; tests include:
    TestUpdateRTOFirstSampleInitialises      (NULL avg_rtt_ms/var_rtt_ms -> avg=sample, var=0, not 0/2000)
    TestUpdateRTOSubsequentSampleBlends       (second call blends via EWMA, does not overwrite)
    TestUpdateRTOIncrementsSampleCount        (rto_sample_count += 1 each call)
    TestUpdateRTOThroughputNeverDefaultsToZero

  $ go test -v -run TestPoolMedianRTO ./internal/scoring/
  EXPECT: exit 0; tests include:
    TestPoolMedianRTOIsTrueMedian    (constructed dataset with a skewing outlier -> median differs from mean; assert function returns the median value)
    TestPoolMedianRTOExcludesLowSampleProviders (providers with rto_sample_count < 5 excluded from the pool)
    TestPoolMedianRTOCachedFor5Minutes

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" internal/scoring/rto.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/scoring/
  EXPECT: exit 0; zero output
```

---

### Phase 8.4 — Scoring Package Tests *(new — closes an MVP §8.2 gap)*

**Reference:** MVP §8.2 (`score_test.go` — "dual-window flag, VETTING→ACTIVE transition race guard")

> Same gap pattern as Milestone 7: `mvp.md §8.2` names `score_test.go` with a specific coverage description, but no session in the original plan owned it. This phase adds it.

#### Session 8.4.1 — Implement `score_test.go`

**PRECONDITIONS:**

- Sessions 8.1.1 through 8.3.1 all complete
- CI Postgres instance available with the Milestone 4 migration applied, including `mv_provider_scores` (Session 4.7.1)

**TASK:**

Create `internal/scoring/score_test.go` consolidating the two behaviors `mvp.md §8.2` names for this file, at the integration level (spanning `score.go` + `passes.go` + `rto.go` together, not just re-running each file's own unit tests):

| Test name | What it validates |
|---|---|
| `TestDualWindowFlagAgainstRealView` | Seed `audit_receipts` so the 7-day pass rate is deliberately worse than the 30-day rate by more than 0.20; refresh `mv_provider_scores`; confirm `GetScore` reports `DualWindowFlag == true` — exercising the real materialised view, not a mocked struct |
| `TestVettingActiveTransitionRaceGuard` | Launch N concurrent calls to `IncrementConsecutivePasses` for the same provider at exactly `VettingMinPasses - 1` remaining passes; confirm the provider transitions to `ACTIVE` exactly once (no double-transition, no lost update) under `-race` |
| `TestDemoAndProductionProfilesBothReachActive` | Run the full pass sequence once with `config.DemoProfile` (expect transition at 5 passes) and once with `config.ProductionProfile` (expect transition at 80 passes) against the same schema, confirming the profile — not a hardcoded literal — drives the outcome |

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/scoring/score_test.go && echo PASS || echo FAIL

TEST_NAMES_PRESENT:
  $ grep -c "^func TestDualWindowFlagAgainstRealView\|^func TestVettingActiveTransitionRaceGuard\|^func TestDemoAndProductionProfilesBothReachActive" \
      internal/scoring/score_test.go
  EXPECT: 3

ALL_TESTS_PASS_WITH_RACE_DETECTOR:
  $ go test -race -count=1 -v ./internal/scoring/
  EXPECT: exit 0; every test in this file and every earlier M8 session's UNIT_TESTS block shows "--- PASS:"

RACE_GUARD_ACTUALLY_CONCURRENT:
  $ grep -A20 "^func TestVettingActiveTransitionRaceGuard" internal/scoring/score_test.go | grep -c "go func("
  EXPECT: >= 2   # must launch multiple goroutines, not call sequentially

NO_HARDCODED_THRESHOLD_IN_TEST:
  # The test itself must reference profile.VettingMinPasses, not a bare "80" or "5"
  $ grep -A15 "^func TestDemoAndProductionProfilesBothReachActive" internal/scoring/score_test.go | \
      grep -c "config.DemoProfile\|config.ProductionProfile"
  EXPECT: 2

CI_RACE_GATE (M0 CI check 4 must stay green with this package included):
  $ go test -race -count=1 ./...
  EXPECT: exit 0

FULL_PACKAGE_BUILD:
  $ go build ./internal/scoring/
  $ go vet ./internal/scoring/
  EXPECT: both exit 0
```

---

## Milestone 9 — Repair System (`internal/repair`)

**Deliverable:** Departure detector, repair job queue with priority ordering, repair executor, replacement-provider selection, and vetting-chunk exclusion.

**Package import constraints:** `internal/repair` must **NOT** import `internal/payment`, `internal/audit`, or `internal/scoring` — IC §9's closing paragraph ("the microservice entrypoint wires `internal/audit`, `internal/scoring`, `internal/repair`, and `internal/payment` together; none of these four packages imports any of the others directly") is the source for this, since IC §9's table itself never gives `internal/repair` its own row. `internal/repair` **MAY** import `internal/config` (`NetworkProfile`) and `internal/erasure` (RS decode/re-encode during repair).

> **Flagged — `internal/repair` cannot avoid touching the network while also "not importing p2p."** Session 9.2.1 requires opening libp2p streams to download and re-upload shards, which needs something shaped like `p2p.Host.NewStream`/`SetStreamHandler` (IC §5.4). Resolution used below (same pattern as `internal/audit`'s `SecretsManagerClient` in the Milestones 7–8 revision): `internal/repair` declares its own narrow, package-local transport interface; `internal/p2p.Host` satisfies it structurally without either package importing the other. The microservice entrypoint (Milestone 12) constructs the real `p2p.Host` and passes it in as that interface type.

**Reference:** IC §5.7 (full package contract), DM §4.10 (`repair_jobs` schema), DM §3 Invariant 6 (no repair for synthetic chunks), DM §3 Invariant 3 (no physical provider deletion), IC §4.4.1–§4.4.2 (repair download/upload wire protocol), IC §6 (`providers`/`repair_jobs` DML contract), FR-042–FR-045, FR-065–FR-067 (repair and vetting-exclusion functional requirements), ADR-004 (repair protocol), ADR-030 (synthetic vetting chunks), ADR-014 (ASN cap — replacement selection), MVP §8.2 (file inventory)

**Milestone review notes:**

- Session 9.2.1 (executor) depends on Session 9.4.1 (replacement selection) to choose the new provider before it can call the upload protocol — these two are more tightly coupled than the phase numbering suggests; implement 9.4.1's selection logic before finishing 9.2.1's upload step.
- Session 9.3.1 (departure detector) needs a `PenaliseFunc`-shaped callback injected at construction time, supplied by Milestone 12 Session 12.1.1 — see the flagged note in Phase 9.3.
- Session 9.1.1 declares `Priority` and `TriggerType`; it also closes the `.golangci.yml` placeholder `# M9: internal/repair.Priority, internal/repair.TriggerType` left by Session 0.2.1.

---

### Phase 9.1 — Repair Job Queue

**Reference:** IC §5.7 (`EnqueueJob`, `DequeueNextJob`, `IsVettingChunk`, `DeleteVettingChunksOnDeparture`, `MarkJobComplete`), DM §4.10 (`repair_priority`/`repair_trigger_type` ENUMs, `repair_jobs_priority_matches_trigger` CHECK), DM §3 Invariant 6, ADR-004, ADR-030, FR-043

> **Flagged — `Priority` is missing a value.** IC §5.7 declares `type Priority int` with exactly two constants (`PriorityPermanentDeparture`, `PriorityPreWarning`), despite its own comment saying it "mirrors the repair_priority DB enum," which has three values (`EMERGENCY`, `PERMANENT_DEPARTURE`, `PRE_WARNING` — DM §4.10). Consequently, `EnqueueJob`'s documented derivation rule ("priority is set to PERMANENT_DEPARTURE iff triggerType is SilentDeparture or AnnouncedDeparture; PRE_WARNING otherwise") never accounts for `TriggerEmergencyFloor` at all — under that rule, an emergency-floor job would fall into the `PRE_WARNING` tier, exactly backwards from ADR-004 ("Repair must be triggered immediately... if the fragment count for any chunk drops to s=16") and DM §4.10's own comment ("`'EMERGENCY'` -- EMERGENCY_FLOOR: s=16, immediate, front of queue"). Fixed below by adding `PriorityEmergency` as the first (lowest-ordinal, drains-first) value and completing the derivation logic for all three trigger groups.

#### Session 9.1.1 — Declare `Priority`/`TriggerType`; implement `EnqueueJob()`; initialise `internal/repair/errors.go`

**PRECONDITIONS:**

- `internal/repair/doc.go` exists (stub from Session 0.1.3)
- `internal/config` complete (Milestone 1) — `EnqueueJob` needs `NetworkProfile.DataShards`/`TotalShards` for range validation
- Migration from Session 4.4.5 applied (`repair_jobs` table, including the profile-parameterised `available_shard_count` CHECK)

**TASK:**

1. Create `internal/repair/errors.go` (the single accumulating home for this package's sentinels — mirrors `internal/audit/errors.go` and `internal/scoring/errors.go` from the Milestones 7–8 revision):

```go
package repair

import "errors"

var (
 // ErrShardCountOutOfRange is returned by EnqueueJob when availableShardCount
 // falls outside [profile.DataShards, profile.TotalShards] (DM §4.10).
 ErrShardCountOutOfRange = errors.New("repair: availableShardCount outside [DataShards, TotalShards] for active profile")

 // ErrJobQueueEmpty is returned by DequeueNextJob when no QUEUED job exists.
 // (Distinguishable from a genuine database error — callers should treat
 // this as "nothing to do right now", not retry with backoff.)
 ErrJobQueueEmpty = errors.New("repair: no queued repair job available")
)
```

1. In `internal/repair/queue.go`, declare the corrected enum types:

```go
// Priority mirrors the repair_priority DB enum (DM §4.10) — all three values,
// ordered to match the ENUM's declaration order in PostgreSQL (see Session
// 9.1.2 for why declaration order, not alphabetical order, is what matters).
type Priority int

const (
 PriorityEmergency          Priority = iota // EMERGENCY — s at reconstruction floor; front of queue
 PriorityPermanentDeparture                 // SILENT_DEPARTURE / ANNOUNCED_DEPARTURE — drains next
 PriorityPreWarning                         // THRESHOLD_WARNING — waits behind the above
)

// TriggerType mirrors the repair_trigger_type DB enum (DM §4.10).
type TriggerType int

const (
 TriggerSilentDeparture TriggerType = iota
 TriggerAnnouncedDeparture
 TriggerThresholdWarning
 TriggerEmergencyFloor
)
```

1. Implement `EnqueueJob`:

```go
// EnqueueJob inserts a repair job into repair_jobs. Priority is derived
// automatically and completely from triggerType (DM §4.10
// repair_jobs_priority_matches_trigger CHECK):
//   TriggerEmergencyFloor                          -> PriorityEmergency
//   TriggerSilentDeparture, TriggerAnnouncedDeparture -> PriorityPermanentDeparture
//   TriggerThresholdWarning                        -> PriorityPreWarning
//
// Pre-conditions:
//   - len(chunkID) == 32; segmentID is a valid UUID
//   - availableShardCount is in [profile.DataShards, profile.TotalShards]
//     (16-56 in production, 3-5 in demo — DM §4.10; NEVER hardcode [16,56])
//   - the chunk_assignments row for (chunkID, any active provider) has
//     is_vetting_chunk = FALSE. Calling EnqueueJob for a synthetic vetting
//     chunk is a calling-contract violation: PANICS in debug builds as a
//     second line of defence (the departure handler, Session 9.3.1, must
//     call IsVettingChunk() first — this panic only catches a caller that
//     skipped that check). (ADR-030, DM §3 Invariant 6)
// Post-conditions (on nil error):
//   - a row with status='QUEUED' is inserted into repair_jobs with the
//     derived priority
// Error semantics: ErrShardCountOutOfRange if the pre-condition on
//   availableShardCount fails; other database errors returned as-is.
// Goroutine-safe: yes.
func EnqueueJob(
 ctx context.Context,
 db *sql.DB,
 profile config.NetworkProfile,
 chunkID [32]byte,
 segmentID uuid.UUID,
 providerID *uuid.UUID,
 triggerType TriggerType,
 availableShardCount int,
) error
```

> **Flagged — added `profile` parameter.** IC §5.7's declared signature has no `profile` parameter, yet its own pre-condition ("availableShardCount is in [16, 56]") is stated as a fixed range that the session task already (correctly) overrides to be profile-variable. Adding `profile config.NetworkProfile` here matches the calling convention already used by the sibling function `RepairPromotionTimeout(profile)` in this same package (Session 9.2.2) — this is a minimal, internally-consistent fix, not a new pattern.

1. **Close the `.golangci.yml` placeholder from Session 0.2.1:** replace `# M9: internal/repair.Priority, internal/repair.TriggerType` with both types now that they're fully declared (with `Priority`'s three values, not two).

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/repair/errors.go && echo PASS || echo FAIL
  $ test -f internal/repair/queue.go  && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/repair/
  EXPECT: exit 0

PRIORITY_HAS_THREE_VALUES:
  # The core bug fix: must be 3 constants, not 2
  $ grep -c "PriorityEmergency\|PriorityPermanentDeparture\|PriorityPreWarning" internal/repair/queue.go
  EXPECT: 3
  $ grep -c "^type Priority int" internal/repair/queue.go
  EXPECT: 1

PRIORITY_EMERGENCY_IS_FIRST:
  # Must be iota-0 so it sorts first, matching ORDER BY priority ASC intent
  $ grep -A3 "^const (" internal/repair/queue.go | grep -B2 "PriorityEmergency" | grep -c "iota"
  EXPECT: >= 1

FUNCTION_SIGNATURE:
  $ grep -c "^func EnqueueJob(" internal/repair/queue.go
  EXPECT: 1
  $ grep -c "profile config.NetworkProfile" internal/repair/queue.go
  EXPECT: >= 1

EMERGENCY_FLOOR_MAPPED_CORRECTLY:
  # The specific bug: TriggerEmergencyFloor must map to PriorityEmergency, not fall through
  $ grep -B2 -A2 "TriggerEmergencyFloor" internal/repair/queue.go | grep -c "PriorityEmergency"
  EXPECT: >= 1

SENTINEL_ERRORS:
  $ grep -c "ErrShardCountOutOfRange\|ErrJobQueueEmpty" internal/repair/errors.go
  EXPECT: 2

DEBUG_PANIC_SECOND_LINE_OF_DEFENCE:
  $ grep -c "panic(" internal/repair/queue.go
  EXPECT: >= 1
  $ grep -c "//go:build debug\|debug.Enabled\|buildDebug" internal/repair/queue.go
  EXPECT: >= 1

GOLANGCI_PLACEHOLDER_CLOSED:
  $ grep -c "# M9: internal/repair.Priority, internal/repair.TriggerType" .golangci.yml
  EXPECT: 0
  $ grep -c "internal/repair\.Priority\|internal/repair\.TriggerType" .golangci.yml
  EXPECT: 2

UNIT_TESTS:
  $ go test -v -run TestEnqueueJob ./internal/repair/
  EXPECT: exit 0; tests include:
    TestEnqueueJobEmergencyFloorGetsEmergencyPriority   (TriggerEmergencyFloor -> row inserted with priority='EMERGENCY', not PRE_WARNING)
    TestEnqueueJobDepartureGetsPermanentDeparturePriority (TriggerSilentDeparture / TriggerAnnouncedDeparture -> PERMANENT_DEPARTURE)
    TestEnqueueJobThresholdGetsPreWarningPriority        (TriggerThresholdWarning -> PRE_WARNING)
    TestEnqueueJobRejectsOutOfRangeShardCountDemo        (config.DemoProfile, count=6 -> ErrShardCountOutOfRange)
    TestEnqueueJobRejectsOutOfRangeShardCountProd        (config.ProductionProfile, count=15 -> ErrShardCountOutOfRange)
    TestEnqueueJobPanicsOnVettingChunkInDebugBuild       (is_vetting_chunk=TRUE -> panic, debug build tag only)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/payment\|Vyomanaut_V2/internal/audit\|Vyomanaut_V2/internal/scoring\|Vyomanaut_V2/internal/p2p" \
      internal/repair/queue.go internal/repair/errors.go \
      && echo "FAIL: prohibited import" || echo "PASS"

VET:
  $ go vet ./internal/repair/
  EXPECT: exit 0; zero output
```

#### Session 9.1.2 — Implement `DequeueNextJob()`

**PRECONDITIONS:**

- Session 9.1.1 complete (`Priority`, `TriggerType`, `errors.go` exist)

**TASK:**

Implement, in `internal/repair/queue.go`:

```go
// DequeueNextJob atomically retrieves and marks IN_PROGRESS the highest-priority
// QUEUED repair job, using SELECT ... FOR UPDATE SKIP LOCKED so concurrent
// workers never contend for the same row (IC §5.7).
//
// Ordering: ORDER BY priority ASC, created_at ASC — EMERGENCY drains first,
// then PERMANENT_DEPARTURE, then PRE_WARNING; FIFO within each tier (ADR-004,
// DM §4.10).
//
// This ordering works because PostgreSQL ENUM comparison operators sort by
// the enum's DECLARATION order (the order values were listed in `CREATE TYPE
// ... AS ENUM (...)`), not lexicographically. DM §4.10's own comment
// describes this as "EMERGENCY < PERMANENT_DEPARTURE < PRE_WARNING
// alphabetically" — that phrasing is misleading: it is only a coincidence
// that the declaration order chosen here also happens to be alphabetical. If
// a fourth priority value were ever added via `ALTER TYPE ... ADD VALUE`
// without care for its position, alphabetical order and declaration order
// would diverge, and this comment would break silently. What actually
// guarantees `ORDER BY priority ASC` drains EMERGENCY first is that
// 'EMERGENCY' was declared before 'PERMANENT_DEPARTURE' before 'PRE_WARNING'
// in the CREATE TYPE statement (Session 4.2.1) — not the spelling.
//
// Returns nil, nil if the queue is empty (not an error condition).
// Goroutine-safe: yes.
func DequeueNextJob(ctx context.Context, db *sql.DB) (*RepairJob, error)
```

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/repair/
  EXPECT: exit 0

FUNCTION_SIGNATURE:
  $ grep -c "^func DequeueNextJob(ctx context.Context, db \*sql.DB) (\*RepairJob, error)" internal/repair/queue.go
  EXPECT: 1

ATOMIC_DEQUEUE:
  $ grep -c "FOR UPDATE SKIP LOCKED" internal/repair/queue.go
  EXPECT: >= 1

ORDERING_CLAUSE:
  $ grep -c "ORDER BY priority ASC, created_at ASC" internal/repair/queue.go
  EXPECT: 1

DECLARATION_ORDER_NOTE_PRESENT:
  # Confirms the "alphabetical" misconception from DM §4.10 is corrected here, not repeated
  $ grep -c "declaration order\|DECLARATION order" internal/repair/queue.go
  EXPECT: >= 1
  $ grep -n "alphabetically" internal/repair/queue.go | grep -v "coincidence\|misleading" \
      && echo "FAIL: must not restate 'alphabetically' as the mechanism without the correction" \
      || echo "PASS"

EMPTY_QUEUE_IS_NOT_AN_ERROR:
  $ grep -c "return nil, nil" internal/repair/queue.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestDequeueNextJob ./internal/repair/
  EXPECT: exit 0; tests include:
    TestDequeueNextJobDrainsEmergencyFirst        (mixed-priority queue -> EMERGENCY row returned before PERMANENT_DEPARTURE/PRE_WARNING)
    TestDequeueNextJobFIFOWithinTier              (two PRE_WARNING rows -> older created_at returned first)
    TestDequeueNextJobEmptyQueueReturnsNilNil      (no QUEUED rows -> (nil, nil), not an error)
    TestDequeueNextJobConcurrentWorkersNoDoubleDequeue (N goroutines, SKIP LOCKED -> each job claimed exactly once)

RACE_DETECTOR:
  $ go test -race -count=1 -run TestDequeueNextJobConcurrentWorkersNoDoubleDequeue ./internal/repair/
  EXPECT: exit 0; zero data-race reports

VET:
  $ go vet ./internal/repair/
  EXPECT: exit 0; zero output
```

#### Session 9.1.3 — Implement `IsVettingChunk()`, `DeleteVettingChunksOnDeparture()`, and `MarkJobComplete()`

**PRECONDITIONS:**

- Session 9.1.1 complete

> **Note:** the original Phase 9.1 reference line names five functions (`EnqueueJob`, `DequeueNextJob`, `IsVettingChunk`, `DeleteVettingChunksOnDeparture`, `MarkJobComplete`), but only four had an owning session — `MarkJobComplete` was never assigned anywhere. Added here since it's a queue-lifecycle function like the other two in this session.

**TASK:**

In `internal/repair/queue.go`:

```go
// IsVettingChunk returns true iff the chunk_assignments row for
// (chunkID, providerID) has is_vetting_chunk = TRUE. Must be called by the
// departure handler (Session 9.3.1) and by any threshold monitor BEFORE
// calling EnqueueJob for that chunk (ADR-030).
func IsVettingChunk(ctx context.Context, db *sql.DB, chunkID [32]byte, providerID uuid.UUID) (bool, error)

// DeleteVettingChunksOnDeparture soft-deletes all synthetic chunk_assignments
// for a departing vetting provider (status='DELETED', deleted_at=NOW()).
// Enqueues zero repair_jobs (FR-065, DM §3 Invariant 6).
//
// Pre-conditions:
//   - providerID identifies a provider transitioning to status='DEPARTED'
//   - all of that provider's chunk_assignments have is_vetting_chunk = TRUE
//     (the departure handler must have already verified this — a VETTING
//     provider that somehow holds a real shard is itself a bug elsewhere)
func DeleteVettingChunksOnDeparture(ctx context.Context, db *sql.DB, providerID uuid.UUID) error

// MarkJobComplete sets a repair job's status to COMPLETED or FAILED, and
// stamps completed_at = NOW() (DM §4.10 repair_jobs_completed_after_started
// CHECK: completed_at requires started_at to already be set). Called by the
// repair executor (Session 9.2.1) after the replacement upload succeeds
// (success=true) or exhausts retries (success=false).
func MarkJobComplete(ctx context.Context, db *sql.DB, jobID uuid.UUID, success bool) error
```

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/repair/
  EXPECT: exit 0

FUNCTION_SIGNATURES:
  $ grep -c "^func IsVettingChunk(ctx context.Context, db \*sql.DB, chunkID \[32\]byte, providerID uuid.UUID) (bool, error)" \
      internal/repair/queue.go
  EXPECT: 1
  $ grep -c "^func DeleteVettingChunksOnDeparture(ctx context.Context, db \*sql.DB, providerID uuid.UUID) error" \
      internal/repair/queue.go
  EXPECT: 1
  $ grep -c "^func MarkJobComplete(ctx context.Context, db \*sql.DB, jobID uuid.UUID, success bool) error" \
      internal/repair/queue.go
  EXPECT: 1

ZERO_REPAIR_JOBS_ON_VETTING_DEPARTURE:
  $ grep -A20 "^func DeleteVettingChunksOnDeparture" internal/repair/queue.go | grep -c "EnqueueJob"
  EXPECT: 0

SOFT_DELETE_NOT_HARD_DELETE:
  $ grep -A20 "^func DeleteVettingChunksOnDeparture" internal/repair/queue.go | grep -c "DELETE FROM chunk_assignments"
  EXPECT: 0
  $ grep -A20 "^func DeleteVettingChunksOnDeparture" internal/repair/queue.go | grep -c "status = 'DELETED'\|deleted_at = NOW()"
  EXPECT: >= 1

MARKJOBCOMPLETE_SETS_TIMESTAMP:
  $ grep -A10 "^func MarkJobComplete" internal/repair/queue.go | grep -c "completed_at"
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestIsVettingChunk ./internal/repair/
  EXPECT: exit 0; tests include: TestIsVettingChunkTrueForSynthetic, TestIsVettingChunkFalseForReal

  $ go test -v -run TestDeleteVettingChunksOnDeparture ./internal/repair/
  EXPECT: exit 0; tests include:
    TestDeleteVettingChunksOnDepartureZeroRepairJobs     (repair_jobs table has zero new rows after call)
    TestDeleteVettingChunksOnDepartureSoftDeletesAll     (all rows for provider -> status='DELETED', deleted_at set)

  $ go test -v -run TestMarkJobComplete ./internal/repair/
  EXPECT: exit 0; tests include:
    TestMarkJobCompleteSuccess    (success=true -> status='COMPLETED')
    TestMarkJobCompleteFailure    (success=false -> status='FAILED')
    TestMarkJobCompleteRejectsUnstartedJob (started_at IS NULL -> error, matching repair_jobs_completed_after_started CHECK)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/payment\|Vyomanaut_V2/internal/audit\|Vyomanaut_V2/internal/scoring" \
      internal/repair/queue.go \
      && echo "FAIL: prohibited import" || echo "PASS"

VET:
  $ go vet ./internal/repair/
  EXPECT: exit 0; zero output
```

---

### Phase 9.2 — Repair Executor

**Reference:** IC §4.4.1 (repair download stream — wire format, timeout, authentication), IC §4.4.2 (repair upload stream — reuses `/vyomanaut/chunk-upload/1.0.0`, IC §4.1), IC §5.7 (`RepairJob`, `RepairPromotionTimeout`), IC §5.4 (`Host` interface — structurally consumed, not imported), FR-042–FR-045, ADR-004

> **Flagged — three missing implementation details in the original task.** (1) IC §4.1's capability-token requirement applies to the repair upload exactly as it does to a normal upload (§4.4.2: "IDENTICAL to the standard upload protocol") — the executor must mint a `capability_token` for the replacement provider before uploading, or the upload fails `0x03 NOT_ASSIGNED`. (2) IC §4.4.2 explicitly requires the microservice to INSERT the new `chunk_assignments` row *before* opening the upload stream, so the replacement provider's own `NOT_ASSIGNED` check passes — the original task description never mentioned this ordering requirement. (3) The reconstructed shard being re-uploaded is not necessarily a *parity* shard — RS re-encoding regenerates the full shard set, and the specific index needing replacement could be any data shard (0–15 in production) or parity shard (16–55), matching whichever holder actually departed. "Generate missing parity shards" (the original phrasing) is corrected below.

#### Session 9.2.1 — Implement the repair download/decode/re-encode/upload pipeline

**PRECONDITIONS:**

- Session 9.1.1 complete (`errors.go`, `Priority`/`TriggerType` exist)
- Session 9.4.1 complete (replacement-provider selection — the executor needs a chosen replacement provider *before* it can construct the capability token or open the upload stream)
- `internal/erasure` complete (Milestone 3)
- A package-local transport interface exists for this session to depend on (see the flagged import-constraint note in the Milestone 9 header) — declare it in this session if not already present:

```go
// RepairTransport is the narrow subset of p2p.Host this package needs to open
// download/upload streams. internal/p2p.Host satisfies this interface
// structurally; internal/repair never imports internal/p2p, and internal/p2p
// never imports internal/repair. The microservice entrypoint (Milestone 12,
// Session 12.1.1) constructs the real Host and injects it here as this type.
type RepairTransport interface {
 NewStream(ctx context.Context, peerID peer.ID, protocolID protocol.ID) (network.Stream, error)
}
```

**TASK:**

In `internal/repair/executor.go`, implement the microservice-side client for `/vyomanaut/repair-download/1.0.0` (IC §4.4.1), then the reconstruction and re-upload:

1. **Download.** Contact `profile.DataShards` (16 in production, 3 in demo) surviving shard holders. For each: open a stream via `RepairTransport.NewStream`, send `RepairDownloadRequest` — `chunk_id` (32B) + `repair_auth_sig` (64B, Ed25519 by the microservice signing key over `SHA-256(chunk_id ‖ request_ts_ms ‖ microservice_peer_id)`, IC §4.4.1). 0-RTT is **PROHIBITED** on this stream (set `DisableEarlyData: true`). Timeout: 10,000 ms per holder. Handle all four documented status codes: `0x00` OK (accept `chunk_data`), `0x01` NOT_FOUND / `0x03` CORRUPTION (try the next surviving holder — up to `profile.ParityShards` extra holders are available before running out), `0x02` NOT_AUTHORISED / `0x04` INTERNAL_ERROR (log and retry with backoff, or escalate if all holders exhausted).
2. **Reconstruct.** Once `profile.DataShards` valid shards are collected, call `erasure.Engine.DecodeSegment()` to recover the AONT package, then `erasure.Engine.EncodeSegment()` to **regenerate the full `profile.TotalShards`-shard set**. Extract only the specific shard index (or indices) that correspond to the departed/failed holder(s) — this can be any index from `0` to `profile.TotalShards-1`, i.e. a data shard or a parity shard; it is not necessarily parity-only.
3. **Pre-register, then upload.** Before opening the upload stream: `INSERT` the new `chunk_assignments` row for the replacement provider (chosen by Session 9.4.1) with `status='REPAIRING'` — this must happen *before* the upload stream opens, or the replacement provider's own `NOT_ASSIGNED` check (IC §4.1 status `0x03`) rejects the frame. Mint a `capability_token` (IC §4.1 format: `expiry_unix_ms(8B) || Ed25519_sign(microservice_signing_key, signing_input)`, 1-hour TTL) scoped to the replacement provider's ID and the reconstructed chunk. Upload via the standard `/vyomanaut/chunk-upload/1.0.0` protocol (IC §4.4.2) — identical wire format to a normal client upload; the replacement provider cannot and must not be able to distinguish a repair upload from a normal one.
4. **Confirm.** On receiving `UploadResponse` with `status = 0x00`: call `MarkJobComplete(ctx, db, jobID, success=true)` and update the new provider's `chunk_assignments.status` from `REPAIRING` to `ACTIVE` (IC §4.4.2 post-repair confirmation). On failure after retries: `MarkJobComplete(ctx, db, jobID, success=false)`.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/repair/executor.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/repair/
  EXPECT: exit 0

TRANSPORT_INTERFACE_IS_LOCAL_NOT_IMPORTED:
  $ grep -c "^type RepairTransport interface" internal/repair/executor.go
  EXPECT: 1
  $ grep -n "Vyomanaut_V2/internal/p2p" internal/repair/executor.go \
      && echo "FAIL: must not import internal/p2p directly" || echo "PASS"

DOWNLOAD_USES_PROFILE_DATASHARDS_NOT_HARDCODED_16:
  $ grep -c "profile.DataShards" internal/repair/executor.go
  EXPECT: >= 1
  $ grep -n "== 16\b" internal/repair/executor.go | grep -v "test\|_test" \
      && echo "FAIL: DataShards count must come from profile, not a literal 16" || echo "PASS"

ZERO_RTT_DISABLED_ON_DOWNLOAD:
  $ grep -c "DisableEarlyData" internal/repair/executor.go
  EXPECT: >= 1

ALL_FOUR_STATUS_CODES_HANDLED:
  $ grep -c "0x00\|0x01\|0x02\|0x03\|0x04" internal/repair/executor.go
  EXPECT: >= 5

RECONSTRUCTS_ANY_SHARD_INDEX_NOT_JUST_PARITY:
  $ grep -c "data shard or a parity shard\|any index from\|not necessarily parity" internal/repair/executor.go
  EXPECT: >= 1

PRE_REGISTRATION_BEFORE_UPLOAD:
  # The order-of-operations fix: INSERT chunk_assignments must appear before NewStream for upload
  $ grep -n "INSERT INTO chunk_assignments" internal/repair/executor.go | head -1
  $ grep -n "chunk-upload/1.0.0" internal/repair/executor.go | head -1
  # Manual check: INSERT line number must be smaller than the upload-stream line number

CAPABILITY_TOKEN_MINTED:
  $ grep -c "capability_token\|CapabilityToken\|expiry_unix_ms" internal/repair/executor.go
  EXPECT: >= 1

STATUS_TRANSITION_ON_SUCCESS:
  $ grep -c "REPAIRING.*ACTIVE\|'ACTIVE'" internal/repair/executor.go
  EXPECT: >= 1
  $ grep -c "MarkJobComplete" internal/repair/executor.go
  EXPECT: >= 2   # both success and failure paths

UNIT_TESTS:
  $ go test -v -run TestRepairExecutor ./internal/repair/
  EXPECT: exit 0; tests include:
    TestRepairExecutorDownloadsMinimumShards       (stops after profile.DataShards successful downloads)
    TestRepairExecutorFallsBackOnHolderFailure     (0x01/0x03 from one holder -> tries next candidate, does not abort)
    TestRepairExecutorReconstructsCorrectShardIndex (missing shard was index 3 -> uploaded shard is index 3, not necessarily a parity index)
    TestRepairExecutorPreRegistersBeforeUpload      (chunk_assignments row exists with status=REPAIRING before any upload stream is opened)
    TestRepairExecutorMarksCompleteOnSuccess
    TestRepairExecutorMarksFailedOnExhaustedRetries

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/payment\|Vyomanaut_V2/internal/audit\|Vyomanaut_V2/internal/scoring" \
      internal/repair/executor.go \
      && echo "FAIL: prohibited import" || echo "PASS"

VET:
  $ go vet ./internal/repair/
  EXPECT: exit 0; zero output
```

#### Session 9.2.2 — Implement `RepairPromotionTimeout()` and the stale pre-warning promotion scan

**PRECONDITIONS:**

- Session 9.1.2 complete (`DequeueNextJob` and the queue's basic shape exist in `queue.go`)

> **Note:** the original task described only the one-line accessor. IC §5.7's own comment says "the scheduler must call this," implying a consumer exists — but no session actually implemented the promotion logic (a scan of stale `PRE_WARNING` jobs) that would call it. Added below, closing FR-043's second half ("must be promoted to permanent-departure priority if they have been queued for more than 6 hours without service").

**TASK:**

In `internal/repair/queue.go` (job-lifecycle logic belongs alongside `DequeueNextJob`, not in `executor.go`):

```go
// RepairPromotionTimeout returns the duration after which a QUEUED PRE_WARNING
// job is promoted to PERMANENT_DEPARTURE priority (ADR-031, FR-043).
// 6 hours in production, 3 minutes in demo — the scheduler must call this
// rather than reading a constant.
func RepairPromotionTimeout(profile config.NetworkProfile) time.Duration {
 return profile.RepairPromotionTimeout
}

// PromoteStalePreWarningJobs finds every QUEUED PRE_WARNING job whose
// created_at is older than RepairPromotionTimeout(profile) and updates its
// priority to PERMANENT_DEPARTURE (FR-043). Intended to run on a periodic
// ticker from the microservice entrypoint (Milestone 12), independent of
// DequeueNextJob's own invocation cadence.
//
//   UPDATE repair_jobs
//   SET priority = 'PERMANENT_DEPARTURE'
//   WHERE priority = 'PRE_WARNING'
//     AND status = 'QUEUED'
//     AND created_at < NOW() - $1  -- RepairPromotionTimeout(profile)
//
// Returns the number of rows promoted, for observability.
func PromoteStalePreWarningJobs(ctx context.Context, db *sql.DB, profile config.NetworkProfile) (int, error)
```

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/repair/
  EXPECT: exit 0

FUNCTION_SIGNATURES:
  $ grep -c "^func RepairPromotionTimeout(profile config.NetworkProfile) time.Duration" internal/repair/queue.go
  EXPECT: 1
  $ grep -c "^func PromoteStalePreWarningJobs(ctx context.Context, db \*sql.DB, profile config.NetworkProfile) (int, error)" \
      internal/repair/queue.go
  EXPECT: 1

NOT_HARDCODED:
  $ grep -c "return profile.RepairPromotionTimeout" internal/repair/queue.go
  EXPECT: 1
  $ grep -n "6 \* time.Hour\|3 \* time.Minute" internal/repair/queue.go | grep -v "_test.go" \
      && echo "FAIL: must read profile field, not a literal duration" || echo "PASS"

PROMOTION_SQL_TARGETS_CORRECT_ROWS:
  $ grep -c "priority = 'PRE_WARNING'" internal/repair/queue.go
  EXPECT: >= 1
  $ grep -c "priority = 'PERMANENT_DEPARTURE'" internal/repair/queue.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestRepairPromotionTimeout ./internal/repair/
  EXPECT: exit 0; tests include:
    TestRepairPromotionTimeoutReturnsProfileValue (demo profile -> 3 min; prod profile -> 6h)

  $ go test -v -run TestPromoteStalePreWarningJobs ./internal/repair/
  EXPECT: exit 0; tests include:
    TestPromoteStalePreWarningJobsPromotesOldOnes     (job older than threshold -> priority becomes PERMANENT_DEPARTURE)
    TestPromoteStalePreWarningJobsLeavesFreshOnes     (job younger than threshold -> priority unchanged)
    TestPromoteStalePreWarningJobsIgnoresNonQueued     (IN_PROGRESS/COMPLETED PRE_WARNING rows untouched)

VET:
  $ go vet ./internal/repair/
  EXPECT: exit 0; zero output
```

---

### Phase 9.3 — Departure Detector

**Reference:** IC §3.1 (heartbeat's effect on `last_heartbeat_ts`), DM §4.2 (`departed_at`, `frozen`, `providers.status` state machine), DM §3 Invariant 3 (no physical deletion), IC §6 (`providers.status` DML — "any → DEPARTED" is a valid transition from either `VETTING` or `ACTIVE`), FR-035, FR-065, ARCH §12 (Provider Lifecycle — "Exiting" table, both Silent-departure rows)

> **Flagged and fixed — the detection query excluded VETTING providers entirely.** The original SQL was `WHERE status = 'ACTIVE'`. But ARCH §12's own "Exiting" table lists **two** silent-departure rows keyed on the *same* `last_heartbeat_ts`/threshold mechanism — one for `status = VETTING`, one for `status = ACTIVE` — and FR-065 explicitly presupposes that a VETTING provider *does* cross this same 72-hour (10-minute in demo) threshold. IC §6 also confirms `providers.status` may transition to `DEPARTED` from "any" prior state, not just `ACTIVE`. As originally written, a silently-departing vetting provider would never be selected by this query, never marked `DEPARTED`, never have its escrow seized, and never free its assignment slot — the entire FR-065/vetting-departure code path in Sessions 9.1.3 and downstream would simply never execute. Corrected below to `WHERE status IN ('ACTIVE', 'VETTING')`; only the *downstream handling* (real-shard repair vs. synthetic-chunk soft-delete) differs by status, not the detection step itself.

#### Session 9.3.1 — Implement the departure detector loop

**PRECONDITIONS:**

- Session 9.1.1 complete (`EnqueueJob`, `Priority`/`TriggerType`)
- Session 9.1.3 complete (`IsVettingChunk`, `DeleteVettingChunksOnDeparture`)
- A `PenaliseFunc`-shaped callback is available to be injected at construction — the actual `internal/payment.Penalise` implementation is wired in by Milestone 12, Session 12.1.1, which is the only place in the codebase permitted to import both `internal/repair` and `internal/payment`

**TASK:**

1. Declare the injection point in `internal/repair/departure.go`:

```go
// PenaliseFunc is the shape of internal/payment.Penalise, injected by the
// caller so internal/repair never imports internal/payment (IC §9). Set to a
// real implementation only in Milestone 12, Session 12.1.1.
type PenaliseFunc func(ctx context.Context, providerID uuid.UUID, amountPaise int64, idempotencyKey string) error

// DepartureDetector periodically scans for silently-departed providers.
type DepartureDetector struct {
 db       *sql.DB
 profile  config.NetworkProfile
 penalise PenaliseFunc
}

func NewDepartureDetector(db *sql.DB, profile config.NetworkProfile, penalise PenaliseFunc) *DepartureDetector
```

1. Implement the detection loop:

```go
// Run polls at profile.PollingInterval (reused from the audit-scheduling
// cadence — no dedicated NetworkProfile field exists for this, and reusing
// PollingInterval keeps the detection-latency-to-threshold ratio roughly
// consistent between modes: ~24h:72h in production, ~2min:10min in demo).
// This choice is an inference, not a documented requirement — revisit if a
// tighter detection SLA is ever specified.
func (d *DepartureDetector) Run(ctx context.Context) {
 ticker := time.NewTicker(d.profile.PollingInterval)
 defer ticker.Stop()
 for {
  select {
  case <-ctx.Done():
   return
  case <-ticker.C:
   d.detectOnce(ctx)
  }
 }
}
```

1. The per-cycle detection query — **corrected to include VETTING**:

```sql
SELECT provider_id, status FROM providers
WHERE status IN ('ACTIVE', 'VETTING')
  AND last_heartbeat_ts < NOW() - $1  -- profile.DepartureThreshold
```

1. For each departed provider (within one transaction per provider):
   - Set `status = 'DEPARTED'`, `frozen = TRUE`, `departed_at = NOW()` — a valid transition from either prior status per IC §6 ("any → DEPARTED").
   - Call `d.penalise(ctx, providerID, sealedBalance, idempotencyKey)` where `idempotencyKey = SHA-256(providerID || "seizure" || departedAt)` (IC §5.8) — this is the only side effect requiring `internal/payment`, hence the callback injection.
   - **Branch on the *original* status** (captured before the UPDATE, not re-queried after): if it was `VETTING`, call `DeleteVettingChunksOnDeparture(ctx, db, providerID)` and enqueue **zero** repair jobs (FR-065). If it was `ACTIVE`, for each real chunk assignment call `IsVettingChunk()` (defensive — should always be `false` for an ACTIVE provider, but this check is what Invariant 6 requires as a hard gate) and then `EnqueueJob(ctx, db, profile, chunkID, segmentID, &providerID, TriggerSilentDeparture, availableShardCount)`.
   - Do **not** physically delete the `providers` row in either branch (DM §3 Invariant 3).

**Downstream note (not implemented here):** peer-blocking after departure (FR-036 — a departed provider that reconnects gets HTTP 403) is enforced by every authenticated REST endpoint checking `providers.status != 'DEPARTED'` (IC §3.3 error code `PROVIDER_DEPARTED`), not by this detector.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/repair/departure.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/repair/
  EXPECT: exit 0

THE_CORE_FIX_VETTING_INCLUDED:
  # Must select both statuses — this is the functional bug fix
  $ grep -c "status IN ('ACTIVE', 'VETTING')" internal/repair/departure.go
  EXPECT: >= 1
  $ grep -n "WHERE status = 'ACTIVE'" internal/repair/departure.go | grep -v "_test.go" \
      && echo "FAIL: detection query must also include VETTING (FR-065, ARCH §12)" \
      || echo "PASS"

PENALISE_IS_INJECTED_NOT_IMPORTED:
  $ grep -c "^type PenaliseFunc func(" internal/repair/departure.go
  EXPECT: 1
  $ grep -n "Vyomanaut_V2/internal/payment" internal/repair/departure.go \
      && echo "FAIL: internal/repair must not import internal/payment (IC §9)" || echo "PASS"

BRANCHES_ON_ORIGINAL_STATUS:
  $ grep -c "DeleteVettingChunksOnDeparture" internal/repair/departure.go
  EXPECT: >= 1
  $ grep -c "EnqueueJob" internal/repair/departure.go
  EXPECT: >= 1

NO_PHYSICAL_DELETE:
  $ grep -n "DELETE FROM providers" internal/repair/departure.go \
      && echo "FAIL: providers is never physically deleted (DM §3 Invariant 3)" || echo "PASS"

STATUS_TRANSITION_FIELDS_SET_TOGETHER:
  $ grep -c "status = 'DEPARTED'" internal/repair/departure.go
  EXPECT: >= 1
  $ grep -c "frozen = TRUE\|frozen = true" internal/repair/departure.go
  EXPECT: >= 1
  $ grep -c "departed_at = NOW()" internal/repair/departure.go
  EXPECT: >= 1

POLLING_INTERVAL_IS_PROFILE_DRIVEN:
  $ grep -c "profile.PollingInterval\|d.profile.PollingInterval" internal/repair/departure.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestDepartureDetector ./internal/repair/
  EXPECT: exit 0; tests include:
    TestDepartureDetectorCatchesActiveProviders         (ACTIVE provider past threshold -> detected, DEPARTED, EnqueueJob called)
    TestDepartureDetectorCatchesVettingProviders         (VETTING provider past threshold -> detected, DEPARTED, DeleteVettingChunksOnDeparture called, EnqueueJob NOT called)
    TestDepartureDetectorIgnoresRecentHeartbeats         (last_heartbeat_ts within threshold -> not selected)
    TestDepartureDetectorNeverPhysicallyDeletesRow       (providers row count unchanged before/after)
    TestDepartureDetectorCallsPenaliseWithSeizureIdempotencyKey

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/audit\|Vyomanaut_V2/internal/scoring" internal/repair/departure.go \
      && echo "FAIL: prohibited import" || echo "PASS"

VET:
  $ go vet ./internal/repair/
  EXPECT: exit 0; zero output
```

---

### Phase 9.4 — Replacement Provider Selection *(new — closes an MVP §8.2 gap)*

**Reference:** MVP §8.2 (`assignment.go` — "Power of Two Choices + ASN cap enforcement for replacement selection"), FR-045 (replacement ASN cap requirement), ADR-014 (20% ASN cap), ADR-005 (Power of Two Choices for original assignment — reused here for replacements)

> No session in the original plan owned `assignment.go`, despite `mvp.md §8.2` naming it explicitly and FR-045 requiring its behavior ("During repair, replacement providers must be selected with the same 20% ASN cap constraint that governs original chunk assignment"). Session 9.2.1 cannot correctly select a replacement provider without this — added here as its own phase since it precedes and feeds Session 9.2.1's upload step.

#### Session 9.4.1 — Implement replacement-provider selection

**PRECONDITIONS:**

- `internal/scoring` complete (Milestone 8) — needed to compare two candidate providers' reliability scores for Power of Two Choices (`scoring.GetScore`, read-only; `internal/repair` importing `internal/scoring` for read access is not one of the prohibited directions — only `internal/repair` importing `internal/payment`/`internal/audit`/`internal/scoring`-writes is at issue; a read-only score lookup mirrors how `internal/audit` and `internal/scoring` already independently query shared tables. If this reasoning doesn't hold, fall back to a direct SQL query against `mv_provider_scores` instead of importing `internal/scoring`.)

**TASK:**

In `internal/repair/assignment.go`:

```go
// SelectReplacementProvider chooses a new provider to receive the
// reconstructed shard for segmentID, excluding excludeProviderIDs (the
// departed/failed holder(s) and every other current holder of this
// segment's shards, so the same provider is never assigned two shards of one
// segment). Uses Power of Two Choices (ADR-005): draw two ACTIVE candidates
// at random from the vetted pool, pick the higher-scored one.
//
// The chosen candidate must not push any single ASN's share of this
// segment's TotalShards above floor(TotalShards * profile.ASNCapFraction)
// (FR-045, ADR-014) — the same 20% cap enforced at original assignment time.
// If the higher-scored candidate would violate the cap, fall back to the
// second candidate; if both would violate it, draw again (bounded retries)
// before returning ErrNoEligibleReplacement.
func SelectReplacementProvider(
 ctx context.Context,
 db *sql.DB,
 profile config.NetworkProfile,
 segmentID uuid.UUID,
 excludeProviderIDs []uuid.UUID,
) (uuid.UUID, error)

var ErrNoEligibleReplacement = errors.New("repair: no ASN-cap-eligible replacement provider found after bounded retries")
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/repair/assignment.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/repair/
  EXPECT: exit 0

FUNCTION_SIGNATURE:
  $ grep -c "^func SelectReplacementProvider(" internal/repair/assignment.go
  EXPECT: 1

POWER_OF_TWO_CHOICES:
  $ grep -c "two.*candidates\|Power of Two\|two ACTIVE candidates" internal/repair/assignment.go
  EXPECT: >= 1

ASN_CAP_ENFORCED:
  $ grep -c "ASNCapFraction\|20%\|ASN cap" internal/repair/assignment.go
  EXPECT: >= 1

EXCLUDES_CURRENT_HOLDERS:
  $ grep -c "excludeProviderIDs" internal/repair/assignment.go
  EXPECT: >= 2

SENTINEL_ERROR:
  $ grep -c "ErrNoEligibleReplacement" internal/repair/assignment.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestSelectReplacementProvider ./internal/repair/
  EXPECT: exit 0; tests include:
    TestSelectReplacementProviderPicksHigherScored        (of two candidates, higher scoring.GetScore composite wins)
    TestSelectReplacementProviderRejectsASNCapViolation    (higher-scored candidate would exceed cap -> falls back to lower-scored candidate)
    TestSelectReplacementProviderExcludesCurrentHolders    (never returns a providerID already holding a shard of this segment)
    TestSelectReplacementProviderReturnsErrAfterBoundedRetries (all candidates violate cap -> ErrNoEligibleReplacement, not an infinite loop)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/payment\|Vyomanaut_V2/internal/audit" internal/repair/assignment.go \
      && echo "FAIL: prohibited import" || echo "PASS"

VET:
  $ go vet ./internal/repair/
  EXPECT: exit 0; zero output
```

---

### Phase 9.5 — Repair Package Tests *(new — closes an MVP §8.2 gap)*

**Reference:** MVP §8.2 (`repair_test.go` — "priority ordering, ASN cap, emergency floor, vetting exclusion")

#### Session 9.5.1 — Implement `repair_test.go`

**PRECONDITIONS:**

- Sessions 9.1.1 through 9.4.1 all complete
- CI Postgres instance available with the Milestone 4 migration applied

**TASK:**

Create `internal/repair/repair_test.go` covering the four areas `mvp.md §8.2` names, at the integration level:

| Test name | What it validates |
|---|---|
| `TestPriorityOrderingEndToEnd` | Enqueue one job of each trigger type in reverse-priority order; `DequeueNextJob` returns them EMERGENCY → PERMANENT_DEPARTURE → PRE_WARNING regardless of insertion order — the specific bug this revision fixed |
| `TestASNCapAcrossRepairAndOriginalAssignment` | Simulate a segment already at the ASN cap boundary; confirm `SelectReplacementProvider` never pushes any ASN over `floor(TotalShards * 0.20)`, matching FR-045's "same cap as original assignment" |
| `TestEmergencyFloorBypassesQueueOrder` | A `PRE_WARNING` job is already queued; an `EMERGENCY` job for a different chunk is enqueued after it; `DequeueNextJob` returns the EMERGENCY job first despite arriving later |
| `TestVettingExclusionEndToEnd` | A VETTING provider with only synthetic chunks crosses `DepartureThreshold`; confirm `repair_jobs` gains **zero** new rows and `chunk_assignments` for that provider all become `DELETED` — exercising Sessions 9.1.3 and 9.3.1 together |

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/repair/repair_test.go && echo PASS || echo FAIL

TEST_NAMES_PRESENT:
  $ grep -c "^func TestPriorityOrderingEndToEnd\|^func TestASNCapAcrossRepairAndOriginalAssignment\|^func TestEmergencyFloorBypassesQueueOrder\|^func TestVettingExclusionEndToEnd" \
      internal/repair/repair_test.go
  EXPECT: 4

ALL_TESTS_PASS:
  $ go test -race -count=1 -v ./internal/repair/
  EXPECT: exit 0; every test in this file and every earlier M9 session's UNIT_TESTS block shows "--- PASS:"

EMERGENCY_FIX_SPECIFICALLY_EXERCISED:
  $ grep -A15 "^func TestEmergencyFloorBypassesQueueOrder" internal/repair/repair_test.go | grep -c "PriorityEmergency\|'EMERGENCY'"
  EXPECT: >= 1

VETTING_FIX_SPECIFICALLY_EXERCISED:
  $ grep -A20 "^func TestVettingExclusionEndToEnd" internal/repair/repair_test.go | grep -c "VETTING"
  EXPECT: >= 1
  $ grep -A20 "^func TestVettingExclusionEndToEnd" internal/repair/repair_test.go | grep -c "COUNT(\*)\|len(jobs) == 0\|zero"
  EXPECT: >= 1

CI_RACE_GATE (M0 CI check 4 must stay green with this package included):
  $ go test -race -count=1 ./...
  EXPECT: exit 0

FULL_PACKAGE_BUILD:
  $ go build ./internal/repair/
  $ go vet ./internal/repair/
  EXPECT: both exit 0
```

---

## Milestone 10 — Payment System (`internal/payment`)

**Deliverable:** `PaymentProvider` interface, DB-backed mock provider, Razorpay live implementation, append-only provider escrow ledger, append-only owner escrow ledger, monthly release computation, seizure, owner withdrawal. Zero float arithmetic.

**Package import constraints (IC §9):** `internal/payment` must **NOT** import `internal/repair` or `internal/p2p` — this is one of IC §9's explicitly tabulated rows, unlike Milestone 9's `internal/repair` (which had to be inferred). `internal/payment` **MAY** import `internal/scoring` for read-only score lookups feeding the release multiplier (Phase 10.4) — nothing in IC §9 prohibits this direction, and `internal/scoring` is documented as read-only against the database, so this is a one-way, side-effect-free dependency, not a cycle.

**Reference:** IC §5.8 (full package contract, INVARIANT block), DM §3 Invariant 2 (append-only escrow ledger) and Invariant 4 (integer paise only), DM §4.8 (`escrow_events` schema), DM §4.9 (`owner_escrow_events` schema — not covered by any session in the original plan; see Phase 10.5), DM §7 (`mv_provider_escrow_balance`, `mv_owner_escrow_balance` — both including the `REVERSAL`/`GREATEST(...,0)` amendments), IC §7 (Razorpay webhook contracts), IC §11 (float arithmetic forbidden in `internal/payment/`), IC §6 (`escrow_events` DML — INSERT-only), FR-046–FR-052 (payment system FRs), FR-059 (owner withdrawal), FR-014/FR-021 (owner balance checks — depend on `owner_escrow_events`), ADR-011 (Razorpay abstraction), ADR-012 (payment-per-audit basis), ADR-016 (payment DB schema, PN-counter ledger design), ADR-024 (economic mechanism — release multiplier, dual-window hold), MVP §5.4 (toggle map), MVP §8.4 (CI check 6)

**Milestone review notes:**

- Session 10.4.1 must call `scoring.GetScoreFromPrimary()` (Milestone 8), not `scoring.GetScore()` — DM §7's staleness rule for payment decisions depends on this distinction, which the original session text never mentioned.
- `TestNoFloatArithmetic` (Session 10.4.2, CI check 6) and the grep-based float check (`scripts/ci/grep_checks.sh`, added in Session 0.2.2, CI check 9) are two *independent, intentionally redundant* enforcement mechanisms — the grep check is a fast textual pattern match across all file types including SQL; `TestNoFloatArithmetic` is a semantically-aware `go/ast` parse of Go source specifically. Neither replaces the other.
- Session 10.2.1 declares `EscrowEventType`; it also closes the `.golangci.yml` placeholder `# M10: internal/payment.EscrowEventType` left by Session 0.2.1.

---

### Phase 10.1 — PaymentProvider Interface & Mock

**Reference:** IC §5.8 (`PaymentProvider` interface), FR-059 (owner withdrawal — extends the interface, see flagged note), MVP §6.1 CR-10, MVP §7.7

> **Flagged — ambiguous parameter.** `InitiateEscrow`'s third parameter, `contractID uuid.UUID`, is used in IC §5.8 without being defined anywhere else in scope — no document in this review's context (`architecture.md`, `requirements.md`, `data-model.md`) uses the term "contract" in the escrow/payment context. Implement it as an opaque, caller-supplied correlation ID (e.g. the `file_id` being paid for, or a deposit-request ID) rather than guessing at a specific meaning; flag for clarification against the original design intent before this ships.
>
> **Flagged — no owner-withdrawal method exists anywhere in the interface contract.** FR-059 requires it and Milestone 11 Session 11.5.6 assumes it exists. Added as `WithdrawOwnerEscrow` below, structured to parallel `ReleaseEscrow` as closely as possible.

#### Session 10.1.1 — Define `PaymentProvider` interface (including owner withdrawal)

**PRECONDITIONS:**

- `internal/payment/doc.go` exists (stub from Session 0.1.3)

**TASK:**

Create `internal/payment/provider.go`:

```go
// PaymentProvider abstracts over Razorpay Route + Smart Collect 2.0 +
// RazorpayX (IC §5.8, ADR-011). All amountPaise parameters and return values
// are int64 paise (₹1 = 100 paise) — passing a float64/float32 anywhere in
// this package is a calling-contract violation and panics in debug builds
// (DM §3 Invariant 4, IC §11).
type PaymentProvider interface {
 // InitiateEscrow creates a virtual UPI address for a data owner and
 // records the expected deposit amount. contractID is an opaque
 // caller-supplied correlation ID (e.g. a file_id or deposit-request ID) —
 // see the flagged ambiguity note above; this package does not interpret it.
 InitiateEscrow(ctx context.Context, ownerID uuid.UUID, amountPaise int64, contractID uuid.UUID) (vpa string, qrURL string, err error)

 // ReleaseEscrow initiates a monthly payout to a provider's Razorpay
 // Linked Account. idempotencyKey = SHA-256(providerID || auditPeriodID),
 // hex-encoded, mandatory per Razorpay's X-Payout-Idempotency requirement
 // (ADR-012).
 ReleaseEscrow(ctx context.Context, providerID uuid.UUID, amountPaise int64, auditPeriodID uuid.UUID, idempotencyKey string) error

 // Penalise seizes a departed provider's rolling escrow window (ADR-024 §5).
 // idempotencyKey = SHA-256(providerID || "seizure" || departedAt).
 Penalise(ctx context.Context, providerID uuid.UUID, amountPaise int64, idempotencyKey string) error

 // GetBalance returns a provider's current balance, always
 // SUM(DEPOSIT + REVERSAL) - SUM(RELEASE + SEIZURE); never negative.
 GetBalance(ctx context.Context, providerID uuid.UUID) (int64, error)

 // WithdrawOwnerEscrow pays out a data owner's available balance to their
 // UPI-linked bank account (FR-059). idempotencyKey =
 // SHA-256(ownerID || withdrawalRequestID), hex-encoded. Must be rejected
 // by the caller (Milestone 11, Session 11.5.6) while any upload is
 // in-flight for this owner — this function itself does not check that.
 WithdrawOwnerEscrow(ctx context.Context, ownerID uuid.UUID, amountPaise int64, idempotencyKey string) (payoutID string, err error)
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/payment/provider.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/payment/
  EXPECT: exit 0

INTERFACE_METHODS:
  $ grep -c "InitiateEscrow\|ReleaseEscrow\|Penalise\|GetBalance\|WithdrawOwnerEscrow" internal/payment/provider.go
  EXPECT: >= 5

ALL_AMOUNTS_ARE_INT64:
  $ grep -c "amountPaise int64" internal/payment/provider.go
  EXPECT: >= 4
  $ grep -n "float64\|float32\|FLOAT\|DECIMAL\|NUMERIC" internal/payment/provider.go \
      && echo "FAIL: no float type may appear anywhere in internal/payment/ (Invariant 4, IC §11)" \
      || echo "PASS"

WITHDRAWOWNERESCROW_PRESENT:
  # The gap-fix: FR-059 has no home without this
  $ grep -c "^\tWithdrawOwnerEscrow(ctx context.Context, ownerID uuid.UUID, amountPaise int64, idempotencyKey string) (payoutID string, err error)" \
      internal/payment/provider.go
  EXPECT: 1

CONTRACTID_AMBIGUITY_DOCUMENTED:
  $ grep -c "opaque\|caller-supplied correlation ID" internal/payment/provider.go
  EXPECT: >= 1

VET:
  $ go vet ./internal/payment/
  EXPECT: exit 0; zero output
```

#### Session 10.1.2 — Implement a DB-backed mock payment provider

**PRECONDITIONS:**

- Session 10.1.1 complete
- Session 10.2.1 complete (`InsertEscrowEvent`) — the mock depends on this, so build order matters here even though session numbering doesn't reflect it; if implementing strictly in order, stub `InsertEscrowEvent`'s signature first and fill in the body in 10.2.1

> **Flagged and corrected — the original mock was specified as in-memory, which contradicts two requirements cited in the same original session.** MVP §6.1 CR-10 says the mock "must implement the interface fully, not bypass it... direct DB writes from a mock break the abstraction" (i.e., an *in-memory* implementation is exactly the anti-pattern CR-10 warns against — the mock must go through the same DB-backed ledger the real provider uses). MVP §7.7 says "the DB constraint is the enforcement... the mock does not need additional [idempotency] logic" — only true if the mock's writes actually reach the real `escrow_events`/`owner_escrow_events` tables and their `UNIQUE(idempotency_key)` constraints. Fixed below: the mock's ledger-writing methods call the same `InsertEscrowEvent`/`InsertOwnerEscrowEvent` (Phase 10.5) as the real Razorpay provider. Only the *external Razorpay HTTP calls* — which have no database representation of their own — are mocked out.

**TASK:**

Create `internal/payment/mock.go`:

```go
// MockProvider implements PaymentProvider for demo mode (profile.PaymentMode
// == "mock") and for tests. It mocks only the external Razorpay HTTP calls;
// every ledger write goes through the same InsertEscrowEvent /
// InsertOwnerEscrowEvent path the real RazorpayProvider uses, so the DB-level
// UNIQUE(idempotency_key) constraint enforces idempotency identically for
// both (MVP §7.7) — this is also why the mock "implements the interface
// fully" rather than bypassing it (MVP §6.1 CR-10).
type MockProvider struct {
 db *sql.DB
}

func NewMockProvider(db *sql.DB) *MockProvider

// InitiateEscrow returns a synthetic vpa/qrURL (no real Razorpay call) and,
// because there is no real Razorpay webhook to wait for in demo mode,
// immediately calls InsertOwnerEscrowEvent(..., OwnerDeposit, ...) — matching
// MVP's own Feature Gap Table row F-09 ("CLI command deposits into mock
// ledger" — no async webhook in demo mode).
func (m *MockProvider) InitiateEscrow(ctx context.Context, ownerID uuid.UUID, amountPaise int64, contractID uuid.UUID) (string, string, error)

// ReleaseEscrow calls InsertEscrowEvent(..., EscrowRelease, ...) directly —
// no real Razorpay payout call, but a real row with a real idempotency
// constraint.
func (m *MockProvider) ReleaseEscrow(ctx context.Context, providerID uuid.UUID, amountPaise int64, auditPeriodID uuid.UUID, idempotencyKey string) error

// Penalise calls InsertEscrowEvent(..., EscrowSeizure, ...) directly.
func (m *MockProvider) Penalise(ctx context.Context, providerID uuid.UUID, amountPaise int64, idempotencyKey string) error

// GetBalance queries mv_provider_escrow_balance — the SAME view the real
// provider's balance queries use. There is no separate in-memory balance
// state to keep in sync.
func (m *MockProvider) GetBalance(ctx context.Context, providerID uuid.UUID) (int64, error)

// WithdrawOwnerEscrow calls InsertOwnerEscrowEvent(..., OwnerWithdrawal, ...)
// directly and returns a synthetic payoutID.
func (m *MockProvider) WithdrawOwnerEscrow(ctx context.Context, ownerID uuid.UUID, amountPaise int64, idempotencyKey string) (string, error)
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/payment/mock.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/payment/
  EXPECT: exit 0

MOCK_SATISFIES_INTERFACE:
  $ grep -c "var _ PaymentProvider = (\*MockProvider)(nil)" internal/payment/mock.go
  EXPECT: 1

THE_CORE_FIX_DB_BACKED_NOT_IN_MEMORY:
  # Must call the shared ledger functions, must not maintain its own map/slice of events
  $ grep -c "InsertEscrowEvent(\|InsertOwnerEscrowEvent(" internal/payment/mock.go
  EXPECT: >= 3
  $ grep -n "map\[string\]\|make(map\[uuid.UUID\]" internal/payment/mock.go \
      && echo "FAIL: mock must not keep a separate in-memory ledger — writes go through the shared DB path" \
      || echo "PASS"

GETBALANCE_USES_SHARED_VIEW:
  $ grep -c "mv_provider_escrow_balance" internal/payment/mock.go
  EXPECT: >= 1

DEMO_DEPOSIT_IS_SYNCHRONOUS:
  # No real webhook in mock mode -> InitiateEscrow itself must credit the ledger
  $ grep -A15 "^func (m \*MockProvider) InitiateEscrow" internal/payment/mock.go | grep -c "InsertOwnerEscrowEvent"
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestMockProvider ./internal/payment/
  EXPECT: exit 0; tests include:
    TestMockProviderInitiateEscrowCreditsLedgerImmediately  (deposit ledger row exists right after the call, no webhook needed)
    TestMockProviderReleaseEscrowWritesRealRow              (escrow_events gains a real RELEASE row)
    TestMockProviderIdempotencyEnforcedByDBConstraint       (duplicate idempotencyKey -> ErrDuplicateIdempotencyKey, not silently accepted, and not caught by mock-side logic)
    TestMockProviderGetBalanceMatchesRealView               (GetBalance result equals a direct query against mv_provider_escrow_balance)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/p2p" internal/payment/mock.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/payment/
  EXPECT: exit 0; zero output
```

---

### Phase 10.2 — Escrow Ledger

**Reference:** IC §5.8 (`InsertEscrowEvent`, `EscrowEventType`, `ErrDuplicateIdempotencyKey`), DM §4.8, DM §3 Invariant 2 (append-only), IC §6 (`escrow_events` — INSERT only, no UPDATE/DELETE policy)

#### Session 10.2.1 — Implement `InsertEscrowEvent()`; declare `EscrowEventType`

**PRECONDITIONS:**

- Session 10.1.1 complete
- Migration from Session 4.4.3 applied (`escrow_events` table, its RSP from Session 4.6.2)

**TASK:**

1. Create `internal/payment/ledger.go`:

```go
type EscrowEventType string

const (
 EscrowDeposit  EscrowEventType = "DEPOSIT"
 EscrowRelease  EscrowEventType = "RELEASE"
 EscrowSeizure  EscrowEventType = "SEIZURE"
 EscrowReversal EscrowEventType = "REVERSAL"
)

// InsertEscrowEvent appends one row to escrow_events — the ONLY permitted
// write to this table (DM §3 Invariant 2). Never accept a float anywhere in
// the call chain; amountPaise is int64 by construction, not by a runtime check.
//
// Pre-conditions:
//   - amountPaise > 0 (sign is implied by eventType, per DM §4.8)
//   - idempotencyKey is globally unique (enforced by the DB UNIQUE constraint,
//     not by this function)
// Error semantics:
//   - ErrDuplicateIdempotencyKey on UNIQUE violation — idempotent retry,
//     caller should treat as success.
func InsertEscrowEvent(ctx context.Context, db *sql.DB, providerID uuid.UUID, eventType EscrowEventType, amountPaise int64, idempotencyKey string, auditPeriodID *uuid.UUID) error

var ErrDuplicateIdempotencyKey = errors.New("payment: idempotency key already exists")
```

1. **Close the `.golangci.yml` placeholder from Session 0.2.1:** replace `# M10: internal/payment.EscrowEventType` with an active entry now that `EscrowEventType` and its four values are declared.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/payment/ledger.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/payment/
  EXPECT: exit 0

ESCROWEVENTTYPE_HAS_FOUR_VALUES:
  $ grep -c "EscrowDeposit\|EscrowRelease\|EscrowSeizure\|EscrowReversal" internal/payment/ledger.go
  EXPECT: 4

FUNCTION_SIGNATURE:
  $ grep -c "^func InsertEscrowEvent(ctx context.Context, db \*sql.DB, providerID uuid.UUID, eventType EscrowEventType, amountPaise int64, idempotencyKey string, auditPeriodID \*uuid.UUID) error" \
      internal/payment/ledger.go
  EXPECT: 1

INT64_ONLY:
  $ grep -c "amountPaise int64" internal/payment/ledger.go
  EXPECT: >= 1
  $ grep -n "float64\|float32\|FLOAT\|DECIMAL\|NUMERIC" internal/payment/ledger.go \
      && echo "FAIL: Invariant 4 violation" || echo "PASS"

SENTINEL_ERROR:
  $ grep -c "ErrDuplicateIdempotencyKey" internal/payment/ledger.go
  EXPECT: >= 1

GOLANGCI_PLACEHOLDER_CLOSED:
  $ grep -c "# M10: internal/payment.EscrowEventType" .golangci.yml
  EXPECT: 0
  $ grep -c "internal/payment\.EscrowEventType" .golangci.yml
  EXPECT: 1

UNIT_TESTS:
  $ go test -v -run TestInsertEscrowEvent ./internal/payment/
  EXPECT: exit 0; tests include:
    TestInsertEscrowEventSucceeds
    TestInsertEscrowEventDuplicateIdempotencyKey  (second insert with same key -> ErrDuplicateIdempotencyKey, exactly one row exists)
    TestInsertEscrowEventOnlyINSERTNoUpdateDelete  (no UPDATE/DELETE statement anywhere in this file)

NEGATIVE_CHECKS:
  $ grep -n "UPDATE escrow_events\|DELETE FROM escrow_events" internal/payment/ledger.go \
      && echo "FAIL: escrow_events is append-only (DM §3 Invariant 2)" || echo "PASS"
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/p2p" internal/payment/ledger.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/payment/
  EXPECT: exit 0; zero output
```

---

### Phase 10.3 — Razorpay Implementation

**Reference:** IC §7 (all three webhook contracts), IC §3.3 (`RAZORPAY_UNAVAILABLE` error code), DM §4.8 (`escrow_events`), DM §4.9 (`owner_escrow_events` — the corrected target for the deposit webhook), ARCH §17 (Smart Collect 2.0 is owner-scoped; Route is provider-scoped)

> **Flagged and corrected — the single most important fix in this milestone.** IC §7.1's given SQL for `virtual_account.payment.captured` inserts into `escrow_events` using an owner-side value in the `provider_id` column:
>
> ```sql
> INSERT INTO escrow_events (
>     event_id, provider_id, event_type, amount_paise,
>     audit_period_id, idempotency_key, created_at
> ) VALUES (
>     gen_random_uuid(), <owner_escrow_account_id>, 'DEPOSIT', ...
> )
> ```
>
> `escrow_events.provider_id` is `NOT NULL REFERENCES providers(provider_id)` (DM §4.8). Smart Collect 2.0 deposits are unambiguously owner-side (ARCH §17: "each data owner is assigned a virtual UPI ID... The microservice credits the data owner's escrow account"; DM §8.1: `owners.smart_collect_vpa`). DM §4.9 exists *specifically* because "the owners table has no balance information" and a separate ledger was needed — inserting an owner's deposit into the provider ledger either violates the foreign key outright or, if somehow forced through, silently defeats the entire reason `owner_escrow_events` was created. **Corrected below:** target `owner_escrow_events`, column `owner_id`, `event_type = 'DEPOSIT'` using the `owner_escrow_event_type` enum (DM §4.9), not `escrow_event_type`.

#### Session 10.3.1 — Implement Razorpay webhook handlers

**PRECONDITIONS:**

- Session 10.2.1 complete (`InsertEscrowEvent`)
- Session 10.5.1 complete (`InsertOwnerEscrowEvent`) — required for the corrected deposit handler below

**TASK:**

Create `internal/payment/razorpay.go`. All three handlers verify `X-Razorpay-Signature` before any DB write.

1. **`virtual_account.payment.captured` (IC §7.1, corrected) — owner deposit:**

```sql
-- CORRECTED target table and column (owner_escrow_events, not escrow_events)
INSERT INTO owner_escrow_events (
    event_id, owner_id, event_type, amount_paise,
    idempotency_key, created_at
) VALUES (
    gen_random_uuid(),
    <owner_id_from_vpa_lookup>,
    'DEPOSIT',
    <amount_paise_from_webhook>,  -- ALREADY in paise; do NOT multiply
    SHA256('deposit' || <owner_id_from_vpa_lookup> || <razorpay_payment_id>),
    NOW()
)
ON CONFLICT (idempotency_key) DO NOTHING;
```

   `amount_paise` from `payload.payment.entity.amount` is already in paise per Razorpay's convention — do not multiply. Look up `owner_id` via the Smart Collect VPA-to-owner mapping (`owners.smart_collect_vpa`), not via any provider table. Idempotency key combines the "deposit" domain-separation prefix (IC §7.1's original convention) with `owner_id` (DM §4.9's convention) and the Razorpay payment ID, harmonizing the two documents' slightly different formats into one unambiguous 64-hex-char key.

1. **`payout.reversed` (IC §7.2) — provider reversal, unchanged, already correctly scoped to `escrow_events`/`provider_id`:**

```sql
INSERT INTO escrow_events (
    event_id, provider_id, event_type, amount_paise,
    audit_period_id, idempotency_key, created_at
) VALUES (
    gen_random_uuid(), <provider_id>, 'REVERSAL', <amount_paise>,
    <original_audit_period_id>,
    SHA256('reversal' || <original_idempotency_key>), NOW()
)
ON CONFLICT (idempotency_key) DO NOTHING;
```

1. **`account.created` (IC §7.3) — provider Linked Account cooling, unchanged, already correctly scoped:**

```sql
UPDATE providers
SET razorpay_linked_account_id = <account_id>,
    razorpay_cooling_until = NOW() + $1  -- profile.RazorpayCoolingPeriod, NOT hardcoded 24h (MVP §5.4)
WHERE provider_id = <provider_id_from_notes>
  AND razorpay_linked_account_id IS NULL;
```

**UPI Collect prohibition (IC §11, NFR-029):** any call to a Razorpay Collect API path is forbidden; all deposit flows use UPI Intent or QR only (deprecated by NPCI 28 February 2026).

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/payment/razorpay.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/payment/
  EXPECT: exit 0

THE_CORE_FIX_DEPOSIT_TARGETS_OWNER_TABLE:
  # The critical bug fix: deposit webhook must write owner_escrow_events, never escrow_events
  $ grep -A8 "payment.captured\|payment_captured" internal/payment/razorpay.go | grep -c "owner_escrow_events"
  EXPECT: >= 1
  $ grep -B2 -A8 "'DEPOSIT'" internal/payment/razorpay.go | grep -c "INSERT INTO escrow_events"
  EXPECT: 0

DEPOSIT_USES_OWNER_ID_NOT_PROVIDER_ID:
  $ grep -A10 "owner_escrow_events" internal/payment/razorpay.go | grep -c "owner_id"
  EXPECT: >= 1

AMOUNT_NOT_MULTIPLIED:
  $ grep -c "already in paise\|do NOT multiply\|do not multiply" internal/payment/razorpay.go
  EXPECT: >= 1

SIGNATURE_VERIFIED_BEFORE_WRITE:
  $ grep -c "X-Razorpay-Signature" internal/payment/razorpay.go
  EXPECT: >= 1

REVERSAL_STAYS_ON_PROVIDER_TABLE:
  # This one was already correct — must not regress
  $ grep -A8 "payout.reversed\|payout_reversed" internal/payment/razorpay.go | grep -c "escrow_events"
  EXPECT: >= 1
  $ grep -A8 "payout.reversed\|payout_reversed" internal/payment/razorpay.go | grep -c "owner_escrow_events"
  EXPECT: 0

COOLING_PERIOD_IS_PROFILE_DRIVEN:
  $ grep -c "profile.RazorpayCoolingPeriod" internal/payment/razorpay.go
  EXPECT: >= 1
  $ grep -n "INTERVAL '24 hours'" internal/payment/razorpay.go \
      && echo "FAIL: cooling period must be profile-variable, not hardcoded 24h (MVP §5.4)" \
      || echo "PASS"

IDEMPOTENCY_GUARD_ON_ACCOUNT_CREATED:
  $ grep -c "razorpay_linked_account_id IS NULL" internal/payment/razorpay.go
  EXPECT: >= 1

NO_UPI_COLLECT:
  $ grep -in "upi/collect\|collect/request\|UpiCollect" internal/payment/razorpay.go \
      && echo "FAIL: UPI Collect is prohibited, deprecated by NPCI (NFR-029)" || echo "PASS"

UNIT_TESTS:
  $ go test -v -run TestRazorpayWebhooks ./internal/payment/
  EXPECT: exit 0; tests include:
    TestDepositWebhookCreditsOwnerLedgerNotProviderLedger  (owner_escrow_events gains a row; escrow_events does not)
    TestDepositWebhookIdempotentOnRedelivery               (same webhook delivered twice -> one row)
    TestDepositWebhookRejectsBadSignature                  (missing/invalid X-Razorpay-Signature -> HTTP 400, no DB write)
    TestReversalWebhookInsertsReversalEvent
    TestAccountCreatedSetsProfileDrivenCoolingPeriod        (demo profile -> cooling_until == NOW(); prod profile -> cooling_until == NOW()+24h)
    TestAccountCreatedIdempotentOnRedelivery

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/p2p" internal/payment/razorpay.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/payment/
  EXPECT: exit 0; zero output
```

---

### Phase 10.4 — Release Computation & Seizure

**Reference:** FR-049, FR-050 (release multiplier, dual-window trigger), IC §5.8 (`ReleaseEscrow`, `Penalise`), DM §7 (`mv_provider_scores` staleness requirement — "must be within 60 minutes"), MVP §5.4 (release computation cycle: calendar in prod, ticker in demo), DM §4.2 (`frozen` column)

> **Flagged — must use `GetScoreFromPrimary`, not `GetScore`.** DM §7's own CRITICAL note ("scores_as_of must be within 60 minutes before this view is used for release multiplier computation... Stale scores produce wrong payments") is exactly the scenario the Milestones 7–8 revision's `GetScoreFromPrimary` (and its `ScoresAsOf` field) exists to guard. The original Session 10.4.1 text never mentioned this distinction. Fixed below.

#### Session 10.4.1 — Implement monthly release computation

**PRECONDITIONS:**

- Session 10.2.1 complete (`InsertEscrowEvent`, `EscrowEventType`)
- `internal/scoring` complete (Milestone 8, as revised) — specifically `scoring.GetScoreFromPrimary` and `ProviderScore.ScoresAsOf`/`DualWindowFlag`

**TASK:**

In `internal/payment/release.go`:

```go
// ComputeMonthlyRelease runs the release computation for every provider with
// release_computed = FALSE in the current audit period (FR-048).
//
// 1. Fetch the score via scoring.GetScoreFromPrimary — NOT scoring.GetScore.
//    Payment decisions require the freshest available data (DM §7 CRITICAL
//    note); a replica-routed read could be stale enough to misapply the
//    multiplier. If score.ScoresAsOf is more than 60 minutes old, skip this
//    provider for the current cycle and retry next tick rather than pay out
//    against stale data.
// 2. Compute the release multiplier from Score30d (FR-049):
//      >= 0.95            -> 1.00 (full release)
//      0.80 - 0.94        -> 0.75
//      0.65 - 0.79        -> 0.50
//      <  0.65            -> 0.00 (hold in full; rolls to next month, FR-049)
// 3. If score.DualWindowFlag is true (FR-050: Score30d - Score7d > 0.20),
//    use the lower of the multiplier computed from Score30d and the
//    multiplier that Score7d would produce under the same table.
// 4. Call ReleaseEscrow(ctx, providerID, releaseAmountPaise, auditPeriodID,
//    idempotencyKey) where idempotencyKey = SHA-256(providerID || auditPeriodID).
// 5. Set audit_periods.release_computed = TRUE.
//
// Amounts held for more than 90 rolling days without release are flagged for
// operations-team review, not auto-seized (FR-049).
//
// Trigger cadence: demo mode fires on profile.ReleaseComputationInterval
// (a ticker); production fires on the 23rd of the calendar month
// (profile.ReleaseComputationInterval == 0 means calendar-driven — MVP §5.4).
func ComputeMonthlyRelease(ctx context.Context, db *sql.DB, primaryDB *sql.DB, profile config.NetworkProfile) error
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/payment/release.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/payment/
  EXPECT: exit 0

USES_PRIMARY_READ_NOT_REPLICA:
  # The gap-fix: must call GetScoreFromPrimary, never GetScore, for payment decisions
  $ grep -c "scoring.GetScoreFromPrimary\|GetScoreFromPrimary(" internal/payment/release.go
  EXPECT: >= 1
  $ grep -n "scoring.GetScore(" internal/payment/release.go \
      && echo "FAIL: release computation must use GetScoreFromPrimary, not GetScore (DM §7)" \
      || echo "PASS"

STALENESS_CHECKED:
  $ grep -c "ScoresAsOf\|60.*Minute\|60 \* time.Minute" internal/payment/release.go
  EXPECT: >= 1

MULTIPLIER_TABLE_CORRECT:
  $ grep -c "0.95\|0.80\|0.65" internal/payment/release.go
  EXPECT: >= 3

DUAL_WINDOW_APPLIED:
  $ grep -c "DualWindowFlag" internal/payment/release.go
  EXPECT: >= 1

IDEMPOTENCY_KEY_FORMAT:
  $ grep -c "providerID || auditPeriodID\|providerID.*auditPeriodID" internal/payment/release.go
  EXPECT: >= 1

RELEASE_COMPUTED_FLAG_SET:
  $ grep -c "release_computed = TRUE\|release_computed = true" internal/payment/release.go
  EXPECT: >= 1

TICKER_VS_CALENDAR_BRANCH:
  $ grep -c "ReleaseComputationInterval == 0\|ReleaseComputationInterval > 0" internal/payment/release.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestComputeMonthlyRelease ./internal/payment/
  EXPECT: exit 0; tests include:
    TestComputeMonthlyReleaseFullMultiplierAboveThreshold   (score30d=0.97 -> multiplier 1.00)
    TestComputeMonthlyReleasePartialMultiplierTiers          (each of the four bands produces the documented multiplier)
    TestComputeMonthlyReleaseDualWindowUsesLowerMultiplier   (DualWindowFlag=true, score7d worse -> lower multiplier applied)
    TestComputeMonthlyReleaseSkipsStaleScore                 (ScoresAsOf > 60min old -> provider skipped this cycle, no ReleaseEscrow call)
    TestComputeMonthlyReleaseIdempotentOnRerun               (running twice for the same audit period -> exactly one RELEASE event)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/p2p" internal/payment/release.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/payment/
  EXPECT: exit 0; zero output
```

#### Session 10.4.2 — Implement `TestNoFloatArithmetic` (CI check 6)

**PRECONDITIONS:**

- All prior `internal/payment` sessions in this milestone complete (this test parses the entire package)

**TASK:**

Create `internal/payment/payment_test.go` with `TestNoFloatArithmetic` (CI check 6, distinct from and complementary to CI check 9's grep-based scan added in Session 0.2.2 — see the milestone review note above; this is not a duplicate). Parse every `.go` file in `internal/payment/` with `go/ast` and fail if `float64`, `float32`, `FLOAT`, `DECIMAL`, or `NUMERIC` appears as an identifier or type in any expression. This test itself, and the CI gate around it, may never be disabled or weakened by any future PR (IC §11).

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/payment/payment_test.go && echo PASS || echo FAIL

COMPILE_AND_RUN:
  $ go test -v -run TestNoFloatArithmetic ./internal/payment/
  EXPECT: exit 0; output contains "--- PASS: TestNoFloatArithmetic"

USES_GO_AST_NOT_JUST_GREP:
  $ grep -c "go/ast\|go/parser" internal/payment/payment_test.go
  EXPECT: >= 1

CHECKS_ALL_FIVE_PATTERNS:
  $ grep -c "float64\|float32\|\"FLOAT\"\|\"DECIMAL\"\|\"NUMERIC\"" internal/payment/payment_test.go
  EXPECT: >= 5

SELF_TEST_CATCHES_A_DELIBERATE_VIOLATION:
  # Inject a float into a scratch copy and confirm the test fails on it
  $ cp internal/payment/ledger.go /tmp/ledger_violation.go
  $ echo 'var _ float64 = 1.0' >> /tmp/ledger_violation.go
  $ go run internal/payment/payment_test.go /tmp/ledger_violation.go 2>&1 | grep -qi "float" \
      && echo "PASS: violation detected" || echo "FAIL: check did not catch injected float"
  $ rm /tmp/ledger_violation.go

DOES_NOT_DUPLICATE_OR_REPLACE_CI_CHECK_9:
  # Both mechanisms must coexist — this test does not remove the grep check from M0
  $ grep -c "NO_FLOAT_PAYMENT" scripts/ci/grep_checks.sh
  EXPECT: 1

CI_GATE_CANNOT_BE_WEAKENED (documentation check, not a runtime assertion):
  $ grep -c "may not be disabled or weakened" internal/payment/payment_test.go
  EXPECT: >= 1

FULL_PACKAGE_BUILD_AND_RACE:
  $ go build ./internal/payment/
  $ go vet ./internal/payment/
  $ go test -race -count=1 ./internal/payment/
  EXPECT: all exit 0
```

---

### Phase 10.5 — Owner Escrow Operations *(new — closes a DM §4.9 / FR-059 gap)*

**Reference:** DM §4.9 (`owner_escrow_events` schema), DM §7 (`mv_owner_escrow_balance`, including the `GREATEST(...,0)` floor), FR-014, FR-021, FR-059 (owner balance check, balance view, and withdrawal), Milestone 11 Session 11.5.6 (`POST /api/v1/owner/withdraw` — the actual caller of this phase's functions)

> Nothing in the original Milestone 10 plan implemented owner-side escrow bookkeeping at all, despite the table existing since Milestone 4 and Milestone 11 assuming a withdrawal function is available to call. Added here.

#### Session 10.5.1 — Implement `InsertOwnerEscrowEvent()` and wire `WithdrawOwnerEscrow()`

**PRECONDITIONS:**

- Session 10.1.1 complete (`WithdrawOwnerEscrow` declared on `PaymentProvider`)
- Migration from Session 4.4.4 applied (`owner_escrow_events` table)

**TASK:**

In `internal/payment/ledger.go` (alongside `InsertEscrowEvent` — both are ledger-write functions and belong in the same file per the established one-ledger-file-per-package convention):

```go
type OwnerEscrowEventType string

const (
 OwnerDeposit    OwnerEscrowEventType = "DEPOSIT"
 OwnerCharge     OwnerEscrowEventType = "CHARGE"
 OwnerWithdrawal OwnerEscrowEventType = "WITHDRAWAL"
 OwnerRefund     OwnerEscrowEventType = "REFUND"
)

// InsertOwnerEscrowEvent appends one row to owner_escrow_events — the ONLY
// permitted write to this table, mirroring InsertEscrowEvent's role for the
// provider-side ledger (DM §4.9).
func InsertOwnerEscrowEvent(ctx context.Context, db *sql.DB, ownerID uuid.UUID, eventType OwnerEscrowEventType, amountPaise int64, idempotencyKey string, fileID *uuid.UUID) error
```

Then in `internal/payment/razorpay.go` (the Razorpay-backed implementation of `PaymentProvider`), implement:

```go
// WithdrawOwnerEscrow (RazorpayProvider) initiates a real Razorpay payout to
// the owner's UPI-linked bank account, then records a WITHDRAWAL event via
// InsertOwnerEscrowEvent. Balance check (available = balance - reserved next
// 30 days) and the in-flight-upload block are enforced by the caller
// (Milestone 11, Session 11.5.6) before this is invoked — this function
// trusts amountPaise as already validated.
func (r *RazorpayProvider) WithdrawOwnerEscrow(ctx context.Context, ownerID uuid.UUID, amountPaise int64, idempotencyKey string) (string, error)
```

**VERIFY:**

```bash
FILES_EXIST:
  $ grep -c "OwnerDeposit\|OwnerCharge\|OwnerWithdrawal\|OwnerRefund" internal/payment/ledger.go
  EXPECT: 4
  $ grep -c "^func InsertOwnerEscrowEvent(" internal/payment/ledger.go
  EXPECT: 1
  $ grep -c "func (r \*RazorpayProvider) WithdrawOwnerEscrow(" internal/payment/razorpay.go
  EXPECT: 1

COMPILE:
  $ go build ./internal/payment/
  EXPECT: exit 0

INT64_ONLY:
  $ grep -A10 "^func InsertOwnerEscrowEvent" internal/payment/ledger.go | grep -c "amountPaise int64"
  EXPECT: 1

TARGETS_CORRECT_TABLE:
  $ grep -A15 "^func InsertOwnerEscrowEvent" internal/payment/ledger.go | grep -c "INTO owner_escrow_events"
  EXPECT: >= 1
  $ grep -A15 "^func InsertOwnerEscrowEvent" internal/payment/ledger.go | grep -c "INTO escrow_events\b"
  EXPECT: 0

BALANCE_QUERY_USES_GREATEST_GUARD:
  # Callers reading balance should hit mv_owner_escrow_balance, which has the GREATEST(...,0) floor from DM §7 — spot check no separate, unguarded balance formula is reimplemented here
  $ grep -c "mv_owner_escrow_balance" internal/payment/ledger.go
  EXPECT: >= 0   # informational — confirm this file defers to the view rather than recomputing the formula inline

UNIT_TESTS:
  $ go test -v -run TestInsertOwnerEscrowEvent ./internal/payment/
  EXPECT: exit 0; tests include:
    TestInsertOwnerEscrowEventSucceeds
    TestInsertOwnerEscrowEventDuplicateIdempotencyKey
    TestInsertOwnerEscrowEventNeverTouchesProviderTable  (escrow_events row count unchanged after call)

  $ go test -v -run TestWithdrawOwnerEscrow ./internal/payment/
  EXPECT: exit 0; tests include:
    TestWithdrawOwnerEscrowRecordsWithdrawalEvent
    TestWithdrawOwnerEscrowIdempotentOnRetry

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/p2p" internal/payment/ledger.go internal/payment/razorpay.go \
      && echo "FAIL: prohibited import (IC §9)" || echo "PASS"

VET:
  $ go vet ./internal/payment/
  EXPECT: exit 0; zero output
```

---

## Milestone 11 — REST API Layer

**Reference:** `docs/api/openapi.yaml` (OAS — authoritative for all request/response schemas, status codes, and error bodies; see IC §3's own precedence statement), IC §3 (REST contract cross-references only — do not treat IC §3.3/§3.4's inline JSON sketches as authoritative where they conflict with OAS; see the two flagged corrections in Phases 11.1 and 11.2), IC §3.3 (error envelope — partial list, reconciled with OAS below), IC §3.4 (readiness gate — condition *logic* is still authoritative here; the response *body shape* is not, see Phase 11.2), IC §7 (webhook contracts implemented in Milestone 10)

**Package:** `internal/api` (new package for HTTP-layer concerns; not previously created). No `internal/` import restrictions apply beyond the general rule that `internal/api` is the one place permitted to call into `internal/audit`, `internal/scoring`, `internal/repair`, and `internal/payment` on the microservice's behalf — actual wiring of those calls happens in Milestone 12, not here; Milestone 11's own deliverable is the routing tree and stub handlers (Session 11.3.1).

---

### Phase 11.1 — Error Envelope

**Reference:** OAS `components/schemas/Error`, IC §3.3 (error envelope — partial, see the flagged correction below)

> **Flagged and corrected — IC §3.3's error-code table is neither exhaustive nor fully consistent with OAS.** Scanning every `error_code:` value that actually appears in `openapi.yaml`'s path definitions and examples turns up more than a dozen codes IC §3.3 never lists (`INVALID_PHONE_NUMBER`, `INVALID_AMOUNT`, `INVALID_CHALLENGE_NONCE`, `WRONG_ROLE`, `INVALID_BODY_SIGNATURE`, `NOT_FOUND`, `DUPLICATE_CHALLENGE_NONCE`, `PHONE_ALREADY_REGISTERED`, `OTP_RATE_LIMITED`, `INVALID_OTP`, `TOKEN_REFRESH_RATE_LIMITED`, `DOWNTIME_ALREADY_ACTIVE`, `FILE_ALREADY_DELETED`), plus two direct naming conflicts on codes both documents share (`UNAUTHENTICATED` vs. OAS's `UNAUTHORIZED`; `INSUFFICIENT_ESCROW` vs. OAS's `INSUFFICIENT_ESCROW_BALANCE`). Since IC §3 states OAS is the exclusive source for REST contracts, OAS's strings win both conflicts. The full reconciled list is given below; "implement all error codes from the IC §3.3 table" (the original instruction) would leave the implementation missing constants that later sessions in this same milestone reference by name.

#### Session 11.1.1 — Implement standard error response

**PRECONDITIONS:**

- `internal/api` package directory exists (create if not; no prior session in this build plan creates it)

**TASK:**

1. Create `internal/api/errors.go`. Implement:

```go
// WriteError writes the standard Vyomanaut error envelope (OAS components/schemas/Error).
//   {"error_code": "...", "message": "...", "request_id": "...", "retry_after": null}
// request_id is a UUIDv7, matching the X-Request-ID response header (OAS, IC §3.3).
// field and details are optional (OAS Error.properties.field/details) — pass "" / nil
// when not applicable. availableASNs is optional (non-nil only for
// INSUFFICIENT_ASN_DIVERSITY, OAS Error.properties.available_asns) — see note below.
func WriteError(w http.ResponseWriter, statusCode int, errorCode ErrorCode, message string, retryAfter *int, field string, availableASNs *int)
```

1. Declare `ErrorCode` as a typed string constant for **every** code found by scanning `openapi.yaml`'s path definitions and component examples — not just IC §3.3's table:

```go
type ErrorCode string

const (
 // From IC §3.3, unchanged from OAS:
 ErrInvalidRequest             ErrorCode = "INVALID_REQUEST"
 ErrProviderDeparted           ErrorCode = "PROVIDER_DEPARTED"
 ErrEscrowFrozen               ErrorCode = "ESCROW_FROZEN"
 ErrNetworkNotReady            ErrorCode = "NETWORK_NOT_READY"
 ErrInsufficientASNDiversity   ErrorCode = "INSUFFICIENT_ASN_DIVERSITY"
 ErrRazorpayUnavailable        ErrorCode = "RAZORPAY_UNAVAILABLE"
 ErrInternal                   ErrorCode = "INTERNAL_ERROR"
 ErrVettingCapExceeded         ErrorCode = "VETTING_CAP_EXCEEDED"
 ErrRealShardOnVettingProvider ErrorCode = "REAL_SHARD_ON_VETTING_PROVIDER"
 ErrDemoModeRealPayment        ErrorCode = "DEMO_MODE_REAL_PAYMENT"
 ErrProdModeEnvSecret          ErrorCode = "PROD_MODE_ENV_SECRET"

 // Corrected from IC §3.3 to match OAS's actual strings (IC §3.3 disagreed with OAS):
 ErrUnauthorized         ErrorCode = "UNAUTHORIZED"          // IC §3.3 wrongly said UNAUTHENTICATED
 ErrInsufficientEscrow   ErrorCode = "INSUFFICIENT_ESCROW_BALANCE" // IC §3.3 wrongly said INSUFFICIENT_ESCROW

 // Present in OAS path/example bodies but absent from IC §3.3's table entirely:
 ErrInvalidPhoneNumber      ErrorCode = "INVALID_PHONE_NUMBER"
 ErrInvalidAmount           ErrorCode = "INVALID_AMOUNT"
 ErrInvalidChallengeNonce   ErrorCode = "INVALID_CHALLENGE_NONCE"
 ErrWrongRole               ErrorCode = "WRONG_ROLE"
 ErrInvalidBodySignature    ErrorCode = "INVALID_BODY_SIGNATURE"
 ErrNotFound                ErrorCode = "NOT_FOUND"
 ErrDuplicateChallengeNonce ErrorCode = "DUPLICATE_CHALLENGE_NONCE"
 ErrPhoneAlreadyRegistered  ErrorCode = "PHONE_ALREADY_REGISTERED"
 ErrOTPRateLimited          ErrorCode = "OTP_RATE_LIMITED"
 ErrInvalidOTP              ErrorCode = "INVALID_OTP"
 ErrTokenRefreshRateLimited ErrorCode = "TOKEN_REFRESH_RATE_LIMITED"
 ErrDowntimeAlreadyActive   ErrorCode = "DOWNTIME_ALREADY_ACTIVE"
 ErrFileAlreadyDeleted      ErrorCode = "FILE_ALREADY_DELETED"

 // New — corrects a wrong-code reuse in Session 11.7.2 (see Phase 11.7 flag):
 ErrFileAlreadyRegistered ErrorCode = "FILE_ALREADY_REGISTERED"

 // Referenced in Session 11.11.1 but not yet present in openapi.yaml — flag for
 // an OAS addition before this ships; implement the constant now so the Go code
 // compiles and is ready the moment OAS catches up:
 ErrInsufficientProviderCapacity ErrorCode = "INSUFFICIENT_PROVIDER_CAPACITY"
)
```

1. **`INSUFFICIENT_ASN_DIVERSITY` special case:** `available_asns` is a field on the same `Error` struct (OAS defines it directly on `components/schemas/Error`, and *separately* re-adds it via `allOf` composition on the specific `/api/v1/upload/assign` 503 response — the second one is redundant since the base schema already has it). Implement as a single optional `AvailableASNs *int` field on the one shared Go `Error` struct; do not create a second wrapper type to mirror OAS's redundant `allOf`.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/api/errors.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

FUNCTION_SIGNATURE:
  $ grep -c "^func WriteError(" internal/api/errors.go
  EXPECT: 1

ALL_OAS_ERROR_CODES_PRESENT:
  # Spot-check the codes IC §3.3 never listed at all
  $ grep -c "ErrInvalidPhoneNumber\|ErrWrongRole\|ErrInvalidBodySignature\|ErrNotFound\|ErrDuplicateChallengeNonce\|ErrPhoneAlreadyRegistered\|ErrOTPRateLimited\|ErrInvalidOTP\|ErrTokenRefreshRateLimited\|ErrDowntimeAlreadyActive\|ErrFileAlreadyDeleted" \
      internal/api/errors.go
  EXPECT: 11

CORRECTED_CODES_MATCH_OAS_NOT_IC:
  # The two naming conflicts — OAS strings must win
  $ grep -c '"UNAUTHORIZED"' internal/api/errors.go
  EXPECT: 1
  $ grep -n '"UNAUTHENTICATED"' internal/api/errors.go \
      && echo "FAIL: IC §3.3's string is stale; OAS's actual 401 example says UNAUTHORIZED" \
      || echo "PASS"
  $ grep -c '"INSUFFICIENT_ESCROW_BALANCE"' internal/api/errors.go
  EXPECT: 1
  $ grep -n '"INSUFFICIENT_ESCROW"$' internal/api/errors.go \
      && echo "FAIL: IC §3.3's string is stale; OAS's actual 409 example says INSUFFICIENT_ESCROW_BALANCE" \
      || echo "PASS"

FILE_ALREADY_REGISTERED_IS_DISTINCT_FROM_DELETED:
  $ grep -c "ErrFileAlreadyRegistered\|ErrFileAlreadyDeleted" internal/api/errors.go
  EXPECT: 2

AVAILABLE_ASNS_SINGLE_FIELD_NOT_WRAPPER:
  $ grep -c "AvailableASNs \*int" internal/api/errors.go
  EXPECT: 1

UNIT_TESTS:
  $ go test -v -run TestWriteError ./internal/api/
  EXPECT: exit 0; tests include:
    TestWriteErrorBodyShape           (JSON body has exactly error_code, message, request_id, retry_after, plus optional fields)
    TestWriteErrorRequestIDIsUUIDv7
    TestWriteErrorSetsXRequestIDHeader (X-Request-ID header equals body's request_id)
    TestWriteErrorAvailableASNsOnlyWhenSet (nil for any code other than INSUFFICIENT_ASN_DIVERSITY)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.2 — Readiness Gate Endpoint

**Reference:** OAS `components/schemas/ReadinessResponse`, `components/schemas/ReadinessCondition` (authoritative response shape), IC §3.4 (authoritative for *condition-evaluation logic and data sources* — see the flagged correction on the response body), FR-053, FR-054, ADR-029

> **Flagged and corrected — response body shape.** IC §3.4 sketches `{"ready": bool, "conditions": {"active_providers": {"value", "threshold", "met"}, ...}, "evaluated_at": "..."}`. OAS's actual `ReadinessResponse` requires `all_conditions_met`, `evaluated_at`, `mode`, and `conditions` keyed by `active_vetted_providers`, `distinct_asns`, `distinct_metro_regions`, `microservice_quorum`, `razorpay_accounts_ready`, `relay_nodes_deployed`, `cluster_audit_secret_loaded` — each a `ReadinessCondition` object with `name`, `satisfied`, `current_value`, `required_value`, and nullable `demo_value`. These are two different, non-overlapping JSON shapes. Per IC §3's own rule, OAS wins. IC §3.4's condition *table* (data source + threshold per condition) is unaffected and remains the right reference for the evaluation logic itself.
>
> **Flagged and corrected — copy-paste error in the quorum condition.** The original task text said: *"The `readiness_condition.threshold` field in the response must read from `NetworkProfile.MinActiveProviders` (not hardcoded 3) per IC §3.4"* — in the middle of describing the **quorum** condition. `NetworkProfile.MinActiveProviders` is the *active-providers* condition's field (bullet 1); it has nothing to do with quorum. There is no dedicated integer profile field for the quorum replica count — OAS's own description confirms this: *"Threshold from NetworkProfile.RequireQuorum. Production: 3 healthy replicas... Demo: 1 instance."* The quorum threshold is **derived** from the boolean `RequireQuorum` (3 if true, 1 if false), not read from any named integer field.

#### Session 11.2.1 — Implement `GET /api/v1/admin/readiness`

**PRECONDITIONS:**

- `internal/api/errors.go` exists (Session 11.1.1)
- `internal/audit`'s `ClusterSecretCache` exists (Milestone 7, Phase 7.4) — this session requires one additional method not in the original Phase 7.4 write-up (see note below)
- A `ClusterMembership` interface (or equivalent) is available, even as a stub — see the demo-mode branch below

> **Retroactive addition needed in Milestone 7, Phase 7.4:** this session calls `clusterSecretCache.IsLoaded() bool` — a trivial accessor reporting whether `Load()` has completed successfully at least once. This method was not included in the Milestones 7–8 revision's `ClusterSecretCache` proposal. Add it there: `func (c *ClusterSecretCache) IsLoaded() bool`.

**TASK:**

In `internal/api/readiness.go`, implement the readiness evaluator and HTTP handler. All seven conditions are evaluated on every call (re-computed at most every 60 seconds by a background goroutine — Milestone 12, Session 12.1.1 — with the handler reading the cached last-evaluated result):

| # | `conditions` key (OAS) | Data source | `required_value` |
|---|---|---|---|
| 1 | `active_vetted_providers` | `COUNT(*) FROM providers WHERE status IN ('VETTING','ACTIVE')` | `profile.MinActiveProviders` |
| 2 | `distinct_asns` | `COUNT(DISTINCT asn) FROM providers WHERE status IN ('VETTING','ACTIVE')` | `profile.MinDistinctASNs` |
| 3 | `distinct_metro_regions` | `COUNT(DISTINCT region) FROM providers WHERE status IN ('VETTING','ACTIVE')` | `profile.MinMetroRegions` |
| 4 | `microservice_quorum` | `clusterMembership.HealthyCount()` when `profile.RequireQuorum`; else always satisfied | **derived**: `3` if `profile.RequireQuorum` else `1` — never `profile.MinActiveProviders` |
| 5 | `razorpay_accounts_ready` | `COUNT(*) FROM providers WHERE razorpay_cooling_until < NOW()` — **live query, never cached, re-run every evaluation cycle** (ADR-029) | `profile.MinCooledAccounts` |
| 6 | `relay_nodes_deployed` | Relay heartbeat table row count | `profile.MinRelayNodes` |
| 7 | `cluster_audit_secret_loaded` | `clusterSecretCache.IsLoaded()` | boolean; no numeric threshold |

Response fields: `all_conditions_met` (AND of all seven `satisfied` flags), `evaluated_at`, `mode` (`profile.Mode`), and `conditions` (the seven `ReadinessCondition` objects above, each with `name`, `satisfied`, `current_value`, `required_value`, and `demo_value` — non-null only when `profile.Mode == "demo"`, mirroring the current `required_value`).

Condition 4's demo-mode branch: when `profile.RequireQuorum == false`, always return `satisfied: true, current_value: 1, required_value: 1, demo_value: 1` without querying `clusterMembership` at all (matches OAS's `DemoReady` example exactly). Until Milestone 17 Phase 17.2.1 delivers the real gossip cluster, use a `MockClusterMembership` stub (Milestone 12 constructs and injects it) that reports `healthy = 3`.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/api/readiness.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

RESPONSE_MATCHES_OAS_NOT_STALE_IC_SKETCH:
  # The core fix: field names must be OAS's, never IC §3.4's older sketch
  $ grep -c "all_conditions_met\|evaluated_at\|active_vetted_providers\|distinct_asns\|distinct_metro_regions\|microservice_quorum\|razorpay_accounts_ready\|relay_nodes_deployed\|cluster_audit_secret_loaded" \
      internal/api/readiness.go
  EXPECT: >= 9
  $ grep -n '"ready"' internal/api/readiness.go \
      && echo "FAIL: 'ready' is IC §3.4's stale field name; OAS calls it all_conditions_met" || echo "PASS"

SEVEN_CONDITIONS_PRESENT:
  $ grep -c "ReadinessCondition{" internal/api/readiness.go
  EXPECT: >= 7

QUORUM_THRESHOLD_IS_DERIVED_NOT_MISATTRIBUTED:
  # The copy-paste bug fix: quorum's required_value must never reference MinActiveProviders
  $ grep -B3 -A3 "microservice_quorum\|RequireQuorum" internal/api/readiness.go | grep -c "MinActiveProviders" \
      && echo "FAIL: quorum threshold copy-pasted the active-providers field" || echo "PASS"
  $ grep -c "RequireQuorum.*3\|3.*RequireQuorum\|if profile.RequireQuorum" internal/api/readiness.go
  EXPECT: >= 1

RAZORPAY_NEVER_CACHED:
  $ grep -c "live query\|never cach\|LIVE QUERY" internal/api/readiness.go
  EXPECT: >= 1

DEMO_QUORUM_SHORT_CIRCUITS:
  $ grep -A5 "RequireQuorum == false\|!profile.RequireQuorum" internal/api/readiness.go | grep -c "current_value.*1\|:= 1"
  EXPECT: >= 1

CLUSTERSECRETCACHE_ISLOADED_USED:
  $ grep -c "clusterSecretCache.IsLoaded()\|\.IsLoaded()" internal/api/readiness.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestReadinessHandler ./internal/api/
  EXPECT: exit 0; tests include:
    TestReadinessAllSevenConditionsEvaluated
    TestReadinessAllConditionsMetWhenAllSatisfied
    TestReadinessDemoQuorumAlwaysSatisfiedWithoutQuery  (RequireQuorum=false -> HealthyCount never called)
    TestReadinessRazorpayIsLiveQueryNotCached           (two calls in the same cycle re-query rather than reuse a cached count)
    TestReadinessModeFieldReflectsProfile               (config.DemoProfile -> mode="demo"; config.ProductionProfile -> mode="prod")
    TestReadinessMatchesOASDemoReadyExample             (fixture built from the OAS DemoReady example -> struct round-trips correctly)

NEGATIVE_CHECKS:
  $ grep -n '"threshold"\|"value"\|"met"' internal/api/readiness.go \
      && echo "FAIL: these are IC §3.4's stale field names, not OAS's" || echo "PASS"

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.3 — Endpoint Routing Stubs

**Reference:** OAS `paths` (every operation listed below is verified against its exact `operationId` in `openapi.yaml`), IC §2 (component communication map — cross-check only, OAS is authoritative for the exact path/method/operationId)

> **Flagged — an endpoint in this table does not exist in OAS.** `GET /api/v1/provider/downtime` / `getActiveDowntime` appears below (carried over from the original plan) and is described in Session 11.6.5, but `openapi.yaml` defines only `POST /api/v1/provider/downtime` (`announceDowntime`) — there is no `GET` sibling path, schema, or operationId anywhere in the spec. This is a genuine prerequisite gap: **add the `GET` path to `openapi.yaml` before implementing this row** (the response shape Session 11.6.5 already describes — `{active, promised_return_at, penalty_fires_at}` — is a reasonable schema to add). Until that's done, treat this row as blocked, not as an OAS-backed contract.

#### Session 11.3.1 — Establish HTTP routing tree with stubs

**PRECONDITIONS:**

- `internal/api/errors.go` exists (Session 11.1.1)

**TASK:**

Create `internal/api/router.go`. For each row below, register a stub handler returning `501 Not Implemented` with `error_code: INTERNAL_ERROR` (Session 11.1.1's `ErrInternal`). Every `operationId` was individually verified against `openapi.yaml`; this table supersedes any other listing.

| Method | Path | OAS `operationId` | Auth |
| --- | --- | --- | --- |
| GET | `/.well-known/jwks.json` | `getJwks` | public |
| POST | `/api/v1/auth/otp/send` | `sendOtp` | public |
| POST | `/api/v1/auth/otp/verify` | `verifyOtp` | public |
| POST | `/api/v1/owner/register` | `registerOwner` | BearerAuth (short-lived registration token from `otp/verify`) |
| POST | `/api/v1/owner/deposit` | `initiateDeposit` | BearerAuth |
| GET | `/api/v1/owner/{owner_id}/balance` | `getOwnerBalance` | BearerAuth |
| GET | `/api/v1/owner/{owner_id}/files` | `listOwnerFiles` | BearerAuth |
| GET | `/api/v1/owner/{owner_id}/escrow` | `getOwnerEscrowHistory` | BearerAuth |
| POST | `/api/v1/owner/withdraw` | `withdrawOwnerEscrow` | BearerAuth |
| POST | `/api/v1/provider/register` | `registerProvider` | BearerAuth (short-lived registration token from `otp/verify`) |
| POST | `/api/v1/provider/heartbeat` | `providerHeartbeat` | BearerAuth |
| POST | `/api/v1/provider/token/refresh` | `refreshProviderToken` | BearerAuth (grace: valid or expired ≤1h) |
| GET | `/api/v1/provider/{provider_id}/status` | `getProviderStatus` | BearerAuth |
| GET | `/api/v1/provider/receipts` | `listProviderReceipts` | BearerAuth |
| POST | `/api/v1/provider/downtime` | `announceDowntime` | BearerAuth |
| GET | `/api/v1/provider/downtime` | `getActiveDowntime` | BearerAuth — **⚠ not yet in `openapi.yaml`; see flagged gap above** |
| POST | `/api/v1/provider/depart` | `announceDeparture` | BearerAuth |
| POST | `/api/v1/upload/assign` | `assignUpload` | BearerAuth |
| POST | `/api/v1/file/register` | `registerFile` | BearerAuth |
| GET | `/api/v1/file/{file_id}/pointer` | `getPointerFile` | BearerAuth |
| DELETE | `/api/v1/file/{file_id}` | `deleteFile` | BearerAuth |
| GET | `/api/v1/pricing/estimate` | `getPricingEstimate` | public |
| GET | `/api/v1/pricing/provider-estimate` | `getProviderEarningsEstimate` | public |
| POST | `/api/v1/audit/challenge` | `dispatchAuditChallenge` | AdminApiKey |
| GET | `/api/v1/admin/readiness` | `getReadiness` | AdminApiKey |
| GET | `/api/v1/admin/repair/queue` | `getRepairQueue` | AdminApiKey |
| POST | `/api/v1/admin/repair/trigger` | `triggerRepair` | AdminApiKey |
| GET | `/api/v1/admin/providers` | `listAdminProviders` | AdminApiKey |
| GET | `/api/v1/admin/audit/stats` | `getAuditStats` | AdminApiKey |
| GET | `/api/v1/admin/vetting/status` | `getVettingStatus` | AdminApiKey |
| POST | `/api/v1/admin/vetting/gc/retry` | `retryVettingGC` | AdminApiKey |
| POST | `/webhooks/razorpay` | *(IC §7 only — confirmed absent from OAS by design; OAS's own intro scopes the spec to the client-facing surface, not server-to-server webhooks)* | signature |

**AdminApiKey middleware** (OAS `components/securitySchemes/AdminApiKey`): `X-Admin-API-Key` header, ≥32 random bytes hex-encoded, rotated quarterly, and — per OAS's own description — must **never** appear in client-facing code or provider daemon builds. Apply to all `/api/v1/admin/*` routes and `/api/v1/audit/challenge`.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/api/router.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

ALL_31_OAS_PATHS_REGISTERED:
  # 30 OAS-defined operations + the one flagged-as-missing-from-OAS row = 31
  $ grep -c "HandleFunc\|Handle(" internal/api/router.go
  EXPECT: >= 31

GET_DOWNTIME_MARKED_AS_GAP:
  # Must not be silently presented as OAS-backed
  $ grep -c "not yet in openapi.yaml\|flagged gap\|getActiveDowntime" internal/api/router.go
  EXPECT: >= 1

ADMIN_MIDDLEWARE_ON_ALL_ADMIN_ROUTES:
  $ grep -c "X-Admin-API-Key" internal/api/router.go
  EXPECT: >= 1
  $ grep -c "AdminApiKey\|adminAuthMiddleware" internal/api/router.go
  EXPECT: >= 9   # 8 admin/* routes + audit/challenge

PUBLIC_ROUTES_HAVE_NO_AUTH_MIDDLEWARE:
  $ grep -B2 "jwks.json\|otp/send\|otp/verify\|pricing/estimate\|pricing/provider-estimate" internal/api/router.go | grep -c "BearerAuth\|AdminApiKey" \
      && echo "FAIL: public routes must not require auth" || echo "PASS"

STUB_HANDLERS_RETURN_501:
  $ grep -c "501\|StatusNotImplemented" internal/api/router.go
  EXPECT: >= 1
  $ grep -c "ErrInternal\|INTERNAL_ERROR" internal/api/router.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestRouterStubs ./internal/api/
  EXPECT: exit 0; tests include:
    TestAllRoutesReturn501Initially
    TestAdminRoutesRejectMissingAPIKey       (no X-Admin-API-Key header -> 401/403, not 501)
    TestAdminRoutesRejectShortAPIKey         (< 32 bytes -> rejected)
    TestPublicRoutesAccessibleWithoutAuth

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.4 — Authentication Endpoints

**Reference:** OAS `paths./api/v1/auth/otp/send`, `paths./api/v1/auth/otp/verify`, `paths./api/v1/provider/token/refresh`, `components/schemas/OtpSendRequest`, `OtpSendResponse`, `OtpVerifyRequest`, `OtpVerifyResponse`, FR-001, FR-024

#### Session 11.4.1 — OTP Send (`POST /api/v1/auth/otp/send`)

**PRECONDITIONS:** Session 11.3.1 complete (route registered as a stub)

**TASK:** Implement per OAS `OtpSendRequest`/`OtpSendResponse`. Required request field `purpose` (enum `OWNER_REGISTER | PROVIDER_REGISTER | LOGIN`) is not optional — it "determines which registration flow the OTP gates," and the subsequent register call must be validated against it (OAS description). Rate limit: 3 attempts per phone number per 10 minutes; on breach, `429` with `Retry-After: 600` and `error_code: OTP_RATE_LIMITED`. Response: `expires_at` (~10 minutes from issuance). Public endpoint.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

REQUEST_VALIDATES_PURPOSE_ENUM:
  $ grep -c "OWNER_REGISTER\|PROVIDER_REGISTER\|LOGIN" internal/api/*.go
  EXPECT: >= 3

RATE_LIMIT_CONSTANTS:
  $ grep -c "3.*10.*[Mm]inute\|600" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestOtpSend ./internal/api/
  EXPECT: exit 0; tests include:
    TestOtpSendSucceedsUnderRateLimit
    TestOtpSendReturns429AfterThreeAttempts   (Retry-After: 600, error_code=OTP_RATE_LIMITED)
    TestOtpSendRejectsInvalidPurpose
    TestOtpSendIsPublicNoAuthRequired

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.4.2 — OTP Verify (`POST /api/v1/auth/otp/verify`)

**PRECONDITIONS:** Session 11.4.1 complete

**TASK:** Per OAS `OtpVerifyResponse`: on success, issue an Ed25519-signed JWT. `is_new_entity: true` → 1-hour registration token, `role: null`. `is_new_entity: false` → 24h (owner) / 7d (provider) JWT with `entity_id` and `role` populated. `401 INVALID_OTP` on incorrect/expired code.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

JWT_TTL_BRANCHING:
  $ grep -c "1.*[Hh]our\|24.*[Hh]our\|7.*day" internal/api/*.go
  EXPECT: >= 3

ROLE_NULL_FOR_NEW_ENTITY:
  $ grep -c "is_new_entity\|IsNewEntity" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestOtpVerify ./internal/api/
  EXPECT: exit 0; tests include:
    TestOtpVerifyNewEntityGetsRegistrationToken   (1h TTL, role=null)
    TestOtpVerifyExistingOwnerGets24HourToken
    TestOtpVerifyExistingProviderGets7DayToken
    TestOtpVerifyRejectsWrongCode                  (401 INVALID_OTP)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.4.3 — Provider Token Refresh (`POST /api/v1/provider/token/refresh`)

**PRECONDITIONS:** Session 11.4.2 complete

**TASK:** Two-factor: (1) current JWT valid OR expired ≤1h ago; (2) Ed25519 signature over `SHA-256(provider_id ‖ timestamp_string)` (exact ISO 8601 string as submitted, no normalisation — OAS description). Rate limit: one successful refresh per 30 minutes per `provider_id`; breach → `429 TOKEN_REFRESH_RATE_LIMITED`. Success → new 7-day JWT.

This endpoint is also called by Milestone 6 Session 6.3.1's heartbeat goroutine whenever `tokenExpiresIn() < 24*time.Hour` — that cross-reference is already correctly bidirectional (6.3.1 names this session; this session names 6.3.1) and needs no fix.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

GRACE_WINDOW_IS_ONE_HOUR:
  $ grep -c "1.*[Hh]our\|time.Hour" internal/api/*.go
  EXPECT: >= 1

SIGNING_INPUT_IS_RAW_TIMESTAMP_STRING:
  # Must not re-serialise/normalise the timestamp before hashing
  $ grep -c "provider_id.*timestamp\|SHA-256(provider_id" internal/api/*.go
  EXPECT: >= 1

RATE_LIMIT_THIRTY_MINUTES:
  $ grep -c "30.*[Mm]inute" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderTokenRefresh ./internal/api/
  EXPECT: exit 0; tests include:
    TestRefreshAcceptsValidJWT
    TestRefreshAcceptsExpiredWithinGraceWindow    (expired 30 min ago -> succeeds)
    TestRefreshRejectsExpiredBeyondGraceWindow    (expired 2h ago -> 401)
    TestRefreshRejectsBadEd25519Signature          (403 INVALID_BODY_SIGNATURE)
    TestRefreshRateLimitedWithinThirtyMinutes      (second successful call before 30 min -> 429 TOKEN_REFRESH_RATE_LIMITED)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.5 — Owner Endpoints

**Reference:** OAS `paths./api/v1/owner/*`, `components/schemas/OwnerRegisterRequest/Response`, `DepositInitiateRequest/Response`, `OwnerBalance`, `FileListItem`, `OwnerEscrowTransaction`, `WithdrawRequest/Response`, FR-001, FR-006, FR-014, FR-019, FR-021, FR-059, DM §4.9 (`owner_escrow_events` — see the flagged corrections below)

#### Session 11.5.1 — Owner Register (`POST /api/v1/owner/register`)

**PRECONDITIONS:** Session 11.4.2 complete (registration token issued by OTP verify)

**TASK:** Verify `owner_sig`: Ed25519 over `{"ed25519_public_key": "<value>"}` (sorted keys, no trailing whitespace, per IC §3.2's canonical procedure). Store `ed25519_public_key` against a new `owner_id`. `201` with a fresh 24-hour JWT (`role=owner`, `sub=owner_id`). `409 PHONE_ALREADY_REGISTERED` if the phone already has an owner account.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

SIGNATURE_VERIFIED_BEFORE_INSERT:
  $ grep -c "owner_sig\|OwnerSig" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestOwnerRegister ./internal/api/
  EXPECT: exit 0; tests include:
    TestOwnerRegisterSucceedsWithValidSig
    TestOwnerRegisterRejectsBadSig
    TestOwnerRegisterRejectsDuplicatePhone   (409 PHONE_ALREADY_REGISTERED)
    TestOwnerRegisterReturns24HourJWT

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.5.2 — Deposit Initiate (`POST /api/v1/owner/deposit`)

**PRECONDITIONS:** Session 11.5.1 complete

> **Flagged and corrected — repeats the M9–M10 table confusion.** The original task said *"No `escrow_events` row is written here — the row is written only on receipt of the `virtual_account.payment.captured` webhook."* `escrow_events` is the **provider**-scoped ledger (`provider_id NOT NULL REFERENCES providers`); an owner deposit belongs in `owner_escrow_events` (DM §4.9), which is exactly what the Milestones 9–10 revision corrected Milestone 10 Session 10.3.1 to target. Fixed below.

**TASK:** Triggers Razorpay Smart Collect 2.0 VPA creation. Returns `vpa`, `qr_code_url`, `expires_at`. **No `owner_escrow_events` row is written here** — the row is written only on receipt of the `virtual_account.payment.captured` webhook (Milestone 10 Phase 10.3.1, as corrected — targets `owner_escrow_events`, not `escrow_events`). UPI Collect must not be used (NFR-029).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

NO_LEDGER_WRITE_AT_INITIATE_TIME:
  $ grep -A15 "func.*[Ii]nitiateDeposit\|func.*DepositInitiate" internal/api/*.go | grep -c "InsertOwnerEscrowEvent\|InsertEscrowEvent" \
      && echo "FAIL: deposit is credited only on webhook receipt, not at initiate time" || echo "PASS"

CORRECT_TABLE_REFERENCED_IN_COMMENTS:
  $ grep -c "owner_escrow_events" internal/api/*.go
  EXPECT: >= 1
  $ grep -n "escrow_events\b" internal/api/*.go | grep -v "owner_escrow_events" \
      && echo "WARN: verify this is not the stale provider-table reference" || echo "PASS"

NO_UPI_COLLECT:
  $ grep -in "upi/collect\|collect/request" internal/api/*.go \
      && echo "FAIL: UPI Collect prohibited (NFR-029)" || echo "PASS"

UNIT_TESTS:
  $ go test -v -run TestDepositInitiate ./internal/api/
  EXPECT: exit 0; tests include:
    TestDepositInitiateReturnsVPAAndQR
    TestDepositInitiateWritesNoLedgerRow

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.5.3 — Owner Balance (`GET /api/v1/owner/{owner_id}/balance`)

**PRECONDITIONS:** Session 11.5.2 complete

**TASK:** Read from `mv_owner_escrow_balance` (≤60s stale). Return `balance_paise`, `reserved_next_30d_paise`, `available_paise = balance − reserved`. All `int64`, minimum 0 (the view already applies `GREATEST(...,0)` — DM §7).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

READS_CORRECT_VIEW:
  $ grep -c "mv_owner_escrow_balance" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestOwnerBalance ./internal/api/
  EXPECT: exit 0; tests include:
    TestOwnerBalanceComputesAvailableCorrectly   (available = balance - reserved)
    TestOwnerBalanceNeverNegative

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.5.4 — Owner File List (`GET /api/v1/owner/{owner_id}/files`)

**PRECONDITIONS:** Session 11.5.3 complete

> **Flagged and corrected — hardcoded production shard thresholds.** OAS's own `FileListItem.availability` description is written with fixed numbers ("OK — available_shard_count ≥ s+r0 = 24... CRITICAL — available_shard_count < s = 16"). These are production-only (`DataShards=16, LazyRepairR0=8`); demo mode uses `DataShards=3, LazyRepairR0=1` (thresholds 4 and 3). The implementation must read `profile.DataShards + profile.LazyRepairR0` and `profile.DataShards`, never the literals 24/16.

**TASK:** Query params: `status` (default `ACTIVE`), `limit` (1–100, default 50), `cursor`. `availability` derived from `mv_segment_shard_counts`: `OK` if `available_shard_count >= profile.DataShards + profile.LazyRepairR0`; `DEGRADED` if below that but `>= profile.DataShards`; `CRITICAL` if `< profile.DataShards`. Return `display_name_ciphertext/nonce/tag` when present (FR-019).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

AVAILABILITY_IS_PROFILE_DRIVEN_NOT_HARDCODED:
  $ grep -c "profile.DataShards" internal/api/*.go
  EXPECT: >= 1
  $ grep -c "profile.LazyRepairR0" internal/api/*.go
  EXPECT: >= 1
  $ grep -n ">= 24\|< 16\b" internal/api/*.go | grep -v "_test.go" \
      && echo "FAIL: availability thresholds must be profile-driven for demo-mode correctness" \
      || echo "PASS"

UNIT_TESTS:
  $ go test -v -run TestOwnerFileList ./internal/api/
  EXPECT: exit 0; tests include:
    TestOwnerFileListAvailabilityProdThresholds   (config.ProductionProfile -> OK at 24, CRITICAL below 16)
    TestOwnerFileListAvailabilityDemoThresholds    (config.DemoProfile -> OK at 4, CRITICAL below 3)
    TestOwnerFileListReturnsDisplayNameWhenPresent

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.5.5 — Owner Escrow History (`GET /api/v1/owner/{owner_id}/escrow`)

**PRECONDITIONS:** Session 11.5.4 complete

**TASK:** Paginated `owner_escrow_events` log with a balance-summary envelope (`balance_paise`, `reserved_next_30d_paise`, `available_paise`) alongside `events[]` and `next_cursor`.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

QUERIES_CORRECT_TABLE:
  $ grep -c "owner_escrow_events" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestOwnerEscrowHistory ./internal/api/
  EXPECT: exit 0; tests include:
    TestOwnerEscrowHistoryPaginates
    TestOwnerEscrowHistoryIncludesBalanceSummary

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.5.6 — Owner Withdraw (`POST /api/v1/owner/withdraw`)

**PRECONDITIONS:** Session 11.5.5 complete; Milestone 10 Phase 10.5 complete (`PaymentProvider.WithdrawOwnerEscrow`)

> **Flagged — the reversal path for owner withdrawals has no clean home.** This session says "final `REVERSED` status arrives via the `payout.reversed` webhook (Milestone 10 Phase 10.3.1)" — but that webhook, as specified (including in the Milestones 9–10 revision), only handles **provider** payout reversals (writing a `REVERSAL` row to `escrow_events`). An owner's withdrawal can equally be reversed by the receiving bank, and `owner_escrow_event_type` (DM §4.9: `DEPOSIT, CHARGE, WITHDRAWAL, REFUND`) has no value representing "a withdrawal came back." Proposed minimal fix (no schema change required): Milestone 10's `payout.reversed` handler branches on whether the reversed payout's reference corresponds to a `RELEASE` (provider) or `WITHDRAWAL` (owner) event; for the owner case, it inserts an `owner_escrow_events` row using the existing `DEPOSIT` type (money is being credited back — that is exactly what `DEPOSIT` represents) with idempotency key `SHA-256("withdrawal-reversal" || original_idempotency_key)`. This is a follow-up to Milestone 10 Session 10.3.1, not a new session here — flagging it at the point where the gap becomes visible (this session's own "final status" claim).

**TASK:** `idempotency_key = SHA-256(owner_id ‖ withdrawal_request_id)`, hex-encoded (OAS). Block while any file upload is in-flight for this owner (FR-059). On success: Razorpay `payout_id`, status `QUEUED`. Calls `payment.WithdrawOwnerEscrow` (Milestone 10 Phase 10.5).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

CALLS_CORRECT_PAYMENT_FUNCTION:
  $ grep -c "WithdrawOwnerEscrow" internal/api/*.go
  EXPECT: >= 1

IN_FLIGHT_UPLOAD_BLOCK_PRESENT:
  $ grep -c "in-flight\|InFlight\|upload.*in.progress" internal/api/*.go
  EXPECT: >= 1

IDEMPOTENCY_KEY_FORMAT:
  $ grep -c "owner_id.*withdrawal_request_id\|ownerID.*withdrawalRequestID" internal/api/*.go
  EXPECT: >= 1

REVERSAL_GAP_DOCUMENTED:
  $ grep -c "withdrawal-reversal\|reversal path\|Phase 10.5" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestOwnerWithdraw ./internal/api/
  EXPECT: exit 0; tests include:
    TestOwnerWithdrawSucceedsWhenNoUploadInFlight
    TestOwnerWithdrawBlockedDuringUpload
    TestOwnerWithdrawRejectsAmountAboveAvailable
    TestOwnerWithdrawIdempotentOnRetry

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.6 — Provider REST Endpoints

**Reference:** OAS `paths./api/v1/provider/*`, `components/schemas/ProviderRegisterRequest/Response`, `HeartbeatRequest/Response`, `ProviderStatusResponse`, `AuditReceiptListItem`, `ProviderDowntimeRequest/Response`, `ProviderDepartRequest/Response`, FR-024–FR-036, FR-058, ADR-005, ADR-007, ADR-030

#### Session 11.6.1 — Provider Register (`POST /api/v1/provider/register`)

**PRECONDITIONS:** Session 11.4.2 complete

**TASK:** Verify `provider_sig` over canonical JSON of all other fields (sorted keys, `provider_sig` omitted — IC §3.2). Store `ed25519_public_key`, `declared_storage_gb`, `city`, `region`, `asn`, `initial_multiaddrs`. Demo mode: accept `demo_asn` (`^SIM-AS\d+$`); auto-assign next available if omitted (FR-068). Production: silently ignore `demo_asn`. `razorpay_cooling_until = NOW() + profile.RazorpayCoolingPeriod` (never hardcoded 24h — MVP §5.4). Return `PENDING_ONBOARDING` and a 7-day JWT.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

COOLING_PERIOD_PROFILE_DRIVEN:
  $ grep -c "profile.RazorpayCoolingPeriod" internal/api/*.go
  EXPECT: >= 1

DEMO_ASN_PATTERN_AND_AUTO_ASSIGN:
  $ grep -c "SIM-AS\|demo_asn" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderRegister ./internal/api/
  EXPECT: exit 0; tests include:
    TestProviderRegisterDemoModeAutoAssignsASN     (demo_asn omitted -> next available SIM-AS{N})
    TestProviderRegisterProdModeIgnoresDemoAsn
    TestProviderRegisterSetsProfileDrivenCooling   (demo -> instant; prod -> +24h)
    TestProviderRegisterReturnsPendingOnboarding

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.6.2 — Provider Heartbeat (`POST /api/v1/provider/heartbeat`)

**PRECONDITIONS:** Session 11.6.1 complete

**TASK:** Verify `provider_sig` over canonical JSON (sorted keys, omitted). Reject timestamp skew > ±5 minutes. Update `providers.last_known_multiaddrs`, `last_heartbeat_ts`, clear `multiaddr_stale`. Return `received_at` and `microservice_sig` (Ed25519 over `{"received_at","provider_id"}` sorted).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

SKEW_CHECK_FIVE_MINUTES:
  $ grep -c "5.*[Mm]inute" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderHeartbeat ./internal/api/
  EXPECT: exit 0; tests include:
    TestHeartbeatUpdatesMultiaddrsAndTimestamp
    TestHeartbeatClearsMultiaddrStale
    TestHeartbeatRejectsSkewBeyondFiveMinutes
    TestHeartbeatReturnsMicroserviceCountersignature

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.6.3 — Provider Status (`GET /api/v1/provider/{provider_id}/status`)

**PRECONDITIONS:** Session 11.6.2 complete; Milestone 8 complete (`mv_provider_scores`)

**TASK:** Join `providers` with `mv_provider_scores` (≤60s stale). Include vetting fields only when `status='VETTING'`: `vetting_chunks_assigned`, `vetting_chunk_cap = floor(declared_storage_gb × 400)`, `vetting_eligible_at = first_chunk_assignment_at + profile.VettingMinDuration` (not hardcoded 120 days — mirrors the Milestone 8 fix), `vetting_gc_pending`. Include `network_mode` from `profile.Mode`.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

VETTING_ELIGIBLE_AT_IS_PROFILE_DRIVEN:
  $ grep -c "profile.VettingMinDuration" internal/api/*.go
  EXPECT: >= 1
  $ grep -n "120.*day\|120 \* 24" internal/api/*.go | grep -v "_test.go\|comment" \
      && echo "WARN: verify this is documentation, not a hardcoded duration" || echo "PASS"

VETTING_FIELDS_NULL_OUTSIDE_VETTING:
  $ grep -c "status.*VETTING\|== .VETTING." internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderStatus ./internal/api/
  EXPECT: exit 0; tests include:
    TestProviderStatusIncludesVettingFieldsWhenVetting
    TestProviderStatusOmitsVettingFieldsWhenActive
    TestProviderStatusNetworkModeReflectsProfile

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.6.4 — Provider Audit Receipts (`GET /api/v1/provider/receipts`)

**PRECONDITIONS:** Session 11.6.3 complete

**TASK:** Filters: `chunk_id`, `from`, `to`, `result`, `limit` (1–200, default 100), `cursor`. Must remain available when `status='DEPARTED'` (FR-058 — dispute evidence). Include both `provider_sig` and `service_sig`. `file_id` nullable for vetting-chunk receipts.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

AVAILABLE_AFTER_DEPARTURE:
  $ grep -c "DEPARTED" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderReceipts ./internal/api/
  EXPECT: exit 0; tests include:
    TestProviderReceiptsAccessibleWhenDeparted   (status=DEPARTED -> still 200, not 403)
    TestProviderReceiptsFiltersByChunkAndResult
    TestProviderReceiptsFileIDNullableForVettingChunks

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.6.5 — Provider Downtime (`POST /api/v1/provider/downtime`; `GET` — blocked, see flagged gap)

**PRECONDITIONS:** Session 11.6.4 complete

**TASK:** `POST`: `promised_return_at` in `[0h, 72h]` from now; past timestamps → `400`. Verify `provider_sig`. `409 DOWNTIME_ALREADY_ACTIVE` if a window is open. `409 PROVIDER_DEPARTED` if departed. `penalty_fires_at = promised_return_at`.

`GET`: **blocked on the flagged OAS gap in Phase 11.3** — implement the handler logic (`{active, promised_return_at, penalty_fires_at}`, matching what this session already specifies) but do not wire it into the router until the path exists in `openapi.yaml`.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

WINDOW_BOUNDS_ENFORCED:
  $ grep -c "72.*[Hh]our\|0.*72" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderDowntime ./internal/api/
  EXPECT: exit 0; tests include:
    TestDowntimePostAccepts0To72Hours
    TestDowntimePostRejectsPastTimestamp           (400)
    TestDowntimePostRejectsSecondWindow            (409 DOWNTIME_ALREADY_ACTIVE)
    TestDowntimePostRejectsDepartedProvider         (409 PROVIDER_DEPARTED)
    TestDowntimeGetLogicNotYetRoutedPendingOAS      (handler exists and is unit-testable directly; not reachable via HTTP until OAS is updated)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.6.6 — Provider Depart (`POST /api/v1/provider/depart`)

**PRECONDITIONS:** Session 11.6.5 complete; Milestone 9 Phase 9.1 complete (`repair.EnqueueJob`); Milestone 10 Phase 10.2 complete (`payment.ReleaseEscrow`)

> **Flagged — idempotency key for the prorated release.** `ReleaseEscrow`'s standard idempotency key is `SHA-256(providerID || auditPeriodID)` (Milestone 10), which assumes a completed monthly audit period. An announced departure fires mid-period, with no "current" `audit_period_id` in the usual sense. Use `SHA-256(providerID || "announced-departure" || departureTimestamp)` for this call site specifically — same `ReleaseEscrow` function, different idempotency-key derivation, since the underlying `INSERT INTO escrow_events (... 'RELEASE' ...)` operation is identical regardless of why the release fired.

**TASK:** Verify `provider_sig`. Optional `depart_at` (omitted/past → immediate). Queues repair for all real chunks (`repair.EnqueueJob`, `TriggerAnnouncedDeparture`). Releases held escrow proportional to the fraction of the current 30-day window completed (`payment.ReleaseEscrow` with the departure-specific idempotency key above). Sets `status='DEPARTED'`. Wired at the microservice entrypoint (Milestone 12) since `internal/api` — not `internal/repair` — is the layer permitted to call both packages.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

DEPARTURE_IDEMPOTENCY_KEY_IS_DISTINCT:
  $ grep -c "announced-departure" internal/api/*.go
  EXPECT: >= 1
  $ grep -n "providerID.*auditPeriodID" internal/api/*.go \
      && echo "WARN: verify this is not misapplied to the departure call site" || echo "PASS"

WIRES_REPAIR_AND_PAYMENT:
  $ grep -c "EnqueueJob\|TriggerAnnouncedDeparture" internal/api/*.go
  EXPECT: >= 1
  $ grep -c "ReleaseEscrow" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderDepart ./internal/api/
  EXPECT: exit 0; tests include:
    TestProviderDepartImmediateWhenDepartAtOmitted
    TestProviderDepartQueuesRepairForAllRealChunks
    TestProviderDepartReleasesProratedEscrow
    TestProviderDepartUsesDistinctIdempotencyKey

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.7 — Upload and File Endpoints

**Reference:** OAS `paths./api/v1/upload/assign`, `paths./api/v1/file/*`, `components/schemas/UploadAssignRequest/Response`, `SegmentAssignment`, `ShardAssignment`, `FileRegisterRequest/Response`, `PointerFileResponse`, FR-007–FR-020, ADR-014, ADR-022

> **Flagged — three hardcoded production shard numbers.** OAS's schema descriptions for this phase are written in fixed production terms: the ASN cap ("no single ASN > 20% of 56 shards"), the shard-index split ("0–15 data, 16–55 parity" on `ShardAssignment.shard_index`), and `SegmentAssignment.providers`'s `minItems/maxItems: 56`. All three must be computed from `NetworkProfile` (`profile.TotalShards`, `profile.DataShards`, `profile.ASNCapFraction`) for demo-mode correctness (5 total shards, 3 data, 1-shard ASN cap) — the same pattern already fixed repeatedly in Milestones 7–10.

#### Session 11.7.1 — Upload Assign (`POST /api/v1/upload/assign`)

**PRECONDITIONS:** Milestone 8 complete (readiness gate feeds off provider/ASN/region counts, Session 11.2.1); Milestone 10 Phase 10.1 complete (escrow balance check)

**TASK:** Checks, in order: (1) readiness gate → `503 NETWORK_NOT_READY`, `retry_after: 60`; (2) escrow balance ≥ 30-day cost → `409 INSUFFICIENT_ESCROW_BALANCE` (corrected code, Phase 11.1); (3) ASN cap — no single ASN holds more than `floor(profile.TotalShards * profile.ASNCapFraction)` shards for the segment (11 in production, 1 in demo — never hardcode "56"/"20% of 56") → `503 INSUFFICIENT_ASN_DIVERSITY`, `available_asns`, `retry_after: 300`. Response includes `monthly_cost_paise`, `required_escrow_paise`, per-shard `capability_token` (72 bytes / 144 hex chars). `shard_index` `0` to `profile.DataShards-1` = data, `profile.DataShards` to `profile.TotalShards-1` = parity (0–15/16–55 in production only). Idempotent on `file_id`.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

ASN_CAP_IS_PROFILE_DRIVEN:
  $ grep -c "profile.TotalShards.*profile.ASNCapFraction\|ASNCapFraction" internal/api/*.go
  EXPECT: >= 1
  $ grep -n "20% of 56\|> 11\b" internal/api/*.go | grep -v "_test.go\|comment" \
      && echo "FAIL: ASN cap must be computed, not hardcoded to the production figure" \
      || echo "PASS"

SHARD_INDEX_SPLIT_IS_PROFILE_DRIVEN:
  $ grep -c "profile.DataShards" internal/api/*.go
  EXPECT: >= 1

CHECKS_IN_CORRECT_ORDER:
  # Readiness -> escrow -> ASN cap; verify by line-number ordering of the three markers
  $ grep -n "NETWORK_NOT_READY\|INSUFFICIENT_ESCROW_BALANCE\|INSUFFICIENT_ASN_DIVERSITY" internal/api/*.go
  # Manual check: NETWORK_NOT_READY line < INSUFFICIENT_ESCROW_BALANCE line < INSUFFICIENT_ASN_DIVERSITY line

CAPABILITY_TOKEN_SIZE:
  $ grep -c "144\|72.*byte" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestUploadAssign ./internal/api/
  EXPECT: exit 0; tests include:
    TestUploadAssignRejectsWhenNetworkNotReady        (503, retry_after=60)
    TestUploadAssignRejectsInsufficientEscrow          (409 INSUFFICIENT_ESCROW_BALANCE)
    TestUploadAssignRejectsASNDiversityDemo             (config.DemoProfile: cap=1 shard/ASN)
    TestUploadAssignRejectsASNDiversityProd             (config.ProductionProfile: cap=11 shards/ASN)
    TestUploadAssignIdempotentOnFileID
    TestUploadAssignShardIndexRangeDemo                 (0-2 data, 3-4 parity for DemoProfile)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.7.2 — File Register (`POST /api/v1/file/register`)

**PRECONDITIONS:** Session 11.7.1 complete

> **Flagged and corrected — wrong error code reused.** The original task returns `409 FILE_ALREADY_DELETED` for "file exists in ACTIVE status" — the opposite of what that code name means (it's correctly used for the actual delete-conflict case in Session 11.7.4). Fixed to a new, accurately-named code.

**TASK:** Verify `owner_sig` over canonical JSON. Store `pointer_ciphertext`, `pointer_nonce`, `pointer_tag`, `original_size_bytes`, `schema_version` (must be `1`). Optionally store `display_name_*` (FR-019). `404` if `file_id` wasn't created by a prior `upload/assign` call. **`409 FILE_ALREADY_REGISTERED`** if the file already exists in `ACTIVE` status (re-registration of a completed upload is not permitted — use `DELETE /api/v1/file/{file_id}` first).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

CORRECT_ERROR_CODE_USED:
  # The core fix: must use the new, accurately-named code, never FILE_ALREADY_DELETED here
  $ grep -c "FILE_ALREADY_REGISTERED" internal/api/*.go
  EXPECT: >= 1
  $ grep -B5 -A5 "already.*ACTIVE\|status.*ACTIVE" internal/api/*.go | grep -c "FILE_ALREADY_DELETED" \
      && echo "FAIL: an ACTIVE file is not a deleted file — wrong code" || echo "PASS"

SCHEMA_VERSION_MUST_BE_ONE:
  $ grep -c "schema_version.*1\|SchemaVersion.*1" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestFileRegister ./internal/api/
  EXPECT: exit 0; tests include:
    TestFileRegisterSucceeds
    TestFileRegisterRejectsUnknownFileID           (404)
    TestFileRegisterRejectsReRegistrationOfActive  (409 FILE_ALREADY_REGISTERED, not FILE_ALREADY_DELETED)
    TestFileRegisterStoresDisplayNameWhenPresent

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.7.3 — Pointer File Retrieval (`GET /api/v1/file/{file_id}/pointer`)

**PRECONDITIONS:** Session 11.7.2 complete

**TASK:** Return `pointer_ciphertext`, `pointer_nonce`, `pointer_tag`, `schema_version`, `original_size_bytes`, optional `display_name_*`. Client decrypts locally; the microservice never decrypts (ADR-020, NFR-014).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

NEVER_DECRYPTS:
  $ grep -in "decrypt" internal/api/*.go | grep -v "_test.go\|comment\|never" \
      && echo "FAIL: the microservice must never attempt decryption (NFR-014)" || echo "PASS"

UNIT_TESTS:
  $ go test -v -run TestPointerFileRetrieval ./internal/api/
  EXPECT: exit 0; tests include:
    TestPointerFileReturnsCiphertextFieldsVerbatim
    TestPointerFileRejectsUnknownFileID  (404)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.7.4 — File Delete (`DELETE /api/v1/file/{file_id}`)

**PRECONDITIONS:** Session 11.7.3 complete

**TASK:** Mark all `chunk_assignments` `PENDING_DELETION`. Notify reachable providers immediately (unreachable ones retried on next heartbeat, FR-020). Return `assignments_marked`, `providers_notified`, `providers_pending`, `status: "DELETED"`. `409 FILE_ALREADY_DELETED` if already deleted (this is the **correct** use of this code — contrast with the fix in Session 11.7.2).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

MARKS_PENDING_DELETION_NOT_HARD_DELETE:
  $ grep -c "PENDING_DELETION" internal/api/*.go
  EXPECT: >= 1
  $ grep -n "DELETE FROM chunk_assignments\|DELETE FROM files" internal/api/*.go \
      && echo "FAIL: soft-delete only, chunk_assignments/files are never physically deleted here" \
      || echo "PASS"

CORRECT_USE_OF_ALREADY_DELETED_CODE:
  $ grep -c "FILE_ALREADY_DELETED" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestFileDelete ./internal/api/
  EXPECT: exit 0; tests include:
    TestFileDeleteMarksAllAssignmentsPendingDeletion
    TestFileDeleteRejectsAlreadyDeleted   (409 FILE_ALREADY_DELETED)
    TestFileDeleteReportsUnreachableProvidersAsPending

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.8 — Pricing Endpoints

**Reference:** OAS `paths./api/v1/pricing/*`, `components/schemas/PricingEstimateResponse`, FR-013, FR-057, ADR-024, FR-051

#### Session 11.8.1 — Storage Pricing Estimate (`GET /api/v1/pricing/estimate`)

**PRECONDITIONS:** None beyond Session 11.3.1

**TASK:** Public. Optional `file_size_bytes` (default 1 GB). Return `storage_rate_paise_per_gb_per_month`, `monthly_cost_paise`, `annual_cost_paise = monthly × 12`, `min_escrow_balance_paise = 30 days cost`. All `int64` paise.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

UNIT_TESTS:
  $ go test -v -run TestPricingEstimate ./internal/api/
  EXPECT: exit 0; tests include:
    TestPricingEstimateDefaultsToOneGB
    TestPricingEstimateAnnualIsTwelveXMonthly
    TestPricingEstimateAllValuesAreIntegerPaise

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.8.2 — Provider Earnings Estimate (`GET /api/v1/pricing/provider-estimate`)

**PRECONDITIONS:** Session 11.8.1 complete

> **Flagged and corrected — OAS disagrees with two agreeing sources on the vetting hold percentage.** OAS's `estimated_escrow_hold_vetting_paise` description says "30% hold during vetting." FR-051 ("the release cap must be 50%") and architecture.md's Payment System section ("the hold window is 60 days and the release cap is 50%... at most 50% of any month's earnings") both independently state 50%. Two agreeing, detailed sources beat one outlier description string; use 50%, and flag the OAS text for correction.

**TASK:** Public. Required: `storage_gb` (10–100000), `uptime_target_pct` (0–100). Return `gross_monthly_paise = storage_gb × rate × uptime_pct/100`; `estimated_escrow_hold_vetting_paise = gross × 0.50` (corrected from OAS's "30%"; FR-051, ARCH Payment System); `estimated_net_paise_vetting = gross × 0.50` (the 50% release cap, FR-051); `estimated_net_paise_post_vetting = gross × release_multiplier` at the assumed ≥0.95 score tier (ADR-024).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

VETTING_HOLD_USES_FIFTY_PERCENT:
  # The core fix: OAS's 30% figure disagrees with FR-051 and ARCH; use 50%
  $ grep -c "0\.50\|50%\|\* 0\.5\b" internal/api/*.go
  EXPECT: >= 2   # both the hold and the release-cap computation
  $ grep -n "0\.30\|30%" internal/api/*.go | grep -v "_test.go\|comment" \
      && echo "FAIL: 30% contradicts FR-051 and architecture.md's Payment System section (both say 50%)" \
      || echo "PASS"

UNIT_TESTS:
  $ go test -v -run TestProviderEarningsEstimate ./internal/api/
  EXPECT: exit 0; tests include:
    TestProviderEarningsGrossMonthlyFormula
    TestProviderEarningsVettingHoldIsFiftyPercent    (matches FR-051, not OAS's stale 30%)
    TestProviderEarningsPostVettingUsesFullMultiplierAtHighScore

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.9 — Audit Admin Endpoints

**Reference:** OAS `paths./api/v1/audit/challenge`, `components/schemas/AuditChallengeDispatchRequest/Response`, FR-037, ADR-002, ADR-006, ADR-014

#### Session 11.9.1 — Manual Audit Challenge (`POST /api/v1/audit/challenge`)

**PRECONDITIONS:** Milestone 7 complete (`audit.ChallengeNonce`)

**TASK:** AdminApiKey. Dispatches an immediate out-of-cycle challenge to `(provider_id, chunk_id)` — does **not** bypass the 24-hour (or `profile.PollingInterval` in demo) deduplication window. `202` with `challenge_nonce` (66 hex chars), `server_challenge_ts`, `deadline_ms = ceil((256 / p95_throughput_kbps) × 1500)`. `403` if provider is `DEPARTED`.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

USES_MILESTONE_7_CHALLENGENONCE:
  $ grep -c "audit.ChallengeNonce\|ChallengeNonce(" internal/api/*.go
  EXPECT: >= 1

DEADLINE_FORMULA:
  $ grep -c "1500\|× 1.5\|\* 1.5" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestManualAuditChallenge ./internal/api/
  EXPECT: exit 0; tests include:
    TestManualChallengeDoesNotBypassDedup
    TestManualChallengeRejectsDepartedProvider   (403)
    TestManualChallengeReturnsCorrectDeadlineMs

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.10 — Admin Endpoints

**Reference:** OAS `paths./api/v1/admin/*`, `components/schemas/AdminProvidersResponse`, `RepairQueueResponse`, `RepairJobItem`, `AuditStatsResponse`, `VettingStatusSummary`, NFR-027, FR-044, ADR-004, ADR-030

#### Session 11.10.1 — Admin Providers List (`GET /api/v1/admin/providers`)

**PRECONDITIONS:** Session 11.3.1 complete

**TASK:** Filters: `status`, `asn`, `region`, `multiaddr_stale`, `accelerated_reaudit`, `vetting_gc_pending`, `limit` (1–200, default 100), `cursor`. Include `vetting_chunks_assigned`, `vetting_chunk_cap`, `vetting_gc_pending` for VETTING providers.

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

UNIT_TESTS:
  $ go test -v -run TestAdminProvidersList ./internal/api/
  EXPECT: exit 0; tests include:
    TestAdminProvidersFiltersByStatus
    TestAdminProvidersFiltersByVettingGCPending
    TestAdminProvidersRequiresAdminAPIKey

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.10.2 — Repair Queue (`GET /api/v1/admin/repair/queue`)

**PRECONDITIONS:** Milestone 9 complete (`repair_jobs` queue populated)

> **Flagged and corrected — wrong NFR citation.** The original task attributes "`emergency_queued > 0` fires immediately" to NFR-027. NFR-027 lists four alert thresholds and none of them is per-priority-tier queue depth — it covers `total_queued > 1,000` only. OAS's own `emergency_queued` field description cites FR-044/ADR-004 instead ("these represent files that may be unrecoverable if one more provider fails"). Fixed.

**TASK:** Filters: `status` (default `QUEUED`), `priority`, `limit` (1–100, default 50), `cursor`. Counts by priority: `emergency_queued`, `permanent_departure_queued`, `pre_warning_queued`, `total_queued`. Alerts: `total_queued > 1000` (NFR-027); `emergency_queued > 0` fires immediately (FR-044, ADR-004 — corrected citation).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

CORRECT_CITATION_FOR_EMERGENCY_ALERT:
  $ grep -c "FR-044\|ADR-004" internal/api/*.go
  EXPECT: >= 1

PRIORITY_COUNTS_PRESENT:
  $ grep -c "emergency_queued\|permanent_departure_queued\|pre_warning_queued\|total_queued" internal/api/*.go
  EXPECT: >= 4

UNIT_TESTS:
  $ go test -v -run TestRepairQueueAdmin ./internal/api/
  EXPECT: exit 0; tests include:
    TestRepairQueueCountsByPriority
    TestRepairQueueFiltersByStatus

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.10.3 — Manual Repair Trigger (`POST /api/v1/admin/repair/trigger`)

**PRECONDITIONS:** Milestone 9 Phase 9.1 complete (`repair.EnqueueJob`)

**TASK:** Required: `chunk_id`, `segment_id`, `trigger_type`. Optional: `provider_id` (nullable for threshold-triggered repairs). `202` with the created `RepairJobItem`. Calls `repair.EnqueueJob` directly (with the profile parameter added in the Milestones 9–10 revision).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

WIRES_ENQUEUEJOB_WITH_PROFILE_PARAM:
  $ grep -c "repair.EnqueueJob(" internal/api/*.go
  EXPECT: >= 1
  $ grep -A3 "repair.EnqueueJob(" internal/api/*.go | grep -c "profile"
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestManualRepairTrigger ./internal/api/
  EXPECT: exit 0; tests include:
    TestManualRepairTriggerCreatesJob
    TestManualRepairTriggerAllowsNullProviderID

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.10.4 — Audit Statistics (`GET /api/v1/admin/audit/stats`)

**PRECONDITIONS:** Milestone 7 complete (`audit_receipts`)

**TASK:** `from`/`to` (default last 1h), optional `provider_id`. Backed by `audit_receipts WHERE abandoned_at IS NULL`. Return `challenges_issued`, `{pass, fail, timeout, pending}`, `pass_rate = pass/(pass+fail+timeout)`, `timeout_rate`, `content_hash_failures`, `jit_flags_raised`. Alert: `timeout_rate > 0.05` (NFR-027 — correctly cited here).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

EXCLUDES_ABANDONED_ROWS:
  $ grep -c "abandoned_at IS NULL" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestAuditStats ./internal/api/
  EXPECT: exit 0; tests include:
    TestAuditStatsDefaultsToLastHour
    TestAuditStatsExcludesAbandonedRows
    TestAuditStatsPassRateFormula

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.10.5 — Vetting Status (`GET /api/v1/admin/vetting/status`)

**PRECONDITIONS:** Milestone 9 Phase 9.1 complete (`IsVettingChunk`)

**TASK:** `include_gc_pending_only` (default false). Return `total_vetting_providers`, `total_synthetic_chunks_active`, `total_synthetic_chunks_pending_gc`, per-provider `vetting_summary` (`chunks_assigned`, `chunk_cap`, `cap_utilisation_pct`, `chunks_pending_gc`).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

UNIT_TESTS:
  $ go test -v -run TestVettingStatus ./internal/api/
  EXPECT: exit 0; tests include:
    TestVettingStatusAggregatesAcrossProviders
    TestVettingStatusIncludeGCPendingOnlyFilter

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

#### Session 11.10.6 — Vetting GC Retry (`POST /api/v1/admin/vetting/gc/retry`)

**PRECONDITIONS:** Milestone 14 stub acceptable at this stage (`vettingchunk.DeliverGCInstruction` may not exist yet — see note)

**TASK:** Required: `provider_id` (must be `ACTIVE` with `chunks_pending_gc > 0`). Attempts immediate delivery. Returns `delivery_attempted`, `chunks_pending_gc_before`, `chunks_pending_gc_after`. `200` with `delivery_attempted: false` if unreachable (background retry continues). Wires to `vettingchunk.DeliverGCInstruction()` (Milestone 14 — built after this milestone in the dependency graph; stub the call here and revisit once Milestone 14 lands).

**VERIFY:**

```bash
COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

REJECTS_NON_ACTIVE_OR_NO_PENDING_GC:
  $ grep -c "chunks_pending_gc > 0\|ChunksPendingGC > 0" internal/api/*.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestVettingGCRetry ./internal/api/
  EXPECT: exit 0; tests include:
    TestVettingGCRetryRejectsNonActiveProvider     (400)
    TestVettingGCRetryReturns200WhenUnreachable    (delivery_attempted: false, not an error)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

### Phase 11.11 — Per-Provider Chunk Count Ceiling

**Reference:** NFR-044, architecture.md §27.3 (`Per-provider storage ceiling at 100 Kbps budget` table — two reference points: 180d→~70GB, 300d→~130GB)

> **Flagged and corrected — the given formula doesn't reproduce its own reference points.** The original task computes `storage_advisory_gb = ceil(mttf_days / 300 * 130)`. At `mttf_days=300` this correctly gives 130 GB, but at `mttf_days=180` it gives `ceil(0.6 × 130) = 78` GB — not the ~70 GB architecture.md §27.3 documents. The relationship between MTTF and the bandwidth-driven storage ceiling is not linear (Giroire's formula scales BWavg with `D/N` in a way that doesn't reduce to simple proportionality in MTTF alone), so a naive straight-line formula between the two points silently produces a wrong number for any MTTF that isn't exactly 300 days. Fixed to use the two documented reference points as a lookup, rather than interpolating past a formula that already fails at one of its own two anchors.

#### Session 11.11.1 — Enforce chunk ceiling in assignment service

**PRECONDITIONS:** Session 11.7.1 complete (`POST /api/v1/upload/assign`)

**TASK:** Before generating shard assignments, exclude candidate providers whose current chunk allocation would push steady-state repair bandwidth above 100 Kbps (NFR-044). Ceiling by MTTF tier, from architecture.md §27.3's actual reference table — **do not linearly interpolate between them without deriving the true Giroire-formula relationship first**:

| MTTF tier | Storage ceiling |
|---|---|
| 180 days (desktop minimum) | ~70 GB |
| 300 days (planning target) | ~130 GB |

If the pool drops below `profile.MinActiveProviders` eligible candidates after applying the ceiling, return `503 INSUFFICIENT_PROVIDER_CAPACITY` (Session 11.1.1's `ErrInsufficientProviderCapacity` — flag for an `openapi.yaml` addition, this code doesn't exist in OAS yet either), `retry_after: 3600`. Surface `providers_near_ceiling_count` as an informational (non-gating) field on the readiness response. Add `storage_advisory_gb` to `ProviderRegisterResponse` and `ProviderStatusResponse`, populated from the two-point table above (nearest tier, or flagged for a proper interpolation formula once the underlying Giroire relationship is derived — do not ship a formula that contradicts architecture.md's own documented values).

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/api/router.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/api/
  EXPECT: exit 0

CEILING_MATCHES_DOCUMENTED_REFERENCE_POINTS:
  # The core fix: 180-day tier must resolve to ~70GB, not the formula's 78GB
  $ grep -c "70.*GB\|70 GB" internal/api/*.go
  EXPECT: >= 1
  $ grep -n "mttf_days / 300 \* 130\|mttfDays / 300 \* 130" internal/api/*.go \
      && echo "FAIL: this formula gives 78GB at 180 days, not the documented ~70GB" \
      || echo "PASS"

CAPACITY_ERROR_CODE_PRESENT:
  $ grep -c "InsufficientProviderCapacity\|INSUFFICIENT_PROVIDER_CAPACITY" internal/api/*.go
  EXPECT: >= 1

STORAGE_ADVISORY_ON_BOTH_RESPONSES:
  $ grep -c "storage_advisory_gb" internal/api/*.go
  EXPECT: >= 2   # ProviderRegisterResponse and ProviderStatusResponse

UNIT_TESTS:
  $ go test -v -run TestChunkCeiling ./internal/api/
  EXPECT: exit 0; tests include:
    TestChunkCeilingAt180DaysMatchesArchitectureDoc  (returns ~70GB, not 78GB)
    TestChunkCeilingAt300DaysMatchesArchitectureDoc  (returns ~130GB)
    TestChunkCeilingExcludesProvidersOverLimit
    TestChunkCeilingReturns503WhenPoolTooSmall        (INSUFFICIENT_PROVIDER_CAPACITY, retry_after=3600)

VET:
  $ go vet ./internal/api/
  EXPECT: exit 0; zero output
```

---

## Milestone 12 — Coordination Microservice (`cmd/microservice`)

**Deliverable:** A fully wired microservice that starts up, passes guard rails, loads the cluster secret, passes the readiness gate (in demo mode), dispatches and records audit challenges, and drives repair/release/departure background loops.

**Reference:** MVP §2.3 (startup guard rails), MVP §5.3 (profile selection), MVP §5.5 (application-layer `mv_provider_scores` regeneration at startup), IC §8 (secrets manager, fail-closed startup), IC §3.4 (readiness gate), IC §4.2 (audit challenge wire format), IC §9 (import graph — this is the one place `internal/audit`, `internal/scoring`, `internal/repair`, and `internal/payment` are all wired together), IC §2 (component communication map), ARCH §18 (gossip cluster, client-driven routing, background throttling), NFR-028

---

### Phase 12.1 — Microservice Startup

**Reference:** MVP §5.3, MVP §5.5, IC §8, IC §3.4, ARCH §18, NFR-028

> **Corrected formatting.** The original reference line had an unclosed markdown code span: `` OAS `components/securitySchemes, ARCH §18, REQ NFR-028 `` (missing backtick after `securitySchemes`). Fixed above.

> **Flagged — this session is the single place every deferred cross-package wire-up from Milestones 7–10 was pointed at; the original task text never made most of them explicit.** Rewritten below as one fully-ordered sequence rather than a numbered list followed by disconnected prose asides, so every ordering dependency (gossip ack before the readiness loop starts; stubs for pieces M17 hasn't built yet) is visible in one place.

#### Session 12.1.1 — Wire microservice `main()`

**PRECONDITIONS:**

- Milestones 1–10 complete (config, crypto, erasure, storage, p2p, audit, scoring, repair, payment) and Milestone 11 Phase 11.1–11.3 complete (error envelope, readiness handler, router)
- `internal/cluster/mock_cluster.go`'s `MockClusterMembership` is usable as a startup-time stub even though its permanent home is Milestone 17 Phase 17.2.1 — same for `internal/cluster/router.go`'s `ResponsibleReplica`

**TASK:**

Replace the stub in `cmd/microservice/main.go` with the following fully-ordered sequence — each step's prerequisites are the steps above it; do not reorder without re-checking the ordering notes inline:

```
 1. profile := config.SelectProfile()                          — print startup banner (MVP §5.3)
 2. err := config.ValidateStartupGuards(profile); halt on error
 3. secretsClient := <real Vault/AWS-SSM/GCP adapter (Milestone 17 Session 17.1.1)
      if profile.RequireSecretsManager, else an env-var adapter reading
      VYOMANAUT_CLUSTER_MASTER_SEED (IC §8; guarded against prod misuse by
      M1 Session 1.3.2's PROD_MODE_ENV_SECRET check, not re-checked here)>
    cache := audit.NewClusterSecretCache(secretsClient)
    if err := cache.Load(ctx); err != nil { fail-closed: do not start (IC §8) }
 4. db := <PostgreSQL connection pool>
 5. Drop and recreate mv_provider_scores using profile.ScoreWindow{Short,Medium,Long}
    (DM §7, MVP §5.5 — application-layer step, not a migration)
 6. IF profile.RequireQuorum:
      a. Read VYOMANAUT_SEED_NODE_1 / VYOMANAUT_SEED_NODE_2 (fail-fast if absent in prod)
      b. gossipCluster := cluster.NewGossipCluster(seeds)  — 1-second reconciliation ticker
         (until Milestone 17 Phase 17.2.1 lands, this is internal/cluster's
         MockClusterMembership returning healthy=3 — same stub-until-M17
         pattern as step 3's env-var secrets fallback)
      c. BLOCK until >= 2 peers ack membership (prevents split-brain false-ready
         on cold start) — this wait MUST complete before step 7
    ELSE:
      clusterMembership := a stub reporting healthy=1 always (demo mode)
 7. Start the readiness gate evaluator goroutine (Milestone 11 Session 11.2.1),
    60-second cycle (IC §3.4) — safe to start now that step 6 has resolved
 8. Start the HTTP server using the router from Milestone 11 Session 11.3.1
 9. router := cluster.NewRouter(clusterMembership)  — ResponsibleReplica(opType)
    for audit-dispatch/chunk-assignment hot paths (ARCH §18). Until Milestone 17
    Phase 17.2.1 provides the real gossip-aware implementation, THIS MUST BE A
    STUB that always returns the load balancer's address (i.e., the latency
    optimisation is a no-op until M17) — do not attempt the real
    membership-aware routing logic here.
10. p2pHost := <construct the real p2p.Host, Milestone 6>
11. repairTransport := p2pHost   — satisfies repair.RepairTransport (Milestone 9
    Session 9.2.1) structurally; internal/repair never imports internal/p2p
12. paymentProvider := <RazorpayProvider or MockProvider per profile.PaymentMode>
13. departureDetector := repair.NewDepartureDetector(db, profile, paymentProvider.Penalise)
    — this is the exact site Milestone 9 Session 9.3.1 deferred payment.Penalise
    to, and Milestone 9's own text explicitly names this session as the fix location
    go departureDetector.Run(ctx)
14. go func() { ticker := time.NewTicker(profile.PollingInterval); for range ticker.C {
       repair.PromoteStalePreWarningJobs(ctx, db, profile) } }()
    — nothing else in the build plan drives this; Milestone 9 Session 9.2.2 only
    defined the function, assuming a caller would exist here
15. go func() {  // repair executor loop — nothing else in the build plan
       for { job, err := repair.DequeueNextJob(ctx, db); ... ;
             run the download/reconstruct/upload pipeline (Milestone 9 Session 9.2.1,
             using repairTransport and a replacement chosen by
             repair.SelectReplacementProvider, Session 9.4.1) } }()
16. Start the audit challenge dispatch loop (Session 12.1.2 below),
    interval profile.PollingInterval
17. Start the release computation goroutine:
    IF profile.ReleaseComputationInterval == 0 (production): schedule for the
      23rd of each calendar month
    ELSE (demo): ticker at profile.ReleaseComputationInterval
    calls payment.ComputeMonthlyRelease(ctx, db, primaryDB, profile)
18. backgroundThrottle goroutine: samples db_read_p99_latency_ms every 60s
    (NFR-028); reduces backgroundSemaphore concurrency as it approaches 50ms,
    restores at < 30ms
```

**The authoritative JWT claim set** (OAS `components/securitySchemes/BearerAuth`): `sub` (entity UUID), `role` (`"owner"|"provider"`), `iss` (`"vyomanaut-microservice-v1"`), `exp` (Unix timestamp). TTL: 24h owners / 7d providers; registration token 1h, single-use, valid only for the matching register call.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f cmd/microservice/main.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./cmd/microservice/
  EXPECT: exit 0

STARTUP_ORDER_IS_A_SINGLE_SEQUENCE:
  # Every ordering dependency should now live in one place, not scattered prose
  $ grep -c "^// Step [0-9]\|^\t[0-9]\+\." cmd/microservice/main.go
  EXPECT: >= 15

FAIL_CLOSED_ON_SECRET:
  $ grep -c "cache.Load(ctx)\|ClusterSecretCache" cmd/microservice/main.go
  EXPECT: >= 1
  $ grep -A5 "cache.Load(ctx)" cmd/microservice/main.go | grep -c "os.Exit\|log.Fatal\|return err"
  EXPECT: >= 1

GOSSIP_ACK_BEFORE_READINESS_LOOP:
  # The core restructuring fix: the wait must appear before the readiness evaluator starts
  $ grep -n "2.*peer.*ack\|gossipCluster\|WaitForQuorum" cmd/microservice/main.go | head -1
  $ grep -n "readiness.*[Ee]valuator\|StartReadinessLoop" cmd/microservice/main.go | head -1
  # Manual check: gossip-ack line number < readiness-evaluator-start line number

DEPARTURE_DETECTOR_WIRES_PENALISE:
  # Closes the M9 Session 9.3.1 forward reference
  $ grep -c "NewDepartureDetector(db, profile, paymentProvider.Penalise)\|NewDepartureDetector.*Penalise" \
      cmd/microservice/main.go
  EXPECT: >= 1

REPAIR_TRANSPORT_IS_REAL_HOST_NOT_IMPORTED_DIRECTLY:
  # Closes the M9 Phase 9.2 forward reference — p2p.Host satisfies RepairTransport structurally
  $ grep -c "p2pHost\|repairTransport" cmd/microservice/main.go
  EXPECT: >= 1

PROMOTION_TICKER_PRESENT:
  # Closes the M9 Session 9.2.2 forward reference — nothing else calls this function
  $ grep -c "PromoteStalePreWarningJobs" cmd/microservice/main.go
  EXPECT: >= 1

REPAIR_EXECUTOR_LOOP_PRESENT:
  # Closes a gap: nothing else in the build plan consumes the repair queue
  $ grep -c "DequeueNextJob" cmd/microservice/main.go
  EXPECT: >= 1

RELEASE_COMPUTATION_WIRED:
  $ grep -c "ComputeMonthlyRelease" cmd/microservice/main.go
  EXPECT: >= 1

CLIENT_DRIVEN_ROUTING_IS_STUBBED_NOT_FULLY_BUILT:
  # Must not attempt the real gossip-aware router before M17 exists
  $ grep -c "stub.*load balancer\|no-op until M17\|until Milestone 17" cmd/microservice/main.go
  EXPECT: >= 1

BACKGROUND_THROTTLE_THRESHOLDS:
  $ grep -c "50.*ms\|50\*time.Millisecond" cmd/microservice/main.go
  EXPECT: >= 1
  $ grep -c "30.*ms\|30\*time.Millisecond" cmd/microservice/main.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestMicroserviceStartup ./cmd/microservice/
  EXPECT: exit 0; tests include:
    TestStartupFailsClosedOnUnreachableSecretsManager   (RequireSecretsManager=true, client errors -> process does not start)
    TestStartupDemoModeSkipsGossipWait                  (RequireQuorum=false -> no blocking wait, healthy=1 stub used)
    TestStartupProdModeBlocksUntilTwoPeerAck
    TestStartupWiresDepartureDetectorPenaliseCallback
    TestStartupRejectsProdModeWithEnvSecretPresent       (PROD_MODE_ENV_SECRET guard from M1 still fires)

NEGATIVE_CHECKS:
  $ grep -n "VYOMANAUT_CLUSTER_MASTER_SEED" cmd/microservice/main.go | grep -v "profile.RequireSecretsManager\|comment" \
      && echo "WARN: verify this read is gated on RequireSecretsManager==false, matching M1's guard" \
      || echo "PASS"

VET:
  $ go vet ./cmd/microservice/
  EXPECT: exit 0; zero output
```

#### Session 12.1.2 — Audit challenge dispatch loop

**PRECONDITIONS:**

- Session 12.1.1 complete (`ClusterSecretCache` loaded, DB pool available)
- Milestone 7 revision complete — specifically the corrected `ValidateResponse` signature (6 parameters, including `serverChallengeTsMs` and `providerID`) and `ClusterSecretCache.IsVersionValid`
- Milestone 8 revision complete — specifically `IncrementConsecutivePasses`'s added `profile` parameter and `UpdateRTO`

> **Flagged — this is the caller Milestone 7's `ValidateResponse` revision explicitly deferred the secret-version check to, and it was never made explicit here.** The original step 7 ("On response: call `audit.ValidateResponse()`, then `audit.WriteReceiptPhase2()`") is missing the `cache.IsVersionValid(nonce[0])` check that Milestone 7 Phase 7.2 requires the caller to perform — without it, item 2 of `ValidateResponse`'s own documented contract is never actually checked by anyone. Added below.

**TASK:**

For each active chunk in `active_chunk_assignments`:

```
1. Determine provider multiaddrs: providers.last_known_multiaddrs (primary);
   DHT fallback if multiaddr_stale = true
2. versionByte, serverSecret, _ := cache.CurrentSecret()   // Milestone 7 Phase 7.4
   nonce := audit.ChallengeNonce(serverSecret, versionByte, chunkID, serverTsMs)
3. Build fields := audit.ReceiptFields{ChunkID: chunkID, FileID: fileID (nil for
   vetting chunks), ProviderID: providerID, ChallengeNonce: nonce,
   ServerChallengeTs: now, AddressWasStale: multiaddrStale}
   receiptID, err := audit.WriteReceiptPhase1(ctx, db, fields)   // before dispatch
4. Open libp2p stream to the provider via /vyomanaut/audit-challenge/1.0.0
   (repairTransport-equivalent for audit — the real p2p.Host from Session 12.1.1)
5. Send ChallengeRequest frame (IC §4.2: chunk_id(32) || challenge_nonce(33) ||
   server_challenge_ts_ms(8) = 73 bytes)
6. Apply the per-provider RTO timeout: scoring.PoolMedianRTO(ctx, db) if
   rto_sample_count < 5, else avg_rtt_ms + 4*var_rtt_ms (IC §4.2, Milestone 8)
7. On response:
   a. IF NOT cache.IsVersionValid(nonce[0]): treat as invalid — do not proceed
      to a PASS/FAIL write (Milestone 7 Phase 7.2's item 2, deferred to here)
   b. err := audit.ValidateResponse(nonce, responseHash, serverChallengeTsMs,
      providerID, providerSig, providerPubKey)   // corrected 6-param signature
   c. audit.WriteReceiptPhase2(ctx, db, receiptID, result, serviceSig, serviceTS)
8. IF result == AuditPass:
     scoring.IncrementConsecutivePasses(ctx, db, providerID, profile)  // profile param added in M8 revision
   ELSE IF NOT (result == AuditTimeout AND fields.AddressWasStale):
     scoring.ResetConsecutivePasses(ctx, db, providerID)
   // else: a stale-address TIMEOUT — do nothing, per DM §4.7 (Milestone 8 Phase 8.2)
9. IF result != AuditTimeout:
     scoring.UpdateRTO(ctx, db, providerID, responseLatencyMs, throughputKbps)
     // never called for TIMEOUT — no response_latency_ms to sample (Milestone 8 Phase 8.3)
```

**Concurrency** (IC §4.2): the microservice may open multiple concurrent challenge streams to a single provider — one goroutine per chunk assignment, bounded by a semaphore.

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f cmd/microservice/audit_dispatch.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./cmd/microservice/
  EXPECT: exit 0

VERSION_VALIDITY_CHECK_PRESENT:
  # The core fix: closes the gap Milestone 7's ValidateResponse revision deferred here
  $ grep -c "IsVersionValid" cmd/microservice/audit_dispatch.go
  EXPECT: >= 1

VALIDATERESPONSE_USES_CORRECTED_SIGNATURE:
  $ grep -c "ValidateResponse(nonce, responseHash, serverChallengeTsMs, providerID, providerSig, providerPubKey)\|ValidateResponse(.*serverChallengeTsMs.*providerID" \
      cmd/microservice/audit_dispatch.go
  EXPECT: >= 1

INCREMENTCONSECUTIVEPASSES_HAS_PROFILE_PARAM:
  $ grep -A2 "IncrementConsecutivePasses(" cmd/microservice/audit_dispatch.go | grep -c "profile"
  EXPECT: >= 1

STALE_ADDRESS_TIMEOUT_DOES_NOT_RESET:
  $ grep -c "AddressWasStale" cmd/microservice/audit_dispatch.go
  EXPECT: >= 1

UPDATERTO_NEVER_CALLED_FOR_TIMEOUT:
  $ grep -B3 "scoring.UpdateRTO(" cmd/microservice/audit_dispatch.go | grep -c "!= AuditTimeout\|result == AuditPass\|result == AuditFail"
  EXPECT: >= 1

PHASE1_BEFORE_DISPATCH:
  $ grep -n "WriteReceiptPhase1" cmd/microservice/audit_dispatch.go | head -1
  $ grep -n "NewStream\|audit-challenge/1.0.0" cmd/microservice/audit_dispatch.go | head -1
  # Manual check: WriteReceiptPhase1 line number < stream-open line number

FRAME_SIZE_IS_73_BYTES:
  $ grep -c "73" cmd/microservice/audit_dispatch.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestAuditDispatchLoop ./cmd/microservice/
  EXPECT: exit 0; tests include:
    TestDispatchRejectsResponseWithInvalidSecretVersion   (IsVersionValid=false -> no PASS/FAIL written)
    TestDispatchCallsValidateResponseWithFullSigningInput  (serverChallengeTsMs and providerID both passed)
    TestDispatchIncrementsPassesOnPass
    TestDispatchDoesNotResetOnStaleAddressTimeout
    TestDispatchResetsOnGenuineTimeout                     (fresh address, no response -> counter resets)
    TestDispatchSkipsUpdateRTOForTimeout
    TestDispatchUsesPoolMedianRTOForNewProviders           (rto_sample_count < 5)

NEGATIVE_CHECKS:
  $ grep -n "Vyomanaut_V2/internal/repair\|Vyomanaut_V2/internal/payment" cmd/microservice/audit_dispatch.go \
      && echo "WARN: confirm this file is the microservice entrypoint, where such imports ARE permitted (IC §9)" \
      || echo "PASS (no such imports needed in this specific file)"

VET:
  $ go vet ./cmd/microservice/
  EXPECT: exit 0; zero output
```

---

### Appendix — Cross-milestone follow-ups surfaced by this revision

These are not new sessions in Milestones 11–12 — they're one-line retroactive additions to earlier, already-revised milestones, surfaced only because this pass wired everything together end-to-end:

| Follow-up | Where it belongs | What to add |
|---|---|---|
| `ClusterSecretCache.IsLoaded() bool` | Milestones 7–8 revision, Phase 7.4 | Trivial accessor; Session 11.2.1 (readiness condition 7) and Session 12.1.1 (startup) both need it |
| `GET /api/v1/provider/downtime` path + `getActiveDowntime` schema | `docs/api/openapi.yaml` (not a build_part2.md session) | Response shape already specified in Session 11.6.5: `{active, promised_return_at, penalty_fires_at}` |
| `INSUFFICIENT_PROVIDER_CAPACITY` error code | `docs/api/openapi.yaml` | Used by Session 11.11.1; not yet in the OAS `Error` schema examples |
| Owner-withdrawal reversal path | Milestone 10 Phase 10.3 (Session 10.3.1's `payout.reversed` handler) | Branch on RELEASE-vs-WITHDRAWAL reference; insert `owner_escrow_events` `DEPOSIT` row for the owner case (proposed in Phase 11.5, Session 11.5.6) |

---

## Milestone M-OBS — Observability & Metrics

**Deliverable:** All Prometheus metrics defined in NFR-025 and NFR-026 are exported with the correct naming convention (NFR-046). Grafana alert rules are defined. The background throttle metric is wired. The `TestNoOrphanMetricName` CI check passes.

**Reference:** architecture.md §23 (metric catalogue, alert thresholds), requirements.md §5.6 (NFR-025, NFR-026, NFR-027, NFR-028, NFR-046)

---

### Phase OBS.1 — Microservice Metrics

#### Session OBS.1.1 — Register all microservice Prometheus metrics

**Reference:** NFR-025 (requirements.md §5.6), architecture.md §23, NFR-046

Create `internal/metrics/microservice.go`. Register every metric from NFR-025 using the `vyomanaut_{subsystem}_{name}_{unit}` naming pattern from NFR-046, where `{subsystem}` matches the `internal/` package name:

| Metric name | Type | Labels | Package |
| --- | --- | --- | --- |
| `vyomanaut_audit_challenges_issued_total` | Counter | — | audit |
| `vyomanaut_audit_results_total` | Counter | `result` (PASS/FAIL/TIMEOUT) | audit |
| `vyomanaut_provider_score` | Histogram | — | scoring |
| `vyomanaut_repair_queue_depth` | Gauge | — | repair |
| `vyomanaut_repair_jobs_completed_total` | Counter | — | repair |
| `vyomanaut_escrow_events_total` | Counter | `type` (DEPOSIT/RELEASE/SEIZURE/REVERSAL) | payment |
| `vyomanaut_microservice_replica_count` | Gauge | `state` (healthy/degraded) | cluster |
| `vyomanaut_db_read_latency_seconds` | Histogram | percentile p99 | audit |

The `vyomanaut_db_read_latency_seconds` histogram bucket boundaries must include 0.01, 0.025, 0.05 (the throttle threshold), 0.1 seconds. The background throttle goroutine in M12 Session 12.1.1 reads `vyomanaut_db_read_latency_seconds` p99 from the in-process registry — no external scrape needed for the throttle decision.

Expose all metrics at `/metrics` on a separate admin port (configurable, default 9090).

#### Session OBS.1.2 — Wire metric increments at call sites

Add `metrics.AuditResultsTotal.With(prometheus.Labels{"result": result.String()}).Inc()` at the Phase 2 receipt write call site (M7 Session 7.3.2). Add `metrics.RepairQueueDepth.Set(float64(depth))` in the repair enqueue/dequeue call sites (M9 Phase 9.1). Add `metrics.EscrowEventsTotal.With(...).Inc()` in `InsertEscrowEvent` (M10 Phase 10.2.1). These increments must use the exact metric names from Session OBS.1.1 — any rename is a breaking change per NFR-046.

---

### Phase OBS.2 — Provider Daemon Metrics

#### Session OBS.2.1 — Register all provider daemon Prometheus metrics

**Reference:** NFR-026, architecture.md §23

Create `internal/metrics/daemon.go`. Register:

| Metric name | Type |
| --- | --- |
| `vyomanaut_chunks_stored_total` | Counter |
| `vyomanaut_audit_responses_sent_total` | Counter |
| `vyomanaut_audit_response_latency_milliseconds` | Histogram |
| `vyomanaut_vlog_append_latency_milliseconds` | Histogram |
| `vyomanaut_content_hash_failures_total` | Counter |
| `vyomanaut_heartbeat_sent_total` | Counter |
| `vyomanaut_daemon_ram_constrained` | Gauge (label: `constrained`) |

Expose at a local-only HTTP server on `localhost:9091` (not publicly reachable). The provider daemon CLI status interface (FR-029) reads its data from this endpoint rather than maintaining a separate in-memory state.

---

### Phase OBS.3 — Operational Alerts

#### Session OBS.3.1 — Define Grafana alert rules**

**Reference:** NFR-027 (requirements.md §5.6), architecture.md §23

Create `deployments/grafana/alerts.yaml` with the four mandatory alert rules from NFR-027:

| Alert name | Condition | Severity |
| --- | --- | --- |
| `RepairQueueDepthHigh` | `vyomanaut_repair_queue_depth > 1000` | warning |
| `AuditTimeoutRateHigh` | `rate(vyomanaut_audit_results_total{result="TIMEOUT"}[1h]) / rate(vyomanaut_audit_results_total[1h]) > 0.05` | critical |
| `ContentHashFailureDetected` | `increase(vyomanaut_content_hash_failures_total[7d]) > 0` | critical |
| `MicroserviceReplicasDegraded` | `vyomanaut_microservice_replica_count{state="healthy"} < 3` | critical |

`AuditTimeoutRateHigh` at > 5% in a 1-hour window triggers the relay infrastructure runbook. `ContentHashFailureDetected` for any provider in a 7-day window triggers accelerated re-audit of all that provider's chunks (architecture.md §23).

**Add the Grafana dashboard JSON:** to `deployments/grafana/dashboards/vyomanaut.json`. The dashboard JSON must reference metric names by exact string — any metric rename without simultaneous dashboard update is a breaking change (NFR-046). Add a CI check `TestGrafanaMetricNamesMatchRegistry` in `scripts/ci/grep_checks.sh` that greps all metric name strings from the dashboard JSON and verifies they exist in `internal/metrics/*.go`.

---

### Phase OBS.4 — Prometheus Metric Naming CI Gate

#### Session OBS.4.1 — Implement `TestNoOrphanMetricName`

**Reference:** NFR-046 (requirements.md §5.5)

Add to `scripts/ci/grep_checks.sh` a fifth grep-fail check: extract all metric name strings from `deployments/grafana/dashboards/vyomanaut.json` and `deployments/grafana/alerts.yaml`, then verify each appears in at least one `.go` file under `internal/metrics/`. A metric name present in dashboards/alerts but not in the Go registry (or vice versa) fails the check with the message `"orphan metric name: {name} — update metrics/*.go and dashboards simultaneously"`. This check is CI check 16 (add it to Phase 0.3 Session 0.3.1 step list).

---

## Milestone 13 — Provider Daemon Core (`cmd/provider`)

**Deliverable:** Provider daemon with stream handlers for all four libp2p protocols,
startup identity, and heartbeat integration.

**Reference:** IC §4.1 (chunk upload stream), IC §4.2 (audit challenge stream),
IC §4.4.1 (repair download stream), IC §4.5 (vetting GC stream), IC §3.1 (heartbeat),
IC §4 (common rules: transport auth, 0-RTT, framing)

---

### Phase 13.1 — Provider Startup

**Reference:** MVP §5.3, IC §3.1

#### Session 13.1.1 — Wire provider `main()`

**Task:** Replace the stub in `cmd/provider/main.go` with:

1. Parse flags per MVP §8.3 table (`--microservice-url`, `--data-dir`, `--declared-storage-gb`,
   `--relay-addrs`, `--sim-count`, `--sim-base-port`, `--sim-data-dir`, `--sim-asn-count`)
2. Call `config.SelectProfile()` and `config.ValidateStartupGuards(profile)`
3. Load or generate Ed25519 identity (`internal/p2p/identity.go`)
4. Initialise `ChunkStore` — call `RecoverFromCrash()` before starting writer goroutine
5. Start the writer goroutine (the only goroutine that may call `AppendChunk`)
6. Initialise libp2p `Host` with QUIC+TCP transports
7. Register stream handlers (Phases 13.2–13.5)
8. Start heartbeat goroutine + DHT republication (IC §3.1, IC §12.2)
9. Register the DHT custom validator using the constant from `dht_namespace.go` (IC §12)

---

### Phase 13.2 — Chunk Upload Stream Handler

**Reference:** OAS `components/schemas/ShardAssignment.capability_token` description and `UploadAssignResponse`, OAS `components/schemas/ShardAssignment.properties`, IC §4.1 (complete wire format, all status codes, capability token verification)

#### Session 13.2.1 — Implement `/vyomanaut/chunk-upload/1.0.0` handler

**Task:** Register the stream handler for `/vyomanaut/chunk-upload/1.0.0` on the
provider daemon. The handler must implement the full Frame 1 verification per IC §4.1:

1. Read the 4-byte length prefix; reject with `0x01` if `length > 262252`
2. Parse `chunk_id` (32B), `shard_index` (4B), `capability_token` (72B), `chunk_data` (262144B)
3. Capability token verification steps 1–5 exactly per IC §4.1:
   - `len(capability_token) == 72 = 8-byte expiry_unix_ms (big-endian int64) ‖ 64-byte Ed25519 signature` → reject `0x03` (Wire encoding on the HTTP side: hex string, pattern `^[0-9a-f]{144}$`)
   - Parse bytes 0–7 as big-endian int64 `expiry_unix_ms`
   - Check `expiry_unix_ms > NOW_unix_ms - 30_000` → reject `0x07` (CAPABILITY_EXPIRED)
   - Verify Ed25519 signature (bytes 8–71) → reject `0x03` if invalid
   - `chunk_id` mismatch via signing input → same `0x03` path
4. Verify `SHA-256(chunk_data) == chunk_id` → respond `0x02` before any disk write
5. Write to vLog via the writer goroutine channel (not directly — single-writer rule)
6. On success (`0x00`): compute `provider_sig` over
   `SHA-256(chunk_id || shard_index || provider_id_bytes || timestamp_unix_ms)` (IC §4.1)
7. Return Frame 2 with `status` byte and (on success) `provider_sig`

Handle `0x06 ALREADY_STORED` for idempotent re-uploads (IC §4.1).

Pre-condition check: `providers.status` must be `ACTIVE` or `VETTING` — if `DEPARTED`,
reset the stream immediately (IC §4.1).

---

### Phase 13.3 — Audit Challenge Stream Handler

**Reference:** IC §4.2 (complete wire format, all status codes)

#### Session 13.3.1 — Implement `/vyomanaut/audit-challenge/1.0.0` handler

**Task:** Register the stream handler. 0-RTT is PROHIBITED (IC §4.2) — the `Host`
middleware from M6 Phase 6.1.1 handles this automatically via protocol ID ending in
`-challenge`. The handler must:

1. Read Frame 1: `chunk_id` (32B), `challenge_nonce` (33B), `server_challenge_ts_ms` (8B)
   — reject with `0x03` if nonce is not 33 bytes (IC §4.2)
2. Check nonce version byte against valid `server_secret_vN` versions
3. Bloom filter check → `LookupChunk()`
4. If not found: respond `0x01` with `provider_sig` (IC §4.2 Frame 2 note: `0x01/0x02`
   responses are 1+64 bytes, not 1 byte)
5. Verify `SHA-256(chunk_data) == content_hash` → respond `0x02` on mismatch (IC §4.2)
6. Compute `response_hash = SHA-256(chunk_data || challenge_nonce)`
7. Compute `provider_sig` over `SHA-256(response_hash || challenge_nonce ||
   server_challenge_ts_ms || provider_id)` (IC §4.2)
8. Respond Frame 2 with `0x00`, `response_hash`, `provider_sig`

Concurrency: handle at least 32 concurrent challenge streams (IC §4.2). Each stream
runs in a goroutine; `LookupChunk` is goroutine-safe.

---

### Phase 13.4 — Repair Download Stream Handler

**Reference:** IC §4.4.1 (repair download stream, 0-RTT prohibited)

#### Session 13.4.1 — Implement `/vyomanaut/repair-download/1.0.0` handler

**Task:** Register the stream handler. 0-RTT PROHIBITED (IC §4.4.1). The handler must:

1. Verify the requesting Peer ID is a registered microservice replica (locally-cached
   microservice peer list refreshed via DHT and heartbeat acknowledgements — IC §4.4.1)
2. Reject unregistered Peer IDs with `0x02 NOT_AUTHORISED` immediately
3. Verify `repair_auth_sig`: Ed25519 over
   `SHA-256(chunk_id || request_ts_ms || microservice_peer_id)` (IC §4.4.1)
4. Call `LookupChunk()` — respond `0x01` if not found, `0x03` if corruption
5. Respond Frame 2 with `0x00` and `chunk_data`
6. Timeout: 10,000ms (IC §4.4.1)

---

### Phase 13.5 — Vetting GC Stream Handler

**Reference:** IC §4.5 (vetting GC protocol, complete wire format), IC §5.3 (`DeleteChunk`)

#### Session 13.5.1 — Implement `/vyomanaut/vetting-gc/1.0.0` handler

**Task:** Register the stream handler. 0-RTT PROHIBITED (IC §4.5). The handler must:

1. Read Frame 1: `chunk_count` (4B), `chunk_ids` (`chunk_count × 32B`)
2. For each chunk ID: call `DeleteChunk()` from `ChunkStore`
3. Construct `failure_bitmap`: bit N set if `DeleteChunk(chunk_ids[N])` failed
4. Respond Frame 2: `0x00` if all succeeded, `0x01` with bitmap on partial failure,
   `0x02` on `INTERNAL_ERROR`
5. Maximum 10,000 chunk IDs per frame (IC §4.5); multiple sequential frames on same stream
6. Timeout: 30,000ms per frame (IC §4.5 — longer than audit challenge timeout)

The protocol handler must remain in the daemon binary indefinitely (IC §13 versioning):
"Removing the protocol handler from the daemon requires a coordinated network-wide migration."

---

### Phase 13.6 — Provider RAM Check at Installation

#### Session 13.6.1 — Implement pre-installation RAM check**

**Reference:** NFR-045 (requirements.md §5.5), architecture.md §27.5

In `cmd/provider/main.go` during the startup sequence, before `ChunkStore` is initialised, compute the minimum free RAM required for the DHT record cache: `required_mb = ceil(declared_storage_gb × 400 × 200 / 1_048_576)`. At 50 GB declared this is approximately 40 MB; at 200 GB approximately 160 MB; at 500 GB approximately 400 MB. Read available free memory via a platform-appropriate syscall (`/proc/meminfo` on Linux, `sysctl` on macOS, `GlobalMemoryStatusEx` on Windows — implement in `internal/storage/memcheck_linux.go`, `_darwin.go`, `_windows.go`). If available free RAM < required, print a human-readable warning: `"[WARN] Declared storage requires ~{required_mb} MB free RAM for DHT cache; only {available_mb} MB detected. Chunk assignment will be limited until RAM is freed."` — and reduce `declared_storage_gb` to the safe ceiling for the available RAM. Do not halt the daemon; log the advisory at WARN level and emit the `vyomanaut_daemon_ram_constrained{constrained="true"}` gauge metric. The installer (packaged binary) must run this check before completing the installation wizard and surface the shortfall with a plain-language advisory screen.

---

## Milestone 14 — Vetting & Synthetic Chunks (`internal/vettingchunk`)

**Deliverable:** Synthetic chunk lifecycle: generation, upload to providers, GC delivery
on ACTIVE transition, departure cleanup. Enforces Invariant 6 end-to-end.

**Reference:** IC §5.10 (`Generator` and `GCDelivery` interfaces, sentinel errors),
IC §4.5 (vetting GC protocol — client side), DM §3 Invariant 6, DM §4.5 (`is_vetting_chunk`
column semantics)

---

### Phase 14.1 — Synthetic Chunk Generator

**Reference:** IC §5.10 (`Generator` interface: `GenerateChunk`, `CurrentCount`, `Cap`)

#### Session 14.1.1 — Implement `Generator`

**Task:** In `internal/vettingchunk/generator.go`, implement `GenerateChunk()` per
IC §5.10:

1. Generate 256 KB via `crypto/rand` — NOT retained by the microservice after confirmation
2. Compute `chunkID = SHA-256(data)`
3. Upload via `/vyomanaut/chunk-upload/1.0.0` to the provider (using the same client
   as the upload orchestrator in M15)
4. On success: INSERT `chunk_assignments` row with `is_vetting_chunk = TRUE`,
   `segment_id = NULL`, `shard_index = NULL` (IC §5.10, DM §4.5)

**Cap enforcement (IC §5.10):** `Cap(declaredStorageGB int) int = floor(declaredStorageGB × 400)`.
The caller is responsible for checking `CurrentCount < Cap` before calling `GenerateChunk`
(IC §5.10 pre-condition).

---

### Phase 14.2 — GC Delivery

**Reference:** IC §5.10 (`GCDelivery` interface, `ErrProviderOffline`), IC §4.5
(vetting GC protocol — the libp2p client side initiated by the microservice)

#### Session 14.2.1 — Implement `DeliverGCInstruction()`

**Task:** In `internal/vettingchunk/gc.go`, implement `DeliverGCInstruction()` per
IC §5.10. Triggered immediately after `providers.status` transitions to `ACTIVE`:

1. Query all synthetic chunk IDs: `WHERE is_vetting_chunk = TRUE AND provider_id = $1
   AND status = 'ACTIVE'`
2. Batch into frames of ≤ 10,000 chunk IDs (IC §4.5)
3. Open libp2p stream `/vyomanaut/vetting-gc/1.0.0` to the provider
4. For each frame: send `VettingGCRequest`, await `VettingGCResponse` within 30,000ms
5. On `0x00`: mark batch as `DELETED` in `chunk_assignments`
6. On `0x01` (partial failure): retry failed entries on next connection
7. On provider offline: set all rows to `PENDING_DELETION`, return `ErrProviderOffline`
   for retry on next heartbeat (IC §4.5, IC §5.10)

Retry backoff: `profile.GCRetryBackoff` (IC §4.5: prod is `[5m, 15m, 60m]`, demo is
`[10s, 30s, 2m]` — profile-variable, NOT hardcoded).

---

## Milestone 15 — Client SDK (`internal/client`)

**Deliverable:** Upload orchestrator, retrieval orchestrator, account management, and
file management. The client SDK does NOT import `cmd/` (IC §9).

**Reference:** IC §5.9 (`UploadOrchestrator` interface, ERRATA note, all sentinel errors),
IC §4.1 (upload stream — client initiator side), MVP §8.2 (file inventory per package),
IC §11 (forbidden: no business logic in cmd/)

---

### Phase 15.1 — Account Management

**Reference:** IC §5.1 (crypto primitives used here), MVP §8.2 (`internal/client/account/`)

#### Session 15.1.1 — Implement registration and keystore

**Task:** In `internal/client/account/register.go` and `keystore.go`:

- `Register()`: generate Ed25519 key pair, derive master secret via `DeriveMasterSecret`
  with `profile.Argon2*` params (NOT hardcoded), display BIP-39 mnemonic, run
  `SelectConfirmationWords()` gate if `!profile.SkipMnemonicConfirm`
- Keystore: encrypt Ed25519 key + pointer file nonce counter under `DeriveKeystoreEncKey()`
- Recovery paths: passphrase (re-derive master secret) and mnemonic (via
  `MnemonicToMasterSecret`)

---

### Phase 15.2 — Upload Orchestrator

**Reference:** IC §5.9 (`UploadOrchestrator` ERRATA note — capability tokens),
IC §4.1 (chunk upload stream client side), IC §5.9 (session state for FR-060 resume), `OAS §components/schemas/PointerFilePlaintextSegment` for the pointer file struct layout when building the ciphertext payload passed to `POST /api/v1/file/register`.

#### Session 15.2.1 — Implement `UploadFile()` with capability token handling

**Task:** In `internal/client/upload/orchestrator.go`, implement the upload lifecycle:

1. Segment the file (padding to `DataShards × ShardSize` minimum)
2. AONT encode each segment via `crypto.AONTEncodeSegment()`
3. RS encode via `erasure.Engine.EncodeSegment()`
4. Call `POST /api/v1/upload/assign` to get provider assignments AND capability tokens
5. **ERRATA (IC §5.9):** Include each `capability_token` verbatim in the corresponding
   `UploadRequest` frame. On `0x07 CAPABILITY_EXPIRED`: re-call the assignment endpoint
   with the same `file_id` (idempotent — returns same providers with fresh tokens)
6. Upload all shards in parallel (bounded goroutine pool)
7. Register the encrypted pointer file with the microservice
8. Persist session state for resume (FR-060): `file_id`, `chunk_ids`, `ack_status[TotalShards]`

**Note:** The OAS `FileRegisterRequest` includes `display_name_ciphertext`, `display_name_nonce`, `display_name_tag` as optional fields (FR-019). Populate these from the client's locally-encrypted filename before calling `POST /api/v1/file/register`. Key derivation: `HKDF-SHA256(master_secret, salt=owner_id, info="vyomanaut-filename-v1" ‖ file_id`(OAS `FileRegisterRequest.properties.display_name_ciphertext` description).

#### Session 15.2.2 — Implement `ResumeUpload()`

**Task:** Load persisted session state, identify unacknowledged shards, re-upload only
those (FR-060 — do not retransmit acknowledged shards, IC §5.9).

---

### Phase 15.3 — Retrieval Orchestrator

**Reference:** IC §5.9 (`RetrieveFile`, `ErrPointerTagMismatch`, `ErrTooFewShards`,
`ErrCanaryMismatch`), IC §5.1 (`DecryptPointerFile`, `AONTDecodePackage`), `OAS §components/schemas/PointerFilePlaintextSegment` while parsing the decrypted pointer file on retrieval.

#### Session 15.3.1 — Implement `RetrieveFile()`

**Task:** In `internal/client/retrieve/orchestrator.go`:

1. Fetch pointer ciphertext from microservice
2. Derive pointer key via `DerivePointerEncKey()`
3. Call `DecryptPointerFile()` — on `ErrTagMismatch`, return `ErrPointerTagMismatch`
   with no plaintext (IC §5.9)
4. For each segment: dial providers in parallel, cancel after `DataShards` valid shards
5. Verify each shard's content address before passing to RS decode
6. RS decode via `erasure.Engine.DecodeSegment()`
7. `AONTDecodePackage()` — on `ErrCanaryMismatch`: zero the buffer and return
   `ErrCanaryMismatch` (IC §5.1, IC §5.9: "caller MUST NOT return any plaintext")
8. Strip padding to `original_size_bytes`
9. Concatenate segments in `segment_index` order

---

## Milestone 16 — Demo Mode Validation

**Deliverable:** A fully runnable demo that completes the 30-minute lifecycle from MVP
§3.6 timeline. `go test` suite validates all demo-specific behaviours and mode-invariant
properties.

**Reference:** MVP §3.6 (demo timeline), MVP §7 (viability fact-checks), MVP §6 (switching
requirements), MVP §8.4 (CI checks 14–15)

---

### Phase 16.1 — End-to-End Demo Test

**Reference:** MVP §3.6

#### Session 16.1.1 — Implement `TestDemoTimeline` integration test

**Task:** Create `scripts/test/demo_timeline_test.go`. Using `--sim-count=5
--sim-asn-count=5` and `VYOMANAUT_MODE=demo`, run the demo timeline from MVP §3.6:

- Assert readiness gate passes within 60s of startup
- Assert file upload succeeds for a file ≤ 1.25 MB
- Assert first audit PASS is recorded within 3 minutes
- Assert VETTING→ACTIVE transition completes within 12 minutes
- Assert synthetic chunk GC is delivered after ACTIVE transition
- Kill one simulated daemon; assert departure detection within `profile.DepartureThreshold`
  (10 min in demo)
- Assert repair job created and completed

**One more assertion:** Readiness response `mode` field equals `"demo"` when `VYOMANAUT_MODE=demo`. The OAS `ReadinessResponse.properties.mode` and the `DemoReady` example confirm this field is present in both modes (OAS `paths./api/v1/admin/readiness.get.responses.200.examples.DemoReady`). Demo thresholds: `active_vetted_providers.required_value=5`, `distinct_asns.required_value=5`, `distinct_metro_regions.required_value=1`, `microservice_quorum.required_value=1`, `relay_nodes_deployed.required_value=0`.

This test is tagged `//go:build integration` and runs separately from unit tests.

#### Session 16.1.2 — Validate all viability fact-checks from MVP §7

**Task:** Add individual tests for each fact-check in MVP §7:

- MVP §7.1 (ASN cap): assert `MinDistinctASNs=5` in DemoProfile (already in M1 tests)
- MVP §7.2 (RS math): assert repair succeeds when 2 of 5 providers are offline
- MVP §7.3 (vetting timing): assert ACTIVE transition happens at ~10 min in demo
- MVP §7.7 (mock idempotency): assert duplicate webhook delivery produces exactly one
  `escrow_events` row

---

### Phase 16.2 — Simulation Mode

**Reference:** MVP §8.3 (`--sim-count` flag), IC §10 (simulation mode paths)

#### Session 16.2.1 — Implement `--sim-count` multi-instance provider

**Task:** In `cmd/provider/main.go`, when `--sim-count=N`, spawn N goroutines each
running an independent provider daemon instance with:

- Separate Ed25519 identity
- Separate RocksDB instance at `/tmp/vyomanaut-sim/{zero-padded-index}/db/`
- Separate vLog at `/tmp/vyomanaut-sim/{zero-padded-index}/vlog/chunks.vlog`
- Auto-assigned synthetic ASN `SIM-AS{1..N}` from `--sim-asn-count`

Paths must use the exact format from IC §10 naming conventions: `{instance_id}` is
zero-padded to 4 digits (e.g. `0000`, `0001`).

---

## Milestone 17 — Production Hardening

**Status:** Circuit relay integration and HA microservice clustering require deployment
topology details from `architecture.md`. The Razorpay live integration and secrets manager
adapters (Vault, AWS SSM, GCP) can proceed.

**Reference:** IC §4.3 (circuit relay), IC §8 (secrets manager — interface defined;
adapters not yet specified without architecture.md), MVP §6.2 (infrastructure requirements)

---

### Phase 17.1 — Secrets Manager Adapters

**Reference:** IC §8, MVP §6.2 (IR-03)

#### Session 17.1.1 — Implement secrets manager adapters

**Task:** Implement `SecretsManagerClient` (IC §8 interface) for three backends:

- HashiCorp Vault (`internal/secrets/vault.go`)
- AWS SSM Parameter Store (`internal/secrets/aws_ssm.go`)
- GCP Secret Manager (`internal/secrets/gcp_secret.go`)

Each adapter reads the secret at path `/vyomanaut/audit-secret/v{N}` (IC §8 path
convention). Each must handle the 24-hour rotation overlap window: read both `v{N}` and
`v{N+1}` when both exist (IC §8 rotation contract). Selection among adapters is via
`VYOMANAUT_SECRETS_BACKEND` environment variable.

---

### Phase 17.2 — HA Microservice & Relay Nodes

#### Session 17.2.1 — Implement gossip cluster (`internal/cluster`)

**Reference:** ARCH §18

Create `internal/cluster/gossip.go` implementing the three-replica gossip membership per architecture.md §18. The `GossipCluster` struct must expose:

- `HealthyCount() int` — returns the count of peers with a last-seen timestamp within the last 5 seconds
- `MemberAddresses() []url.URL` — returns the current membership list for client-driven routing
- A `reconcile()` loop running at 1-second intervals: select one randomly-chosen peer, exchange membership histories via a `POST /internal/membership/sync` HTTP call carrying a vector clock
- Two pre-configured seed node addresses read from `VYOMANAUT_SEED_NODE_1` and `VYOMANAUT_SEED_NODE_2`; these prevent partition on restart (architecture.md §18)

Quorum check: `HealthyCount() >= 2` satisfies the (3,2,2) write quorum (architecture.md §18). A read or write that cannot reach 2 replicas must return `ErrQuorumUnavailable`.

Add `internal/cluster/router.go` implementing `ResponsibleReplica(opType string) *url.URL` per the client-driven routing description in M12 Session 12.1.1.

Add `internal/cluster/mock_cluster.go` (build tag `test`) providing `MockClusterMembership` that returns configurable healthy counts for unit testing without a live cluster.

Wire into `cmd/microservice/main.go`: after guard rails pass, initialise `GossipCluster`, wait for 2-peer ack, then start the readiness evaluator.

#### Session 17.2.2 — Relay node binary and deployment configuration

**Reference:** ARCH §13, §24, §27.5

Create `cmd/relay/main.go` as the relay node binary. The relay runs a libp2p host with Circuit Relay v2 enabled and no DHT, chunk storage, or audit logic. Configuration:

- 128 concurrent relay reservations per node (architecture.md §13, §27.5)
- Reservation TTL: 30 minutes (libp2p default)
- Relay multiaddrs are reported via a `GET /relay/status` HTTP health endpoint returning `{"reservation_count": N, "capacity": 128}`
- Metrics: expose `vyomanaut_relay_reservations_active` gauge at `/metrics`

Create `deployments/production/relay/docker-compose.yml` for the three-node relay deployment:

- Node 1: Mumbai AZ1 (`ap-south-1a`)
- Node 2: Mumbai AZ2 (`ap-south-1b`)
- Node 3: Chennai/Hyderabad (`ap-south-2` or `ap-southeast-1`)
- Minimum spec per node: 1 vCPU, 1 GB RAM, 1 Gbps network (architecture.md §24)

---

## Milestone 18 — Launch Readiness

**Status:** Runbooks and benchmark scripts are structurally defined by document references.

**Reference:** MVP §8.5 (runbooks list), MVP §8.5 (benchmark scripts), IC §10 (runbook
filenames — must match exactly)

---

### Phase 18.1 — Runbooks

**Reference:** IC §10 (naming conventions — exact filenames), MVP §8.5 (list of 8
required runbooks, all must exist before M8 private beta)

#### Session 18.1.1 — Create the 8 required runbooks

**Task:** Create all 8 runbook files in `runbooks/` using EXACT filenames per IC §10
(Grafana alert links depend on these names):

1. `microservice-failover.md`
2. `postgres-failover.md`
3. `relay-node-replacement.md`
4. `secrets-manager-outage.md`
5. `razorpay-api-outage.md`
6. `provider-mass-departure.md`
7. `rbi-holiday-table-update.md`
8. `audit-secret-rotation.md`

Each runbook must include:

- Trigger conditions (Prometheus alert name)
- Step-by-step recovery procedure
- Rollback procedure
- For `rbi-holiday-table-update.md`: the annual update procedure for `rbi_holidays.go`
  (IC §11 — hardcoded RBI holiday data is forbidden outside that file)
- For `audit-secret-rotation.md`: the full rotation procedure per IC §8 rotation contract
  (24-hour overlap window, `v{N}` and `v{N+1}` both present during overlap)

---

### Phase 18.2 — Benchmark Scripts

**Reference:** MVP §8.5 (four benchmark scripts, minimum-spec hardware requirements)

#### Session 18.2.1 — Create benchmark scripts

**Task:** Create `scripts/benchmarks/` with the four scripts from MVP §8.5:

- `aont_encode.sh` (Q16-1): AONT encode benchmarks on minimum-spec hardware
  (dual-core, no AES-NI, 2 GB RAM, 7200 RPM HDD)
- `argon2id.sh` (Q18-1): Argon2id performance benchmarks (must show demo vs prod params)
- `rocksdb_ssd.sh` (Q27-1 SSD): RocksDB throughput on SSD
- `rocksdb_hdd.sh` (Q27-1 HDD): RocksDB throughput on HDD

Each script must output pass/fail against the following thresholds from `requirements.md §7.4`:

- **`aont_encode.sh` (Q16-1):** median ≤ 200 ms, p99 ≤ 400 ms. If median > 200 ms in the Go implementation, re-evaluate segment size; any change requires a coordinated ADR-003 + ADR-004 update.
- **`argon2id.sh` (Q18-1):** median ≤ 500 ms at production parameters (t=3, m=64 MB, p=4). Acceptable with spinner UI up to 1000 ms. Fallback ladder: try t=2/m=64 MB, then t=3/m=32 MB, then t=2/m=32 MB. Do not go below m=32768 KiB (OWASP 2023 minimum). Demo parameters (t=1, m=4 MB, p=1) are not benchmarked on this script.
- **`rocksdb_ssd.sh` (Q27-1 SSD):** p99 audit latency ≤ 100 ms at the highest RocksDB rate limiter setting that does not violate it. Write throughput at the chosen rate must be ≥ 2 MB/s.
- **`rocksdb_hdd.sh` (Q27-1 HDD):** p99 audit latency ≤ 200 ms on a 7200 RPM consumer HDD under concurrent compaction.

A fifth script must be added:

- **`postgres_insert_ceiling.sh` (NFR-043):** measure the sustained INSERT rate on the `audit_receipts` schema with row security policy enabled at which p99 write latency first exceeds 50 ms. This measured ceiling must be written back into `architecture.md §27.4` replacing the `5,000–10,000 rows/sec` planning estimate. This benchmark is a V2 launch blocker per NFR-043.

A sixth script for end-to-end upload latency:

- **`e2e_upload.sh` (NFR-033):** measure p50 time from upload start (encoding begins) to all 56 signed receipts collected and pointer file stored. Pass criterion: p50 ≤ 3 minutes for a 100 MB file on a provider network where p50 upload throughput is 10 Mbps (requirements.md §5.9 NFR-033).

---

### Phase 18.3 — Security Verification Checklist

**Reference:** MVP `§Security Verification Checklist` (referenced in IC §12 — the
TestDHTKeyValidatorPersists test references this section)

#### Session 18.3.1 — Document and run security checklist

**Task:** Create `docs/system-design/security-verification-checklist.md` with all
verifiable security properties derivable from the three documents in context:

- [ ] `TestDHTKeyValidatorPersists` passes (IC §12, CI check 5)
- [ ] `TestNoFloatArithmetic` passes (IC §11, CI check 6)
- [ ] Grep: no `challenge_nonce BYTEA(32)` in any file (CI check 8)
- [ ] All Ed25519 signing inputs use fixed-layout bytes, not JSON (IC §3.2)
- [ ] `ErrTagMismatch` path returns no plaintext bytes (IC §5.1)
- [ ] `ErrCanaryMismatch` path zeros the buffer before returning (IC §5.1)
- [ ] AONT key K is fresh `crypto/rand` on every segment encode (IC §11)
- [ ] 0-RTT disabled for audit-challenge and vetting-gc streams (IC §4)
- [ ] `VYOMANAUT_CLUSTER_MASTER_SEED` absent in all production env configs (IC §8)
- [ ] Razorpay error bodies never forwarded to API callers (IC §3.3)
- [ ] No Razorpay live keys in source files (IC §11)
- [ ] Demo and production databases are separate instances (MVP §6.2 IR-01)
- [ ] `challenge_nonce CHECK (octet_length(challenge_nonce) = 33)` in schema (DM §3 Invariant 5)
- [ ] `escrow_events` has no FLOAT columns (DM §3 Invariant 4)
- [ ] Row security policies enabled on `audit_receipts` and `escrow_events` (DM §6)
- [ ] `TestDHTKeyValidatorPersists` re-run after every `go-libp2p` upgrade (IC §12)
- [ ] Relay overhead < 50ms RTT from Indian cloud nodes (IC §4.3 NFR-006) — measured

Add the following entries to `docs/system-design/security-verification-checklist.md`:

**From requirements.md §5.4 (NFR-014 through NFR-020):**

- [ ]  NFR-014: `grep -r "pointer_key\|pointer_enc_key\|file_key" internal/api/` returns zero results — decryption keys never flow through the microservice layer
- [ ]  NFR-015: `ValidateResponse()` in `internal/audit/validate.go` only verifies the Ed25519 signature, never the `response_hash` preimage — and the comment `LIMITATION: microservice cannot verify response_hash` is present (IC §5.5)
- [ ]  NFR-016: All libp2p connections use TLS 1.3 (QUIC) or Noise XX (TCP); `TestTransportAuthentication` passes in M6 test suite
- [ ]  NFR-017: `DeriveD HTKey()` in `internal/crypto/hkdf.go` uses HMAC-SHA256 and the DHT validator rejects plaintext CID-format keys — `TestDHTKeyValidatorPersists` (CI check 5)
- [ ]  NFR-018: `VYOMANAUT_CLUSTER_MASTER_SEED` is absent from all production environment configs; `TestProdModeEnvSecretGuard` (M1 Session 1.3.3) passes
- [ ]  NFR-019: `DecryptPointerFile()` uses `crypto/subtle.ConstantTimeCompare` for the Poly1305 tag — confirmed by `TestConstantTimeTagComparison` in M2 Phase 2.5
- [ ]  NFR-020: Every `ChallengeNonce()` call returns a `[33]byte` with version byte at index 0 — confirmed by CI grep check 8 and the return type annotation

**From requirements.md §5.5 (NFR-021 through NFR-031):**

- [ ]  NFR-021: The `audit_receipts` RSP (DM §6) allows only the single PENDING→terminal UPDATE; `TestAuditReceiptRSPBlocksDelete` in M4 migration check script passes
- [ ]  NFR-022: No UPDATE or DELETE policy exists on `escrow_events`; `TestEscrowEventsRSPInsertOnly` passes
- [ ]  NFR-023: `TestSingleWriterGoroutine` (CI check 4) passes with `race` flag
- [ ]  NFR-024: `RecoverFromCrash()` is called before the writer goroutine starts; confirmed by startup sequence order in M13 Session 13.1.1
- [ ]  NFR-029: No UPI Collect API endpoint strings in any source file — confirmed by CI grep check 11
- [ ]  NFR-030: Every Razorpay payout call in `internal/payment/razorpay.go` includes `X-Payout-Idempotency` header — confirmed by `TestPayoutIdempotencyHeaderPresent`
- [ ]  NFR-031: `rbi_bank_holidays_YYYY` table is present for the current year and next year in `internal/payment/`; the `rbi-holiday-table-update.md` runbook has been executed for the current December cycle

---

### Phase 18.4 — Final CI Gate

**Reference:** MVP §8.4 (all 15 CI checks), MVP §6.4 MR-03 (migration checklist against both profiles)

#### Session 18.4.1 — All-green CI verification

**Task:** Verify all 15 CI checks from MVP §8.4 pass on a clean build:

1. `go build ./...` — zero warnings
2. `go vet ./...` — zero output
3. `golangci-lint run` — zero findings
4. `go test ./... -race` — all pass
5. `TestDHTKeyValidatorPersists` — pass
6. `TestNoFloatArithmetic` — pass
7. Migration apply + rollback against CI Postgres (both profiles per MVP §6.4 MR-03)
8. Grep: no `challenge_nonce BYTEA(32)` — pass
9. Grep: no float in payment — pass
10. Grep: no non-existent ADR reference — pass
11. Grep: no UPI Collect endpoint — pass
12. Mermaid render check — pass
13. Hyperlink check — pass
14. `TestProfileShardSizeIsConstant` — pass
15. `TestProfileBothFullySpecified` — pass

All 15 must pass simultaneously. Document the passing state in a milestone sign-off
commit with the message `milestone: M18 launch-readiness all-green CI`.

---
