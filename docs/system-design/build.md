# Vyomanaut V2 — BUILD Procedure

**Version:** 1.0
**Status:** Active — evolves with the build
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research
**Derived from:** `docs/system-design/architecture.md`, `docs/system-design/data-model.md`, `docs/system-design/mvp.md`
**Awaiting integration:** `docs/system-design/requirements.md`, `docs/system-design/interface-contracts.md`, `docs/system-design/api/openapi.yaml`

> **Primary build principle (mvp.md §9.3):** Build the full system against `DemoProfile` first. Every code path must execute in a 30-minute demo session before production infrastructure is introduced. `VYOMANAUT_MODE=prod` is a configuration switch, not a code change.

> **How to read this file:** Each **Session** is the atomic unit of work — a self-contained task with a clear definition of done. The **Doc Ref** on each session is the exact section that governs its constraints and invariants. When a referenced document is not yet in context, the session is marked `[BLOCKED: missing doc]` — do not begin it until the doc is loaded. Every implementation decision that conflicts with the referenced section must be logged as a candidate fix before merging.

---

## Milestone Index

| # | Milestone | Build phase | Depends on |
|---|---|---|---|
| M0 | Project Foundation | Scaffold | — |
| M1 | NetworkProfile & Mode Flag | Scaffold | M0 |
| M2 | Cryptographic Primitives | Core pipeline | M1 |
| M3 | Erasure Coding Engine | Core pipeline | M2 |
| M4 | Provider Storage Engine (WiscKey) | Core pipeline | M1 |
| M5 | P2P Network Layer | Core pipeline | M1 |
| M6 | Microservice Database Schema | Core pipeline | M1 |
| M7 | Provider Daemon — Registration & Heartbeat | Demo lifecycle | M4, M5, M6 |
| M8 | Data Owner Client — Encode & Upload | Demo lifecycle | M2, M3, M5 |
| M9 | Audit System | Demo lifecycle | M5, M6, M7 |
| M10 | Provider Lifecycle — Vetting & ACTIVE Transition | Demo lifecycle | M9 |
| M11 | Repair System | Demo lifecycle | M9, M10 |
| M12 | Scoring & Payment (Mock) | Demo lifecycle | M9, M10 |
| M13 | Data Owner Client — Retrieval | Demo lifecycle | M8, M9 |
| M14 | Demo Integration & End-to-End Validation | Demo complete | M7–M13 |
| M15 | Secrets Manager & Cluster Audit Secret | Production infra | M9, M14 |
| M16 | Razorpay Payment Integration | Production infra | M12, M14 |
| M17 | Relay Nodes & Production NAT Traversal | Production infra | M5, M14 |
| M18 | Microservice HA Quorum | Production infra | M6, M14 |
| M19 | Production Hardening & Launch Gate | Launch | M15–M18 |

---

## Milestone 0 — Project Foundation

**Goal:** A compiling, linting, and test-running Go module with the exact directory layout specified in the system design. No business logic yet.

**Doc Ref:** `mvp.md §8.1` (top-level directory map), `mvp.md §8.4` (CI pipeline structure), `arch §4` (technology stack — Go ≥ 1.22, single module)

---

### Phase 0.1 — Repository Layout

**Doc Ref:** `mvp.md §8.1`

#### Session 0.1.1 — Initialise Go module

**Task:** Create the Go module `github.com/masamasaowl/vyomanaut` with Go ≥ 1.22. Create `go.mod`. Verify `go build ./...` produces no errors on an empty module.

**Definition of done:** `go build ./...` exits 0. `go.mod` contains `go 1.22` or higher.

**Note:** The module path is a fixed identifier used in DHT namespace strings (`/vyomanaut/dht-key/1.0.0`) and in external documentation. Do not change it after this session.

#### Session 0.1.2 — Create directory skeleton

**Task:** Create every directory listed in `mvp.md §8.1` with a `.gitkeep` placeholder. Do not create any `.go` files yet. Directories to create:
`cmd/microservice`, `cmd/provider`, `cmd/client`, `internal/config`, `internal/crypto`, `internal/erasure`, `internal/storage`, `internal/p2p`, `internal/audit`, `internal/scoring`, `internal/repair`, `internal/payment`, `internal/client/account`, `internal/client/upload`, `internal/client/retrieve`, `internal/client/manage`, `migrations`, `deployments/production`, `deployments/staging`, `deployments/dev`, `scripts/benchmarks`, `runbooks`, `docs/decisions`, `docs/research`, `docs/system-design/api`, `.github/workflows`.

**Definition of done:** All directories exist. `find . -type d` matches the spec.

#### Session 0.1.3 — Create cmd entrypoint stubs

**Task:** Create minimal `main.go` stubs in `cmd/microservice`, `cmd/provider`, and `cmd/client` that do nothing but print `"not implemented"` and exit 0.

**Definition of done:** `go build ./cmd/microservice`, `./cmd/provider`, `./cmd/client` all succeed.

---

### Phase 0.2 — CI Pipeline

**Doc Ref:** `mvp.md §8.4` (CI pipeline — all 15 required checks)

#### Session 0.2.1 — Create `.github/workflows/ci.yml`

**Task:** Create the CI workflow file with all 15 required checks from `mvp.md §8.4`. For checks 5–15 (which require actual code), add placeholder steps that `echo "TODO: not yet implemented — check will be enforced once target code is merged"` and exit 0. The placeholder must be replaced before the relevant milestone closes. Mark each placeholder with the milestone that owns it.

Required checks to scaffold:
1. `go build ./...`
2. `go vet ./...`
3. `golangci-lint run`
4. `go test ./... -race`
5. `TestDHTKeyValidatorPersists` — owned by M5
6. `TestNoFloatArithmetic` — owned by M12
7. Migration apply + rollback — owned by M6
8. Grep fail: `challenge_nonce BYTEA(32)` — owned by M9
9. Grep fail: `float64|float32|FLOAT|DECIMAL|NUMERIC` in payment context — owned by M12
10. Grep fail: `ADR-039` (non-existent ADR) — active from day one
11. Grep fail: UPI Collect API endpoint string — active from day one
12. Mermaid render check — active from day one
13. Hyperlink check — active from day one
14. `TestProfileShardSizeIsConstant` — owned by M1
15. `TestProfileBothFullySpecified` — owned by M1

**Definition of done:** CI workflow file exists. PR triggers all 15 steps without failing.

#### Session 0.2.2 — Create `.golangci.yml`

**Task:** Create the linter configuration file with all mandatory linters from `mvp.md §8.4`: `gofmt`, `govet`, `errcheck`, `exhaustive`, `godot`, `gomnd`. Configure `exhaustive` to require switches on `AuditResult`, `ProviderStatus`, `EscrowEventType`, `RepairPriority` to handle all cases.

**Definition of done:** `golangci-lint run` passes on the stub `main.go` files.

#### Session 0.2.3 — Create `CODEOWNERS`

**Task:** Create `.github/CODEOWNERS`. The `/migrations/` directory requires 3 reviewers. The `/internal/crypto/` and `/internal/payment/` directories require 2 reviewers from separate ownership tracks.

**Definition of done:** CODEOWNERS file exists with correct path-to-reviewer mappings.

---

### Phase 0.3 — Development Environment

**Doc Ref:** `mvp.md §8.5` (infrastructure directory conventions — dev docker-compose)

#### Session 0.3.1 — Create `deployments/dev/docker-compose.yml`

**Task:** Create a docker-compose file that starts: one Postgres 15+ instance (with `btree_gist` pre-installed via init script), one stub microservice container, one stub provider daemon with `--sim-count=5 --sim-asn-count=5`. The dev environment must reach a healthy state (all containers running) within 10 seconds of `docker compose up`.

**Doc Ref:** `arch §4` (PostgreSQL ≥ 15 required for `NULLS NOT DISTINCT`), `data §9` (migration checklist — `btree_gist` must be installed before any migration)

**Definition of done:** `docker compose up` succeeds. Postgres is reachable on port 5432.

#### Session 0.3.2 — Create Postgres init script

**Task:** Create `deployments/dev/postgres/init.sql` that runs `CREATE EXTENSION IF NOT EXISTS btree_gist;` and creates the `vyomanaut_app` and `vyomanaut_gc` Postgres roles with appropriate grants. These roles are required before any migration in `data §6` (row security policies) can be applied.

**Definition of done:** Postgres starts with both roles and `btree_gist` installed. Verified by `\dx` and `\du`.

---

## Milestone 1 — NetworkProfile & Mode Flag

**Goal:** The single source of truth for all mode-variable parameters exists, compiles, passes compiler-enforced invariant tests, and is injected correctly at startup.

**Doc Ref:** `mvp.md §2` (mode flag definition), `mvp.md §3` (exact demo specs), `mvp.md §5.1` (NetworkProfile struct), `mvp.md §5.2` (profile instances), `mvp.md §5.3` (profile selection), `arch §9` (core design principles — profile-driven configuration), `data §3 Invariant 7` (ShardSize is a compile-time constant)

---

### Phase 1.1 — NetworkProfile Struct

#### Session 1.1.1 — Define `NetworkProfile` in `internal/config/network_profile.go`

**Task:** Implement the `NetworkProfile` struct exactly as specified in `mvp.md §5.1`. All fields must be present. `ShardSize int` must be present but its value must be `262144` in both profiles — it is included in the struct so the compiler can enforce it via `TestProfileShardSizeIsConstant`, but no subsystem should ever vary it based on the profile. Add a doc comment on the struct that explicitly states it is the sole source of mode-variable values and that subsystems must never read `VYOMANAUT_MODE` directly.

**Definition of done:** File compiles. All fields from `mvp.md §5.1` are present with correct Go types.

#### Session 1.1.2 — Implement `ProductionProfile` and `DemoProfile` in `internal/config/profiles.go`

**Task:** Implement both profile instances with the exact values from `mvp.md §5.2`. Every field must be explicitly set — no zero-value defaults. The Go struct literal must be exhaustive; if a new field is added to `NetworkProfile`, the compiler fails until both profiles are updated.

**Doc Ref:** `mvp.md §5.2`, `mvp.md §3` (exact demo specs for every parameter)

**Definition of done:** Both profile vars compile. Every field value matches `mvp.md §3` exactly.

#### Session 1.1.3 — Write compiler-enforced tests in `internal/config/profiles_test.go`

**Task:** Implement two tests:

`TestProfileShardSizeIsConstant`: asserts `DemoProfile.ShardSize == ProductionProfile.ShardSize == 262144`. If either differs, the test must fail with a message that names `data §3 Invariant 7` and `mvp.md §5.1`.

`TestProfileBothFullySpecified`: uses reflection to verify no field in `ProductionProfile` or `DemoProfile` holds a zero value. Zero values are bugs — every parameter must be a deliberate choice. Fields permitted to be zero: `ReleaseComputationInterval` (production uses a calendar date, not a ticker — zero is the sentinel), `RazorpayCoolingPeriod` in `DemoProfile` (intentionally zero).

**Definition of done:** Both tests pass. CI check #14 and #15 in `mvp.md §8.4` activate.

---

### Phase 1.2 — Mode Selection & Startup Guards

#### Session 1.2.1 — Implement `selectProfile()` in shared startup logic

**Task:** Implement the `selectProfile()` function from `mvp.md §5.3` in `internal/config/select.go`. The function reads `VYOMANAUT_MODE` env var, falls back to a CLI flag `--mode`, defaults to `prod` with a warning if unset. The function must print the exact startup banner strings from `mvp.md §2.1`.

**Definition of done:** Function compiles. Unit test verifies all three cases: explicit demo, explicit prod, unset (defaults to prod with warning logged).

#### Session 1.2.2 — Implement startup guard checks

**Task:** Implement the three mandatory startup guard checks from `mvp.md §2.3`:
1. `VYOMANAUT_MODE=prod` AND `VYOMANAUT_CLUSTER_MASTER_SEED` present in env → `log.Fatalf`
2. `VYOMANAUT_MODE=demo` AND real Razorpay live API endpoint string present → `log.Fatalf`
3. `VYOMANAUT_MODE` absent → default to `prod` and log a warning

**Doc Ref:** `mvp.md §2.3`, `arch §18` (secrets manager — fail-closed design), `mvp.md §6.2 IR-05`

**Definition of done:** Unit tests cover all three guard conditions. Guards are called from `selectProfile()` before returning.

#### Session 1.2.3 — Wire profile into `cmd/` entrypoints

**Task:** Update all three `main.go` stubs to call `selectProfile()` and print the full `NetworkProfile` struct at startup (field values, no secrets). The profile instance is the first argument passed to every subsystem constructor. No subsystem may call `os.Getenv("VYOMANAUT_MODE")` directly — this is enforced by the linter.

**Doc Ref:** `mvp.md §5.3`, `mvp.md §6.3 OR-01` (profile must be printed at startup)

**Definition of done:** Running any binary prints the profile fields. `go vet` and `golangci-lint` pass.

---

## Milestone 2 — Cryptographic Primitives

**Goal:** All cryptographic operations compile, pass known-answer vector tests, and satisfy the performance NFRs on minimum-spec hardware. No integration with other systems yet.

**Doc Ref:** `arch §10` (data encoding pipeline — cipher paths), `arch §11` (key hierarchy), `arch §4` (technology stack — crypto library selections), `arch §4.1` (technology rationale — cipher choice), `mvp.md §3.5` (demo cryptographic parameters), `data §3 Invariant 5` (33-byte challenge nonce)

> **Note:** The full cryptographic requirements and NFRs governing performance targets (NFR-009, NFR-010, NFR-019) are in `requirements.md`, which is not yet in context. Build to the performance targets stated in `arch §3` (quality attributes) and `arch §4.1` (performance contracts). **[BLOCKED: full NFR list pending `requirements.md`]** — tag any implementation detail that needs NFR cross-check.

---

### Phase 2.1 — CPUID Detection & Cipher Selection

**Doc Ref:** `arch §10 Stage 1` (AONT cipher selection), `arch §4.1` (ChaCha20 vs AES-CTR rationale)

#### Session 2.1.1 — Implement `internal/crypto/aesni.go` and `aesni_other.go`

**Task:** Implement CPUID-based AES-NI detection in `aesni.go` with build tag `//go:build amd64`. The function signature is `HasAESNI() bool`. Create a stub `aesni_other.go` with the same signature that always returns `false` for non-amd64 platforms. The cipher selection result is read once at daemon startup and stored as a package-level constant — it is never re-checked at runtime.

**Doc Ref:** `arch §10 Stage 1` ("The cipher is ChaCha20-256 on hardware without AES-NI... detected at startup via CPUID and sets a global constant"), `arch §4.1` (ChaCha20 performance contract: 75 MB/s no-AES-NI; AES-256-CTR: ~900 MB/s on AES-NI)

**Definition of done:** `HasAESNI()` compiles on amd64 and non-amd64. Unit test verifies the stub returns false on the CI platform.

---

### Phase 2.2 — AONT Cipher (ChaCha20 / AES-256-CTR)

**Doc Ref:** `arch §10 Stage 1` (AONT step-by-step algorithm), `arch §4.1` (cipher rationale — RC4 explicitly rejected)

#### Session 2.2.1 — Implement `internal/crypto/aont.go`

**Task:** Implement the AONT transform as described in `arch §10 Stage 1`. The five steps must map exactly to the spec:
1. Append 16-byte canary to the segment.
2. Generate `K = SecureRandom(256 bits)`.
3. For each 16-byte word `d_i`: encrypt via the selected cipher path (ChaCha20 path or AES-256-CTR path based on `HasAESNI()`).
4. Compute `h = SHA-256(c_0 || c_1 || ... || c_s)` (over all codewords including canary-appended segment).
5. Append key-embedding block: `c_{s+1} = K XOR h`.

The ChaCha20 path uses `ChaCha20_keystream_word(K, block=⌊i/16⌋, offset=i%16)`. The AES-CTR path uses `AES-256-ECB(K, i+1)` (counter starts at `i+1` per the AONT-RS spec cited in `arch §10`).

**Explicit prohibition (arch §4.1):** RC4-128 + MD5 is the explicitly insecure AONT-RS configuration from Paper 16. It must never be implemented or accepted.

**Definition of done:** Encode + decode round-trip test with a known 4 MB segment. Canary verification passes. K recovery test: given all s+1 codewords, K is correctly recovered.

#### Session 2.2.2 — Implement `internal/crypto/aont_canary.go`

**Task:** Define the canary as a fixed-value `[16]byte` constant (not a var). The value must be a fixed, non-zero 16-byte sequence defined in the code. Document that the canary is specified in `arch §10 Stage 4` ("verified on decode to detect corruption — if the canary fails, the segment is corrupt").

**Definition of done:** `const AONTCanary [16]byte` compiles. Unit test: corrupt one byte of a decoded segment and verify `ErrCanaryMismatch` is returned.

---

### Phase 2.3 — Key Derivation (HKDF & Argon2id)

**Doc Ref:** `arch §11` (key hierarchy diagram), `arch §4.1` (HKDF-SHA256 + Argon2id rationale), `mvp.md §3.5` (demo Argon2id parameters)

#### Session 2.3.1 — Implement `internal/crypto/argon2.go`

**Task:** Implement `DeriveMasterSecret(passphrase string, ownerID string, profile config.NetworkProfile) []byte`. The function uses Argon2id with parameters from the profile: `profile.Argon2Time`, `profile.Argon2Memory`, `profile.Argon2Threads`. Output: 32 bytes. The function must never log or print its output.

Production values (from `mvp.md §5.2`): t=3, m=65536 KiB, p=4.
Demo values: t=1, m=4096 KiB, p=1.

Performance contract (`arch §4.1`): production parameters complete in ≈200 ms on minimum-spec hardware. **[NFR-010 reference pending `requirements.md`]**

**Definition of done:** Unit test with known-answer vector for both parameter sets. Timing test: demo params complete in < 100 ms on CI hardware.

#### Session 2.3.2 — Implement `internal/crypto/hkdf.go`

**Task:** Implement HKDF-SHA256 derivations matching the key hierarchy in `arch §11`:
- `DeriveFileKey(masterSecret []byte, fileID string) []byte` — info: `"vyomanaut-file-v1" || fileID`
- `DerivePointerKey(masterSecret []byte, fileID string) []byte` — info: `"vyomanaut-pointer-v1" || fileID`
- `DeriveKeystoreKey(masterSecret []byte) []byte` — info: `"vyomanaut-keystore-v1"`
- `DeriveDHTKey(masterSecret []byte, fileID string) []byte` — info: `"vyomanaut-dht-v1" || fileID` (used for DHT privacy, `arch §9.5`)
- `DeriveFilenameKey(masterSecret []byte, fileID string) []byte` — info: `"vyomanaut-filename-v1" || fileID` (used for `files.display_name_ciphertext`, `data §4.3`)

All domain separation strings must be defined as named constants — not inline strings. The domain strings are specified in `arch §4.1` ("documented in ADR-020").

**Definition of done:** Unit tests with known-answer vectors for all five derivation paths. A test verifies that all five outputs are distinct for the same master secret and file ID.

---

### Phase 2.4 — Authenticated Encryption (Pointer File)

**Doc Ref:** `arch §10 Stage 3` (pointer file encryption: AEAD_CHACHA20_POLY1305), `arch §4.1` (AEAD rationale), `data §4.3` (pointer_ciphertext, pointer_nonce BYTEA(12), pointer_tag BYTEA(16))

#### Session 2.4.1 — Implement `internal/crypto/chacha20poly1305.go`

**Task:** Implement pointer file encryption and decryption using `AEAD_CHACHA20_POLY1305` (RFC 8439). Functions:
- `EncryptPointerFile(key []byte, nonce [12]byte, plaintext []byte, aad []byte) (ciphertext []byte, tag [16]byte)`
- `DecryptPointerFile(key []byte, nonce [12]byte, ciphertext []byte, tag [16]byte, aad []byte) ([]byte, error)`

The tag comparison in `DecryptPointerFile` must use `crypto/subtle.ConstantTimeCompare`. A tampered tag must return `ErrTagMismatch` before any decryption attempt — the check precedes the decryption step. **[NFR-019 reference pending `requirements.md`]**

The returned `tag` from encrypt is always exactly 16 bytes — matching `data §4.3` `pointer_tag BYTEA(16) CHECK (octet_length(pointer_tag) = 16)`.

**Definition of done:** Round-trip test. Tampered-ciphertext test returns `ErrTagMismatch`. Tampered-tag test returns `ErrTagMismatch` (constant-time, no oracle). Tampered-AAD test returns error.

---

### Phase 2.5 — Digital Signatures (Ed25519)

**Doc Ref:** `arch §4.1` (Ed25519 rationale), `arch §14` (audit receipt — provider_sig and service_sig), `data §4.2` (ed25519_public_key BYTEA(32))

#### Session 2.5.1 — Implement Ed25519 helpers in `internal/crypto/` (stdlib only)

**Task:** Implement thin wrappers over `crypto/ed25519` (Go stdlib — no external dependency):
- `GenerateKeyPair() (publicKey [32]byte, privateKey [64]byte, err error)`
- `Sign(privateKey [64]byte, message []byte) [64]byte`
- `Verify(publicKey [32]byte, message []byte, signature [64]byte) bool`

Both `publicKey` and `signature` are fixed-size arrays, matching the BYTEA CHECK constraints in `data §4.2` and `data §4.7`. No field in any database table stores an Ed25519 private key.

**Definition of done:** Sign-then-verify round-trip test. Corrupted-signature test returns false. Sub-millisecond signing confirmed via benchmark.

---

### Phase 2.6 — BIP-39 Mnemonic

**Doc Ref:** `arch §11` (BIP-39 mnemonic as offline backup for master_secret), `mvp.md §3.5` (mnemonic confirmation skipped in demo), `mvp.md §8.2` (package file inventory — `bip39.go`)

#### Session 2.6.1 — Implement `internal/crypto/bip39.go`

**Task:** Implement:
- `MasterSecretToMnemonic(masterSecret []byte) ([]string, error)` — returns 24 words
- `MnemonicToMasterSecret(words []string) ([]byte, error)` — inverse
- `SelectConfirmationWords(words []string) (indices [2]int)` — returns 2 random word indices for the production confirmation gate

**[BLOCKED: exact function signatures pending `interface-contracts.md §5.1`]** — implement with the signatures above as placeholder; replace with the frozen signatures when `interface-contracts.md` is available.

**Doc Ref:** `arch §11` (recovery paths), `mvp.md §3.5` (`SkipMnemonicConfirm` field in `DemoProfile`)

**Definition of done:** Round-trip test: encode master_secret → 24 words → decode → same bytes. Confirmation gate: demo mode with `SkipMnemonicConfirm=true` must skip the two-word check.

---

## Milestone 3 — Erasure Coding Engine

**Goal:** RS(16,56) encodes a 4 MB segment into 56 × 256 KB shards and decodes any 16 back to the original. In demo mode, RS(3,5) encodes a segment into 5 × 256 KB shards. All parameters come from `NetworkProfile`.

**Doc Ref:** `arch §10 Stage 2` (Reed-Solomon parameters and rationale), `arch §4.1` (`klauspost/reedsolomon` rationale), `mvp.md §3.2` (demo RS parameters), `data §3 Invariant 7` (ShardSize is constant)

---

### Phase 3.1 — RS Engine

#### Session 3.1.1 — Add `klauspost/reedsolomon` dependency

**Task:** Add `klauspost/reedsolomon` to `go.mod` and pin the version. Document the pinned version in `go.sum`. Add a comment in `go.mod` noting that this dependency must be vendored if `TestDHTKeyValidatorPersists` fails after an upgrade (this policy from `arch §4` applies to `go-libp2p` but the vendoring decision pattern applies here too for any CI-required check dependency).

**[Version pin: TBD — must be resolved before M3 closes, per `arch §4`]**

**Definition of done:** `go mod tidy` succeeds. `klauspost/reedsolomon` is in `go.sum`.

#### Session 3.1.2 — Implement `internal/erasure/engine.go`

**Task:** Implement `NewEngine(profile config.NetworkProfile) (*Engine, error)` that creates an RS encoder/decoder using `profile.DataShards`, `profile.ParityShards`, and `profile.TotalShards`.

Implement:
- `EncodeSegment(data []byte) ([][]byte, error)` — takes plaintext segment (padded to `DataShards × ShardSize` bytes), returns `TotalShards` shards each of `ShardSize` bytes. The first `DataShards` shards are the systematic (identity-mapped) codewords per `arch §10 Stage 2`.
- `DecodeSegment(shards [][]byte, available []bool) ([]byte, error)` — takes `TotalShards` shards (nil for missing), returns reconstructed data. Requires at least `DataShards` non-nil shards.

`ShardSize` is `profile.ShardSize` (always 262144 — `data §3 Invariant 7`). Padding: segments smaller than `DataShards × ShardSize` must be zero-padded before encoding. The original size is stored in `files.original_size_bytes` for padding removal at decode (`data §4.3`).

**Definition of done:** Full round-trip test at both production (16/40/56) and demo (3/2/5) parameters. Test with exactly `DataShards` shards available (minimum reconstruction). Test with shards 0..`DataShards-1` missing but `DataShards..TotalShards-1` available.

#### Session 3.1.3 — Implement `internal/erasure/params.go`

**Task:** Define exported constants that document the production parameter values for cross-package reference. These are never used for conditional logic — they exist for documentation and as a sanity check in `TestProfileBothFullySpecified`. Constants: `ProductionDataShards = 16`, `ProductionParityShards = 40`, `ProductionTotalShards = 56`, `ProductionShardSize = 262144`.

**Definition of done:** Constants compile. A test asserts that `ProductionShardSize == config.ProductionProfile.ShardSize`.

---

## Milestone 4 — Provider Storage Engine (WiscKey)

**Goal:** The provider daemon can store, retrieve, and garbage-collect 256 KB chunks using the WiscKey key-value separation architecture. Write amplification is ~1.0 at 256 KB values. Single-writer goroutine invariant is verified by a deadlock-detecting test.

**Doc Ref:** `arch §16` (provider storage engine — full spec), `arch §4.1` (RocksDB + vLog rationale), `data §3 Invariant 7` (vLog entry size: 262,212 bytes), `arch §4` (RocksDB CGo build note)

---

### Phase 4.1 — RocksDB Index

**Doc Ref:** `arch §16` (chunk index: `chunk_id (32 bytes) → vlog_offset uint64, chunk_size uint32`), `arch §16` (Bloom filter — 10 bits/key, ~1% FP rate)

#### Session 4.1.1 — Configure RocksDB CGo dependency

**Task:** Add `linxGnu/grocksdb` to `go.mod`. Configure the CI Docker image with the pinned `librocksdb` shared library pre-installed. Document the exact RocksDB version and Docker image tag in `.github/workflows/ci.yml` and confirm they match the Go binding version — a mismatch is a link-time failure, not runtime. **[Version pin: TBD — must be resolved before M4 closes, per `arch §4.1`]**

**Definition of done:** `go build ./internal/storage` succeeds on the CI Docker image.

#### Session 4.1.2 — Implement `internal/storage/index.go`

**Task:** Implement a RocksDB wrapper with:
- `InsertChunkIndex(chunkID [32]byte, vlogOffset uint64, chunkSize uint32) error`
- `LookupChunkIndex(chunkID [32]byte) (vlogOffset uint64, chunkSize uint32, exists bool, err error)`
- `DeleteChunkIndex(chunkID [32]byte) error`

Configure the RocksDB column family with a Bloom filter of 10 bits per key (~1% FP rate) per `arch §16`. The Bloom filter eliminates disk I/O for audit challenges on unassigned chunks — a challenge for a non-existent chunk must be answerable from memory alone.

**Definition of done:** Insert-then-lookup round-trip test. Bloom filter test: 10,000 lookups for non-existent keys produce zero disk reads (verified via RocksDB stats). Delete test.

---

### Phase 4.2 — Value Log (vLog)

**Doc Ref:** `arch §16` (vLog spec: fixed-size entries, single writer, content_hash verification), `data §3 Invariant 7` (vLog entry layout: 32+4+262144+32 = 262,212 bytes)

#### Session 4.2.1 — Implement `internal/storage/vlog.go`

**Task:** Implement the append-only vLog with the fixed entry format from `arch §16`:

```
chunk_id     (32 bytes)  — copy for GC validation
chunk_size   (4 bytes)   — always 262,144 in V2
chunk_data   (262,144 bytes)
content_hash (32 bytes)  — SHA-256(chunk_data), verified on every read
```

Functions:
- `AppendChunk(chunkID [32]byte, data [262144]byte) (vlogOffset uint64, err error)` — must be called only from the single writer goroutine
- `ReadChunk(vlogOffset uint64) (chunkID [32]byte, data [262144]byte, err error)` — verifies `SHA-256(data) == stored content_hash` before returning; returns `ErrContentHashMismatch` on failure
- `RecoverFromCrash() error` — scans from the last known head pointer in RocksDB and re-inserts any entries not yet indexed

All appends serialised through one goroutine via a buffered channel (per `arch §16`: "POSIX O_APPEND atomicity does not hold for writes above ~4 KB; for 262 KB entries, serialisation is required").

**Definition of done:** Write-then-read round-trip. Content hash corruption test returns `ErrContentHashMismatch`. Single-writer test: 100 concurrent goroutines submit writes — no data corruption, no deadlock (uses `-race` detector).

#### Session 4.2.2 — Implement `TestSingleWriterGoroutine` in `internal/storage/single_writer_test.go`

**Task:** Implement the required test per `mvp.md §8.2`. Launch 100 goroutines simultaneously calling `AppendChunk`. Verify: (1) all writes complete without error, (2) RocksDB index offsets are consistent with vLog entries, (3) the `-race` detector reports no races. The test must run under `go test -race`.

**Definition of done:** Test passes under `-race`. This test is the CI gate for single-writer correctness.

---

### Phase 4.3 — ChunkStore Interface & Crash Recovery

**Doc Ref:** `arch §16` (crash recovery spec), `arch §16` (audit lookup path — 7 steps)

#### Session 4.3.1 — Implement `internal/storage/store.go`

**Task:** Define the `ChunkStore` interface and implement `NewChunkStore(dataDir string, profile config.NetworkProfile) (*ChunkStore, error)`.

The interface (to be frozen against `interface-contracts.md` when available — **[BLOCKED: pending `interface-contracts.md`]**):
```go
type ChunkStore interface {
    Store(chunkID [32]byte, data [262144]byte) error
    Retrieve(chunkID [32]byte) ([262144]byte, error)  // verifies content_hash
    Exists(chunkID [32]byte) bool                      // Bloom filter only, no disk I/O
    Delete(chunkID [32]byte) error
    RecoverFromCrash() error
    Close() error
}
```

On startup, `RecoverFromCrash()` must be called before the store is used. The daemon's `main.go` must call it and handle the error.

**Definition of done:** Full audit lookup path test (7 steps from `arch §16`): receive challenge → Bloom filter (absent: return FAIL) → RocksDB lookup → vLog read → content hash verify → response computed. Times for SSD (<1 ms) and HDD (<15 ms) verified by benchmark.

---

## Milestone 5 — P2P Network Layer

**Goal:** Provider daemons can discover each other, establish authenticated QUIC connections, traverse NAT (DCUtR in demo; Circuit Relay v2 in production), and perform chunk-address DHT lookups using privacy-preserving HMAC keys.

**Doc Ref:** `arch §13` (P2P transfer layer), `arch §4.1` (libp2p rationale), `arch §5` (system context — libp2p/QUIC), `arch §9.5` (DHT privacy via HMAC keys), `mvp.md §3.1` (demo: 0 relay nodes, all local)

> **[BLOCKED: libp2p version pin TBD — must be resolved before M5 closes, per `arch §4`]**
> **[BLOCKED: go-libp2p vendoring decision pending `TestDHTKeyValidatorPersists` — per `arch §4`]**
> **[BLOCKED: full wire format contracts pending `interface-contracts.md`]**

---

### Phase 5.1 — libp2p Host & Transports

**Doc Ref:** `arch §13` (QUIC v1 primary, TCP+Noise+yamux fallback), `arch §4.1` (QUIC rationale — connection migration, independent streams)

#### Session 5.1.1 — Implement `internal/p2p/host.go`

**Task:** Create a libp2p host with:
- QUIC v1 (RFC 9000) as primary transport
- TCP + Noise XX + yamux as fallback transport
- Ed25519 identity (loaded from keystore at startup)
- 0-RTT session resumption **disabled** for all protocols with the suffix `/audit` or `/receipt` — per `arch §13` ("0-RTT disabled for audit interactions") and `mvp.md §5.4` (0-RTT disabled for signed receipts). 0-RTT may be enabled for pure chunk transfer protocols.

The libp2p Peer ID is `multihash(public_key)` per `arch §13`.

**Definition of done:** Host starts, generates a Peer ID, and accepts incoming connections. Connection established between two hosts on localhost using QUIC. Fallback TCP connection established when QUIC is blocked (simulated).

#### Session 5.1.2 — Implement NAT traversal in `internal/p2p/nat.go`

**Task:** Configure AutoNAT (Tier 1), DCUtR (Tier 2), and Circuit Relay v2 registration (Tier 3) per `arch §13` (three-tier NAT traversal table).

DCUtR configuration: `maxHolePunchRetries = 1` (not the libp2p default of 3). Justification: `arch §13` ("97.6% of successes happen on the first attempt — retry count set to 1").

In demo mode (`profile.MinRelayNodes == 0`): Circuit Relay v2 registration is skipped. The relay stack code is compiled in but the relay registration step is gated on `len(profile.RelayAddrs) > 0`. **[BLOCKED: relay configuration fields pending `interface-contracts.md` — add `RelayAddrs []string` to `NetworkProfile` if not already present]**

**Definition of done:** AutoNAT classifies a localhost connection as "publicly reachable". DCUtR hole-punch configured with `maxHolePunchRetries = 1`. Test verifies the retry constant is 1.

---

### Phase 5.2 — Kademlia DHT with HMAC Key Validator

**Doc Ref:** `arch §13` (DHT configuration table), `arch §9.5` (HMAC-pseudonymised DHT keys), `mvp.md §8.2` (DHT namespace constant), `arch §4.1` (libp2p rationale — custom DHT key validator)

#### Session 5.2.1 — Implement `internal/p2p/dht_namespace.go`

**Task:** Define the DHT namespace constant. This is the sole definition of this string in the entire repository:
```go
const dhtKeyNamespace = "/vyomanaut/dht-key/1.0.0"
```
Do not duplicate this string anywhere else. All DHT key construction must import this constant.

**Definition of done:** Constant defined. Grep confirms it appears exactly once in the codebase.

#### Session 5.2.2 — Implement the custom DHT key validator in `internal/p2p/dht.go`

**Task:** Implement the HMAC-SHA256 DHT key validator. DHT lookup keys use:
```
dht_key = HMAC-SHA256(chunk_hash, file_owner_key)
where file_owner_key = HKDF(master_secret, "vyomanaut-dht-v1", file_id)
```
per `arch §9.5` and `arch §13`.

The validator must implement the `go-libp2p-kad-dht` `Validator` interface for the namespace `dhtKeyNamespace`. The validator must survive every `go-libp2p` upgrade — this is the trigger for `TestDHTKeyValidatorPersists`.

Also implement DHT republication: `PublishChunkAddress(chunkID [32]byte, fileOwnerKey []byte)` and `LookupChunkAddress(chunkID [32]byte, fileOwnerKey []byte) ([]peer.AddrInfo, error)`. Republication interval: `profile.DHTRepublishInterval`. Record expiry: `profile.DHTExpiryDuration`. The `dht_key` for each chunk is cached in RocksDB alongside `vlog_offset` so republication does not require the data owner to be online.

**Definition of done:** Round-trip DHT publish and lookup on a local 3-node DHT. HMAC key is non-reversible without the `file_owner_key` (verified by a test that shows two different keys produce different DHT addresses for the same chunk).

#### Session 5.2.3 — Implement `TestDHTKeyValidatorPersists` in `internal/p2p/dht_test.go`

**Task:** Implement the CI required check (`mvp.md §8.4`, check #5). The test:
1. Creates a DHT key using the custom validator with the namespace `/vyomanaut/dht-key/1.0.0`.
2. Upgrades `go-libp2p` (or simulates the test by checking the namespace has not been reset by the validator framework).
3. Verifies the key is still valid and the namespace is unchanged.

The test must fail if any `go-libp2p` upgrade silently resets the custom key namespace. This is the vendoring gate: if the test fails after a dependency upgrade, `go-libp2p` is vendored until the upstream fix lands.

**Definition of done:** CI check #5 activates and passes.

---

### Phase 5.3 — Heartbeat Protocol

**Doc Ref:** `arch §13` (heartbeat: "4-hour address update"), `arch §18` (microservice dispatches challenges using `providers.last_known_multiaddrs`, not DHT), `data §4.2` (`last_known_multiaddrs JSONB`, `last_heartbeat_ts`, `multiaddr_stale`), `mvp.md §3.4` (demo heartbeat: 30 s interval, ±5 s jitter)

#### Session 5.3.1 — Implement `internal/p2p/heartbeat.go`

**Task:** Implement the provider-side heartbeat sender. Every `profile.HeartbeatInterval` ± `profile.HeartbeatJitter`, the daemon signs and sends the current libp2p multiaddresses to the microservice endpoint `POST /api/v1/provider/heartbeat`.

The heartbeat payload must be signed with the provider's Ed25519 private key. **[BLOCKED: exact heartbeat wire format pending `interface-contracts.md`]** — use a placeholder struct until wire format is frozen.

The microservice-side handler (saves multiaddrs to `providers.last_known_multiaddrs`, updates `last_heartbeat_ts`, clears `multiaddr_stale`) is implemented in M7.

**Definition of done:** Heartbeat fires at the correct interval with jitter. Signed payload test: tampered heartbeat fails signature verification.

---

## Milestone 6 — Microservice Database Schema

**Goal:** All Postgres tables, indexes, row security policies, materialised views, and the migration checklist are applied to the dev database. The schema exactly matches `data-model.md`. The migration generator produces correct CHECK constraint bounds for both demo and production profiles.

**Doc Ref:** `data-model.md` (entire document), `arch §4` (PostgreSQL ≥ 15), `mvp.md §5.5` (schema parameterisation via `migrations/generator.go`), `data §9` (migration checklist — must be completed in full)

---

### Phase 6.1 — Migration Generator

**Doc Ref:** `mvp.md §5.5`, `data §3 Invariant 7`, `data §8.23` (profile-parameterised CHECK bounds)

#### Session 6.1.1 — Implement `migrations/generator.go`

**Task:** Implement the migration generator that emits the two profile-parameterised CHECK constraints:
```go
func GenerateInitialSchema(profile config.NetworkProfile) string {
    // Emits shard_index CHECK bound: BETWEEN 0 AND profile.TotalShards-1
    // Emits available_shard_count CHECK range: BETWEEN profile.DataShards AND profile.TotalShards
}
```
per `mvp.md §5.5`. The generator must be invoked with the active profile before any migration is applied to a new database. A `--profile=demo|prod` flag selects the profile.

Add to `data §9` migration checklist: a CI check that runs the generator for both profiles and verifies the emitted SQL parses without error.

**Definition of done:** Generator emits correct SQL for both profiles. Demo: `BETWEEN 0 AND 4`, `BETWEEN 3 AND 5`. Production: `BETWEEN 0 AND 55`, `BETWEEN 16 AND 56`.

---

### Phase 6.2 — Initial Schema Migration

**Doc Ref:** `data §4` (all table definitions), `data §5` (index catalogue), `data §6` (row security policies), `data §7` (materialised views), `data §9` (migration ordering — must follow the exact DDL order in §9)

#### Session 6.2.1 — Create `migrations/001_initial_schema.sql`

**Task:** Write the initial schema migration applying ALL tables, indexes, RSPs, and views in the exact ordering from `data §9`:
1. `CREATE EXTENSION IF NOT EXISTS btree_gist`
2. All `CREATE TYPE` statements (in dependency order)
3. All `CREATE TABLE` statements (in FK-dependency order)
4. All `CREATE INDEX` and `CREATE UNIQUE INDEX` statements — note: `idx_chunk_assignments_one_active_per_shard` is a standalone `CREATE UNIQUE INDEX`, NOT an inline table constraint (per `data §9` migration checklist)
5. All `ALTER TABLE ENABLE ROW LEVEL SECURITY` and `CREATE POLICY` statements
6. All `CREATE MATERIALIZED VIEW` and their unique indexes

Every nullable column must match its justification in `data §8`. Every CHECK constraint must match its table definition in `data §4`. The challenge_nonce constraint must be `octet_length = 33` (not 32) per `data §3 Invariant 5`.

**Definition of done:** `psql -f migrations/001_initial_schema.sql` succeeds on a fresh Postgres 15 instance. All 35+ checklist items from `data §9` pass. CI check #7 (migration apply + rollback) activates.

#### Session 6.2.2 — Verify all 7 Invariants from `data §3`

**Task:** For each of the 7 invariants in `data §3`, write a test or a psql verification command that confirms the invariant is enforced at the database level:
1. `audit_receipts` INSERT-only → attempt UPDATE of a non-PENDING row → must fail
2. `escrow_events` INSERT-only → attempt UPDATE → must fail
3. No physical deletion of `providers` rows → attempt DELETE → must fail
4. `escrow_events.amount_paise` is BIGINT → attempt INSERT with fractional paise → must fail
5. `challenge_nonce` is 33 bytes → attempt INSERT with 32-byte nonce → must fail
6. Real shard on vetting provider → attempt INSERT of `is_vetting_chunk=FALSE` on VETTING provider → must fail at application layer
7. `ShardSize` compile-time constant → `TestProfileShardSizeIsConstant` already covers this

**Definition of done:** All 7 invariants are enforced. Each has a failing test that demonstrates the enforcement.

---

### Phase 6.3 — Materialised View Lifecycle

**Doc Ref:** `data §7`, `mvp.md §5.5` (mv_provider_scores dropped and recreated at startup from profile scoring windows), `mvp.md §6.3 OR-04`

#### Session 6.3.1 — Implement view regeneration at microservice startup

**Task:** The `mv_provider_scores` view is dropped and recreated at microservice startup using the active profile's scoring window intervals. Implement `RegenerateScoringView(db *sql.DB, profile config.NetworkProfile) error` in the microservice startup sequence. The view SQL is generated from `profile.ScoreWindowShort`, `profile.ScoreWindowMedium`, `profile.ScoreWindowLong`. The view must include `NOW() AS scores_as_of` per `data §7` and `mvp.md §6.3 OR-04`.

**Definition of done:** Starting the microservice with `DemoProfile` generates the view with 2-minute, 6-minute, 20-minute intervals. Starting with `ProductionProfile` generates the view with 24-hour, 7-day, 30-day intervals. Verified by `\d+ mv_provider_scores`.

---

## Milestone 7 — Provider Daemon: Registration & Heartbeat

**Goal:** A provider daemon can register with the microservice (creating a DB row, initiating Razorpay account setup in mock mode), receive its first heartbeat acknowledgement, and advance to `VETTING` status.

**Doc Ref:** `arch §12` (provider lifecycle — joining), `data §4.2` (`providers` table), `arch §18` (microservice API — `/api/v1/provider/register`, `/api/v1/provider/heartbeat`), `mvp.md §3.3` (demo readiness gate), `mvp.md §3.6` (demo timeline — T+00:00 to T+00:30)

> **[BLOCKED: exact request/response wire formats pending `interface-contracts.md` and `openapi.yaml`]** — implement with placeholder structs; replace with frozen contracts when docs are available.

---

### Phase 7.1 — Provider Registration

**Doc Ref:** `arch §12` (registration step), `data §4.2` (providers table constraints — `phone_number UNIQUE`, `ed25519_public_key CHECK(32 bytes)`, `status DEFAULT 'PENDING_ONBOARDING'`)

#### Session 7.1.1 — Implement `POST /api/v1/provider/register` handler

**Task:** Implement the microservice endpoint that:
1. Validates OTP-verified phone number (mock OTP in demo mode)
2. Checks `phone_number` uniqueness against `providers.phone_number`
3. Inserts a `providers` row with `status = 'PENDING_ONBOARDING'`
4. Calls `PaymentProvider.CreateLinkedAccount(providerID)` — mock in demo, Razorpay in prod
5. Sets `razorpay_cooling_until = NOW() + profile.RazorpayCoolingPeriod`

**Demo behaviour:** `RazorpayCoolingPeriod = 0` so `razorpay_cooling_until = NOW()` (instant cooling).

**Definition of done:** POST to `/api/v1/provider/register` creates a DB row. Status is `PENDING_ONBOARDING`. `razorpay_cooling_until` is set correctly for both demo and production profiles.

#### Session 7.1.2 — Implement `POST /api/v1/provider/heartbeat` handler

**Task:** Implement the microservice endpoint that:
1. Verifies the Ed25519 signature on the heartbeat payload
2. Extracts the provider's current libp2p multiaddresses from the payload
3. Updates `providers.last_known_multiaddrs` (JSONB array of multiaddr strings)
4. Updates `providers.last_heartbeat_ts = NOW()`
5. Clears `providers.multiaddr_stale = FALSE`
6. If `status = 'PENDING_ONBOARDING'` AND cooling period has elapsed → advances to `VETTING`

**Doc Ref:** `data §4.2` (`last_known_multiaddrs JSONB`, `last_heartbeat_ts`, `multiaddr_stale`), `arch §12` ("Status advances to VETTING on first heartbeat")

**Definition of done:** Heartbeat from a `PENDING_ONBOARDING` provider with elapsed cooling advances it to `VETTING`. Stale address flag is cleared. Invalid signature returns HTTP 401.

---

### Phase 7.2 — Readiness Gate

**Doc Ref:** `arch §8` (minimum viable network — 7 conditions), `mvp.md §3.3` (demo thresholds), `data §4.2` (`asn` field, `region` field)

#### Session 7.2.1 — Implement `GET /api/v1/admin/readiness` endpoint

**Task:** Implement the readiness gate check that evaluates all 7 conditions from `arch §8` against the active `NetworkProfile` thresholds:
1. Active vetted providers ≥ `profile.MinActiveProviders`
2. Distinct ASNs ≥ `profile.MinDistinctASNs`
3. Distinct metro regions ≥ `profile.MinMetroRegions`
4. Quorum state (skipped in demo if `!profile.RequireQuorum`)
5. Linked accounts with cooling elapsed ≥ `profile.MinCooledAccounts`
6. Relay nodes ≥ `profile.MinRelayNodes`
7. Cluster audit secret loaded on all replicas (skipped in demo)

Returns HTTP 200 with a JSON summary of all conditions and their pass/fail state. Returns HTTP 503 if any condition fails. Upload requests in M8 must check this gate before proceeding.

**Doc Ref:** `arch §8` ("Upload requests return HTTP 503 until all conditions are met"), `mvp.md §3.3` (demo thresholds: 5 providers, 5 ASNs, 1 region, etc.)

**Definition of done:** Gate passes with 5 demo providers correctly registered with distinct `SIM-AS1..5` ASNs. Gate fails if any condition is unmet.

---

## Milestone 8 — Data Owner Client: Encode & Upload

**Goal:** A data owner client can register, derive a master secret, encode a file through the 4-stage AONT-RS pipeline, and upload 56 (or 5) shards directly to providers over libp2p. The pointer file is encrypted and registered with the microservice.

**Doc Ref:** `arch §10` (data encoding pipeline — all 4 stages), `arch §11` (key hierarchy), `arch §5` (system context — data plane never touches microservice), `data §4.3` (files table), `data §4.4` (segments table), `data §4.5` (chunk_assignments table), `mvp.md §3.6` (demo timeline — T+01:00 upload)

---

### Phase 8.1 — Data Owner Registration

**Doc Ref:** `arch §11` (master secret derivation), `data §4.1` (owners table)

#### Session 8.1.1 — Implement `internal/client/account/register.go`

**Task:** Implement `cmd/client register` subcommand:
1. Prompt for phone number + OTP verification
2. Prompt for passphrase
3. Call `DeriveMasterSecret(passphrase, ownerID, profile)` — the master secret never leaves this function's stack frame
4. Generate Ed25519 key pair
5. Derive keystore key via `DeriveKeystoreKey(masterSecret)`
6. Encrypt private key + pointer nonce counter into keystore file on disk
7. Call `MasterSecretToMnemonic` and display mnemonic
8. If `!profile.SkipMnemonicConfirm`: gate on two-word confirmation
9. POST to microservice to create the `owners` row

**Doc Ref:** `mvp.md §3.5` (`SkipMnemonicConfirm = true` in demo), `arch §11` (recovery paths)

**Definition of done:** `register` command completes in demo mode without mnemonic confirmation prompt. Keystore file created. Master secret never written to disk (verified by test that checks no secret bytes appear in the keystore file).

---

### Phase 8.2 — Upload Orchestrator

**Doc Ref:** `arch §10` (4-stage pipeline), `arch §10 Stage 3` (pointer file creation), `data §4.3`, `data §4.4`, `data §4.5`

#### Session 8.2.1 — Implement segmentation in `internal/client/upload/orchestrator.go`

**Task:** Implement file segmentation: split the input file into segments of `DataShards × ShardSize` bytes (`profile.DataShards × 262144`). Zero-pad the last segment if necessary. Store `original_size_bytes` for later padding removal. Each segment is processed independently through Stages 1–4.

**Definition of done:** A 5 MB file with `DemoProfile` (DataShards=3, ShardSize=262144 = 786,432 bytes per segment) produces `ceil(5MB / 786KB) = 7` segments.

#### Session 8.2.2 — Implement AONT + RS pipeline per segment

**Task:** For each segment, execute Stages 1 and 2 from `arch §10`:
1. Stage 1: AONT encode → `(s+1)` words, with K embedded
2. Stage 2: RS encode → `TotalShards` shards each of `ShardSize` bytes

Both operations use `profile.DataShards`, `profile.ParityShards`, `profile.TotalShards`.

**Definition of done:** Full encode-then-decode round-trip for a multi-segment file. Canary verified after decode.

#### Session 8.2.3 — Implement assignment request and parallel upload

**Task:** Call `POST /api/v1/upload/assign` to get `TotalShards` provider assignments from the microservice. Then upload each shard to its assigned provider over libp2p in parallel. Each upload is one QUIC stream.

**[BLOCKED: upload assignment request/response format pending `interface-contracts.md`]**

The assignment service enforces the 20% ASN cap at assignment time — do not re-implement this in the client. The client simply uses the assignments returned by the microservice.

Check the readiness gate (`GET /api/v1/admin/readiness`) before attempting upload. Return a clear error if the gate is not passing.

**Definition of done:** File uploads to 5 demo providers. All 5 libp2p connections established. All shards transferred.

#### Session 8.2.4 — Implement pointer file construction and registration

**Task:** After all shards are confirmed uploaded:
1. Construct the pointer file struct: 56 provider IDs, 56 chunk content addresses (SHA-256 of each shard), erasure parameters
2. Derive pointer key: `DerivePointerKey(masterSecret, fileID)`
3. Encrypt with `EncryptPointerFile`
4. POST the three fields to microservice: `pointer_ciphertext`, `pointer_nonce`, `pointer_tag` — matching `data §4.3`
5. Create a `segments` row and `chunk_assignments` rows in the DB (via microservice API)

**Doc Ref:** `arch §10 Stage 3`, `data §4.3` (files table schema), `data §4.5` (chunk_assignments — `is_vetting_chunk=FALSE`, `segment_id NOT NULL`, `shard_index NOT NULL`)

**Definition of done:** `files` row created. `segments` row created. `chunk_assignments` rows created with correct `shard_index` (0..`TotalShards-1`). `is_vetting_chunk = FALSE` for all.

#### Session 8.2.5 — Implement upload crash recovery in `internal/client/upload/session.go`

**Task:** Implement the upload session crash recovery per `mvp.md §8.2` (session.go spec): persist `file_id`, `chunk_ids`, `ack_status[TotalShards]` to disk during the upload. On restart, detect an incomplete session file and resume from the last confirmed shard.

**Definition of done:** Kill the process mid-upload. Restart. Upload completes without re-sending already-confirmed shards.

---

## Milestone 9 — Audit System

**Goal:** The microservice issues one audit challenge per active chunk per `profile.PollingInterval`. Providers respond within their RTO. The microservice writes receipts using the two-phase write protocol. All 7 invariants in `data §3` are maintained under concurrent writes.

**Doc Ref:** `arch §14` (audit system — full spec), `data §4.7` (audit_receipts table), `data §6` (row security policies), `arch §9.5` (security boundary — nonce replay prevention), `mvp.md §3.4` (demo audit interval: 2 minutes)

---

### Phase 9.1 — Challenge Generation & Dispatch

**Doc Ref:** `arch §14` (challenge issuance), `arch §18` (cluster audit secret), `data §4.7` (`challenge_nonce BYTEA(33)` — `data §3 Invariant 5`)

#### Session 9.1.1 — Implement `internal/audit/challenge.go`

**Task:** Implement `GenerateChallenge(serverSecret []byte, chunkID [32]byte, serverTs time.Time) ChallengeNonce` where:

```
challenge_nonce = version_byte (1 byte) || HMAC-SHA256(server_secret_vN, chunk_id || server_ts)
```

The result is always 33 bytes. The version byte identifies the `server_secret` version used for cross-replica validation (`arch §18`, `arch §14`).

**Critical (from `data §3 Invariant 5`):** The CHECK constraint is `octet_length(challenge_nonce) = 33`. A 32-byte nonce silently breaks cross-replica validation after failover. CI grep check #8 (`mvp.md §8.4`) must catch any `BYTEA(32)` in challenge context.

**Definition of done:** Output is always 33 bytes. Unit test with known HMAC vector. The CI grep check fires and fails on a `BYTEA(32)` string (verified manually during this session).

#### Session 9.1.2 — Implement per-provider RTO in `internal/scoring/rto.go`

**Task:** Implement EWMA-based per-provider RTO calculation:
```
RTO = avg_rtt_ms + 4 × var_rtt_ms
```
New providers: substitute pool median (below `rto_sample_count < 5`). Update EWMA after each PASS receipt per `arch §14` ("per-provider RTO = AVG + 4 × VAR of recent audit response latencies").

Also implement `p95_throughput_kbps` EWMA update — used for JIT detection deadline: `(chunk_size_kb / p95_throughput_kbps) × 1.5`.

**Definition of done:** RTO test: a provider with 5 samples uses per-provider formula. Provider with 4 samples uses pool median. EWMA convergence test.

#### Session 9.1.3 — Implement challenge scheduler loop

**Task:** Implement the audit challenge dispatch loop: once per `profile.PollingInterval`, for every `ACTIVE` chunk_assignment row, dispatch a challenge to the provider at their `last_known_multiaddrs` (fall back to DHT if `multiaddr_stale = TRUE`).

**Doc Ref:** `arch §14` ("microservice dispatches challenges using the address from `providers.last_known_multiaddrs`... DHT is fallback if no heartbeat within 8 hours")

Track `address_was_stale` on each challenge (used in `data §4.7` for `address_was_stale BOOLEAN` — TIMEOUTs from stale addresses do NOT reset `consecutive_audit_passes`, per `data §4.2` comment on that column).

**Definition of done:** In demo mode, challenges fire every 2 minutes for all active chunks. Stale-address fallback to DHT activates correctly.

---

### Phase 9.2 — Provider Audit Response

**Doc Ref:** `arch §14` (provider response — 5 steps), `arch §16` (audit lookup path — 7 steps), `data §4.7` (response_hash, provider_sig)

#### Session 9.2.1 — Implement provider-side challenge handler

**Task:** Implement the daemon-side challenge response per `arch §14`:
1. Lookup `chunk_id` in RocksDB. If absent → return FAIL immediately (no disk I/O)
2. Read chunk from vLog using offset
3. Verify `SHA-256(chunk_data) == stored content_hash`. If wrong → return FAIL with `ErrContentHashMismatch` error code
4. Compute `response_hash = SHA-256(chunk_data || challenge_nonce)`
5. Sign the receipt with the provider's Ed25519 private key
6. Return signed audit receipt to microservice

**Definition of done:** Full challenge-response cycle works between a test microservice and a test provider. Content hash corruption returns FAIL with correct error code.

---

### Phase 9.3 — Two-Phase Receipt Write

**Doc Ref:** `arch §14` (crash-safe receipt writing — 4 steps), `data §4.7` (audit_receipts schema), `data §3 Invariant 1` (INSERT-only), `data §6` (row security policies)

#### Session 9.3.1 — Implement `internal/audit/receipt.go`

**Task:** Implement the two-phase write protocol from `arch §14`:

Phase 1 (`WriteReceiptPhase1`): INSERT a PENDING row (`audit_result = NULL`) with the full `provider_sig`. This INSERT is durable before any further processing.

Phase 2 (`WriteReceiptPhase2`): Verify the Ed25519 signature. UPDATE the row: set `audit_result`, `service_sig`, `service_countersign_ts`. Return the countersignature to the provider.

The only permitted UPDATE on `audit_receipts` is the PENDING→final transition per `data §3 Invariant 1`. The row security policy `audit_receipts_phase2_update` enforces this at the database level.

**JIT flag evaluation:** After setting `audit_result`, evaluate the JIT flag:
```
jit_flag = (response_latency_ms < (chunk_size_kb / p95_throughput_kbps) × 0.3)
```
per `arch §14` and `arch §20` (JIT defence).

**Idempotent retry:** If the provider retries the same `challenge_nonce`, detect the duplicate unique constraint violation and return the existing countersignature.

**Definition of done:** Two-phase write test: simulate a crash between Phase 1 and Phase 2 → verify the orphaned PENDING row exists and is GC'd within `profile.PendingReceiptGCAge`. Idempotent retry test.

#### Session 9.3.2 — Implement pending receipt GC in `internal/audit/` (background)

**Task:** Background GC process: every `profile.PendingReceiptGCAge / 2`, scan for PENDING rows older than `profile.PendingReceiptGCAge` and mark them `abandoned_at = NOW()` per `data §4.7`. The row security policy `audit_receipts_gc_abandon` governs this UPDATE.

ABANDONED rows are excluded from all scoring window queries (`WHERE abandoned_at IS NULL`).

**Definition of done:** GC test: insert a PENDING row, wait `profile.PendingReceiptGCAge` (use demo value of 5 minutes), verify `abandoned_at` is set.

---

## Milestone 10 — Provider Lifecycle: Vetting & ACTIVE Transition

**Goal:** Synthetic vetting chunks are assigned to VETTING providers, providers accumulate consecutive audit passes, and transition to ACTIVE after `VettingMinPasses` AND `VettingMinDuration`. The GC instruction delivers the synthetic chunk IDs to the daemon. In demo mode, this lifecycle completes in ~10 minutes.

**Doc Ref:** `arch §12` (vetting period spec), `arch §20` (synthetic vetting chunks — ADR-030), `data §4.5` (chunk_assignments — `is_vetting_chunk=TRUE`, `segment_id=NULL`, `shard_index=NULL`), `data §3 Invariant 6`, `mvp.md §3.6` (demo timeline — T+00:30 to T+10:30)

---

### Phase 10.1 — Synthetic Chunk Assignment

**Doc Ref:** `arch §12` (vetting period — synthetic chunks, 10% cap), `data §4.5` (CHECK constraint: `segment_id IS NULL AND shard_index IS NULL` when `is_vetting_chunk=TRUE`), `data §6` (row security policy — assignment service must check provider status before INSERT)

#### Session 10.1.1 — Implement synthetic chunk generation and assignment

**Task:** When a provider in VETTING status is eligible for chunk assignment:
1. Generate a random 256 KB block (the synthetic vetting chunk)
2. Compute `content_hash = SHA-256(block)`
3. Send the block to the provider via libp2p — provider stores it identically to a real shard
4. INSERT a `chunk_assignments` row with `is_vetting_chunk=TRUE`, `segment_id=NULL`, `shard_index=NULL`
5. Cap: total synthetic chunks ≤ `floor(declared_storage_gb × 400)` (10% of declared storage per `arch §12`)

**Doc Ref:** `data §6` (the assignment service MUST check `providers.status = 'VETTING'` before INSERT and must set `is_vetting_chunk = TRUE` — described in the row security note)

**Definition of done:** Synthetic chunk inserted with correct NULL fields. Cap enforced: a provider with `declared_storage_gb = 10` gets at most 4,000 synthetic chunks. `data §3 Invariant 6` test: attempt to INSERT `is_vetting_chunk=FALSE` for a VETTING provider → must fail at application layer.

---

### Phase 10.2 — ACTIVE Transition

**Doc Ref:** `arch §12` ("After 80 consecutive audit passes AND 120 days since first chunk assignment → ACTIVE"), `mvp.md §5.4` (toggle map — `VettingMinPasses` and `VettingMinDuration`), `data §4.2` (`consecutive_audit_passes`, `first_chunk_assignment_at`)

#### Session 10.2.1 — Implement consecutive pass counter and ACTIVE transition check

**Task:** In `internal/scoring/passes.go`:
- `IncrementConsecutivePasses(providerID)` — increments `consecutive_audit_passes`; checks `profile.VettingMinPasses` AND `(NOW() - first_chunk_assignment_at) >= profile.VettingMinDuration` → if both true: advance to ACTIVE
- `ResetConsecutivePasses(providerID)` — resets to 0 on any non-PASS result

TIMEOUT with `address_was_stale=TRUE` does NOT reset consecutive passes per `data §4.2` comment.

**Definition of done:** Demo: 5 consecutive passes AND 5 minutes elapsed → status becomes ACTIVE. TIMEOUT from stale address does not reset counter. Non-stale TIMEOUT resets counter.

#### Session 10.2.2 — Implement vetting GC instruction delivery

**Task:** On ACTIVE transition:
1. Query all `chunk_assignments WHERE provider_id = $1 AND is_vetting_chunk = TRUE AND status = 'ACTIVE'`
2. Send the list of `chunk_id`s to the provider via the `/vyomanaut/vetting-gc/1.0.0` libp2p protocol
3. Provider deletes these from its vLog
4. Microservice marks the rows `status = 'PENDING_DELETION'` then `'DELETED'`

**[BLOCKED: `/vyomanaut/vetting-gc/1.0.0` wire format pending `interface-contracts.md`]**

Use the retry backoff from `profile.GCRetryBackoff` on delivery failure.

**Doc Ref:** `arch §12` ("On the ACTIVE transition the microservice sends a GC instruction... listing all synthetic chunk IDs"), `mvp.md §3.4` (GC retry backoff: demo `[10s, 30s, 2min]`)

**Definition of done:** GC instruction delivered. Provider deletes synthetic chunks. Rows soft-deleted in DB.

---

## Milestone 11 — Repair System

**Goal:** When a provider is detected as silently departed (absent ≥ `profile.DepartureThreshold`), the repair scheduler enqueues jobs for all affected real shards (not synthetic chunks), downloads surviving shards, reconstructs missing shards, and uploads them to replacement providers. Escrow is seized.

**Doc Ref:** `arch §15` (repair system), `arch §12` (provider exit states), `data §4.9` (repair_jobs table), `data §3 Invariant 6` (no repair jobs for vetting chunks), `mvp.md §3.6` (demo timeline — T+20:00 to T+32:00)

---

### Phase 11.1 — Departure Detection

**Doc Ref:** `arch §12` (silent departure: "absent ≥ 72 hours"), `arch §6` (72-hour threshold calibration — Bolosky's bimodal distribution), `data §4.2` (`departed_at`, `frozen`), `mvp.md §3.4` (demo departure threshold: 10 minutes)

#### Session 11.1.1 — Implement departure detector in `internal/repair/departure.go`

**Task:** Background loop: every `profile.HeartbeatInterval / 2`, scan `providers WHERE status = 'ACTIVE' AND last_heartbeat_ts < NOW() - profile.DepartureThreshold`. For each match:
1. Set `status = 'DEPARTED'`, `departed_at = NOW()`, `frozen = TRUE` in a single transaction
2. Seize escrow (SEIZURE event in `escrow_events`)
3. For all `chunk_assignments WHERE provider_id = $1 AND is_vetting_chunk = FALSE AND status = 'ACTIVE'` → enqueue repair jobs (**NOT** for `is_vetting_chunk = TRUE` rows — `data §3 Invariant 6`)
4. Set those `chunk_assignments.status = 'REPAIRING'`

**Doc Ref:** `arch §15` ("Vetting chunk exclusion: before enqueueing any repair job, check `is_vetting_chunk`"), `arch §12` (vetting departure path — zero repair jobs enqueued)

**Definition of done:** Demo: kill a provider daemon. After 10 minutes, the detector marks it DEPARTED. Escrow seized. Repair jobs enqueued only for non-vetting chunks.

---

### Phase 11.2 — Repair Executor

**Doc Ref:** `arch §15` (what repair does), `data §4.9` (repair_jobs priority ordering), `arch §4.1` (repair priority: EMERGENCY > PERMANENT_DEPARTURE > PRE_WARNING)

#### Session 11.2.1 — Implement repair scheduler and executor in `internal/repair/`

**Task:** Implement:
- `EnqueueJob(chunkID, segmentID, triggerType, availableShardCount)` — pre-condition: `is_vetting_chunk` must be FALSE; function must check this (`data §3 Invariant 6`)
- `DequeueNextJob()` — `ORDER BY priority ASC, created_at ASC` (EMERGENCY first, then PERMANENT_DEPARTURE, then PRE_WARNING)
- `ExecuteRepairJob(job)`:
  1. Query 16 (or `DataShards`) surviving shard holders from `chunk_assignments`
  2. Download their shards over libp2p
  3. RS decode to reconstruct missing shards
  4. Select replacement providers (Power of Two Choices + ASN cap, same as assignment)
  5. Upload missing shards to replacements
  6. Update `chunk_assignments`: new rows for replacement providers, mark departed provider's row DELETED

**Definition of done:** Demo repair completes in < 2 minutes (from T+30 departure → T+32 repair complete). Fragment count restored to 5. File still retrievable after repair.

---

## Milestone 12 — Scoring & Payment (Mock Mode)

**Goal:** The three-window reliability score drives the monthly (or 2-minute in demo) release multiplier. Escrow events are recorded correctly. The mock PaymentProvider logs all operations. No floating-point arithmetic anywhere.

**Doc Ref:** `arch §17` (payment system), `arch §19` (reliability scoring), `data §4.8` (escrow_events), `data §4.6` (audit_periods), `data §7` (materialised views — mv_provider_scores, mv_provider_escrow_balance), `arch §4.1` (Razorpay rationale — mock in demo)

---

### Phase 12.1 — Reliability Scoring

**Doc Ref:** `arch §19`, `data §7` (mv_provider_scores), `mvp.md §5.4` (scoring window toggle), `arch §17` (release multiplier table)

#### Session 12.1.1 — Implement `internal/scoring/score.go`

**Task:** Implement score queries against `mv_provider_scores`:
- `GetScore(providerID) (score24h, score7d, score30d, scoreComposite float64)` — uses the materialised view
- `GetScoreFromPrimary(providerID) (...)` — for monthly release computation, must read from the primary replica, not a read replica (per `arch §19` comment that "monthly release multiplier must use primary")

Implement the dual-window deterioration check:
```
if score7d < score30d - 0.20 → use score7d as the release multiplier input
```
per `arch §17` ("If the 7-day score drops more than 0.20 below the 30-day score").

**Definition of done:** Score computation matches expected values from a known audit receipt sequence. Dual-window flag triggers correctly when 7d score drops 0.20 below 30d score.

---

### Phase 12.2 — Payment Mock & Escrow Ledger

**Doc Ref:** `arch §17` (payment system), `data §4.8` (escrow_events — append-only, integer paise), `data §3 Invariant 2 and 4`, `mvp.md §6.1 CR-04` (mock must enforce integer paise)

#### Session 12.2.1 — Implement `internal/payment/provider.go` and `internal/payment/mock.go`

**Task:** Define the `PaymentProvider` interface. **[BLOCKED: exact interface signature pending `interface-contracts.md`]**. Implement the mock payment provider that:
1. Logs all operations to stdout with timestamps
2. Enforces the `PaiseAmount int64` type — rejects any fractional value
3. Implements `CreateLinkedAccount`, `ReleaseEarnings`, `SeizeEscrow`, `ReverseTransfer`

**Definition of done:** All mock calls succeed. `TestNoFloatArithmetic` (CI check #9) activates: scanning `internal/payment/` for `float64|float32|FLOAT|DECIMAL|NUMERIC` must return zero results in payment context.

#### Session 12.2.2 — Implement `internal/payment/ledger.go`

**Task:** Implement escrow event insertion:
- `InsertEscrowEvent(providerID, eventType, amountPaise int64, auditPeriodID *uuid.UUID, idempotencyKey string)` — INSERT only, never UPDATE. The idempotency key for RELEASE events: `SHA-256(provider_id || audit_period)` as 64 hex chars.
- REVERSAL event: idempotency key = `SHA-256("reversal" || original_idempotency_key)` per `data §7` comment

`escrow_events.amount_paise` is `BIGINT` — the application type is `int64`. Any path that uses float arithmetic in payment context is a build-breaking CI failure.

**Definition of done:** INSERT test. Duplicate idempotency key test returns the existing event (idempotent). Balance query test: `SUM(DEPOSIT + REVERSAL) - SUM(RELEASE + SEIZURE)` returns correct paise value.

#### Session 12.2.3 — Implement release computation in `internal/payment/release.go`

**Task:** Implement the monthly (or `profile.ReleaseComputationInterval`) release computation:
1. For each active provider, get `score30d` and `score7d`
2. Apply the release multiplier table from `arch §17`
3. Apply dual-window deterioration signal
4. Compute `releasable_paise = floor(held_balance × multiplier)` — integer division, never float
5. Call `PaymentProvider.ReleaseEarnings(providerID, releasable_paise, auditPeriodID, idempotencyKey)`
6. Set `audit_periods.release_computed = TRUE`

In demo mode: triggered by `profile.ReleaseComputationInterval` ticker (2 minutes). In production: triggered on the 23rd of the month using `LastWorkingDayOfMonth` from `rbi_holidays.go`.

**Definition of done:** Demo release fires every 2 minutes after vetting completes. Multiplier table applied correctly for each score tier.

---

## Milestone 13 — Data Owner Client: Retrieval

**Goal:** A data owner can retrieve any file by downloading any `DataShards` of its `TotalShards` shards, RS-decoding, AONT-decoding (K recovered automatically), canary-verified, padding stripped.

**Doc Ref:** `arch §10 Stage 4` (decoding — full algorithm), `arch §11` (pointer file key derivation and decryption), `data §4.3` (files table — pointer fields), `arch §9.5` (K recovery from AONT package)

---

### Phase 13.1 — Pointer File Retrieval

#### Session 13.1.1 — Implement `internal/client/retrieve/pointer.go`

**Task:**
1. Fetch `pointer_ciphertext`, `pointer_nonce`, `pointer_tag` from microservice
2. Derive pointer key: `DerivePointerKey(masterSecret, fileID)`
3. Verify Poly1305 tag with `crypto/subtle.ConstantTimeCompare` before any decryption
4. Decrypt pointer ciphertext
5. Parse pointer file struct: extract provider IDs, chunk content addresses, erasure parameters

**Definition of done:** Decryption round-trip. Tampered-tag rejection before decryption. Tampered-ciphertext rejection.

---

### Phase 13.2 — Parallel Shard Download & Decode

#### Session 13.2.1 — Implement `internal/client/retrieve/download.go` and `decode.go`

**Task:**
1. Dial all `TotalShards` providers in parallel (cancel after any `DataShards` respond successfully)
2. Verify content address of each shard: `SHA-256(shard_data) == stored_chunk_id`
3. RS decode to reconstruct the AONT package
4. Recover K: `K = c_{s+1} XOR SHA-256(all codewords)` per `arch §10 Stage 4`
5. Decrypt each word to recover the segment
6. Verify canary — if fail: `ErrCanaryMismatch`, escalate to audit subsystem
7. Strip padding to `original_size_bytes`
8. Zero the decrypted buffer on canary failure (security)

**Definition of done:** Full retrieval round-trip for a multi-segment file. Retrieval with only `DataShards` providers online. Content address mismatch test. Canary failure test.

---

## Milestone 14 — Demo Integration & End-to-End Validation

**Goal:** The full lifecycle from `mvp.md §3.6` completes end-to-end on a single machine with `--sim-count=5 --sim-asn-count=5` in approximately 30–35 minutes. All 5 demo time windows are observable live.

**Doc Ref:** `mvp.md §3.6` (demo timeline — T+00:00 to T+32:00), `mvp.md §7` (viability fact-check — all 13 items must pass)

---

### Phase 14.1 — Demo Integration Test

#### Session 14.1.1 — Create end-to-end integration test script

**Task:** Create `scripts/demo_e2e_test.sh` that:
1. Starts 5 provider daemons with `--sim-count=5 --sim-asn-count=5`
2. Starts the microservice with `VYOMANAUT_MODE=demo`
3. Registers a data owner and uploads a 1 MB test file
4. Waits for vetting to complete (polls status every 10 seconds)
5. Verifies the file is retrievable
6. Kills one provider daemon (simulates departure)
7. Waits for departure detection (10 minutes in demo)
8. Verifies repair completes and file remains retrievable
9. Verifies a mock payout was logged

**Definition of done:** Script completes without errors in ≤ 35 minutes. All lifecycle transitions verified.

#### Session 14.1.2 — Verify all 13 viability items from `mvp.md §7`

**Task:** For each item in `mvp.md §7.1` through `7.13`, add an assertion to the integration test or a standalone test that confirms the mathematical consistency holds:
- 7.1: ASN cap enforcement with 5 distinct ASNs
- 7.2: RS(3,5) reconstruction math
- 7.3: Vetting timing (10 min, never violates VettingMinDuration)
- 7.7: Idempotency key uniqueness at DB level
- 7.8: Repair completes in < 2 minutes on LAN

**Definition of done:** All 13 viability checks pass. Any failing check is documented as a candidate fix.

---

## Milestone 15 — Secrets Manager & Cluster Audit Secret (Production)

**Goal:** The cluster audit secret is loaded from a secrets manager (not env var) at microservice startup. Nonce rotation works across a 24-hour window.

**Doc Ref:** `arch §18` (cluster audit secret — HKDF derivation, versioned rotation), `arch §4.1` (secrets manager rationale), `mvp.md §6.2 IR-03` (demo and prod secrets must be separate)

> **Prerequisite:** M14 complete (demo validated). This milestone is production infrastructure only.

---

### Phase 15.1 — SecretsManagerClient Interface

#### Session 15.1.1 — Implement `internal/audit/secrets_iface.go`

**Task:** Define the `SecretsManagerClient` interface. **[BLOCKED: exact interface signature pending `interface-contracts.md §8`]**. Implement three concrete providers:
- `HashiCorpVaultClient`
- `AWSSSMClient`
- `GCPSecretManagerClient`

All three support per-path versioning (`/vyomanaut/audit-secret/v{N}`) and IAM-gated access. Selected at deployment time.

**Definition of done:** Interface defined. Each implementation loads a secret from a test secrets manager.

#### Session 15.1.2 — Implement cluster secret rotation

**Task:** Implement HKDF derivation of the audit secret:
```
server_secret_v{N} = HKDF-SHA256(
    ikm  = cluster_master_seed,
    salt = cluster_id,
    info = "vyomanaut-audit-secret-v" + N,
    len  = 32
)
```
On rotation: old version accepted for 24 hours (one full audit cycle) before retirement. The `ClusterSecretCache` has a 5-minute TTL — fail-closed if the secrets manager is unreachable at startup.

**Definition of done:** Rotation test: a challenge nonce generated with `v1` secret is validated correctly by a replica that has loaded `v2` but still accepts `v1` for 24 hours.

---

## Milestone 16 — Razorpay Payment Integration (Production)

**Goal:** The `RazorpayProvider` implementation of `PaymentProvider` handles deposits (Smart Collect 2.0 webhook), release transfers (Route with `on_hold`), seizures, payouts (RazorpayX), and idempotency.

**Doc Ref:** `arch §17` (payment system — Razorpay products), `arch §4.1` (Razorpay rationale + mandatory compliance notes), `data §4.8` (escrow_events — idempotency_key), `mvp.md §6.2 IR-02` (demo and prod Razorpay credentials must be separate)

---

### Phase 16.1 — Razorpay Provider Implementation

**Doc Ref:** `arch §17` mandatory compliance notes: (1) UPI Collect deprecated 28 Feb 2026 — use UPI Intent/QR, (2) `X-Payout-Idempotency` header mandatory since 15 Mar 2025, (3) RBI bank holiday table updated each December

#### Session 16.1.1 — Implement `internal/payment/razorpay.go`

**Task:** Implement `RazorpayProvider` with:
- Smart Collect 2.0 webhook handler for UPI deposits (UPI Intent/QR — NOT UPI Collect)
- Route transfer creation with `on_hold: true`
- `Modify Settlement Hold API` to update `on_hold_until`
- `on_hold_until` set to last working day of current month via `LastWorkingDayOfMonth` from `rbi_holidays.go`
- `RazorpayX Payouts` with mandatory `X-Payout-Idempotency` header
- `payout.reversed` webhook handler: sets REVERSAL event in `escrow_events`

**CI check #11** (`mvp.md §8.4`): UPI Collect API endpoint string must never appear in the codebase.

**Definition of done:** Webhook handler test with Razorpay test credentials. Idempotency key test. `on_hold_until` computation test against RBI holiday table.

---

## Milestone 17 — Relay Nodes & Production NAT Traversal

**Goal:** Three relay nodes (one per Indian AZ) are deployed and handle Circuit Relay v2 reservations for the approximately 30% of providers behind symmetric NAT.

**Doc Ref:** `arch §13` (Circuit Relay v2 — Tier 3 NAT), `arch §24` (deployment topology — relay nodes: 1 vCPU, 1 GB RAM, minimum 1 Gbps), `arch §27.5` (relay infrastructure scaling)

---

### Phase 17.1 — Relay Node Deployment

#### Session 17.1.1 — Configure libp2p Circuit Relay v2 on relay nodes

**Task:** Run the provider daemon binary with relay mode enabled. Each relay node configured for 128 concurrent relay reservations. Deploy 3 instances (Mumbai AZ1, Mumbai AZ2, Chennai/Hyderabad) per `arch §24`.

Scale trigger per `arch §27.5`: add a 4th relay node before provider count reaches 400 (under the 45% Indian CGNAT pessimistic assumption).

**Definition of done:** Relay node starts and accepts incoming reservation requests. 128 concurrent reservation test. Provider behind simulated symmetric NAT connects via relay.

---

## Milestone 18 — Microservice HA Quorum (Production)

**Goal:** Three microservice replicas with a (3,2,2) quorum — reads require 2 replicas, writes require 2 acknowledgements. One replica can fail without interrupting service.

**Doc Ref:** `arch §18` (cluster configuration — (3,2,2) quorum, gossip membership), `arch §4.1` (quorum rationale — etcd/Consul explicitly rejected for 3-node cluster), `arch §25` (consistency model — 14 coordination-free, 6 coordinated operations)

---

### Phase 18.1 — Gossip Membership

**Doc Ref:** `arch §18` (gossip: "each replica contacts one randomly chosen peer per second", two seed node addresses)

#### Session 18.1.1 — Implement gossip membership in microservice

**Task:** Implement gossip-based cluster membership with two pre-configured seed node addresses. Each replica contacts one random peer per second to reconcile membership histories. Client-driven routing: for latency-sensitive paths, the service client caches membership and routes directly to the responsible replica, bypassing the load balancer.

**Doc Ref:** `arch §18` ("client-driven routing reduces 99.9th-percentile latency by 30+ ms compared to load-balancer indirection")

**Definition of done:** Three-replica cluster starts. One replica killed → reads and writes continue on remaining two. Killed replica rejoins and reconciles state via anti-entropy.

---

## Milestone 19 — Production Hardening & Launch Gate

**Goal:** All benchmark scripts pass on minimum-spec hardware. All 8 runbooks exist. All CI checks are active (no placeholder steps). The production readiness gate passes.

**Doc Ref:** `mvp.md §8.5` (runbooks — 8 required files), `mvp.md §8.5` (benchmark scripts — 4 required), `arch §27.7` (hard ceilings — relay saturation, repair window, Postgres audit INSERT), `arch §3` (quality attributes — performance targets)

---

### Phase 19.1 — Benchmark Validation

**Doc Ref:** `mvp.md §8.5` (scripts/benchmarks — 4 required scripts), `arch §3` (performance contracts)

#### Session 19.1.1 — Run all 4 benchmark scripts on minimum-spec hardware

**Task:** Execute `aont_encode.sh`, `argon2id.sh`, `rocksdb_ssd.sh`, `rocksdb_hdd.sh` on a machine with dual-core, no AES-NI, 2 GB RAM, 7200 RPM HDD. Verify:
- AONT encode: ≤ 186 ms for a 14 MB segment (ChaCha20, no AES-NI) — `arch §3`
- Argon2id production params: ≤ 500 ms — `arch §4.1`
- RocksDB SSD audit lookup: ≤ 1 ms p99 — `arch §16`
- RocksDB HDD audit lookup: ≤ 15 ms p99 — `arch §16`

**Definition of done:** All 4 benchmarks pass. Results recorded in `docs/research/` per the benchmarking protocol.

---

### Phase 19.2 — Runbooks

**Doc Ref:** `mvp.md §8.5` (8 required runbooks before M19 closes)

#### Session 19.2.1 — Create all 8 operational runbooks

**Task:** Create the 8 runbook files in `runbooks/`:
1. `microservice-failover.md` — (3,2,2) quorum failover steps
2. `postgres-failover.md` — managed Postgres primary failover
3. `relay-node-replacement.md` — replace a failed relay node
4. `secrets-manager-outage.md` — behaviour during secrets manager outage (fail-closed, existing replicas continue with cached secret)
5. `razorpay-api-outage.md` — payout failure handling, `payout.reversed` webhook
6. `provider-mass-departure.md` — alert thresholds, manual monitoring at N < 500
7. `rbi-holiday-table-update.md` — December update procedure for `rbi_holidays.go`
8. `audit-secret-rotation.md` — 24-hour overlap window procedure

**Definition of done:** All 8 files exist with complete step-by-step procedures.

---

### Phase 19.3 — Production Launch Checklist

#### Session 19.3.1 — Activate all CI placeholder checks

**Task:** Replace every placeholder CI step (from M0 Phase 0.2) with the actual check. By this session, all 15 CI checks from `mvp.md §8.4` must be active and passing. No `echo "TODO"` remains in `ci.yml`.

**Definition of done:** All 15 CI checks pass on a clean PR. No placeholder steps.

#### Session 19.3.2 — Run production readiness gate

**Task:** With `VYOMANAUT_MODE=prod`, verify `GET /api/v1/admin/readiness` returns HTTP 200 with all 7 conditions from `arch §8` satisfied:
1. ≥ 56 active vetted providers
2. ≥ 5 distinct ASNs
3. ≥ 3 distinct Indian metro regions
4. Full (3,2,2) quorum
5. ≥ 56 Razorpay accounts with 24h cooling complete
6. ≥ 3 relay nodes deployed
7. Cluster audit secret loaded on all replicas

**Definition of done:** Production readiness gate passes. First real upload accepted.

---

## Appendix A — Pending Interface-Contracts Integrations

The following sessions are marked `[BLOCKED: pending interface-contracts.md]`. Each must be revisited when that document is loaded. Search for this tag before beginning any session.

| Session | What is blocked | Section in interface-contracts.md expected |
|---|---|---|
| 2.6.1 | `MasterSecretToMnemonic` exact signature | §5.1 |
| 5.1.1 | Libp2p host configuration — relay addr fields | §5.2 |
| 5.3.1 | Heartbeat wire format | §4.x |
| 7.1.1 | Provider registration request/response format | §4.x |
| 7.1.2 | Heartbeat request/response format | §4.x |
| 7.2.1 | Admin readiness response JSON format | §4.x |
| 8.2.3 | Upload assignment request/response | §4.x |
| 10.2.2 | `/vyomanaut/vetting-gc/1.0.0` wire format | §4.5 |
| 12.2.1 | `PaymentProvider` interface signature | §5.x |
| 15.1.1 | `SecretsManagerClient` interface signature | §8 |

---

## Appendix B — Pending Requirements Document Integrations

The following sessions need NFR/FR cross-checks once `requirements.md` is available. Search for `[BLOCKED: requirements.md]` and `[NFR-XXX reference pending]` in the build sessions.

| Session | Referenced NFR/FR | Likely location in requirements.md |
|---|---|---|
| 2.4.1 | NFR-019 (constant-time tag comparison) | §NFR-019 |
| 2.3.1 | NFR-010 (Argon2id latency p50 ≤ 500 ms) | §NFR-010 |
| 4.3.1 | NFR-013 (write amplification), NFR-008 (audit lookup p99) | §NFR-008, §NFR-013 |
| 7.1.1 | FR-001 (OTP-verified registration) | §FR-001 |
| 8.2.4 | FR-030 (assignment ceiling enforcement) | §FR-030 |
| 10.2.1 | FR-026 (120-day vetting check) | §FR-026 |
| 13.2.1 | FR-008 (padding strip to original_size_bytes) | §FR-008 |

---

## Appendix C — Known Candidate Fixes

These are potential inconsistencies identified during document analysis. Each must be resolved during the affected milestone — not silently worked around.

| # | Location | Observation | Resolution required during |
|---|---|---|---|
| CF-01 | `data §4.2` `p95_throughput_kbps` | Changed from `NOT NULL DEFAULT 0` to `NULL` — division-by-zero risk if application reads 0 before checking for NULL. Application must substitute pool median when value is NULL. | M9 (challenge generation uses this value) |
| CF-02 | `data §4.2` `avg_rtt_ms` | Changed from `NOT NULL DEFAULT 2000` to `NULL` — hardcoded 2000ms default is no longer the initial state. Pool median must be used when NULL. | M9 |
| CF-03 | `data §3 Invariant 6` | States "must never be assigned to a provider whose `status = 'VETTING'`" for real shards — but the assignment service only checks status at INSERT time. If a provider's status changes between assignment time and the check, a race is possible. | M10 (assignment service, test for race condition) |
| CF-04 | `arch §14` / `data §4.7` | Architecture states "the microservice independently has the expected hash and verifies it" for `response_hash` — but architecture also states "the microservice never verifies the response_hash, it knows the chunk_id but not the 256 KB chunk data". These two statements contradict each other. The NOTE in `arch §14` is authoritative: the microservice does NOT verify `response_hash` — only the Ed25519 signature. Implement accordingly. | M9 |
| CF-05 | `data §9` migration checklist | Checklist item: `chunk_assignments partial unique index created as a standalone CREATE UNIQUE INDEX, NOT as an inline table constraint`. The table definition in `data §4.5` shows the partial index inline in the CREATE TABLE — this is invalid SQL syntax per the checklist note. Implement as standalone `CREATE UNIQUE INDEX`. | M6 |
| CF-06 | `data §4.9` `repair_jobs` | The removed `UNIQUE (chunk_id, provider_id, trigger_type)` constraint leaves threshold-based repairs (where `provider_id IS NULL`) without deduplication. The comment says "deduplication must be at the application layer". Implement in `EnqueueJob` as a pre-INSERT check for existing `QUEUED/IN_PROGRESS` job for the same `chunk_id`. | M11 |

---

*Repository: https://github.com/masamasaowl/Vyomanaut_Research*
*Primary build sequence derived from: `mvp.md §9.4`*
*All section references are to the three documents shared in context. References to `requirements.md`, `interface-contracts.md`, and `openapi.yaml` are marked `[BLOCKED]` and will be filled in when those documents are loaded.*