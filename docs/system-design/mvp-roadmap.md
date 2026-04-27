# Vyomanaut V2 — MVP Roadmap

**Status:** Build-ready  
**Date:** April 2026  
**Audience:** Every engineer who writes the first line of production code  
**Architecture reference:** `docs/system-design/architecture.md`  
**Requirements reference:** `docs/system-design/requirements.md`  
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research

---

## What the MVP Is

The MVP is the smallest version of the system that:

1. Exercises every critical path end-to-end under real conditions
2. Is safe to run with real users (not a prototype — correctness first)
3. Is honest about what it does not yet do

A critical path is any sequence of steps where a bug anywhere in the chain produces a
wrong outcome that the user cannot detect and cannot recover from. Vyomanaut has exactly
four critical paths:

| Path | Failure mode if broken |
|---|---|
| Encode → upload → store | Corrupted data that passes audits and is unrecoverable |
| Challenge → respond → score | Providers earning without holding data |
| Departure → repair | Files silently falling below the reconstruction floor |
| Audit pass → escrow release | Providers working for free, or paid twice |

The MVP must exercise all four. Nothing is "nice to have" if it sits on one of these paths.

---

## What the MVP Is Not

These are deliberately excluded from the MVP scope. They are recorded here so no one
builds them accidentally before the critical paths work.

| Excluded | Lives in | Why |
|---|---|---|
| Provider local dashboard (tray app / web UI) | V2 post-MVP | CLI-only is fine for private beta |
| Hot Storage Band | V3 | Separate erasure parameters, separate payment model |
| Transparent Merkle Log | V3 | Signed receipts are sufficient for V2 trust |
| Hitchhiker repair bandwidth optimisation | V3 | BWavg is well under budget at launch scale |
| Mobile providers | V3 | BWavg exceeds 100 Kbps budget at mobile MTTF |
| International payments (Stripe Connect) | Post-India-launch | PaymentProvider interface is ready; implementation is not |
| File versioning | Never in V2 | Files are immutable by design |
| Convergent encryption / deduplication | Never | Privacy is the product |
| Geographic proximity routing (Coral DSHT) | V3 | Inter-city latency is homogeneous in India |

---

## Reading the Milestones

Each milestone is a **sprint epic**. A sprint epic ends when:

1. Every component in scope passes its **exit criterion** — a concrete test you can run
2. The system is **runnable** — not just compilable, but executable in a state a reviewer
   can observe
3. Nothing from a previous milestone has regressed

The milestones are strictly sequential. Milestone N depends on Milestone N-1 being correct.
Parallelising them is possible within a team only when the dependency graph allows it —
the dependency notes in each milestone make this explicit.

---

## Milestone 0 — Foundation

**Goal:** A monorepo with tooling, CI, and a 56-node simulated provider network running
on a single developer laptop. No business logic yet. This milestone exists to establish
the substrate everything else builds on.

### Scope

**Repository layout:**
```
/
  cmd/
    microservice/   ← coordination microservice entrypoint
    provider/       ← provider daemon entrypoint (supports --sim-count flag)
    client/         ← data owner CLI entrypoint
  internal/
    erasure/        ← (empty, filled in M1)
    crypto/         ← (empty, filled in M1)
    storage/        ← (empty, filled in M2)
    p2p/            ← (empty, filled in M3)
    audit/          ← (empty, filled in M4)
    scoring/        ← (empty, filled in M5)
    repair/         ← (empty, filled in M5)
    payment/        ← (empty, filled in M6)
  migrations/       ← SQL schema migrations (Postgres)
  docs/             ← existing research and system-design (read-only during build)
  scripts/          ← dev tooling (lint, test, sim)
```

**Postgres schema — all tables, created now, populated later:**
Create every table defined in the ADRs in a single `migrations/001_initial_schema.sql`.
Tables that cannot be filled yet get their rows inserted by later milestones. The schema
is the contract — get it right once, here.

Key constraints to enforce from day one:
- `audit_receipts`: row security policy blocking UPDATE and DELETE (FR-039, NFR-021)
- `escrow_events`: row security policy blocking UPDATE and DELETE (NFR-022)
- `audit_receipts.challenge_nonce`: BYTEA(33) not BYTEA(32) — includes 1-byte version
  prefix per ADR-027 (cross-document correction noted in requirements.md §9.3)
- `audit_receipts.audit_result`: accepts NULL (in-flight PENDING state, ADR-015)
- `audit_receipts.challenge_nonce`: UNIQUE index for idempotent retry
- `providers.last_known_multiaddrs`: JSONB column (ADR-028)

**Provider daemon simulation mode (FR-055, FR-056):**  
The `--sim-count=N` flag launches N independent provider instances in one process. Each
gets its own:
- Ed25519 key pair (generated fresh, persisted under `/tmp/vyomanaut-sim/{id}/keys/`)
- RocksDB instance at `/tmp/vyomanaut-sim/{id}/db/`
- vLog file at `/tmp/vyomanaut-sim/{id}/vlog/chunks.vlog`
- libp2p listen address at `127.0.0.1:{base_port + i}`
- Synthetic ASN: `SIM-AS{(i % sim_asn_count) + 1}` (default `--sim-asn-count=5`)
- Synthetic region: `["Delhi", "Mumbai", "Bangalore"][i % 3]`

Simulation mode must not bypass the network readiness gate. The simulation with
`--sim-count=56 --sim-asn-count=5` must satisfy all seven readiness conditions
(FR-056) before any upload can proceed. The gate will trivially fail in M0 (most
conditions not yet implemented) — that is correct.

**CI pipeline:**
- `go build ./...` must pass with zero warnings (strict mode from day one)
- `go vet ./...` must pass
- `go test ./...` must pass (no tests yet — the harness is wired, tests come with each milestone)
- `golangci-lint run` with a checked-in `.golangci.yml` config
- Postgres migration applied and rolled back cleanly against a CI Postgres instance
- All CI checks run on every PR; nothing merges red

### Exit Criterion

```bash
# This command runs and produces structured output (even if all gates show FAIL)
go run ./cmd/provider --sim-count=56 --sim-asn-count=5 \
  --microservice-url=http://localhost:8080 \
  --sim-data-dir=/tmp/vyomanaut-sim

# This shows the readiness gate status (all FAIL is expected in M0)
curl http://localhost:8080/api/v1/admin/readiness | jq .

# CI passes green
```

---

## Milestone 1 — Encoding Pipeline

**Goal:** A correct, benchmarked, and tested encoding pipeline that transforms plaintext
into 56 encrypted RS fragments and back. This is the zero-knowledge property in code.

**Dependency:** M0 (repository layout, Go toolchain)  
**Parallelisable with:** Nothing — every downstream milestone depends on this being correct.

### Scope

**Package: `internal/crypto`**

Key hierarchy (ADR-019, ADR-020):
- `Argon2id(passphrase, owner_id, t=3, m=65536, p=4) → [32]byte` master secret
- `HKDF-SHA256(master_secret, salt=owner_id, info="vyomanaut-file-v1"+file_id) → file_key`
- `HKDF-SHA256(master_secret, salt=owner_id, info="vyomanaut-pointer-v1"+file_id) → pointer_enc_key`
- `HKDF-SHA256(master_secret, salt=owner_id, info="vyomanaut-keystore-v1") → keystore_enc_key`
- AES-NI detection at package init via CPUID (x86) — sets a package-level constant

ChaCha20 AONT pass (no AES-NI path, ADR-019):
```
K = SecureRandom(256 bits)
for i, word in enumerate(16-byte words of segment + canary):
    c_i = word XOR ChaCha20(key=K, counter=i/4, nonce=zeros)[word_offset]
h = SHA-256(c_0 || ... || c_s)
c_{s+1} = K XOR h[:32]
```

AES-256-CTR AONT pass (AES-NI path, ADR-019):
```
K = SecureRandom(256 bits)
for i, word in enumerate(words):
    c_i = word XOR AES-256-ECB(K, i+1)    // counter mode
h = SHA-256(c_0 || ... || c_s)
c_{s+1} = K XOR h[:32]
```

Canary: 16-byte fixed value appended before AONT. Verified during decode.

Pointer file schema and AEAD encryption (AEAD_CHACHA20_POLY1305):
- Plaintext schema exactly as defined in ADR-022 and architecture.md §10
- 96-bit nonce: counter from keystore, never random
- AAD: `owner_id || file_id || schema_version`
- Ed25519 signature by the data owner over all fields

**Package: `internal/erasure`**

Reed-Solomon RS(s=16, r=40):
- Use a production-quality GF(2^8) Reed-Solomon library (e.g., `klauspost/reedsolomon`)
- Systematic form: first 16 output shards are identity-mapped from the AONT package
- Shards are exactly lf=256 KB (262,144 bytes) each
- Segment padding to fill the minimum segment size (fr-008: files smaller than 4 MB
  padded; original size recorded in pointer file for stripping on retrieval)

The full pipeline in order (ADR-022):
```
plaintext → [pad if < 4 MB] → [segment at 14 MB boundaries]
         → [AONT transform] → [RS(16,56) dispersal]
         → [56 shards of 256 KB]
```

The full decode pipeline:
```
[any 16 shards] → [RS decode → AONT package]
               → [recover K: h=SHA256(all codewords), K=c_{s+1} XOR h]
               → [decrypt each word] → [verify canary] → [strip padding] → plaintext
```

### Tests

Every test here uses table-driven subtests. Fuzzing is strongly encouraged for the AONT
round-trip.

```
TestAONTRoundTrip/ChaCha20     — encode then decode, canary passes, plaintext identical
TestAONTRoundTrip/AES          — same with AES path
TestAONTKeyRecovery            — K must not be computable with < s+1 codewords
TestAONTCorruptionDetection    — flip any bit in any codeword, canary fails
TestRSRoundTrip                — encode then decode with exactly 16 shards
TestRSAnyKShards               — 100 random subsets of size 16 each reconstruct correctly
TestRSShardSize                — each output shard is exactly 262,144 bytes
TestFullPipeline               — plaintext → 56 shards → reconstruct from any 16 → plaintext
TestPaddingRoundTrip           — file smaller than 4 MB pads and strips correctly
TestKeyHierarchy               — HKDF derivation is deterministic given same inputs
TestArgon2idDeterminism        — same passphrase + owner_id always produces same master_secret
TestPointerFileEncryption      — encrypt then decrypt, AAD verified, tampered ciphertext fails
```

### Benchmarks (must pass before milestone closes)

Run on minimum-spec hardware (dual-core, no AES-NI, 2 GB RAM, HDD) per
`docs/research/benchmarking-protocol.md` Q16-1:

```
BenchmarkAONTEncode14MB/ChaCha20  — target: p50 ≤ 200 ms, p99 ≤ 400 ms
BenchmarkArgon2id                 — target: p50 ≤ 500 ms at t=3, m=64 MB (Q18-1)
```

Record hardware spec alongside benchmark output. If Argon2id fails on minimum-spec,
apply the fallback protocol in `docs/research/benchmarking-protocol.md` Q18-1 and
update ADR-020 with the confirmed parameters.

### Exit Criterion

```bash
go test ./internal/crypto/... ./internal/erasure/... -v -race
# All tests pass, zero data races

go test ./internal/crypto/... -bench=. -benchtime=100x
# Benchmark output recorded, p50 targets met on minimum-spec machine
```

---

## Milestone 2 — Provider Storage Engine

**Goal:** A provider daemon that can store and retrieve 256 KB chunks from local disk
correctly and efficiently, with corruption detection, write-safety, and crash recovery.

**Dependency:** M0 (repository layout)  
**Parallelisable with:** M1 (independent code paths)

### Scope

**Package: `internal/storage`**

WiscKey key-value separation (ADR-023):

*The chunk index (RocksDB):*
- Key: `chunk_id` (32 bytes, SHA-256 of chunk content)
- Value: `vlog_offset` (uint64) + `chunk_size` (uint32) = 12 bytes; total entry 44 bytes
- Bloom filter: 10 bits per key (~1% false-positive rate)
- RocksDB options: `max_background_compactions=1`, `max_background_flushes=1`,
  `rate_limiter=10 MB/s` starting value (to be calibrated by Q27-1 benchmark)
- SSD vs HDD detection at daemon startup via `/sys/block/{dev}/queue/rotational` (Linux)
  or platform equivalent; sets rate_limiter from two separate constants

*The value log (vLog):*
- Single append-only file, fixed-size entries of exactly 262,212 bytes:
  `chunk_id(32) + chunk_size(4) + chunk_data(262,144) + content_hash(32)`
- `content_hash = SHA-256(chunk_data)` — verified on every read
- `fsync()` the vLog entry before the RocksDB insert (write ordering)
- Single writer goroutine with a buffered channel (buffer size: 32); all other goroutines
  submit write requests and block on a result channel (ADR-023, concurrent write safety)

*Crash recovery:*
- On startup, read `last_vlog_head_offset` from RocksDB
- Scan vLog from that offset to EOF
- For each complete entry, verify content_hash; if valid and chunk_id absent from
  RocksDB, re-insert the index entry
- Incomplete (truncated) tail entries are discarded

*Garbage collection:*
- Triggered only by explicit chunk deletion events (announced departure, file owner delete,
  repair reassignment)
- Read vLog from tail, check each chunk_id against RocksDB; absent = invalid
- Append valid entries to vLog head; update `last_vlog_tail_offset` in RocksDB
- `fallocate(FALLOC_FL_PUNCH_HOLE)` to release freed regions

*Audit lookup path (must complete within NFR-008 thresholds):*
```
receive (chunk_id, challenge_nonce)
  → Bloom filter check: absent → FAIL immediately (no disk I/O)
  → RocksDB get chunk_id → (vlog_offset, chunk_size) (block cache hit = no disk I/O)
  → read 262,212 bytes from vLog at vlog_offset (one random read)
  → verify SHA-256(chunk_data) == content_hash → FAIL with corruption code if mismatch
  → response_hash = SHA-256(chunk_data || challenge_nonce)
  → return response_hash
```

### Tests

```
TestStoreAndRetrieve           — store a chunk, retrieve by chunk_id, data identical
TestBloomFilterNegative        — lookup of un-stored chunk_id never touches disk
TestContentHashVerification    — corrupt one byte in vLog entry, lookup returns corruption FAIL
TestSingleWriterGoroutine      — 100 concurrent goroutines store chunks; no vLog overlap
TestCrashRecovery              — write 100 chunks, kill without RocksDB flush, restart, all 100 retrievable
TestGarbageCollection          — delete 50 of 100 chunks, GC runs, deleted chunks absent, rest intact
TestRateLimiter                — concurrent reads and compaction, p99 read latency stays within threshold
```

### Benchmarks (must pass before milestone closes)

Q27-1 from `docs/research/benchmarking-protocol.md`:

```
BenchmarkAuditLookup/SSD      — target: p99 ≤ 100 ms under concurrent writes
BenchmarkAuditLookup/HDD      — target: p99 ≤ 200 ms under concurrent compaction
BenchmarkVLogAppend            — report throughput in MB/s; must match declared upload speed
```

### Exit Criterion

```bash
go test ./internal/storage/... -v -race
# All tests pass, zero data races

# Demonstrate crash recovery visually:
scripts/test-crash-recovery.sh
# Stores 1000 chunks, kills process mid-write, restarts, verifies all 1000 retrievable
```

---

## Milestone 3 — P2P Networking

**Goal:** Provider instances can discover each other, establish authenticated libp2p
connections with NAT traversal, maintain the Kademlia DHT with HMAC-pseudonymised chunk
keys, and send 4-hourly signed heartbeats to the microservice.

**Dependency:** M0 (repository layout)  
**Parallelisable with:** M1, M2

### Scope

**Package: `internal/p2p`**

*Peer identity (ADR-021):*
- Generate Ed25519 key pair at first daemon start; persist encrypted under keystore_enc_key
- Derive libp2p Peer ID as `multihash(public_key)` (standard libp2p convention)
- Register public key with the microservice at registration

*Transport stack (ADR-021):*
- Primary: QUIC v1 (RFC 9000) via go-libp2p's QUIC transport
- Fallback: TCP + Noise XX handshake + yamux multiplexer
- Automatic transport selection via libp2p multiaddr dialling (provider does not configure this)
- 0-RTT disabled for all connections where audit receipts or signed receipts flow
  (flag set on connection-level context; pure chunk data transfer connections may allow 0-RTT)

*NAT traversal — three tiers (ADR-021):*
```
AutoNAT → classify provider as: public / cone NAT / symmetric NAT
        → cone NAT: attempt DCUtR hole-punch (max_hole_punch_retries = 1)
        → symmetric NAT: fall back to Circuit Relay v2
```
Relay nodes are hardcoded in the daemon configuration at launch (`--relay-addrs`).
Relay addresses are the three Vyomanaut-operated relay nodes.

*DHT configuration (ADR-001):*
- k-bucket size: k=16 (configured at Kademlia init, never changed after network launches)
- Parallel lookups: alpha=3
- DHT mode: server (all V2 desktop providers)
- Custom key validator: accepts only keys of the form
  `HMAC-SHA256(chunk_hash, file_owner_key)` — rejects plain CIDs
- Republication: disabled internally; driven by the availability microservice

*Heartbeat protocol (ADR-028):*
- Every 4 hours, the daemon POST-s a signed heartbeat to
  `/api/v1/provider/heartbeat` containing:
  - `provider_id`, `current_multiaddrs` (JSON array), `timestamp`, `daemon_version`
  - `provider_sig`: Ed25519 over all fields
- Microservice updates `providers.last_known_multiaddrs` and `providers.last_heartbeat_ts`
- The 4-hour timer resets after each successful heartbeat; missed heartbeats do not
  accumulate — the daemon tries again at the next tick

### Tests

```
TestPeerIdentityPersistence    — key pair survives daemon restart; same Peer ID
TestLibp2pConnection           — two instances on same machine establish QUIC connection
TestNoiseHandshakeFallback     — with QUIC blocked (iptables DROP udp), falls back to TCP+Noise
TestDHTKeyValidator            — plain CID rejected; HMAC-keyed lookup accepted
TestHeartbeatSigned            — heartbeat payload verifies against registered public key
TestHeartbeatInterval          — heartbeat fires within 4h ± 5 minutes
TestRelayFallback              — with AutoNAT classifying as symmetric NAT, relay path used
```

### Integration test (requires simulation mode from M0):

```bash
# Launch 56 simulated providers with sim mode
go run ./cmd/provider --sim-count=56 --sim-asn-count=5 \
  --microservice-url=http://localhost:8080 \
  --sim-data-dir=/tmp/vyomanaut-sim &

# Verify all 56 have sent at least one heartbeat and are visible in the provider registry
curl http://localhost:8080/api/v1/admin/providers | jq '. | length'
# Expected: 56
```

### Exit Criterion

```bash
go test ./internal/p2p/... -v -race
# All tests pass

# Two provider instances connect to each other and exchange a test message
scripts/test-p2p-connection.sh
```

---

## Milestone 4 — Audit System

**Goal:** The microservice issues audit challenges to providers, providers respond with
signed receipts, and the receipts are durably recorded in the INSERT-only audit log.
The cluster audit secret is correctly shared across microservice replicas. The 12-field
receipt schema is complete and correct.

**Dependency:** M0 (Postgres schema), M2 (storage engine — provider needs to read chunks),
M3 (P2P — microservice needs to reach providers)

### Scope

**Package: `internal/audit`**

*Cluster audit secret (ADR-027):*
- `server_secret_vN = HKDF-SHA256(cluster_master_seed, salt=cluster_id, info="vyomanaut-audit-secret-v"+N, len=32)`
- `cluster_master_seed` loaded from secrets manager (or `VYOMANAUT_CLUSTER_MASTER_SEED`
  env var in development and simulation mode — never in production)
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
- Per-provider RTO: `AVG + 4 × VAR` of recent audit response latencies
  (stored as `(avg_rtt_ms FLOAT, var_rtt_ms FLOAT, sample_count INT)` per provider)
- New providers: initialise RTO from pool median; switch to per-provider formula after 5 samples
- Schedule: exactly one challenge per assigned chunk per 24-hour window, with randomised
  jitter within the window (provider cannot anticipate timing)

*Receipt recording — crash-safe two-phase write (ADR-015):*
```
Phase 1: INSERT PENDING row (audit_result = NULL, provider_sig populated)
         — durable before Phase 2
Phase 2: Validate response_hash
         UPDATE row: SET audit_result = PASS|FAIL, service_sig, service_countersign_ts
Phase 3: Return countersignature to provider
```
If microservice crashes between Phase 1 and Phase 2: orphaned PENDING row is
garbage-collected after 48 hours (set `abandoned_at`, exclude from scoring). Provider
receives no countersignature and retries at next challenge cycle. Provider-side retry
within 48 hours: microservice detects duplicate `challenge_nonce` (UNIQUE index)
and returns the existing receipt. (ADR-015)

*Provider-side audit response (ADR-002, ADR-014):*
```
receive (chunk_id, challenge_nonce)
read chunk from storage engine (M2)  ← content_hash verified here
compute response_hash = SHA-256(chunk_data || challenge_nonce)
build receipt struct (all 12 fields)
sign with Ed25519 private key
send to microservice
```

*JIT flag (ADR-014 Defence 3):*
After recording a receipt, the microservice evaluates:
- If `response_latency_ms < (chunk_size / p95_throughput_kbps) × 0.3` → `jit_flag = true`
- 3+ jit_flags from same provider in 7-day window → apply 0.5× weight to audit passes
  in the 24h scoring window for 30 days

*Audit receipt schema — complete 12-field form (ADR-017):*

```sql
audit_receipts (
  receipt_id              UUIDv7       PRIMARY KEY,
  schema_version          SMALLINT     NOT NULL DEFAULT 1,
  chunk_id                BYTEA(32)    NOT NULL,
  file_id                 UUID         NOT NULL,
  provider_id             UUID         NOT NULL,
  challenge_nonce         BYTEA(33)    NOT NULL,  -- 1 version byte + 32 HMAC bytes
  server_challenge_ts     TIMESTAMPTZ  NOT NULL,
  response_hash           BYTEA(32)    NOT NULL,
  response_latency_ms     INT          NOT NULL,
  audit_result            TEXT         CHECK (audit_result IN ('PASS','FAIL','TIMEOUT'))
                                       -- NULL = in-flight PENDING
                                       -- 'ABANDONED' after 48h GC
  provider_sig            BYTEA(64)    NOT NULL,
  service_sig             BYTEA(64)    NOT NULL,
  service_countersign_ts  TIMESTAMPTZ  NOT NULL,
  jit_flag                BOOLEAN      NOT NULL DEFAULT false,
  abandoned_at            TIMESTAMPTZ  -- NULL unless GC'd
)
```

Row security policy: INSERT only. No UPDATE. No DELETE. Enforced at DB level.

### Tests

```
TestChallengeNonceVersion      — nonce prefix encodes correct version byte
TestChallengeNonceDeterminism  — same inputs → same nonce
TestCrossReplicaValidation     — nonce generated by replica A is validated by replica B
  (use same cluster_master_seed; derive secret independently on each)
TestReceiptPhase1Durability    — kill microservice after Phase 1; on restart, PENDING row present
TestReceiptPhase2Idempotency   — provider retries same challenge; second attempt returns existing receipt
TestReceiptInsertOnly          — UPDATE and DELETE return permission denied
TestAuditPassRoundTrip         — challenge issued → provider responds → receipt countersigned → PASS
TestAuditFailOnMissingChunk    — provider without the chunk returns FAIL
TestAuditCorruptionFail        — content_hash mismatch on storage read → FAIL with corruption code
TestJITFlagThreshold           — response faster than 0.3× deadline sets jit_flag=true
TestAbandonedRowGC             — PENDING row older than 48h is marked ABANDONED
TestRTOUpdated                 — after 5 audit responses, RTO switches from pool-median to per-provider formula
```

### Observability

Wire up the following metrics in this milestone (NFR-025, NFR-026):
- `audit_challenges_issued_total`
- `audit_results_total{result="PASS|FAIL|TIMEOUT"}`
- `audit_receipt_latency_ms` (histogram)
- `content_hash_failures_total`

### Exit Criterion

```bash
go test ./internal/audit/... -v -race

# Run a full audit cycle against 56 simulated providers (simulation mode from M0)
scripts/test-audit-cycle.sh
# Expected output:
#   56 challenges issued
#   56 PASS receipts countersigned
#   56 rows in audit_receipts with audit_result='PASS'
#   0 PENDING rows older than 48h
```

---

## Milestone 5 — Reliability Scoring and Repair

**Goal:** The system computes per-provider reliability scores from audit history, detects
provider departures, and automatically repairs affected chunks through the lazy repair
protocol. The self-healing loop is complete.

**Dependency:** M4 (audit system must generate receipts before scoring can compute anything)

### Scope

**Package: `internal/scoring`**

Three-window rolling score (ADR-008):
- Query `audit_receipts` for each provider over the three windows: last 24h, last 7d, last 30d
- Score per window = `PASS_count / (PASS_count + FAIL_count + TIMEOUT_count)` in that window
- Final score = weighted combination (exact weights: product decision, start with 0.5 / 0.3 / 0.2)
- Score decrement on FAIL and TIMEOUT is non-I-confluent (ADR-013): single authoritative scorer,
  no distributed writes
- Materialised view for the aggregate score: refreshed asynchronously after each receipt batch,
  not synchronously per receipt insert

Failure clustering signal (ADR-008 open constraint, Paper 32):
- If a provider has > 1 audit FAIL in a rolling 7-day window, set `accelerated_reaudit = true`
  on all chunks for that provider; schedule re-audit of all their chunks in the next polling cycle

**Package: `internal/repair`**

Departure detection (ADR-007, ADR-006):
- Query `providers` where `last_heartbeat_ts < NOW() - 72 hours` AND `status = 'ACTIVE'`
- On match:
  1. SET `providers.status = 'DEPARTED'` (soft delete; physical row deletion prohibited)
  2. Delete all rows from `chunk_assignments` for this `provider_id` (stops challenge issuance)
  3. Seize escrow (see M6 — repair generates a seizure event; a no-op in M5 until M6 is complete)
  4. Enqueue repair jobs for all affected chunks

Repair scheduler — lazy trigger (ADR-004):
- Track available fragment count per chunk in `chunk_assignments`
- When count drops to `s + r0 = 24`: enqueue a repair job (pre-warning)
- When count drops to `s = 16`: enqueue immediately (emergency floor, ADR-004)
- Emergency floor has infinite priority; pre-warning jobs have lower priority

Repair scheduler — priority ordering (ADR-004, Paper 39):
- Two priority queues: `PERMANENT_DEPARTURE` and `PRE_WARNING`
- Drain all `PERMANENT_DEPARTURE` jobs before accepting any `PRE_WARNING` jobs
- Within each queue: FIFO

Repair execution:
- Contact k=16 surviving fragment holders via libp2p (M3)
- Download 16 fragments, RS decode (M1), re-encode missing fragments
- Select replacement providers using Power of Two Choices + 20% ASN cap (ADR-005, ADR-014)
- Upload to replacements, collect signed upload receipts
- Update `chunk_assignments` in Postgres

### Tests

```
TestThreeWindowScore           — known audit history produces correct per-window scores
TestScoreWeighting             — weighted combination is correctly computed
TestAcceleratedReaudit         — 2 FAILs in 7 days sets accelerated_reaudit on all provider's chunks
TestDepartureDetection         — provider with last_heartbeat_ts 73h ago is detected and DEPARTED
TestRepairPriorityOrdering     — PERMANENT_DEPARTURE jobs drain before PRE_WARNING jobs
TestRepairASNCap               — replacement provider selection respects 20% ASN cap
TestEmergencyFloor             — when fragment count hits s=16, repair fires immediately
  regardless of 72h threshold
TestLazyRepairThreshold        — fragment count at s+r0=24 triggers pre-warning, not immediate
TestRepairBandwidthEstimate    — at N=1000, 50GB/provider, computed BWavg ≈ 39 Kbps
```

### Integration test

```bash
scripts/test-self-healing.sh
# 1. Start 56 simulated providers
# 2. Store a test file (1 segment = 56 shards)
# 3. Kill 8 simulated providers (fragment count drops to 48, then trigger is at 24 — above threshold)
# 4. Verify: no repair triggered (48 > 24)
# 5. Kill 24 more providers (count drops to 24 = s+r0)
# 6. Verify: repair job enqueued within 60 seconds
# 7. Verify: after repair, 56 providers hold the chunk again
```

### Observability

Add in this milestone:
- `provider_score_histogram`
- `repair_queue_depth`
- `repair_jobs_completed_total`

Wire the `repair_queue_depth > 1000` alert (NFR-027).

### Exit Criterion

```bash
go test ./internal/scoring/... ./internal/repair/... -v -race

# Integration test passes end-to-end
scripts/test-self-healing.sh
# Output: repair completes, fragment count restored to 56
```

---

## Milestone 6 — Payment System

**Goal:** Providers accumulate earnings per audit pass, receive monthly partial releases
based on reliability score, and have escrow seized on silent departure. Razorpay integration
is real (test mode at this stage, live mode at M8).

**Dependency:** M5 (scoring must compute release multipliers; repair must trigger seizure)

### Scope

**Package: `internal/payment`**

Escrow ledger — PN-counter CRDT (ADR-016):
- `escrow_events` table: INSERT-only, ENUM(DEPOSIT, RELEASE, SEIZURE), `amount_paise BIGINT`
- `idempotency_key VARCHAR(64) UNIQUE`: `SHA-256(provider_id + audit_period)` for releases;
  `SHA-256(provider_id + withdrawal_request_id)` for data owner withdrawals
- Balance computed as: `SUM(DEPOSIT) - SUM(RELEASE + SEIZURE)` per `provider_id`
- Materialised view for balance, refreshed asynchronously
- No floating-point anywhere in this package (NFR-046)

Per-audit-pass earnings (ADR-012, ADR-024):
- `payout_per_audit_pass = (storage_rate_paise × chunk_size_GB) / audits_per_month`
- `storage_rate_paise` is a runtime config value (OQ-001 — product decision)
- Earn on PASS only; FAIL and TIMEOUT earn nothing; INSERT a DEPOSIT event per PASS

Vetting period economics (ADR-024, ADR-005):
- During vetting (status = 'VETTING'): hold window = 60 days, release cap = 50%
- After vetting (status = 'ACTIVE'): hold window = 30 days, no cap

Monthly release computation — runs on the 23rd of each month (ADR-024):
```
For each active provider:
  score_30d = query scoring package
  score_7d  = query scoring package
  dual_window_flag = (score_30d - score_7d) > 0.20
  effective_multiplier = min(release_multiplier(score_30d),
                             dual_window_flag ? release_multiplier(score_7d) : 1.0)
  releasable_amount_paise = escrow_window_balance × effective_multiplier
  on_hold_until = last_working_day_of_month()  // RBI holiday table
  call Razorpay PATCH /transfers/:id with on_hold_until
  INSERT RELEASE event to escrow_events
```

Release multiplier table (ADR-024):
| score_30d | multiplier |
|---|---|
| ≥ 0.95 | 1.00 |
| 0.80–0.94 | 0.75 |
| 0.65–0.79 | 0.50 |
| < 0.65 | 0.00 |

Withheld amounts roll forward (not seized) unless the provider departs silently.

RBI holiday table:
- Embedded as a Go constant in the payment package
- File: `internal/payment/rbi_holidays.go`, updated each December deployment
- `last_working_day_of_month(year, month) time.Time` — returns the last calendar day
  that is not Saturday, Sunday, or in the RBI holiday set

Escrow seizure on silent departure (ADR-024, M5 hooks):
- Called by the repair scheduler when `providers.status` is set to `DEPARTED`
- Freeze provider escrow account (set a `frozen` flag; refuse new DEPOSIT events)
- INSERT SEIZURE event for the full 30-day rolling window balance
- Call Razorpay Route reversal API if any transfer has not yet settled

Razorpay integration (ADR-011, Paper 35):
- Use Razorpay test credentials during all milestones up to M8
- Products used:
  - Smart Collect 2.0: virtual UPI ID per data owner contract
  - Route with `on_hold/on_hold_until`: monthly release hold
  - RazorpayX Payouts: monthly transfer to provider's bank account
- **X-Payout-Idempotency header mandatory** on every payout call (NFR-030)
- Payout.reversed webhook: append REVERSED event to escrow_events, balance refunded
- Razorpay Escrow+ is never called — it requires NBFC registration

PaymentProvider interface (ADR-011):
```go
type PaymentProvider interface {
    InitiateEscrow(ownerID, amount, contractID string) error
    ReleaseEscrow(providerID string, amount int64, auditPeriod string) error
    Penalise(providerID string, amount int64, reason string) error
    GetBalance(entityID string) (int64, error)
}
```
The Razorpay implementation of this interface is `internal/payment/razorpay.go`.

### Tests

```
TestEscrowLedgerPNCounter      — deposits and releases compute correct balance
TestIdempotentRelease          — duplicate idempotency_key returns existing event, no double credit
TestNoFloatArithmetic          — static analysis: no float64 or float32 in payment package
TestReleaseMultiplier          — all four score bands return correct multiplier
TestDualWindowFlag             — 0.21 drop in 7d vs 30d sets flag; 0.19 drop does not
TestVettingPeriodCap           — VETTING provider release capped at 50%
TestSeizureOnDeparture         — DEPARTED provider's 30-day balance moved to SEIZURE event
TestLastWorkingDay             — returns correct day accounting for weekends and RBI holidays
TestRazorpayIdempotency        — payout API called twice with same key; second call is no-op
TestRazorpayReversedWebhook    — webhook triggers REVERSED event insertion
```

### Observability

Add in this milestone:
- `escrow_events_total{type="DEPOSIT|RELEASE|SEIZURE"}`
- `payment_payout_total_paise`
- `payment_seizure_total_paise`

### Exit Criterion

```bash
go test ./internal/payment/... -v -race

# Smoke test against Razorpay test environment:
scripts/test-payment-cycle.sh
# Expected:
#   Provider accumulates 30 days of simulated audit passes
#   Monthly release computation runs
#   Razorpay test API receives PATCH with on_hold_until
#   Provider account shows releasable balance
#   Simulate departure: seizure event recorded, Razorpay reversal called
```

---

## Milestone 7 — Data Owner Client

**Goal:** A data owner can create an account (OTP + mnemonic backup), upload a real file,
retrieve it on a new machine using only their passphrase, and manage their escrow balance.
The full critical path — encode, upload, audit, retrieve — is exercised with real data.

**Dependency:** M1 (encoding pipeline), M3 (P2P upload to providers), M4 (audits begin on
stored chunks), M6 (escrow deposit required before upload)

### Scope

**Package: `cmd/client`**

Account creation flow (FR-001 through FR-006):
1. Enter phone number → OTP sent → OTP verified
2. Enter passphrase → `Argon2id(passphrase, owner_id) → master_secret` (M1)
3. Display 24-word BIP-39 mnemonic (derived from master_secret); block until two
   randomly chosen words confirmed by the user (FR-003)
4. Derive Ed25519 signing key pair from keystore_enc_key (M1)
5. Register with microservice (phone, public key)

Account restore flow (FR-004):
- Path A: enter passphrase → re-derive master_secret → all keys recoverable
- Path B: enter 24-word mnemonic → reconstruct master_secret → same as Path A

File upload flow (FR-007 through FR-014):
1. Check escrow balance ≥ 30-day cost for file (refuse if insufficient, FR-014)
2. Pad file to minimum segment size if < 4 MB (FR-008)
3. For each segment (max 14 MB):
   - Run AONT-RS encoding (M1) → 56 shards
   - Request 56 provider assignments from microservice `/api/v1/upload/assign`
     (ASN cap enforced by microservice, FR-009)
   - Upload all 56 shards via parallel libp2p connections (M3, FR-010)
   - Collect signed upload receipts from all 56 providers
4. Build pointer file (provider list, chunk IDs, erasure params, original file size)
5. Encrypt pointer file with AEAD_CHACHA20_POLY1305 using pointer_enc_key (M1)
6. POST encrypted ciphertext to microservice `/api/v1/file/register`
7. Persist upload session state locally (keyed by session ID) before starting;
   clean up only after pointer file confirmed stored (FR-060 crash safety)

Upload progress: emit per-shard progress events; display as "X of 56 shards confirmed"

File retrieval flow (FR-015 through FR-018):
1. GET encrypted pointer file ciphertext from microservice
2. Decrypt locally using pointer_enc_key (M1)
3. Attempt retrieval from the 16 fastest-responding providers (FR-016)
4. Verify each fragment: `SHA-256(fragment) == chunk_id` from pointer file (FR-017)
5. RS decode 16 shards (M1)
6. AONT decode, verify canary (M1, FR-018)
7. Strip padding, write plaintext

Escrow deposit — UPI Intent flow (FR-006, NFR-029):
- UPI Collect is deprecated; use UPI Intent only
- Display UPI Intent QR code or app-selector; on payment confirmation, Razorpay webhook
  credits the data owner's escrow account
- UPI Collect (`collect` field in Razorpay API) must not appear anywhere in this code

File management (FR-019 through FR-021, P1 — include in MVP for operator use):
- List files: name, size, upload date, monthly cost, fragment availability status
- Delete file: POST to microservice → microservice removes chunk_assignments,
  notifies providers; `pending_deletion` flag for unreachable providers at deletion time
- Escrow balance view: current balance, 30-day reserved amount, available for withdrawal

### Tests

```
TestAccountCreateAndRestore    — create account, restore on fresh keystore using passphrase
TestMnemonicRoundTrip          — master_secret → 24 words → master_secret (identical)
TestUploadSmallFile            — 1 MB file, padded to 4 MB, encodes and stores across 56 providers
TestUploadLargeFile            — 50 MB file, segments correctly, all segments stored
TestRetrievalFreshMachine      — upload on machine A, restore using passphrase on machine B, retrieve identical
TestUploadCrashResume          — kill client after 30 of 56 shard uploads; restart; only 26 remaining are re-uploaded
TestEscrowInsufficientBlock    — upload refused when escrow < 30-day cost
TestASNCapEnforced             — assignment service returns providers from ≥ 5 ASNs (via simulation)
TestUPICollectAbsent           — static analysis: string "collect" must not appear in UPI-related code paths
```

### Exit Criterion

```bash
go test ./cmd/client/... -v -race

# Full end-to-end smoke test (requires simulation mode from M0):
scripts/test-e2e-upload-retrieve.sh
# 1. Create account (passphrase: "test-passphrase-twelve-words")
# 2. Deposit 1000 paise to escrow (Razorpay test mode)
# 3. Upload a 10 MB test file
# 4. Verify 56 audit challenges issued and 56 PASS receipts recorded
# 5. Delete local client state (simulates machine loss)
# 6. Restore account using passphrase only
# 7. Retrieve file
# 8. SHA-256 of retrieved file == SHA-256 of original (identical)
```

---

## Milestone 8 — Network Readiness Gate and Private Beta

**Goal:** All seven readiness conditions are satisfied simultaneously. The network is
deployed to real cloud infrastructure with real relay nodes. Real provider machines are
recruited and onboarded. The system accepts its first real upload from a real data owner.
Razorpay goes live (test → production credentials). Private beta begins.

**Dependency:** All previous milestones complete and passing CI.

### Scope

**Network readiness gate — all 7 conditions enforced (FR-053, FR-054, ADR-029):**

```go
type ReadinessCheck struct {
    ActiveVettedProviders      int  // required ≥ 56
    DistinctASNs               int  // required ≥ 5
    DistinctMetroRegions       int  // required ≥ 3 (Delhi NCR, Mumbai, one south metro)
    MicroserviceQuorumHealthy  bool // true iff all 3 replicas respond to gossip
    RazorpayAccountsReady      int  // Linked Accounts with 24h cooling past, required ≥ 56
    RelayNodesDeployed         int  // required ≥ 3
    AuditSecretLoaded          bool // true iff server_secret_vN present on all replicas
}

// Returns HTTP 503 if any condition is not met
// Returns HTTP 200 with JSON breakdown if all met
```

**Infrastructure deployment:**
- Three microservice replicas on AWS `t3.medium` or equivalent, one per AZ in `ap-south-1`
- Managed Postgres (`db.t3.medium`) with Multi-AZ replication, 100 GB GP3 SSD
- Three relay nodes (`t3.micro` or equivalent, 1 Gbps network), one per AZ
- Secrets manager: AWS SSM Parameter Store with IAM-gated access
- Load balancer in front of heartbeat endpoint (separate from audit challenge endpoints)
- Prometheus metrics scrape endpoint live; basic Grafana dashboard with the four alert
  thresholds from NFR-027

**Razorpay — switch from test to production:**
- Create production Razorpay master account
- Create Route Linked Accounts for each recruited provider (24h cooling starts now)
- Smart Collect 2.0 virtual UPI IDs for data owner escrow deposits
- X-Payout-Idempotency header verified on every outbound payout call
- UPI Intent flow tested with a real payment (not test mode)

**Pre-beta checklist (all must be green before any upload is permitted):**

```
□ All P0 functional requirements (FR-001 through FR-054) implemented and tested
□ All P0 NFR benchmarks (NFR-007 through NFR-013) passing on minimum-spec hardware
□ audit_receipts row security policy verified: UPDATE and DELETE return permission denied
  in production Postgres
□ challenge_nonce column is BYTEA(33), not BYTEA(32) (cross-document correction)
□ UPI Collect absent from codebase (grep and static analysis pass)
□ X-Payout-Idempotency header present on all Razorpay payout calls
□ BIP-39 mnemonic confirmation blocks until two words entered correctly
□ Simulation mode (--sim-count=56) satisfies all 7 readiness conditions in CI
□ Relay nodes deployed and reachable from at least 3 Indian metro test machines
□ Secrets manager loaded; at least one cluster secret rotation tested (ADR-027)
□ Provider daemon installed on minimum-spec test machine; all NFR benchmarks pass
□ Data owner: uploaded 100 MB file end-to-end, retrieved on fresh machine, SHA-256 matches
□ Repair: deliberately killed 8 of 56 providers; repair completed within 8 hours
□ Payment: monthly release ran in test mode; correct paise amounts transferred
□ Alert thresholds wired and tested (repair_queue_depth, TIMEOUT rate, hash failures, quorum)
□ readiness gate: returns HTTP 200 only when all 7 conditions simultaneously true
□ Razorpay live credentials loaded (not test mode)
□ First real provider (Ravi's NAS) installed, registered, and past 24h Razorpay cooling
```

**Private beta scope:**
- 20 invited data owners; 100 invited providers
- Uploads limited to 100 MB per data owner during beta
- Provider earnings held in full (no releases) until 30-day stability is confirmed
- Weekly check against NFR-027 alert thresholds; any breach pauses onboarding

### Exit Criterion

```bash
# All 7 conditions green:
curl https://api.vyomanaut.in/api/v1/admin/readiness | jq .
# Expected: all conditions true

# Real end-to-end with real providers:
# 1. Data owner uploads 50 MB file from Pune
# 2. 56 providers in 5 ASNs across Delhi, Mumbai, Bangalore receive shards
# 3. Audit challenges fire within 24h; all 56 return PASS
# 4. Data owner retrieves on a new machine; file is identical
# 5. Provider reliability scores are non-zero after 24h

# Private beta onboarding begins
```

---

## What Ships in Each Milestone

| Milestone | What a reviewer can observe | ADRs exercised |
|---|---|---|
| M0 | `--sim-count=56` starts without crash; readiness endpoint shows all FAIL | ADR-029 |
| M1 | `TestFullPipeline` encodes a file and decodes it identically; canary passes | ADR-003, ADR-019, ADR-020, ADR-022 |
| M2 | 200,000 chunks stored; any random chunk retrieved under 15 ms on SSD | ADR-023, ADR-025 (schema) |
| M3 | 56 sim providers send heartbeats; libp2p connection established | ADR-021, ADR-028, ADR-001 |
| M4 | Audit challenge → PASS receipt visible in DB; INSERT-only enforced | ADR-002, ADR-015, ADR-017, ADR-027 |
| M5 | Kill 32 sim providers; repair fires; fragment count restored to 56 | ADR-004, ADR-005, ADR-007, ADR-008 |
| M6 | Provider accumulates paise per PASS; monthly release paise computed | ADR-011, ADR-012, ADR-016, ADR-024 |
| M7 | Real file uploaded, retrieved on fresh machine with only passphrase | ADR-019, ADR-020, ADR-022 + all above |
| M8 | Real network, real providers, real money, first real upload accepted | ADR-029 + all above |

---

## Open Questions Resolved Before Each Milestone Ships

These are the questions from `docs/research/open-questions.md` whose answers affect
implementation decisions within each milestone.

| Milestone | Question | How to resolve |
|---|---|---|
| M1 | Q18-1 — Argon2id parameters on min-spec hardware | Run benchmark on min-spec machine; if p50 > 500ms, apply fallback protocol, update ADR-020 |
| M1 | Q16-1 — AONT encoding throughput without AES-NI | Run benchmark; if p50 > 200ms, investigate (likely Python vs Go difference — repeat in Go) |
| M2 | Q27-1 — RocksDB rate limiter calibration (SSD) | Run Q27-1 benchmark from benchmarking-protocol.md |
| M2 | HDD benchmark — rate limiter calibration (HDD) | Run HDD variant from ADR-023 |
| M3 | Q14-1 — fraction of Indian ISPs blocking UDP | Instrument transport type at connection; report at 30-day mark |
| M4 | Postgres INSERT ceiling — actual schema benchmark | INSERT stress test against production schema with row security policy active; record result |
| M5 | Q06-3 — false-positive departure rate | Track providers declared departed who reconnect within 72h; if > 5%, adjust threshold |
| M6 | OQ-001 — storage rate (paise/GB/month) | Product decision must be made before FR-013 (cost display) and FR-014 (balance check) can be completed |
| M8 | Q20-1 — Indian CGNAT fraction | Log AutoNAT classification per provider at registration; report at 30-day beta mark |

---

## Staffing and Parallel Work

The dependency graph allows this parallel structure within a team:

**Sprint 1 (M0 + M1 + M2 in parallel):**
- Engineer A: M0 — monorepo, CI, Postgres schema, simulation mode skeleton
- Engineer B: M1 — encoding pipeline (AONT, RS, key hierarchy)
- Engineer C: M2 — WiscKey storage engine (RocksDB + vLog)

**Sprint 2 (M3 + M4, M4 depends on M2 and M3):**
- Engineer A: M3 — libp2p networking, heartbeat
- Engineer B+C: M4 — audit system (needs M2 for provider storage read, M3 for dispatch)

**Sprint 3 (M5, depends on M4):**
- Full team: M5 — scoring and repair (most complex integration work)

**Sprint 4 (M6, depends on M5):**
- Engineer A: M6 — payment system, Razorpay integration

**Sprint 5 (M7, depends on M1+M3+M4+M6):**
- Engineer B+C: M7 — data owner client

**Sprint 6 (M8):**
- Full team: cloud deployment, relay nodes, provider recruitment, beta preparation

---

## Invariants That Must Never Break

These are not tests. They are properties that, if violated, make the system wrong in ways
no user can detect. Check them in code review on every PR.

1. **The microservice never holds a decryption key.** If any PR introduces a path where
   the microservice derives `file_key`, `pointer_enc_key`, or `master_secret`, reject it.

2. **The audit receipt table is INSERT-only.** If any application code calls UPDATE or
   DELETE on `audit_receipts`, reject the PR. The row security policy enforces this at the
   DB level, but it should also never appear at the application level.

3. **The 20% ASN cap is enforced at assignment time, not retrospectively.** If any
   assignment code path can produce a state where one ASN holds > ceil(0.20 × 56) = 12
   shards of any file, reject it.

4. **challenge_nonce is 33 bytes.** The 1-byte version prefix is part of the nonce, not
   an outer wrapper. Any code that generates a 32-byte nonce and calls it challenge_nonce
   is wrong.

5. **All amounts are integer paise.** Float64 division, multiplication, or rounding in
   any payment code path is wrong. If `paise / 100.0` appears in the codebase, reject it.

6. **UPI Collect must not be called.** NPCI deprecated it February 2026. Any code
   calling the Razorpay `collect` endpoint for user-facing payment initiation is wrong.

7. **The AONT key K is never stored.** K is recovered from the threshold, not from a
   key store. Any code that persists K to disk or transmits it to the microservice is wrong.

8. **vLog appends go through the single writer goroutine.** Any code that writes to the
   vLog file handle from a goroutine other than the designated writer is wrong.

---

## The First Five Minutes

For a new engineer joining the project after the MVP is shipped:

```bash
# 1. Clone the repo
git clone https://github.com/masamasaowl/Vyomanaut_Research
cd Vyomanaut_Research

# 2. Start a local Postgres
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=dev postgres:16

# 3. Apply the schema
psql postgresql://postgres:dev@localhost:5432/postgres -f migrations/001_initial_schema.sql

# 4. Start the microservice (dev mode: no secrets manager, env var for audit secret)
VYOMANAUT_CLUSTER_MASTER_SEED=$(openssl rand -hex 32) \
  go run ./cmd/microservice --dev-mode --db-url=postgresql://postgres:dev@localhost:5432/postgres

# 5. Start 56 simulated providers
go run ./cmd/provider \
  --sim-count=56 --sim-asn-count=5 \
  --microservice-url=http://localhost:8080 \
  --sim-data-dir=/tmp/vyomanaut-sim

# 6. Run the full end-to-end test
scripts/test-e2e-upload-retrieve.sh

# If this passes, you understand the system.
```