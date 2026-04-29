# Vyomanaut V2 — Testing Strategy

**Status:** Authoritative — read before generating any test code.
**Version:** 1.0
**Date:** April 2026
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research
**Companion documents:**
- [`docs/system-design/mvp.md`](docs/system-design/mvp.md) — milestone exit criteria (the ground truth for "does this work")
- [`docs/system-design/data-model.md`](docs/system-design/data-model.md) — five invariants tests must enforce
- [`coding-guide.md`](coding-guide.md) — test code must follow all coding rules

---

## 0. Testing Philosophy

This system handles encrypted user data and real money. The consequence of a silent correctness bug is permanent, unrecoverable data loss or financial error. Testing is not a quality gate — it is the primary mechanism by which the five invariants in `data-model.md §3` are enforced independently of application code.

**Three testing tiers:**

| Tier | What it tests | When it runs |
|---|---|---|
| Unit | Individual exported functions, pure logic, edge cases | Every `go test ./...` — must be fast (<1s per package) |
| Integration | Cross-package interactions, DB invariants, two-phase writes | On every PR; requires real Postgres and temp filesystem |
| System / E2E | Full critical paths end-to-end against 56 simulated providers | Milestone exit criterion; must pass before milestone closes |

All three tiers run with `-race`. A data race is a critical bug.

---

## 1. The Four Critical Paths (from mvp.md)

Every test in this document traces to at least one of these paths. A test that cannot be traced to a critical path is a candidate for deletion.

| Path | Milestone(s) | Failure consequence |
|---|---|---|
| **P1** Encode → Upload → Store | M1, M2, M3, M7 | Corrupted data that passes audits; permanent loss |
| **P2** Challenge → Respond → Score | M2, M3, M4, M5 | Providers earn without holding data |
| **P3** Departure → Repair | M1, M3, M5, M6 | Files fall below reconstruction floor silently |
| **P4** Audit pass → Escrow release | M5, M6 | Providers work for free, or paid twice |

---

## 2. Unit Tests — Package by Package

### 2.1 `internal/crypto`

Tests are pure (no I/O, no DB). Every function must be deterministic given the same inputs, except those that call `crypto/rand`.

| Test name | Function under test | Critical path | What it asserts |
|---|---|---|---|
| `TestAONTRoundTrip/ChaCha20` | `AONTEncodeSegment`, `AONTDecodePackage` | P1 | Encode then decode, canary passes, plaintext == original |
| `TestAONTRoundTrip/AES` | Same | P1 | Same via AES-NI path |
| `TestAONTCanaryVerification` | `AONTDecodePackage` | P1 | Flip any single bit in any codeword → `ErrCanaryMismatch` |
| `TestAONTKeyNotRecoverable` | `AONTEncodeSegment` | P1 | With fewer than s+1 codewords, K cannot be computed |
| `TestPointerFileEncryptDecrypt` | `EncryptPointerFile`, `DecryptPointerFile` | P1 | Encrypt → decrypt → identical plaintext |
| `TestPointerFileTagTamper` | `DecryptPointerFile` | P1 | One byte changed in ciphertext → `ErrTagMismatch` |
| `TestPointerFileConstantTime` | `DecryptPointerFile` | P1 | Uses `crypto/subtle.ConstantTimeCompare` — verified by code inspection in test |
| `TestKeyHierarchyDeterminism` | `DeriveFileKey`, `DerivePointerEncKey`, `DeriveKeystoreEncKey` | P1 | Same inputs → same 32-byte key, always |
| `TestArgon2idDeterminism` | `DeriveMasterSecret` | P1 | Same passphrase + ownerID → same master secret |
| `TestArgon2idParameters` | `DeriveMasterSecret` | P1 | Panics / errors if passphrase < 8 bytes |
| `TestDomainSeparation` | All `Derive*` functions | P1 | `DeriveFileKey` ≠ `DerivePointerEncKey` for same inputs |

**Fuzz targets (mandatory):**
- `FuzzAONTDecode` — random byte slices as AONT package input; must never crash, must return error
- `FuzzPointerFileDecrypt` — random ciphertext; must return `ErrTagMismatch`, never panic

### 2.2 `internal/erasure`

| Test name | What it asserts | Critical path |
|---|---|---|
| `TestEncodeProducesTotalShards` | `EncodeSegment` returns exactly 56 slices | P1 |
| `TestShardSizeFixed` | Every shard is exactly 262,144 bytes | P1 |
| `TestDecodeFromExactly16` | Decode from exactly 16 shards → original package | P1 |
| `TestDecodeFromAny16` | 100 random subsets of 16 shards → identical result | P1 |
| `TestDecodeFailsBelow16` | 15 shards → `ErrTooFewShards` | P1 |
| `TestRoundTripFullPipeline` | Plaintext → encode → 56 shards → decode any 16 → identical plaintext | P1 |
| `TestPaddingRoundTrip` | File < 4 MB pads to segment size; strips correctly on decode | P1 |

### 2.3 `internal/storage`

Integration tests — require a real filesystem, temp directory per test.

| Test name | What it asserts | Critical path |
|---|---|---|
| `TestStoreAndRetrieve` | Store chunk; retrieve by chunkID; data identical | P2 |
| `TestContentHashVerifiedOnRead` | Corrupt one byte in vLog entry; LookupChunk returns `ErrContentHashMismatch` | P2 |
| `TestBloomFilterNegative` | LookupChunk for un-stored chunkID never reads from disk (measure I/O) | P2 |
| `TestSingleWriterEnforced` | 100 concurrent goroutines store chunks; no vLog overlap; all 100 retrievable | P2 |
| `TestCrashRecovery` | Write 100 chunks; kill without RocksDB flush; restart; all 100 retrievable | P1, P2 |
| `TestGarbageCollection` | Delete 50 of 100 chunks; GC; deleted absent; rest intact | P3 |
| `TestRateLimiterSSD` | Concurrent writes + compaction; p99 audit lookup ≤ 100 ms | P2 |
| `TestRateLimiterHDD` | Same on rotational disk (or simulated 12 ms seek latency) | P2 |
| `TestVLogFsyncFailure` | Inject fsync error; AppendChunk returns `ErrVLogFsync`; vLog is not partially written | P1 |

### 2.4 `internal/p2p`

Integration tests — require a real libp2p host; run in CI with loopback network.

| Test name | What it asserts | Critical path |
|---|---|---|
| `TestPeerIdentityPersistence` | Ed25519 key pair survives daemon restart; Peer ID identical | P2 |
| `TestQUICConnection` | Two hosts on same machine connect and exchange a test message | P1, P2 |
| `TestNoiseXXFallback` | With QUIC blocked (UDP iptables drop), falls back to TCP+Noise XX | P2 |
| `TestDHTKeyValidator` | HMAC-derived key accepted; plain SHA-256 CID rejected; 31-byte key rejected | P2 |
| **`TestDHTKeyValidatorPersists`** | **After go-libp2p version bump, validator still rejects plain CIDs** | P2 |
| `TestHeartbeatSigned` | Heartbeat payload verifies against registered Ed25519 public key | P2 |
| `TestHeartbeatInterval` | Heartbeat fires within 4h ± 5 minutes | P2 |
| `Test0RTTDisabledForAudit` | Audit protocol streams reject early data (DisableEarlyData) | P2 |
| `TestRelayFallback` | AutoNAT classifies as symmetric NAT; relay path used for audit | P2 |

> `TestDHTKeyValidatorPersists` is marked **critical** — it must run on every PR that upgrades `go-libp2p`.

### 2.5 `internal/audit`

| Test name | What it asserts | Critical path |
|---|---|---|
| `TestChallengeNonceIs33Bytes` | `ChallengeNonce` always returns exactly 33 bytes | P2, Invariant 5 |
| `TestChallengeNonceVersionByte` | Byte 0 == `versionByte` argument | P2 |
| `TestChallengeNonceDeterministic` | Same inputs → same nonce | P2 |
| `TestCrossReplicaValidation` | Nonce from replica-A validates on replica-B sharing same `server_secret_vN` | P2 |
| `TestReceiptInsertOnly` | `UPDATE audit_receipts SET chunk_id = ...` returns permission denied from `vyomanaut_app` role | Invariant 1 |
| `TestReceiptDeleteProhibited` | `DELETE FROM audit_receipts` returns permission denied | Invariant 1 |
| `TestReceiptPhase1Durability` | Kill after Phase 1 INSERT; on restart, PENDING row present in DB | P2 |
| `TestReceiptPhase2Idempotency` | Retry with same `challenge_nonce`; returns existing `service_sig`, no duplicate row | P2 |
| `TestAuditPassRoundTrip` | Challenge → provider responds → countersigned → `audit_result = 'PASS'` | P2 |
| `TestAuditFailNotFound` | Provider without chunk responds with status 0x01 → FAIL recorded | P2 |
| `TestAuditCorruptionFail` | `ErrContentHashMismatch` from storage → FAIL with corruption code recorded | P2 |
| `TestJITFlagThreshold` | Response time < 0.3× deadline → `jit_flag = true` | P2 |
| `TestAbandonedRowGC` | PENDING row older than 48h; GC sets `abandoned_at`; row not deleted | Invariant 1 |
| `TestRTOPerProvider` | After 5 audit responses, scheduler uses per-provider formula not pool median | P2 |

### 2.6 `internal/scoring`

| Test name | What it asserts | Critical path |
|---|---|---|
| `TestThreeWindowScore` | Known audit history → correct per-window scores (0.50/0.30/0.20 weights) | P4 |
| `TestDualWindowFlag` | 30d score − 7d score > 0.20 → flag set; ≤ 0.20 → flag clear | P4 |
| `TestVettingTransition` | 80th consecutive PASS → provider status becomes ACTIVE | P3 |
| `TestConsecutivePassReset` | One FAIL resets `consecutive_audit_passes` to 0 | P3 |
| `TestAcceleratedReauditTrigger` | > 1 FAIL in 7-day window → `accelerated_reaudit = true` | P2 |

### 2.7 `internal/repair`

| Test name | What it asserts | Critical path |
|---|---|---|
| `TestEnqueueJob` | SILENT_DEPARTURE trigger → `PERMANENT_DEPARTURE` priority | P3 |
| `TestPriorityOrdering` | `PERMANENT_DEPARTURE` dequeues before `PRE_WARNING` | P3 |
| `TestEmergencyFloorTrigger` | Fragment count at s=16 → immediate enqueue | P3 |
| `TestLazyThreshold` | Fragment count at s+r0=24 → pre-warning enqueue | P3 |
| `TestASNCap` | Replacement provider selection never places > 11 shards on one ASN | P3 |
| `TestRepairUpdatesAssignments` | After repair: `chunk_assignments` updated; old provider absent | P3 |

### 2.8 `internal/payment`

| Test name | What it asserts | Critical path |
|---|---|---|
| **`TestNoFloatArithmetic`** | `grep -rn "float64\|float32" internal/payment/` returns empty | Invariant 4 |
| `TestEscrowLedgerBalance` | SUM(DEPOSIT) − SUM(RELEASE+SEIZURE) = correct balance | P4, Invariant 2 |
| `TestEscrowInsertOnly` | `UPDATE escrow_events` returns permission denied | Invariant 2 |
| `TestEscrowDeleteProhibited` | `DELETE FROM escrow_events` returns permission denied | Invariant 2 |
| `TestIdempotentRelease` | Two RELEASE calls with same `idempotency_key` → one row, `ErrDuplicateIdempotencyKey` | P4 |
| `TestReleaseMultiplierTable` | All four score bands map to correct multiplier (1.00/0.75/0.50/0.00) | P4 |
| `TestVettingPeriodCap` | VETTING provider release capped at 50% | P4 |
| `TestSeizureOnDeparture` | DEPARTED provider's 30-day balance moved to SEIZURE event | P3, P4 |
| `TestLastWorkingDay` | `lastWorkingDayOfMonth` correct for weekends and RBI holidays | P4 |
| `TestRazorpayIdempotencyHeader` | Every mock Razorpay payout call asserts header presence | P4 |
| `TestEscrowNonTransferable` | Transfer to different `provider_id` returns 403 | P4 |

---

## 3. Integration Tests — Cross-Package

These require the full Postgres schema (from `migrations/001_initial_schema.sql`)
and a temp filesystem. Run with `go test -tags=integration ./...`.

| Test name | Packages exercised | Critical path |
|---|---|---|
| `TestFullAuditCycleAgainstDB` | audit + storage + scoring | P2 |
| `TestRepairWithRealDB` | repair + scoring + payment | P3, P4 |
| `TestEscrowReleaseWithRazorpayMock` | payment + audit periods | P4 |
| `TestProviderLifecycleDB` | scoring (vetting) + audit + payment | P3, P4 |
| `TestRowSecurityPolicies` | audit (INSERT-only) + payment (INSERT-only) | Invariants 1, 2 |
| `TestCrashRecoveryWithIndexRebuild` | storage + audit (challenge after recovery) | P1, P2 |

---

## 4. System / E2E Tests — Milestone Exit Criteria

These are the scripts referenced in `mvp.md` milestone exit criteria. They must pass
before a milestone is declared done.

### M0 Exit
```bash
go run ./cmd/provider --sim-count=56 --sim-asn-count=5 \
  --microservice-url=http://localhost:8080 --sim-data-dir=/tmp/vyomanaut-sim &
curl http://localhost:8080/api/v1/admin/readiness | jq .
# All 7 conditions: FAIL (expected at M0)
```

### M1 Exit
```bash
go test ./internal/crypto/... ./internal/erasure/... -v -race
go test ./internal/crypto/... -bench=BenchmarkAONTEncode14MB -benchtime=100x
# p50 ≤ 200 ms on minimum-spec hardware (recorded with hardware spec)
```

### M2 Exit
```bash
go test ./internal/storage/... -v -race -tags=integration
scripts/test-crash-recovery.sh  # 1000 chunks; kill mid-write; restart; verify all 1000
```

### M4 Exit
```bash
scripts/test-audit-cycle.sh
# 56 challenges issued; 56 PASS receipts; 0 PENDING rows > 48h
```

### M5 Exit
```bash
scripts/test-self-healing.sh
# Kill 32 providers (count drops to 24 = s+r0); repair enqueued within 60s;
# after repair: 56 providers hold the chunk
```

### M7 Exit
```bash
scripts/test-end-to-end.sh
# Register owner; upload 10 MB; verify 56 shards in DB; retrieve;
# SHA-256 match; canary pass; delete; 404 on re-retrieve
```

### M8 Exit (Production)
```bash
curl https://api.vyomanaut.com/api/v1/admin/readiness | jq '.all_conditions_met'
# true
scripts/test-live-e2e.sh --file=test-10mb.bin
# Upload → retrieve → SHA-256 match in production
scripts/test-microservice-failover.sh
# 2-replica cluster continues; replica-3 recovers within 30 seconds
```

---

## 5. Benchmark Protocol (from `docs/research/benchmarking-protocol.md`)

All benchmarks that correspond to NFR targets must be run on the minimum-spec machine:
**dual-core, no AES-NI, 2 GB RAM, 7200 RPM HDD.**

Record and commit alongside benchmark output:
```
Hardware: [CPU model, RAM, disk type]
OS: [kernel version]
Go: [version]
Date: [ISO 8601]
Result: [p50, p99 in ms]
Pass/Fail: [vs NFR target]
```

| Benchmark | NFR | Target | Q-number |
|---|---|---|---|
| `BenchmarkAONTEncode14MB/ChaCha20` | NFR-009 | p50 ≤ 200 ms, p99 ≤ 400 ms | Q16-1 |
| `BenchmarkArgon2id` | NFR-010 | p50 ≤ 500 ms | Q18-1 |
| `BenchmarkAuditLookup/SSD` | NFR-008 | p99 ≤ 100 ms | Q27-1 |
| `BenchmarkAuditLookup/HDD` | NFR-008 | p99 ≤ 200 ms | Q27-1 |

If a benchmark fails on minimum-spec hardware, the session must pause.
Apply the fallback protocol from `docs/research/benchmarking-protocol.md` and update the relevant ADR.

---

## 6. What We Explicitly Do Not Test (and Why)

| Skipped | Reason |
|---|---|
| Razorpay live API in unit tests | Mock via `PaymentProvider` interface; live tested only in M8 |
| Secrets manager in unit tests | Injected via `VYOMANAUT_CLUSTER_MASTER_SEED` env var in test mode |
| Cross-ISP NAT traversal | Simulated via loopback; real NAT fraction measured via telemetry (OQ-002) |
| Replay attacks against real QUIC | Covered by `Test0RTTDisabledForAudit` + libp2p's own test suite |
| RBI holiday correctness for years > current | Static table; verified annually in December deployment |

---

## 7. Test File Naming Conventions

```
internal/crypto/
  aont_test.go           — AONT encode/decode unit tests
  aont_fuzz_test.go      — AONT fuzz targets
  pointer_file_test.go   — pointer file encrypt/decrypt
  key_hierarchy_test.go  — HKDF + Argon2id

internal/storage/
  storage_test.go        — store/retrieve/GC unit tests
  crash_recovery_test.go — crash recovery integration test
  bench_test.go          — NFR-008 benchmarks

internal/audit/
  audit_test.go          — challenge, receipt, two-phase write
  invariants_test.go     — row security policy assertions

internal/payment/
  payment_test.go        — escrow ledger, release multiplier
  no_float_test.go       — TestNoFloatArithmetic
```

---

## 8. CI Requirements

Every PR must pass:

```bash
go build ./...            # zero warnings (strict mode)
go vet ./...              # pass
go test ./... -race       # zero data races
go test -tags=integration ./... -race  # integration suite
golangci-lint run         # .golangci.yml config
```

Additionally, any PR touching `internal/payment/` must pass `TestNoFloatArithmetic`.
Any PR upgrading `go-libp2p` must pass `TestDHTKeyValidatorPersists`.

---

*This document defines the testing contract for Vyomanaut V2.
Update it via PR when a milestone adds a new exit criterion or a new invariant is discovered.*