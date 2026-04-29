# Vyomanaut V2 — Coding Guide

**Status:** Authoritative — read this before generating any code for Vyomanaut V2.
**Version:** 1.1
**Date:** April 2026
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research
**Authority chain:** ADR wins over this guide. This guide wins over general Go convention.
**Companion documents:**
- [`docs/system-design/interface-contracts.md`](docs/system-design/interface-contracts.md) — package-level Go interfaces (function signatures are binding)
- [`docs/system-design/data-model.md`](docs/system-design/data-model.md) — five invariants that code must never violate
- [`docs/system-design/requirements.md`](docs/system-design/requirements.md) — FR/NFR IDs code must satisfy
- [`testing-strategy.md`](testing-strategy.md) — test names, tiers, and CI tag requirements

---

## 0. Mandatory Pre-Read for Every Session

Before writing a single line, confirm:

1. The milestone this code belongs to (M0–M8 in `docs/system-design/mvp.md`)
2. The exact FR/NFR IDs it satisfies
3. The invariants it must not violate (`data-model.md §3`)
4. Whether any function signature is pre-defined in `interface-contracts.md §5`
5. Which integration tests this code must pass (`testing-strategy.md §3`)

If any of these is unclear: **stop and ask. Do not proceed.**

---

## 1. Language Version and Toolchain

```
Go ≥ 1.22 (required — generics, rangefunc experiment, slices package)
golangci-lint — checked-in .golangci.yml, zero warnings on every PR
go vet — must pass; no exclusions
go test -race — mandatory; data races are correctness violations in this codebase
```

No `//nolint` directives without a comment explaining why, approved by two engineers.

---

## 2. Package Layout and Dependency Rules

Mirrors the repository structure in `mvp.md §Milestone 0` and `repo-structure.md §3.2`.

```
internal/
  crypto/      — key derivation, AONT, AEAD (pure functions, no shared state)
  erasure/     — Reed-Solomon encode/decode (pure functions)
  storage/     — WiscKey engine (ChunkStore interface)
  p2p/         — libp2p host, DHT, heartbeat
  audit/       — challenge generation, receipt writing, cluster secret
  scoring/     — three-window score queries, consecutive-pass counter
  repair/      — job queue, repair orchestration, departure detector
  payment/     — PaymentProvider interface, escrow ledger
  client/      — upload/retrieve orchestrators
cmd/
  microservice/
  provider/
  client/
migrations/    — SQL only; no application logic
```

### Full dependency order (from `repo-structure.md §3.2.10`)

The permitted import direction flows top-to-bottom. An arrow A→B means A may import B.

```
cmd/*
  → internal/client/*
    → internal/crypto
    → internal/erasure
    → internal/p2p
  → internal/audit
    → internal/crypto
    → internal/p2p
  → internal/scoring
    → (DB reads only — no package imports except stdlib)
  → internal/repair
    → internal/erasure
    → internal/p2p
    → internal/scoring (read-only)
  → internal/payment
    → (DB writes only — no package imports except stdlib)
  → internal/storage
    → (filesystem only — no package imports except stdlib)
```

**Hard prohibitions (build failure or code review rejection):**

| Package | Must NOT import |
|---------|----------------|
| `internal/crypto` | Any other `internal/` package |
| `internal/erasure` | Any other `internal/` package |
| `internal/storage` | `internal/payment`, `internal/scoring`, `internal/repair` |
| `internal/payment` | `internal/repair`, `internal/p2p`, `internal/audit` |
| `internal/scoring` | `internal/repair`, `internal/payment` |
| `internal/audit` | `internal/scoring`, `internal/repair`, `internal/payment` |

Circular imports are a build failure. Fix the abstraction; never fix the import.

### Compile-time interface assertions

Required for every concrete type implementing an exported interface. This catches a missing
method at compile time, not at runtime during an audit cycle.

```go
// At the bottom of the file implementing the interface:
var _ p2p.Host      = (*host)(nil)
var _ p2p.DHT       = (*dht)(nil)
var _ storage.ChunkStore = (*chunkStore)(nil)
```

---

## 3. The Five Invariants in Code

These come from `data-model.md §3`. Violating any one is a critical bug — not a style issue,
not a TODO. A PR that introduces any of these violations must not be merged.

### Invariant 1 — Append-only audit log

The two-phase write is *not* a database transaction. The phases are deliberately separate
calls to `db.ExecContext`. Wrapping them in `db.BeginTx` and `tx.Commit` would destroy
the crash-safety property: a transaction rolled back after Phase 1 loses the PENDING row,
and the provider's signed receipt is gone forever.

```go
// CORRECT — two separate DB calls, not a transaction:

// Phase 1: INSERT PENDING row. Must be durable before Phase 2 begins.
// If the process crashes here, crash recovery finds this PENDING row on restart.
receiptID, err := audit.WriteReceiptPhase1(ctx, db, fields)
if err != nil {
    return err
}

// (validation logic runs here — verify response_hash, check signature)

// Phase 2: UPDATE to terminal state. Only columns audit_result, service_sig,
// service_countersign_ts may be set. All other columns are immutable.
if err := audit.WriteReceiptPhase2(ctx, db, receiptID, result, sig, ts); err != nil {
    return err
}

// PROHIBITED — wrapping in a transaction destroys the crash-safety guarantee:
// tx, _ := db.BeginTx(ctx, nil)
// audit.WriteReceiptPhase1(ctx, tx, fields)  // Phase 1 is NOT durable until Commit
// audit.WriteReceiptPhase2(ctx, tx, ...)
// tx.Commit()  // A crash before this loses Phase 1 entirely
```

**Permitted writes:** INSERT (Phase 1), UPDATE of `audit_result/service_sig/service_countersign_ts`
(Phase 2), UPDATE of `abandoned_at` by the GC goroutine only.

**Prohibited:** Any other UPDATE column, any DELETE, ever, under any condition.

### Invariant 2 — Append-only escrow ledger

```go
// Balance is ALWAYS computed, never stored:
//   SELECT SUM(amount_paise) FILTER (WHERE event_type='DEPOSIT')
//        - SUM(amount_paise) FILTER (WHERE event_type IN ('RELEASE','SEIZURE'))
//   FROM escrow_events WHERE provider_id = $1
//
// Permitted: INSERT only.
// Prohibited: UPDATE on escrow_events. DELETE on escrow_events. A mutable balance column.
```

### Invariant 3 — No physical provider deletion

```go
// Terminal state: providers.status = 'DEPARTED'
// Hard-deleting a provider row orphans audit_receipts, escrow_events, and chunk_assignments.
// DELETE FROM providers — prohibited under any circumstance.
```

### Invariant 4 — Integer paise everywhere

```go
// CORRECT:
var amountPaise int64 = 40_000  // ₹400.00 = 40,000 paise

// PROHIBITED in internal/payment/ and any code path it touches:
var amount float64  // floating-point is a correctness violation in payment paths
```

The CI test `TestNoFloatArithmetic` uses Go AST analysis (not grep) to verify no `float64`
or `float32` type nodes exist in `internal/payment/`. This is enforced at the type system
level and cannot be evaded with type aliases.

### Invariant 5 — challenge_nonce is always 33 bytes

Use the named type to make this impossible to get wrong at the type system level:

```go
// Definition in internal/audit/types.go:
// ChallengeNonce is a 33-byte versioned challenge nonce.
// Byte 0: version byte (N mod 256, identifies which server_secret_vN was used)
// Bytes 1–32: HMAC-SHA256(server_secret_vN, chunk_id || server_ts_ms)
// Using a named type prevents accidental use of [32]byte where [33]byte is required.
type ChallengeNonce [33]byte

// CORRECT:
var nonce audit.ChallengeNonce

// PROHIBITED — silently breaks cross-replica validation on microservice failover:
var nonce [32]byte
```

---

## 4. Cryptographic Code Rules

Bugs here are silent, undetectable, and permanent. A data owner who uploads a file with
a corrupted AONT package loses that data forever — there is no recovery path.

### 4.1 Constant-time comparisons

Any comparison of authentication tags, signatures, or secret material must use constant-time
comparison. Variable-time comparison leaks a timing oracle that allows an attacker to brute-force
the correct value byte-by-byte.

```go
// CORRECT — for Poly1305 tag and all auth-tag comparisons:
import "crypto/subtle"
if subtle.ConstantTimeCompare(got, want) != 1 {
    return nil, ErrTagMismatch
}

// PROHIBITED — leaks timing oracle; enables auth-tag forgery attacks:
if bytes.Equal(got, want) { ... }
if got == want { ... }
```

### 4.2 AONT key K handling

K is the most sensitive value in the system — it can decrypt a segment if recovered.
K must be generated, used, and zeroed entirely within `AONTEncodeSegment`. It must never
appear in a log statement, error string, struct field, return value, or test assertion.

```go
k := make([]byte, 32)
if _, err := io.ReadFull(rand.Reader, k); err != nil {
    return nil, fmt.Errorf("crypto/rand: %w", err)  // fatal; propagate immediately
}
// Zeroing limitation: Go's garbage collector may scan k before the zero executes.
// This is a known Go limitation with no perfect solution below the runtime layer.
// The defer is still required — it prevents K from being readable in a heap dump
// after the function returns.
defer func() {
    for i := range k {
        k[i] = 0
    }
    runtime.KeepAlive(k)  // prevents the compiler from eliminating the zero as dead store
}()

// PROHIBITED anywhere in the codebase:
slog.Debug("AONT key", "key", hex.EncodeToString(k))
os.WriteFile("debug.key", k, 0o600)
return k, nil  // K must never leave this function
```

### 4.3 Nonce discipline for pointer files

**Why this matters:** AEAD_CHACHA20_POLY1305 uses a nonce-based stream cipher internally.
If the same (key, nonce) pair is used twice, an attacker who has both ciphertexts can XOR
them to cancel the keystream, decrypting both plaintexts. The nonce must be unique for every
encryption under the same key. The counter must be persisted *before* the encryption call —
if the process crashes between encrypt and persist, the next run re-derives the same nonce
from the persisted counter, which would cause nonce reuse.

```go
// CORRECT — increment and persist BEFORE the AEAD call:
nonce, err := keystore.IncrementAndGetNonce(fileID)  // writes new counter to disk first
if err != nil {
    return err
}
ciphertext, err := crypto.EncryptPointerFile(key, nonce, aad, plaintext)
if err != nil {
    return err
}
// Now store ciphertext. If we crash here, the nonce is already consumed — safe.

// PROHIBITED — nonce reuse on crash:
ciphertext, err := crypto.EncryptPointerFile(key, currentNonce, aad, plaintext)
keystore.PersistNonce(currentNonce + 1)  // crash here = nonce reuse next run
```

### 4.4 Error propagation from crypto primitives

All errors from `internal/crypto` indicate either a pre-condition violation (programming
error) or a hardware failure (`crypto/rand` exhausted). Both are fatal.

```go
// CORRECT:
encoded, err := crypto.AONTEncodeSegment(segment, aesNI)
if err != nil {
    return fmt.Errorf("AONT encode: %w", err)  // propagate; caller must halt
}

// PROHIBITED — swallowing a crypto error means corrupted data that passes audits:
encoded, _ := crypto.AONTEncodeSegment(segment, aesNI)
```

### 4.5 K-zeroing after AONT decode

After decoding, the recovered K must be zeroed before the function returns, regardless
of whether the canary passed. Never return plaintext if the canary failed.

```go
// After recovering K in AONTDecodePackage:
defer func() {
    for i := range k { k[i] = 0 }
    runtime.KeepAlive(k)
}()

// If canary fails: zero the plaintext buffer before returning the error.
if !bytes.Equal(canaryDecoded, aontCanary[:]) {
    for i := range plaintext { plaintext[i] = 0 }
    return nil, ErrCanaryMismatch
    // NEVER return corrupted plaintext to the caller, even partially.
}
```

---

## 5. Storage Engine Rules (`internal/storage`)

### 5.1 Single writer goroutine — enforced by design

The vLog is an append-only file. POSIX `O_APPEND` atomicity does not hold for writes
above ~4 KB; for 262 KB vLog entries, concurrent appends without serialisation produce
overlapping and corrupt entries.

```go
// The writer goroutine owns the vLog file handle.
// All other goroutines submit write requests via a buffered channel:
type writeRequest struct {
    chunkID  [32]byte
    data     []byte
    resultCh chan<- writeResult
}

// AppendChunk is called ONLY from the single writer goroutine.
// Calling it from any other goroutine is a pre-condition violation.
// In debug builds: panic. In release builds: return ErrWriterViolation.
```

### 5.2 fsync ordering

The fsync must complete before the RocksDB index insert. If the process crashes after
the index insert but before the fsync, the index points to unwritten data. The reverse
— crash after fsync but before index insert — is recoverable by the crash recovery
tail-scan on next startup.

```go
// CORRECT ordering:
if err := vlog.writeEntry(entry); err != nil { return 0, err }
if err := vlog.file.Sync(); err != nil { return 0, ErrVLogFsync }  // halt on failure
if err := db.Put(chunkID[:], offsetBytes); err != nil { return 0, ErrRocksDBInsert }
// If we crash here: crash recovery re-inserts the missing index entry on next start.
```

### 5.3 Content hash on every read — no exceptions

```go
entry, err := vlog.readAt(offset)
if err != nil { return nil, ErrVLogRead }

computed := sha256.Sum256(entry.ChunkData)
if computed != entry.ContentHash {
    // Disk corruption. The provider must return audit FAIL with status 0x02.
    // Do not attempt repair here — the microservice will queue accelerated re-audit.
    return nil, ErrContentHashMismatch
}
```

---

## 6. Concurrency Patterns

### 6.1 Context propagation

Every function that touches a database, network, or file system takes `context.Context`
as its first argument. No goroutine does I/O without a context that can be cancelled.

```go
// CORRECT:
func (s *Store) LookupChunk(ctx context.Context, chunkID [32]byte) ([]byte, error)

// PROHIBITED:
func (s *Store) LookupChunk(chunkID [32]byte) ([]byte, error)
```

### 6.2 Context deadlines for the audit RTO

Per-provider deadlines use `context.WithTimeout`. The timeout is computed from the
per-provider RTO stored in `providers.avg_rtt_ms` and `providers.var_rtt_ms`.

```go
// CORRECT — per-provider deadline from RTO formula (ADR-006):
rtoMs := providerAvgRTT + 4*providerVarRTT
ctx, cancel := context.WithTimeout(parentCtx, time.Duration(rtoMs)*time.Millisecond)
defer cancel()
response, err := p2pHost.SendChallenge(ctx, peerID, challenge)
// If ctx.Err() == context.DeadlineExceeded: record audit_result = TIMEOUT
```

### 6.3 Goroutine ownership

Every goroutine must have a named owner and a defined exit condition. Goroutines that
outlive their owner are resource leaks that accumulate silently.

```go
// CORRECT:
func (m *AuditScheduler) runChallengeDispatcher(ctx context.Context, wg *sync.WaitGroup) {
    defer wg.Done()
    for {
        select {
        case <-ctx.Done():
            return
        case job := <-m.challengeQueue:
            m.dispatchChallenge(ctx, job)
        }
    }
}
```

### 6.4 Buffered channels for write requests

```go
const writerBufferSize = 32  // documented in ADR-023 §Single writer goroutine

writerCh := make(chan writeRequest, writerBufferSize)
```

---

## 7. Database Interaction Rules

### 7.1 Table ownership

Each table has one owning package. SQL touching that table lives only in that package.
Cross-package DB access violates the ownership model and makes invariant enforcement impossible.

| Table | Owning package |
|---|---|
| `audit_receipts` | `internal/audit` |
| `escrow_events` | `internal/payment` |
| `providers` (writes) | `internal/scoring` (status, counters), `internal/repair` (departure), `internal/audit` (heartbeat update) |
| `providers` (reads) | Any package via materialised view |
| `chunk_assignments` | `internal/repair` |
| `repair_jobs` | `internal/repair` |
| `audit_periods` | `internal/payment` |

### 7.2 Parameterised queries only

```go
// CORRECT:
row := db.QueryRowContext(ctx,
    `SELECT vlog_offset FROM chunk_index WHERE chunk_id = $1`, chunkID[:])

// PROHIBITED — SQL injection and correctness risk:
query := fmt.Sprintf("SELECT ... WHERE chunk_id = '%x'", chunkID)
```

### 7.3 Two-phase writes are NOT transactions

The crash-safe audit receipt write uses two *separate, non-transactional* database calls.
This is deliberate — Phase 1 must be independently durable before Phase 2 begins.

```go
// CORRECT — two separate calls, no transaction wrapper:

// Phase 1: INSERT PENDING row (audit_result = NULL, provider_sig populated)
// This is durable on disk before Phase 2 begins.
receiptID, err := audit.WriteReceiptPhase1(ctx, db, fields)
if err != nil {
    return err
}

// (validate response_hash and provider_sig here)

// Phase 2: UPDATE to terminal state.
// Only audit_result, service_sig, service_countersign_ts are written.
if err := audit.WriteReceiptPhase2(ctx, db, receiptID, result, sig, ts); err != nil {
    if errors.Is(err, audit.ErrReceiptAlreadyFinal) {
        // Idempotent: provider retried with same nonce. Return existing receipt.
        return existingServiceSig, nil
    }
    return err
}

// PROHIBITED — single transaction destroys crash safety:
// tx, _ := db.BeginTx(ctx, nil)
// defer tx.Rollback()
// audit.WriteReceiptPhase1(ctx, tx, fields)  // NOT durable until tx.Commit
// audit.WriteReceiptPhase2(ctx, tx, ...)
// tx.Commit()  // crash before this = Phase 1 lost forever
```

### 7.4 Transactions for other multi-step writes

For operations that are NOT the two-phase audit receipt write, standard transactions are
correct and should be used when atomicity is required (e.g., departure: seize escrow +
update provider status + enqueue repair jobs as one atomic operation).

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil { return err }
defer tx.Rollback()  // no-op if already committed

if err := setProviderDeparted(ctx, tx, providerID); err != nil { return err }
if err := payment.InsertSeizure(ctx, tx, providerID, amount, key); err != nil { return err }
if err := repair.EnqueueJobs(ctx, tx, affectedChunks); err != nil { return err }

return tx.Commit()
```

### 7.5 Integer paise in all DB interactions

```go
// CORRECT — bind int64:
_, err = db.ExecContext(ctx,
    `INSERT INTO escrow_events (amount_paise) VALUES ($1)`, amountPaise)

// PROHIBITED — float in any position in the payment path:
_, err = db.ExecContext(ctx,
    `INSERT INTO escrow_events (amount_paise) VALUES ($1)`, float64(amountPaise))
```

---

## 8. Error Handling

### 8.1 Sentinel errors for public APIs

All exported errors from `internal/` packages are typed sentinels, never format strings.
Callers use `errors.Is()` for matching.

```go
// In errors.go for the package:
var (
    ErrChunkNotFound        = errors.New("storage: chunk not found")
    ErrContentHashMismatch  = errors.New("storage: content hash mismatch (disk corruption)")
    ErrVLogFsync            = errors.New("storage: vLog fsync failed")
    ErrVLogRead             = errors.New("storage: vLog read failed")
    ErrRocksDBInsert        = errors.New("storage: RocksDB index insert failed")
)

// Callers:
if errors.Is(err, storage.ErrContentHashMismatch) {
    return auditFAILWithCorruptionCode(challengeNonce)
}
```

### 8.2 Error classification

| Class | Example | Response |
|---|---|---|
| Pre-condition violation | `len(nonce) != 33` | `panic` in debug; sentinel in release |
| Crypto failure | `crypto/rand` exhausted | Propagate immediately; calling goroutine halts |
| Disk corruption | `ErrContentHashMismatch` | Return audit FAIL status 0x02; do not retry |
| Network transient | libp2p dial timeout | Retry with exponential backoff; max 3 attempts |
| DB idempotency hit | duplicate `idempotency_key` | Return `ErrDuplicateIdempotencyKey`; treat as success |
| Fatal infrastructure | vLog fsync failure | `slog.Error`; goroutine halts; daemon restarts on crash |

### 8.3 Never swallow errors

```go
// PROHIBITED:
result, _ := someImportantFunction()

// PROHIBITED even in tests (use t.Fatal or t.Fatalf):
if err != nil { t.Log(err) }
```

---

## 9. Logging

Use `log/slog` (stdlib, Go 1.21+). No `fmt.Println` in production paths.

```go
// CORRECT — structured, no sensitive material:
slog.Info("audit challenge dispatched",
    "provider_id", providerID,
    "chunk_id", hex.EncodeToString(chunkID[:]),
    "rto_ms", rto.Milliseconds(),
)

// PROHIBITED — unstructured:
log.Printf("dispatched to %s", providerID)

// PROHIBITED — sensitive material:
slog.Debug("AONT key", "key", hex.EncodeToString(k))
slog.Debug("master secret", "secret", hex.EncodeToString(ms))
slog.Debug("pointer file key", "key", hex.EncodeToString(pfk))
```

**Log levels:**
- `DEBUG` — development only; gated by build tag or env var; absent from release binary
- `INFO` — normal operational events (challenge dispatched, heartbeat received, shard stored)
- `WARN` — unexpected but recoverable (heartbeat late, multiaddr stale, jit_flag set)
- `ERROR` — failure requiring operator attention (fsync failure, replica count drop, seizure triggered)

---

## 10. Testing Standards

Full detail in `testing-strategy.md`. Summary of mandatory rules:

Every package in `internal/` must have:
- Table-driven unit tests covering all exported functions
- `t.Parallel()` on every test function (exception must be justified in a comment)
- At least one fuzz test for any function handling untrusted input
- `go test -race` passing with zero data races
- Benchmark functions for every code path with an NFR latency target

**Integration test files require a build tag:**
```go
//go:build integration
// +build integration

package audit_test
```
Without this tag, integration tests run in every `go test ./...` — including on machines
without Postgres, causing false CI failures.

**Table-driven test skeleton:**
```go
func TestAONTRoundTrip(t *testing.T) {
    t.Parallel()
    cases := []struct {
        name    string
        size    int
        wantErr bool
    }{
        {"minimum segment (4 MB)", 4 * 1024 * 1024, false},
        {"full segment (14 MB)", 14 * 1024 * 1024, false},
        {"empty input", 0, true},
        {"one byte below minimum", 4*1024*1024 - 1, true},
    }
    for _, tc := range cases {
        tc := tc
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            // ...
        })
    }
}
```

---

## 11. Performance-Sensitive Paths

These paths have NFR latency targets. Every change to them must be accompanied by a
benchmark run on the minimum-spec machine (dual-core, no AES-NI, 2 GB RAM, 7200 RPM HDD).

| Path | Target | NFR | Benchmark name |
|---|---|---|---|
| AONT encode 14 MB | p50 ≤ 200 ms | NFR-009 | `BenchmarkAONTEncode14MB/ChaCha20` |
| Argon2id session start | p50 ≤ 500 ms | NFR-010 | `BenchmarkArgon2id` |
| Audit lookup (SSD) | p99 ≤ 100 ms | NFR-008 | `BenchmarkAuditLookup/SSD` |
| Audit lookup (HDD) | p99 ≤ 200 ms | NFR-008 | `BenchmarkAuditLookup/HDD` |
| vLog append throughput | report MB/s | ADR-023 | `BenchmarkVLogAppend` |

Record hardware spec alongside benchmark output. A benchmark passing on a developer MacBook
but failing on the minimum-spec machine is a failing benchmark.

---

## 12. Code Generation Heuristics for Build Sessions

**Scope per session = one package's files, driven by `mvp.md` milestone scope.**

**File count per session:**
- M0–M2 (foundational packages): 1–3 files, full content
- M3–M5 (network, scoring, repair): 1–2 files; complex files get scaffold + BUILD-TODO markers for next session
- M6–M8 (payment, client, deployment): 1 file, driven by `interface-contracts.md`

**BUILD-TODO markers:**
```go
// BUILD-TODO(M3-S2): implement relay reservation refresh loop
// Requires: ADR-021 §Circuit Relay v2, interface-contracts.md §4.4
func (h *host) maintainRelayReservation(ctx context.Context) error {
    panic("not implemented — BUILD-TODO(M3-S2)")
}
```

`panic` ensures the test suite fails loudly on any call to an incomplete function.
Never use `return nil` as a stub — it would silently pass where a correct implementation
should fail.

---

## 13. Prohibited Patterns

| Pattern | Reason | Enforced by |
|---|---|---|
| `float64`/`float32` in `internal/payment/` | Invariant 4 | `TestNoFloatArithmetic` (AST analysis) |
| `bytes.Equal` for auth-tag comparison | Timing oracle | Code review + `go vet` rule |
| `DELETE FROM audit_receipts` | Invariant 1 | Row security policy + code review |
| `DELETE FROM escrow_events` | Invariant 2 | Row security policy + code review |
| `UPDATE escrow_events` | Invariant 2 | Row security policy + code review |
| `DELETE FROM providers` | Invariant 3 | Row security policy + code review |
| Two-phase write in a single transaction | Destroys crash safety (see §7.3) | Code review + `TestReceiptPhase1Durability` |
| AONT key K in log/error/struct/return | Security critical | Code review |
| Nonce persist after AEAD call | Nonce reuse on crash | Code review + `TestPointerFileNonceReuse` |
| `[32]byte` for challenge_nonce | Invariant 5 | `type ChallengeNonce [33]byte` enforces at type level |
| `_ = someErr` | Silent failure | `golangci-lint errcheck` |
| Raw SQL string concatenation | SQL injection | `golangci-lint sqlclosecheck` |
| Goroutine without context | Resource leak | Code review |
| Missing `//go:build integration` on integration tests | Tests run without Postgres | `golangci-lint` + CI |
| UPI Collect API calls | Deprecated by NPCI 28 Feb 2026 | Code review; blocked by NFR-029 |
| `BYTEA(32)` for `challenge_nonce` in any migration | Invariant 5 | CI grep check |

---

## 14. Dependency Pinning

Pin before the milestone that introduces the dependency. Document in `go.mod` and `go.sum`.
Vendor with `go mod vendor` if CGo is involved (RocksDB).

| Dependency | Pin before | Notes |
|---|---|---|
| `golang.org/x/crypto` | M1 | ChaCha20, Poly1305, HKDF, Argon2id |
| `klauspost/reedsolomon` | M1 | Vendor; verify systematic form support in test |
| `linxGnu/grocksdb` | M2 | CGo; vendor; test on Windows 10+, macOS 12+, Ubuntu 22.04+ |
| `libp2p/go-libp2p` | M3 | Pin exact tag; `TestDHTKeyValidatorPersists` must pass after every upgrade |
| `prometheus/client_golang` | M4 | Pin minor version |
| Razorpay Go SDK | M6 | Pin patch version; abstract behind `PaymentProvider` interface |

---

*This document is authoritative for all code generated in build sessions.
Update it via PR when an ADR changes a code-level decision.
Version history is tracked in the repository commit log.*