# Vyomanaut V2 — Build Procedure

**Status:** Authoritative build specification  
**Version:** 1.0  
**Date:** May 2026  
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research  
**Derived from:**
- `docs/system-design/interface-contracts.md` (IC) — wire contracts, Go package interfaces, forbidden patterns
- `docs/system-design/data-model.md` (DM) — PostgreSQL schema, invariants, indexes, row security policies
- `docs/system-design/mvp.md` (MVP) — NetworkProfile, demo/prod mode, repository layout, CI pipeline

**Awaiting (documents not yet in context):**
- `docs/system-design/architecture.md` (ARCH) — system overview, deployment topology, relay infrastructure
- `docs/system-design/requirements.md` (REQ) — FR/NFR completeness gates, capacity calculations
- `docs/system-design/api/openapi.yaml` (OAS) — authoritative REST/HTTP surface; all endpoint schemas

> **Reading convention.** Each session carries a `[REF]` tag pointing to the exact document
> section that governs it. Sessions marked `⚠️ AWAITING` cannot be completed until the
> missing document is provided; a stub must be committed that compiles and fails its own
> placeholder test. Every session is atomic: it produces a passing `go test` or a passing
> migration apply before the next session begins. Logical errors discovered during
> implementation are fixed in the current session with a note appended to the session log —
> do not silently carry forward broken invariants.

---

## Build Dependency Graph

``` tree
M0 (Setup)
 └─ M1 (Config / NetworkProfile)
     ├─ M2 (Cryptography)
     │   └─ M3 (Erasure Coding)
     ├─ M4 (Database Schema)
     └─ M5 (Storage Engine)
         └─ M6 (P2P Network Layer)
             ├─ M7 (Provider Daemon Core)
             ├─ M8 (Audit System)
             │   ├─ M9 (Scoring)
             │   └─ M10 (Repair)
             ├─ M11 (Payment)
             └─ M12 (REST API Layer)  ← ⚠️ AWAITING openapi.yaml
                 └─ M13 (Coordination Microservice)
                     └─ M14 (Vetting & Synthetic Chunks)
                         └─ M15 (Client SDK)
                             └─ M16 (Demo Mode Validation)
                                 └─ M17 (Production Hardening)  ← ⚠️ AWAITING architecture.md
                                     └─ M18 (Launch Readiness)  ← ⚠️ AWAITING requirements.md
```

The import constraint DAG (IC §9) enforces that `crypto` and `erasure` have zero internal
dependencies; all data-layer packages flow upward only to the microservice entrypoint.

---

## Milestone 0 — Project Setup & Repository Foundation

**Deliverable:** A compilable Go module with the full directory skeleton, a passing CI
pipeline skeleton, and all tooling configured. No business logic yet.

**Reference:** MVP §8.1 (directory map), MVP §8.4 (CI pipeline structure),
IC §10 (naming conventions), IC §11 (forbidden code patterns — linter rules set here)

---

### Phase 0.1 — Repository Layout

**Reference:** MVP §8.1

#### Session 0.1.1 — Initialise Go module and top-level directories

**Task:** Create the repository at `github.com/masamasaowl/Vyomanaut_Research`. Run
`go mod init github.com/masamasaowl/vyomanaut`. Create every top-level directory listed
in MVP §8.1: `cmd/`, `internal/`, `migrations/`, `deployments/`, `scripts/`, `runbooks/`,
`docs/`. Within each, place a `.gitkeep` or a stub `doc.go` so the tree is committed.

Verify: `ls` matches the tree in MVP §8.1 exactly. No extra top-level directories without
a corresponding ADR (IC §11 — no communication link not shown in the diagram).

#### Session 0.1.2 — Create `cmd/` entrypoint stubs

**Task:** Create three entrypoint packages: `cmd/microservice/main.go`,
`cmd/provider/main.go`, `cmd/client/main.go`. Each file must contain only:
1. `package main`
2. An empty `func main()` that prints the startup banner format from MVP §2.1:
   `[STARTUP] Vyomanaut <binary> v0.1.0 — mode=UNKNOWN — stub`
3. A `// TODO: wire subsystems` comment

Verify: `go build ./cmd/...` succeeds with zero warnings.

**Note:** Business logic — including mode flag parsing — is deferred to M1. These stubs
exist only to confirm the package structure compiles.

#### Session 0.1.3 — Create `internal/` package stubs

**Task:** For every package listed in MVP §8.2, create the package directory and a
`doc.go` file containing only the package declaration and a one-line doc comment matching
the description in MVP §8.2. Packages to stub: `internal/config`, `internal/crypto`,
`internal/erasure`, `internal/storage`, `internal/p2p`, `internal/audit`,
`internal/scoring`, `internal/repair`, `internal/payment`, `internal/vettingchunk`,
`internal/client/account`, `internal/client/upload`, `internal/client/retrieve`,
`internal/client/manage`.

Verify: `go build ./internal/...` succeeds. `go vet ./internal/...` produces zero output.

---

### Phase 0.2 — Toolchain & Linter Configuration

**Reference:** MVP §8.4 (linter list), IC §11 (forbidden patterns that linters enforce)

#### Session 0.2.1 — Configure `.golangci.yml`

**Task:** Create `.golangci.yml` enabling exactly the linters listed in MVP §8.4:
`gofmt`, `govet`, `errcheck`, `exhaustive`, `godot`, `gomnd`. Configure `exhaustive` to
require all cases on the types: `AuditResult`, `ProviderStatus`, `EscrowEventType`,
`RepairPriority` (these types do not exist yet; add them to the `exhaustive.check` list
by name so they are enforced once defined). Configure `gomnd` to catch magic numbers that
should be `NetworkProfile` fields (IC §11 — forbidden: hardcoded Argon2id parameters,
hardcoded shard counts).

Add a `# TODO: add type names as they are defined` comment for the `exhaustive` section.

Verify: `golangci-lint run ./...` on the stub-only repository produces zero lint errors
(stubs have no logic to trigger the configured rules).

#### Session 0.2.2 — Configure forbidden-pattern grep checks

**Task:** Create `scripts/ci/grep_checks.sh` implementing the exact grep-fail checks
listed in MVP §8.4 checks 8–11:
- Fail if `challenge_nonce BYTEA(32)` appears in any file (IC §11, DM §3 Invariant 5)
- Fail if `float64|float32|FLOAT|DECIMAL|NUMERIC` appears in `internal/payment/` context (IC §11)
- Fail if any ADR reference above ADR-031 appears (IC §11 — no non-existent ADR references)
- Fail if the UPI Collect API endpoint string appears (IC §11)

Each check must print the offending file and line on failure. The script exits 0 only if
all four checks pass.

Verify: Running the script against the current stub-only repository exits 0.

---

### Phase 0.3 — CI Pipeline Skeleton

**Reference:** MVP §8.4

#### Session 0.3.1 — Create `.github/workflows/ci.yml`

**Task:** Implement the CI workflow with all 15 checks listed in MVP §8.4, in order.
Checks that depend on code not yet written (5, 6, 7, 14, 15) must be present as steps
but guarded with `continue-on-error: false` and a placeholder Go test file that
deliberately fails with `t.Skip("not yet implemented — M1+")`. This ensures the CI
step exists and will fail loudly until the corresponding milestone is complete, rather
than being silently absent.

The migration step (check 7) must target the CI Postgres instance; document the required
`btree_gist` extension setup in `.github/workflows/ci.yml` comments (DM §9 migration
checklist first item).

Verify: The CI workflow file parses valid YAML. `act` (local GitHub Actions runner) or
manual inspection confirms all 15 steps are present.

#### Session 0.3.2 — Create `.github/CODEOWNERS`

**Task:** Create `.github/CODEOWNERS` with the rule from MVP §8.4: the `/migrations/`
directory requires three reviewers. Add placeholder ownership entries for all `internal/`
packages (two reviewers each) and `cmd/` (one reviewer). Use placeholder GitHub handles
marked `# TODO: replace with real handles`.

Verify: File parses per GitHub CODEOWNERS syntax. The `/migrations/` entry specifically
lists at least three owner handles (can be duplicates at this stage).

---

### Phase 0.4 — Development Docker Compose

**Reference:** MVP §8.5 (dev docker-compose requirements)

#### Session 0.4.1 — Create `deployments/dev/docker-compose.yml`

**Task:** Create a docker-compose file satisfying the requirements in MVP §8.5:
- One microservice replica (points to stub binary; will start and exit immediately at
  this stage)
- One Postgres 16 instance with `btree_gist` pre-installed (init SQL:
  `CREATE EXTENSION IF NOT EXISTS btree_gist;`)
- One relay node stub (can be a simple echo server at this stage; full implementation
  in M17)
- One provider daemon in `--sim-count=5 --sim-asn-count=5` mode (stub binary)

The Postgres instance must expose port 5432 and have health-check configured. The
connection string format must match what the migration runner expects.

Verify: `docker-compose up` brings up Postgres. `psql` connects. `\dx` shows
`btree_gist` installed.

---

## Milestone 1 — Configuration & NetworkProfile

**Deliverable:** `internal/config` package fully implemented, both profiles defined,
all profile-selection and guard-rail logic tested. This is the single source of truth
for all mode-variable parameters. Every subsequent milestone reads from this package.

**Reference:** MVP §2 (mode flag), MVP §5.1 (NetworkProfile struct), MVP §5.2 (profiles),
MVP §5.3 (profile selection), MVP §5.4 (toggle map), MVP §6 (switching requirements),
IC §11 (guard rails — DEMO_MODE_REAL_PAYMENT, PROD_MODE_ENV_SECRET error codes from IC §3.3)

---

### Phase 1.1 — NetworkProfile Struct

**Reference:** MVP §5.1

#### Session 1.1.1 — Define the `NetworkProfile` struct

**Task:** Create `internal/config/network_profile.go` containing the `NetworkProfile`
struct with every field listed in MVP §5.1, in the exact groupings shown (erasure coding,
readiness gate, ASN cap, time windows, scoring windows, vetting, cryptographic cost,
infrastructure, release cycle, GC retry backoff). Every field must have a comment citing
the ADR that governs it (as shown in MVP §5.1).

**Critical constraint:** `ShardSize int` must be present in the struct (for compiler
enforcement in tests) but its value must always equal the constant `262144` (DM Invariant
7). Add a compile-time assertion: `var _ = [1]struct{}{}[ShardSize-262144]` — this will
not compile if anyone sets ShardSize to a non-262144 value in a profile. Also add a
`const ShardSize = 262144` to `internal/erasure/` (set up in M3) as the canonical
definition; the profile field must match it.

Verify: `go build ./internal/config/` succeeds. The struct has exactly the fields in
MVP §5.1 — no additions, no omissions. Use `go vet` struct-literal completeness to
enforce this once profiles are defined.

#### Session 1.1.2 — Define `ProductionProfile` and `DemoProfile`

**Task:** Create `internal/config/profiles.go` with `var ProductionProfile` and
`var DemoProfile` populated with every value from MVP §5.2. Use Go struct literal
syntax with all fields explicitly set (no omitted fields — MVP §6.3 requirement OR-03:
Go compiler enforces completeness). Both literals must be on the same file so a diff
immediately shows any field present in one but not the other.

Verify: `go build` succeeds. Both structs compile without zero-value defaults silently
filling any field.

---

### Phase 1.2 — Profile Tests

**Reference:** MVP §8.2 (profiles_test.go), MVP §6.3 (OR-02, OR-03), DM §3 Invariant 7

#### Session 1.2.1 — `TestProfileShardSizeIsConstant`

**Task:** In `internal/config/profiles_test.go`, implement `TestProfileShardSizeIsConstant`
(CI check 14 in MVP §8.4). The test must assert:
- `ProductionProfile.ShardSize == 262144`
- `DemoProfile.ShardSize == 262144`
- Both equal the constant exported from `internal/erasure` (import will be added once
  M3 is complete; use a `// TODO: cross-check with erasure.ShardSize after M3` comment
  and assert the literal 262144 for now)

This test must be in CI check 14 and must block merges on failure (MVP §8.4).

#### Session 1.2.2 — `TestProfileBothFullySpecified`

**Task:** Implement `TestProfileBothFullySpecified` (CI check 15 in MVP §8.4). Use
reflection to verify that every field in `ProductionProfile` and `DemoProfile` is
non-zero (except fields where zero is explicitly the correct production value, e.g.
`ReleaseComputationInterval: 0` in ProductionProfile — these must be listed in an
explicit allowlist in the test with a comment explaining why zero is correct per
MVP §5.2).

#### Session 1.2.3 — `TestDemoDiffersFromProduction`

**Task:** Assert that demo and production profiles differ on every field listed in
MVP §3.2 through §3.5 (erasure params, time windows, crypto params). Assert they are
identical on: `ShardSize`, `ASNCapFraction`, `VettingCapFraction`, `DualWindowDrop`
(per MVP §5.1 comments that these are "never mode-variable"). This test documents the
exact boundary between what changes and what stays the same.

---

### Phase 1.3 — Profile Selection & Guard Rails

**Reference:** MVP §2.3 (guard rails), MVP §5.3 (profile selection), IC §3.3
(DEMO_MODE_REAL_PAYMENT, PROD_MODE_ENV_SECRET error codes)

#### Session 1.3.1 — Implement `selectProfile()`

**Task:** Create `internal/config/select.go` implementing the `SelectProfile()` function
(exported for use by both `cmd/microservice` and `cmd/provider`) matching the logic in
MVP §5.3. The function must:
1. Read `VYOMANAUT_MODE` from environment
2. Allow `--mode` CLI flag override (accept the flag value as a parameter so `cmd/`
   wiring passes it in)
3. Log the startup banner format exactly as shown in MVP §2.1
4. Return `DemoProfile` for `"demo"`, `ProductionProfile` for `"prod"` or `""`
5. Log a WARNING when mode is absent (defaults to prod)
6. Call `os.Exit(1)` with a fatal log for any unknown value

#### Session 1.3.2 — Implement startup guard rails

**Task:** Create `internal/config/guards.go` implementing `ValidateStartupGuards(profile
NetworkProfile)`. This function must implement the two mandatory guard rails from MVP §2.3:
1. If `profile.Mode == "prod"` AND `os.Getenv("VYOMANAUT_CLUSTER_MASTER_SEED") != ""`
   → return error with code `PROD_MODE_ENV_SECRET` (IC §3.3 error code table)
2. If `profile.Mode == "demo"` AND `profile.PaymentMode == "razorpay_live"`
   → return error with code `DEMO_MODE_REAL_PAYMENT` (IC §3.3 error code table)

These correspond to the `500` error codes in IC §3.3. The function returns a typed error
so callers can distinguish which guard fired. Both `cmd/microservice` and `cmd/provider`
must call this at startup before any other initialisation.

#### Session 1.3.3 — Guard rail tests

**Task:** Test all guard rail combinations: prod + seed env present (must error),
prod + seed env absent (must pass), demo + razorpay_live (must error), demo + mock (must
pass). Use `t.Setenv` for environment isolation.

---

## Milestone 2 — Core Cryptography (`internal/crypto`)

**Deliverable:** All cryptographic primitives fully implemented, goroutine-safe, with
known-answer tests and fuzz targets. This package has zero imports from other `internal/`
packages (IC §9 — hardest import constraint).

**Reference:** IC §5.1 (full function signatures, pre/post-conditions, sentinel errors),
MVP §8.2 (file inventory for internal/crypto), IC §11 (forbidden: float params, convergent
encryption, K reuse), DM §3 Invariant 4 (no float in monetary path — enforced in crypto
by the package invariant in IC §5.1)

---

### Phase 2.1 — AES-NI Detection

**Reference:** IC §5.1 (`DetectAESNI`)

#### Session 2.1.1 — Implement `DetectAESNI()`

**Task:** Create `internal/crypto/aesni.go` (build tag `//go:build amd64`) implementing
`DetectAESNI() bool` via CPUID instruction per IC §5.1. Create
`internal/crypto/aesni_other.go` (build tag `//go:build !amd64`) returning `false`.
The function is called once at startup and stored; it must never be called again at
runtime (IC §5.1 post-condition: "Never re-checked at runtime"). Store the result in a
package-level `var aesNIAvailable = DetectAESNI()` that callers use — but this variable
must not be exported (callers pass the result as a parameter per IC §5.1).

Verify: Compiles on amd64 and arm64. On amd64 with AES-NI hardware, returns true.

---

### Phase 2.2 — HKDF Key Derivation

**Reference:** IC §5.1 (`DeriveFileKey`, `DerivePointerEncKey`, `DeriveKeystoreEncKey`,
`DeriveDHTOwnerKey`, `DeriveDHTKey`)

#### Session 2.2.1 — Implement HKDF-SHA256 derivation functions

**Task:** Create `internal/crypto/hkdf.go`. Implement the five HKDF functions from
IC §5.1, each with the exact `info` string specified:
- `DeriveFileKey`: info = `"vyomanaut-file-v1" || fileID`
- `DerivePointerEncKey`: info = `"vyomanaut-pointer-v1" || fileID`
- `DeriveKeystoreEncKey`: info = `"vyomanaut-keystore-v1"` (no fileID)
- `DeriveDHTOwnerKey`: info = `"vyomanaut-dht-v1" || fileID`
- `DeriveDHTKey`: output of HMAC-SHA256(chunkHash, fileOwnerKey) per IC §5.1 and IC §12

All pre-conditions from IC §5.1 must be enforced: `len(masterSecret) == 32`,
`len(ownerID) == 16`, `len(fileID) == 16` — panic in debug build (`//go:build debug`),
return sentinel error in release build. All functions are pure (no side effects).

**Critical:** `DeriveKeystoreEncKey` is named `DeriveKeystoreEncKey` here but IC §3.2
refers to it as `DeriveDKSKeystoreEncKey()` in the heartbeat section. Resolve: the
canonical name per IC §5.1 exported interface is `DeriveKeystoreEncKey`. The IC §3.2
reference contains an inconsistency — use `DeriveKeystoreEncKey` and add a code comment
noting the discrepancy for a future PR to fix IC §3.2.

#### Session 2.2.2 — HKDF known-answer tests

**Task:** Add `internal/crypto/hkdf_test.go` with known-answer vectors derived from
fixed inputs (not from a live run — compute expected output offline using the HKDF-SHA256
RFC 5869 test vector approach). Add a round-trip test: `DeriveFileKey(ms, o, f) ==
DeriveFileKey(ms, o, f)` (determinism). Add a non-collision test: two different fileIDs
produce different keys.

---

### Phase 2.3 — Argon2id Master Secret Derivation

**Reference:** IC §5.1 (`DeriveMasterSecret`), MVP §3.5 (crypto parameters),
MVP §5.4 (caller responsibility: pass profile.Argon2* fields, never hardcode)

#### Session 2.3.1 — Implement `DeriveMasterSecret()`

**Task:** Create `internal/crypto/argon2.go` implementing `DeriveMasterSecret()` per
IC §5.1 signature exactly: `func DeriveMasterSecret(passphrase, ownerID []byte,
argon2Time uint32, argon2Memory uint32, argon2Threads uint8) [32]byte`. Use
`golang.org/x/crypto/argon2` Argon2id variant. The ownerID (UUID bytes) is the salt.

Pre-condition enforcement: `len(passphrase) >= 8`, `len(ownerID) == 16`,
`argon2Time >= 1`, `argon2Memory >= 4096`, `argon2Threads >= 1` — panic in debug,
sentinel error otherwise (no errors returned per IC §5.1 since it panics; note that
IC §5.1 says "no errors returned; pre-condition violations panic").

Add the mandatory caller-responsibility comment from IC §5.1: callers must pass
`profile.Argon2Time`, `profile.Argon2Memory`, `profile.Argon2Threads` — never hardcode.

#### Session 2.3.2 — `DeriveMasterSecret` performance test

**Task:** Add a test `TestArgon2idProduction` that runs with production parameters
(t=3, m=65536, p=4) and asserts completion time >= 100ms (conservative floor — the
IC §5.1 post-condition states "Production: >= 200ms" but 100ms covers slower CI
hardware). Add `TestArgon2idDemo` that runs with demo parameters (t=1, m=4096, p=1)
and asserts completion time < 200ms. These tests must be tagged `//go:build !short`
so they are skipped in fast CI runs.

---

### Phase 2.4 — AONT Cipher

**Reference:** IC §5.1 (`AONTEncodeSegment`, `AONTDecodePackage`), IC §11 (forbidden:
convergent encryption, K reuse), MVP §8.2 (`aont_canary.go`)

#### Session 2.4.1 — Implement the AONT canary constant

**Task:** Create `internal/crypto/aont_canary.go` defining the fixed 16-byte canary
value as `var aontCanary = [16]byte{...}` (choose a deterministic value, e.g. the
first 16 bytes of SHA-256("vyomanaut-aont-canary-v1"), computed once and hardcoded).
This must be a `var`, not a `const`, because Go does not allow array constants — but it
must never be reassigned. Add a `// canary must never be changed — it is an on-disk format commitment` comment. The canary is the second-to-last 16-byte word of every AONT
package (IC §5.1 post-condition for `AONTEncodeSegment`).

#### Session 2.4.2 — Implement `AONTEncodeSegment()`

**Task:** Create `internal/crypto/aont.go`. Implement `AONTEncodeSegment(segment []byte,
aesNIAvailable bool) ([]byte, error)` per IC §5.1:
- Generate fresh random K via `crypto/rand` (never reuse — IC §11 forbidden: K reuse)
- Select cipher: ChaCha20-256 if `aesNIAvailable == false`, AES-256-CTR if true (IC §5.1,
  ADR-019, ADR-022)
- Produce AONT package: `(s+1)` 16-byte words where `s = len(segment)/16`
- Embed canary as second-to-last word
- Last word = `K XOR SHA-256(all preceding codewords)` (IC §5.1 post-condition)
- K must not be returned — it is embedded and inaccessible without all words

Pre-conditions: `len(segment)` must be a multiple of 16; caller is responsible for
padding to 4 MB minimum before calling.

#### Session 2.4.3 — Implement `AONTDecodePackage()`

**Task:** Implement `AONTDecodePackage(aontPackage []byte, aesNIAvailable bool) ([]byte,
error)` in `internal/crypto/aont.go`. Must:
1. Recover K from the last word XOR SHA-256(all preceding words)
2. Decrypt
3. Verify the canary word (second-to-last) equals `aontCanary`
4. If canary mismatches: zero the buffer before returning `ErrCanaryMismatch` (IC §5.1
   post-condition: "caller MUST NOT return any plaintext to the data owner. Zero the
   buffer before returning.")

#### Session 2.4.4 — AONT tests

**Task:** Add `internal/crypto/aont_test.go`:
- Round-trip test: `AONTDecodePackage(AONTEncodeSegment(data))` == original data
- Test that each call to `AONTEncodeSegment` produces a different ciphertext (K freshness)
- Test that corrupting any single byte in the AONT package causes `ErrCanaryMismatch`
  (all-or-nothing property)
- Cross-cipher test: encode with AES-NI=true, decode with AES-NI=false — must produce
  `ErrCanaryMismatch` (different cipher = different output; test documents this is not
  cross-compatible and is expected to fail)

---

### Phase 2.5 — Pointer File AEAD

**Reference:** IC §5.1 (`EncryptPointerFile`, `DecryptPointerFile`, `ErrTagMismatch`),
IC §11 (constant-time tag comparison: NFR-019)

#### Session 2.5.1 — Implement pointer file AEAD

**Task:** Create `internal/crypto/chacha20poly1305.go`. Implement `EncryptPointerFile`
and `DecryptPointerFile` using `golang.org/x/crypto/chacha20poly1305` (AEAD_CHACHA20_POLY1305,
RFC 8439 per IC §5.1). 

**Critical for `DecryptPointerFile`:** The Poly1305 tag must be verified with
`crypto/subtle.ConstantTimeCompare` before any plaintext is returned (IC §5.1
post-condition, NFR-019). If tag verification fails, return `ErrTagMismatch` with
nil plaintext. The caller must not use any returned bytes on `ErrTagMismatch` (IC §5.1).

AAD pre-condition: `len(aad) > 0`; the AAD must include `ownerID || fileID ||
schemaVersion` — this is the caller's responsibility (documented in IC §5.1 pre-condition).

Define `ErrTagMismatch` in `internal/crypto/errors.go`.

---

### Phase 2.6 — BIP-39 Mnemonic System

**Reference:** IC §5.1 (`MasterSecretToMnemonic`, `MnemonicToMasterSecret`,
`SelectConfirmationWords`, `ErrInvalidMnemonic`), MVP §3.5 (mnemonic confirmation),
MVP §8.2 (`bip39.go`), IC §11 (forbidden: BIP-39 mnemonic words for real accounts)

#### Session 2.6.1 — Embed BIP-39 English wordlist

**Task:** Embed the BIP-39 English wordlist (2048 words) as a `//go:embed` file at
`internal/crypto/wordlist_en.txt`. This is the only permitted wordlist (IC §5.1).
Vyomanaut never uses passphrases on top of the mnemonic — the mnemonic IS the master
secret directly, not a BIP-39 seed input (IC §5.1 comment). Add a test that the
wordlist contains exactly 2048 words.

#### Session 2.6.2 — Implement `MasterSecretToMnemonic()`

**Task:** Implement per IC §5.1: 32 bytes of entropy (256 bits) → 24 words with 8-bit
checksum, per BIP-39 §Generating the mnemonic. The round-trip property
`MnemonicToMasterSecret(MasterSecretToMnemonic(ms)) == ms` must hold (IC §5.1
post-condition).

#### Session 2.6.3 — Implement `MnemonicToMasterSecret()`

**Task:** Implement the recovery path per IC §5.1. On invalid mnemonic (wrong word
count, unknown word, checksum failure), return `ErrInvalidMnemonic` without revealing
which word failed (IC §5.1: "Do not expose which word failed — timing oracle"). All
validation branches must take equal time (use constant-time word lookup).

#### Session 2.6.4 — Implement `SelectConfirmationWords()`

**Task:** Implement per IC §5.1. Return two distinct random indices (0–23) drawn via
`crypto/rand`. The function may be called even in demo mode where `SkipMnemonicConfirm
== true` — the caller simply does not block on the result (IC §5.1 note).

#### Session 2.6.5 — BIP-39 round-trip and error tests

**Task:** Add known-answer vectors from BIP-39 test vectors (RFC-documented, fixed
entropy — not from real accounts per IC §11). Test `ErrInvalidMnemonic` on: wrong count,
unknown word, bad checksum. Test that `SelectConfirmationWords` never returns two equal
indices across 1000 calls.

---

### Phase 2.7 — Ed25519 Signing Conventions

**Reference:** IC §3.2 (canonical signing input serialisation, verification procedure),
IC §5.1 (no JSON serialisation for signing inputs)

#### Session 2.7.1 — Document Ed25519 signing conventions in `internal/crypto`

**Task:** Create `internal/crypto/ed25519.go`. This file does NOT wrap the standard
library — it provides:
1. A `SignBytes(privateKey ed25519.PrivateKey, inputBytes []byte) [64]byte` helper that
   computes `Ed25519(private_key, SHA-256(inputBytes))` per IC §3.2 canonical procedure
2. A `VerifyBytes(publicKey [32]byte, inputBytes []byte, sig [64]byte) bool` helper
3. A compile-time assertion that Go's `ed25519.PublicKeySize == 32`

**Critical:** IC §3.2 states: "JSON serialisation MUST NOT be used for signing inputs —
field ordering is not guaranteed across Go versions." Add this as a package-level comment
and as a `// SIGNING_INPUT_RULE: use fixed-layout byte sequence, never JSON` comment on
both functions.

Verify: The functions are pure and goroutine-safe (IC §5.1 package invariant).

---

## Milestone 3 — Erasure Coding Engine (`internal/erasure`)

**Deliverable:** `internal/erasure` fully implemented with profile-parameterised
DataShards/TotalShards. Zero imports from other `internal/` packages (IC §9).

**Reference:** IC §5.2 (full interface), MVP §3.2 (erasure params table),
DM §3 Invariant 7 (ShardSize constant), IC §9 (import constraints)

---

### Phase 3.1 — Engine Construction

**Reference:** IC §5.2 (`NewEngine`, `Engine` struct, `ShardSize` constant)

#### Session 3.1.1 — Define `ShardSize` constant and `Engine` struct

**Task:** Create `internal/erasure/params.go`. Define `const ShardSize = 262144`
(IC §5.2 comment: "fixed in both modes; never profile-variable"). Define the `Engine`
struct per IC §5.2: `DataShards`, `ParityShards`, `TotalShards` as `int` fields.

Now update M1 Session 1.2.1: add the cross-check import of `erasure.ShardSize` to
`TestProfileShardSizeIsConstant`.

#### Session 3.1.2 — Implement `NewEngine()`

**Task:** In `internal/erasure/engine.go`, implement `NewEngine(profile
config.NetworkProfile) (*Engine, error)`. Pre-conditions per IC §5.2:
- `profile.DataShards >= 1`
- `profile.TotalShards == profile.DataShards + profile.ParityShards`
- `profile.ShardSize == ShardSize` (compile-time constant cross-check)

Use `github.com/klauspost/reedsolomon` as the underlying library. Return an error if the
library constructor fails.

---

### Phase 3.2 — Encode & Decode

**Reference:** IC §5.2 (`EncodeSegment`, `DecodeSegment`, `ErrTooFewShards`, `ErrShardSize`)

#### Session 3.2.1 — Implement `EncodeSegment()`

**Task:** Per IC §5.2 pre-condition: `len(aontPackage) == e.DataShards * ShardSize`
exactly. Returns exactly `e.TotalShards` byte slices each of length `ShardSize`.

#### Session 3.2.2 — Implement `DecodeSegment()`

**Task:** Per IC §5.2. Accept `e.TotalShards` shards with at least `e.DataShards`
non-nil. Fill nil entries in-place. Return `ErrTooFewShards` if fewer than DataShards
non-nil. Return `ErrShardSize` if a non-nil shard has wrong length.

#### Session 3.2.3 — Erasure coding tests

**Task:** In `internal/erasure/engine_test.go`:
- Round-trip test for both production (16,56) and demo (3,5) engines
- "Any k shards" test: for every combination of DataShards from TotalShards shards,
  reconstruction succeeds (for demo engine only — production would be 56-choose-16 combinations)
- ShardSize constant test: assert `ShardSize == 262144` (belt-and-suspenders for DM Invariant 7)
- Cross-engine test: data encoded with demo engine cannot be decoded by production engine
  (documents the separation)

---

## Milestone 4 — Database Schema & Migrations

**Deliverable:** A fully applied initial migration (`migrations/001_initial_schema.sql`)
with all tables, indexes, row security policies, and materialised views. The migration
must pass the DM §9 checklist and must apply cleanly against both demo-profile and
production-profile schema generators.

**Reference:** DM §2 through §9 (complete schema), DM §3 (invariants — all 7 must be
enforced), IC §6 (row-level DML contracts — these constrain the schema's RSP definitions),
MVP §5.5 (schema parameterisation), MVP §6.4 (migration requirements)

---

### Phase 4.1 — Migration Generator

**Reference:** MVP §5.5 (`migrations/generator.go`), DM §9 (profile rule)

#### Session 4.1.1 — Implement `migrations/generator.go`

**Task:** Create `migrations/generator.go` implementing `GenerateInitialSchema(profile
config.NetworkProfile) string` per MVP §5.5. The function produces two profile-dependent
CHECK constraints:
1. `chunk_assignments.shard_index`: `BETWEEN 0 AND {profile.TotalShards-1} OR shard_index IS NULL`
2. `repair_jobs.available_shard_count`: `BETWEEN {profile.DataShards} AND {profile.TotalShards}`

The generator is invoked as: `go run migrations/generator.go --profile=prod` or `--profile=demo`.
It outputs the full migration SQL to stdout. The output is redirected to
`migrations/001_initial_schema.sql` (production) or `migrations/001_initial_schema_demo.sql`
(demo).

**Warning from DM §9:** "Never apply a demo schema to a production database or vice versa."
The generator must embed the profile name as a comment header in the output SQL:
`-- Generated for profile: {prod|demo}` so a human reviewer can verify the schema
before applying.

---

### Phase 4.2 — Core Type Definitions

**Reference:** DM §4 (CREATE TYPE statements, DM §9 migration ordering rule: types before tables)

#### Session 4.2.1 — Define all PostgreSQL ENUMs

**Task:** Add to the migration SQL in order (DM §9: all `CREATE TYPE` statements before
`CREATE TABLE`):
- `provider_status` ENUM: `PENDING_ONBOARDING`, `VETTING`, `ACTIVE`, `DEPARTED` (DM §4.2)
- `file_status` ENUM: `ACTIVE`, `DELETION_PENDING`, `DELETED` (DM §4.3, DM §9 checklist)
- `assignment_status` ENUM: `ACTIVE`, `REPAIRING`, `PENDING_DELETION`, `DELETED` (DM §4.5)
- `audit_result_type` ENUM: `PASS`, `FAIL`, `TIMEOUT` (DM §4.7)
- `escrow_event_type` ENUM: `DEPOSIT`, `RELEASE`, `SEIZURE`, `REVERSAL` (DM §4.8 and DM §9
  checklist item "Add REVERSAL to escrow_event_type ENUM")
- `owner_escrow_event_type` ENUM: `DEPOSIT`, `CHARGE`, `WITHDRAWAL`, `REFUND` (DM §4.9)
- `repair_trigger_type` ENUM: `SILENT_DEPARTURE`, `ANNOUNCED_DEPARTURE`,
  `THRESHOLD_WARNING`, `EMERGENCY_FLOOR` (DM §4.10)
- `repair_priority` ENUM: `EMERGENCY`, `PERMANENT_DEPARTURE`, `PRE_WARNING`
  (DM §4.10, DM §9 checklist item "repair_priority ENUM has three values")
- `repair_job_status` ENUM: `QUEUED`, `IN_PROGRESS`, `COMPLETED`, `FAILED` (DM §4.10)

**Verify:** DM §9 checklist items for ENUMs are all checked. The `repair_priority` ENUM
ordering matters for `ORDER BY priority ASC` — document that EMERGENCY sorts first
alphabetically (E < P < P) in the migration comment.

---

### Phase 4.3 — Core Tables

**Reference:** DM §4.1 through §4.5

#### Session 4.3.1 — `owners` table

**Task:** Implement the `owners` DDL from DM §4.1 exactly. Columns: `owner_id` (UUID PK),
`phone_number` (VARCHAR(15) NOT NULL UNIQUE), `ed25519_public_key` (BYTEA with
`octet_length = 32` CHECK), `smart_collect_vpa` (VARCHAR(255) nullable — justified in
DM §8.1), `created_at` (TIMESTAMPTZ NOT NULL DEFAULT NOW()). Add table and column
COMMENTs from DM §4.1.

#### Session 4.3.2 — `providers` table

**Task:** Implement the `providers` DDL from DM §4.2 with all columns and constraints.
Pay particular attention to:
- `p95_throughput_kbps FLOAT NULL` (DM §4.2 comment: default must be NULL not 0;
  DM §9 checklist: "p95_throughput_kbps and avg_rtt_ms default to NULL, not 0/2000")
- `avg_rtt_ms FLOAT NULL` (same reason)
- `var_rtt_ms FLOAT NOT NULL DEFAULT 0` (DM §4.2: zero variance is safe initial assumption)
- `first_chunk_assignment_at TIMESTAMPTZ` nullable (DM §8.6)
- The `providers_departed_status` CHECK: `departed_at IS NULL OR status = 'DEPARTED'`
- `providers_throughput_nonneg`, `providers_avg_rtt_nonneg`, `providers_var_rtt_nonneg`,
  `providers_passes_nonneg` CHECK constraints
- Physical DELETE is prohibited — the RSP in Phase 4.7 enforces this at the DB layer;
  the DDL comment must state: "No DELETE ever — see RSP (DM §6, IC §6, Invariant 3)"

#### Session 4.3.3 — `files` table

**Task:** Implement the `files` DDL from DM §4.3. Include the `display_name_*` columns
(DM §9 checklist: "files.display_name_ciphertext/nonce/tag columns present for FR-019").
The `display_name_nonce` and `display_name_tag` CHECKs must allow NULL (DM §4.3).

**⚠️ AWAITING:** `FR-019` is referenced here but `requirements.md` is not in context.
The column is included per DM §4.3 DDL as written. When `requirements.md` is shared,
verify FR-019 scope matches the column design.

#### Session 4.3.4 — `segments` table

**Task:** Implement the `segments` DDL from DM §4.4. The `segments_unique_index` UNIQUE
constraint on `(file_id, segment_index)` prevents two segments at the same position in
a file.

#### Session 4.3.5 — `chunk_assignments` table

**Task:** Implement the `chunk_assignments` DDL from DM §4.5. Critical items:
- `is_vetting_chunk BOOLEAN NOT NULL DEFAULT FALSE` (DM §9 checklist)
- `segment_id` changed to nullable (DM §9 checklist, DM §8.21)
- `shard_index` changed to nullable (DM §9 checklist, DM §8.22)
- `chunk_assignments_segment_and_shard_null_iff_vetting` CHECK constraint (DM §4.5,
  DM §9 checklist) — this is the DB-layer enforcement of Invariant 6
- `chunk_assignments_one_per_provider_per_chunk` UNIQUE on `(chunk_id, provider_id)`
- The partial unique index `idx_chunk_assignments_one_active_per_shard` must be
  a standalone `CREATE UNIQUE INDEX` statement, NOT an inline constraint
  (DM §9: "inline WHERE on UNIQUE is invalid syntax")

---

### Phase 4.4 — Audit, Escrow, and Repair Tables

**Reference:** DM §4.6 through §4.10

#### Session 4.4.1 — `audit_periods` table

**Task:** Implement DM §4.6 including the `audit_periods_no_overlap` exclusion constraint.
This requires `btree_gist` (DM §9 first checklist item). Add the prerequisite comment:
`-- PREREQUISITE: CREATE EXTENSION IF NOT EXISTS btree_gist;`
The `period_start < period_end` CHECK must be present.

#### Session 4.4.2 — `audit_receipts` table

**Task:** Implement DM §4.7. Critical items enforcing Invariant 1 (append-only audit log):
- `challenge_nonce BYTEA NOT NULL CHECK (octet_length(challenge_nonce) = 33)` —
  must be 33, never 32 (DM §3 Invariant 5, DM §9 checklist, IC §11 forbidden pattern)
- `audit_result audit_result_type` — nullable, no DEFAULT (DM §9 checklist: "no DEFAULT
  on audit_result — NULL is the intended initial PENDING state")
- `file_id UUID REFERENCES files(file_id)` — nullable (DM §9 checklist, DM §8.20)
- `address_was_stale BOOLEAN NOT NULL DEFAULT FALSE` (DM §4.7)
- The `audit_receipts_response_consistency` CHECK (DM §4.7)
- The `audit_receipts_service_sig_consistency` CHECK (DM §4.7)
- The `audit_receipts_nonce_unique` UNIQUE constraint (prevents replay attacks)
- Add the nightly data integrity monitoring query as a SQL comment (DM §4.7 note,
  DM §9 checklist): the query must return 0

#### Session 4.4.3 — `escrow_events` table

**Task:** Implement DM §4.8. Invariant 2 enforcement:
- `amount_paise BIGINT NOT NULL CHECK (amount_paise > 0)` — never FLOAT, NUMERIC, DECIMAL
  (DM §3 Invariant 4)
- `idempotency_key VARCHAR(64) NOT NULL UNIQUE`
- `escrow_event_type` must include `REVERSAL` (DM §9 checklist item)
- The RSP in Phase 4.7 adds the INSERT-only policy at the DB layer

#### Session 4.4.4 — `owner_escrow_events` table

**Task:** Implement DM §4.9. This table is required for FR-014, FR-021, FR-059 (DM §9
checklist). Same `amount_paise BIGINT NOT NULL CHECK (amount_paise > 0)` constraint.
Add the balance query as a SQL comment: `SUM(DEPOSIT) - SUM(CHARGE + WITHDRAWAL) + SUM(REFUND)`.

#### Session 4.4.5 — `repair_jobs` table

**Task:** Implement DM §4.10. Critical items:
- `repair_jobs_priority_matches_trigger` CHECK — priority derived from trigger_type
  (IC §5.7 pre-condition also enforces this)
- `repair_jobs_completed_after_started` CHECK
- The `available_shard_count` CHECK bounds come from the generator (Phase 4.1)
- `provider_id` is nullable — justified in DM §8.17
- The `repair_jobs_no_duplicate_departure` constraint note (DM §4.10): the UNIQUE has
  been removed; deduplication is at the application layer
- The `idx_repair_jobs_threshold_no_dup` partial unique index handles threshold-triggered
  deduplication (DM §5)

---

### Phase 4.5 — Index Catalogue

**Reference:** DM §5 (complete index catalogue with named query patterns)

#### Session 4.5.1 — Create all indexes

**Task:** Create every index listed in DM §5 as a standalone `CREATE INDEX` or
`CREATE UNIQUE INDEX` statement, in the order they appear in DM §5. Each must be
accompanied by its named query pattern comment from DM §5 (e.g., "Query: departure
detector — find providers with last_heartbeat_ts > 72h ago").

Special attention to vetting-related indexes (DM §5):
- `idx_chunk_assignments_vetting_provider_active` — for ACTIVE transition GC
- `idx_chunk_assignments_vetting_provider` — for departure handler bulk delete
  These are new indexes added with ADR-030 (DM §9 checklist).

The `idx_chunk_assignments_one_active_per_shard` partial unique index must include
`WHERE is_vetting_chunk = FALSE` (DM §9 checklist: "old inline constraint without
this filter must be dropped").

---

### Phase 4.6 — Row Security Policies

**Reference:** DM §6, IC §6 (DML contracts that RSPs enforce)

#### Session 4.6.1 — `audit_receipts` RSP (Invariant 1)

**Task:** Implement per DM §6:
- `ALTER TABLE audit_receipts ENABLE ROW LEVEL SECURITY`
- `audit_receipts_insert_only` policy for `vyomanaut_app` — INSERT only
- `audit_receipts_phase2_update` policy for `vyomanaut_app` — UPDATE only where
  `audit_result IS NULL AND abandoned_at IS NULL`, and only to set
  `audit_result IN ('PASS','FAIL','TIMEOUT')` with non-null `service_sig` and
  `service_countersign_ts`
- `audit_receipts_gc_abandon` policy for `vyomanaut_gc` — UPDATE only for stale
  PENDING rows older than 48h (in production; the profile's `PendingReceiptGCAge`
  governs the actual query, but the RSP uses the production interval for safety)
- No DELETE policy created — any DELETE returns permission denied

#### Session 4.6.2 — `escrow_events` RSP (Invariant 2)

**Task:** Implement per DM §6:
- `ALTER TABLE escrow_events ENABLE ROW LEVEL SECURITY`
- `escrow_events_insert_only` policy for `vyomanaut_app` — INSERT only
- No UPDATE or DELETE policy

#### Session 4.6.3 — `chunk_assignments` soft-delete policy

**Task:** Add the `chunk_assignments` RSP per DM §6:
- A policy that prohibits hard DELETE for all roles
- The only permitted status-to-`DELETED` transition is via `UPDATE status = 'DELETED'`
  by `vyomanaut_app`

**Note from DM §6:** "Previously a HARD-DELETE was issued; now a soft-delete policy is
implemented." This RSP exists specifically to prevent recurrence.

---

### Phase 4.7 — Materialised Views

**Reference:** DM §7, MVP §5.5 (mv_provider_scores is profile-parameterised), DM §9
(migration ordering: views after tables)

#### Session 4.7.1 — `mv_provider_scores`

**Task:** Implement per DM §7. Critical items:
- The `NOW() AS scores_as_of` column (DM §9 checklist, DM §7: "consumers must check age")
- The view uses a subquery (DM §9 checklist: "column aliases not referenced in same SELECT level")
- The interval literals are PLACEHOLDERS in the migration — this view is dropped and
  recreated at microservice startup from `NetworkProfile.ScoreWindow{Short,Medium,Long}`
  (DM §7 note, MVP §5.5). Add a SQL comment: `-- DROP AND RECREATE at startup from NetworkProfile`

#### Session 4.7.2 — `mv_provider_escrow_balance`

**Task:** Implement per DM §7 with the REVERSAL amendment (DM §7 comment: "WE MADE AN
AMMEND ADDING REVERSAL"). Balance formula: `SUM(DEPOSIT + REVERSAL) - SUM(RELEASE + SEIZURE)`.
DM §9 checklist: "mv_provider_escrow_balance includes REVERSAL in the DEPOSIT-side SUM".

#### Session 4.7.3 — `mv_owner_escrow_balance`

**Task:** Implement per DM §7 with `GREATEST(..., 0)` on the final balance to prevent
negative values (DM §7 comment). Formula: `SUM(DEPOSIT + REFUND) - SUM(CHARGE + WITHDRAWAL)`.

#### Session 4.7.4 — `mv_segment_shard_counts`

**Task:** Implement per DM §7. Both `available_shard_count` (ACTIVE + REPAIRING) and
`active_shard_count` (ACTIVE only) columns are required for the repair trigger detector
and owner dashboard.

#### Session 4.7.5 — Unique indexes on all materialised views

**Task:** Add `CREATE UNIQUE INDEX ON mv_provider_scores (provider_id)` and equivalent
for all four materialised views. Required for `REFRESH MATERIALIZED VIEW CONCURRENTLY`
(DM §9 checklist).

---

### Phase 4.8 — Migration Checklist Verification

**Reference:** DM §9 (complete checklist)

#### Session 4.8.1 — Run and document DM §9 checklist

**Task:** Execute every item in the DM §9 migration checklist as a CI-runnable test
suite in `scripts/ci/migration_check.sh`. Each check must be a `psql` query or `\d`
inspection that passes or fails explicitly. Failing checks must print the exact DM §9
checklist item text. The checks include:
- `btree_gist` installed
- `challenge_nonce` is BYTEA with `octet_length = 33` (not 32)
- No FLOAT columns in escrow_events
- `audit_result` has no DEFAULT
- All ENUM values are present (including `REVERSAL`)
- `is_vetting_chunk` column present on `chunk_assignments`
- `file_id` on `audit_receipts` is nullable
- Partial unique index includes `WHERE is_vetting_chunk = FALSE`
- `scores_as_of` column present in `mv_provider_scores`
- `REVERSAL` in `mv_provider_escrow_balance` balance formula

The migration checklist script is CI check 7 (MVP §8.4).

---

## Milestone 5 — Provider Storage Engine (`internal/storage`)

**Deliverable:** WiscKey-style storage engine: RocksDB index + append-only vLog, with
crash recovery, GC, and the single-writer goroutine constraint enforced by test.

**Reference:** IC §5.3 (full `ChunkStore` interface, all methods, sentinel errors,
concurrency contract), MVP §8.2 (file inventory: store.go, vlog.go, index.go,
single_writer_test.go), IC §11 (forbidden: breaking the single-writer invariant)

---

### Phase 5.1 — ChunkStore Interface & vLog

**Reference:** IC §5.3

#### Session 5.1.1 — Define `ChunkStore` interface and sentinel errors

**Task:** Create `internal/storage/store.go` with the `ChunkStore` interface from IC §5.3
and `internal/storage/errors.go` with all sentinel errors: `ErrChunkNotFound`,
`ErrContentHashMismatch`, `ErrVLogFsync`, `ErrVLogRead`, `ErrRocksDBInsert`.

**Critical concurrency comment (IC §5.3):** The `AppendChunk` method comment must include
the exact warning: `*** SINGLE WRITER ONLY — NOT goroutine-safe ***` as specified in IC §5.3.

#### Session 5.1.2 — Implement append-only vLog

**Task:** Create `internal/storage/vlog.go`. The vLog is an append-only file. Each entry
stores: `chunk_id` (32 bytes), `content_hash = SHA-256(chunk_data)` (32 bytes), chunk
length (4 bytes), raw `chunk_data` (262144 bytes). The `AppendChunk` method must:
1. Verify `SHA-256(chunkData) == chunkID` before writing (IC §5.3 pre-condition)
2. Write the entry to the vLog file
3. Call `fsync` before returning
4. On fsync failure: return `ErrVLogFsync`; the daemon must halt and restart

Add HDD/SSD detection via `internal/storage/rotational.go` (build tag `linux`) and
`internal/storage/rotational_other.go` (stub: assume SSD). This affects fsync strategy
(document the decision in a comment per MVP §8.2).

#### Session 5.1.3 — Implement RocksDB index wrapper

**Task:** Create `internal/storage/index.go`. Wrap `github.com/linxGnu/grocksdb` (or
equivalent RocksDB Go binding). The index maps `chunk_id → (vlog_offset, chunk_size)`.
Include a Bloom filter (a RocksDB column family option) so `LookupChunk` can exit fast
on "not present" without a full disk seek (IC §4.2: "Bloom filter check" in the audit
challenge flow diagram).

#### Session 5.1.4 — Implement `LookupChunk()`

**Task:** Per IC §5.3: read the chunk from the vLog using the stored offset. Verify
`SHA-256(returned data) == chunkID` internally before returning. If hash mismatch:
return `ErrContentHashMismatch` (this is the signal for audit FAIL with `status=0x02`
per IC §4.2). Goroutine-safe (uses `pread` which is thread-safe on POSIX).

#### Session 5.1.5 — Implement `DeleteChunk()`, `RecoverFromCrash()`, `RunGC()`

**Task:** Implement the three remaining `ChunkStore` methods per IC §5.3:
- `DeleteChunk`: removes RocksDB index entry; vLog entry is reclaimed by GC
- `RecoverFromCrash`: scans vLog from last known head offset, re-inserts missing
  RocksDB entries; must be called before the writer goroutine starts
- `RunGC`: reclaims vLog space from deleted entries; runs in a background goroutine

**IC §5.3 note on `DeleteChunk`:** "The daemon has no visibility into whether the deleted
chunk was synthetic or real." Both real and synthetic chunk deletions use the same code path.

---

### Phase 5.2 — Storage Tests

**Reference:** MVP §8.2 (single_writer_test.go), IC §5.3

#### Session 5.2.1 — Implement `TestSingleWriterGoroutine`

**Task:** In `internal/storage/single_writer_test.go`, implement the test from MVP §8.2:
launch 100 goroutines all calling `AppendChunk` simultaneously. The expected outcome is
a deadlock or race detection failure (not silent data corruption) — the test is structured
so that the channel-based serialisation in the upload manager (tested separately) is what
prevents this. This test documents that `AppendChunk` is intentionally not goroutine-safe
and should not be made goroutine-safe.

**Note:** This test must be run with `-race` to detect race conditions. It is part of
CI check 4 (MVP §8.4: `go test ./... -race`).

#### Session 5.2.2 — Storage round-trip and crash recovery tests

**Task:** Test: write N chunks, restart (simulate crash by skipping `Close()`),
call `RecoverFromCrash()`, verify all chunks are readable. Test: `LookupChunk` on
a chunk with corrupted vLog bytes returns `ErrContentHashMismatch`. Test: `DeleteChunk`
followed by `LookupChunk` returns `ErrChunkNotFound`.

---

## Milestone 6 — P2P Network Layer (`internal/p2p`)

**Deliverable:** libp2p host with QUIC/TCP+Noise transports, NAT traversal stack,
Kademlia DHT with custom HMAC key validator, and 0-RTT policy enforcement per protocol ID.

**Reference:** IC §4 (common libp2p rules), IC §4.3 (Circuit Relay v2), IC §5.4
(`Host` and `DHT` interfaces, sentinel errors), IC §12 (DHT key contract),
MVP §8.2 (file inventory), MVP §8.4 (TestDHTKeyValidatorPersists — CI check 5)

---

### Phase 6.1 — libp2p Host

**Reference:** IC §5.4 (`Host` interface), IC §4 (common rules: transport auth, stream lifecycle)

#### Session 6.1.1 — Implement `Host` interface

**Task:** Create `internal/p2p/host.go`. Implement the `Host` interface from IC §5.4
using `github.com/libp2p/go-libp2p`. The host must use:
- QUIC v1 as primary transport (IC §4.3 ADR-021)
- TCP+Noise XX as fallback transport
- Ed25519-based Peer ID (IC §5.4: `PeerID() = multihash(ed25519_public_key)`)

**0-RTT policy enforcement (IC §4, IC §5.4):** The `Host` must automatically set
`DisableEarlyData: true` for any stream whose protocol ID suffix matches `-audit` or
`-challenge`. IC §5.4: "The caller does not need to enforce this; the Host enforces it
automatically based on the protocol ID." Implement this in `host.go` as a protocol
middleware that wraps `NewStream`.

#### Session 6.1.2 — NAT traversal stack

**Task:** Create `internal/p2p/nat.go`. Configure AutoNAT, DCUtR (hole-punching), and
Circuit Relay v2 reservation per IC §4.3. Set `maxHolePunchRetries = 1` (MVP §8.2).
The `NATType()` method returns the current AutoNAT classification.

**IC §4.3 circuit relay:** Relay reservation TTL = 30 minutes (libp2p default). The daemon
includes relay multiaddrs in heartbeat `current_multiaddrs[]` when a reservation is active
(IC §4.3). Relay overhead must add < 50ms RTT from Indian cloud nodes (IC §4.3 NFR-006)
— this is a milestone gate, not code, and will be validated in M18.

#### Session 6.1.3 — Provider identity persistence

**Task:** Create `internal/p2p/identity.go`. Generate Ed25519 key pair at first startup
via `crypto/rand`. Persist encrypted to the daemon's keystore using `DeriveKeystoreEncKey`
from `internal/crypto` (IC §3.2: "Stored in the daemon's local keystore encrypted under
a key derived from DeriveDKSKeystoreEncKey()"). On subsequent startups, load and decrypt.

---

### Phase 6.2 — Kademlia DHT with Custom HMAC Validator

**Reference:** IC §12 (DHT key contract), IC §5.4 (`DHT` interface), MVP §8.2 (dht_namespace.go),
MVP §8.4 (TestDHTKeyValidatorPersists — CI check 5)

#### Session 6.2.1 — Define the DHT namespace constant

**Task:** Create `internal/p2p/dht_namespace.go` with:
```go
const dhtKeyNamespace = "/vyomanaut/dht-key/1.0.0"
```
This is the sole definition in the entire repository (MVP §8.2). No other file may define
this string inline. The CI namespace pinning requirement (IC §12) means a global search
for this string must find only this file and the registration point in `cmd/provider/main.go`.

#### Session 6.2.2 — Implement custom HMAC key validator

**Task:** Create `internal/p2p/dht.go`. Implement the custom key validator that accepts:
- Any 32-byte key (HMAC-SHA256 output is always 32 bytes per IC §12)

And rejects:
- Keys shorter or longer than 32 bytes
- Keys with a `vyom-chunk:` prefix (plaintext SHA-256 hashes per IC §12)
- libp2p default CID namespace keys (multihash encoding)

Register the validator with namespace `dhtKeyNamespace`. Implement `PutProviderRecord`,
`FindProviders`, `Bootstrap` per IC §5.4. The `PutProviderRecord` pre-condition
(IC §5.4): the daemon must NOT recompute the dht_key at republication time — the key
is pre-computed at upload time and cached locally (IC §5.4, IC §12.2).

#### Session 6.2.3 — Implement `TestDHTKeyValidatorPersists` (CI check 5)

**Task:** In `internal/p2p/dht_test.go`, implement the mandatory CI test from IC §12:
```go
func TestDHTKeyValidatorPersists(t *testing.T) {
    // A valid HMAC-derived key must be accepted
    // A plain CID must be rejected with ErrDHTKeyInvalid
    // A 31-byte key must be rejected with ErrDHTKeyInvalid
}
```
This test catches `go-libp2p` version upgrades that silently reset the namespace
configuration to defaults (IC §12, MVP §8.4 CI check 5). This test MUST pass on every
commit touching `internal/p2p`.

---

### Phase 6.3 — Heartbeat Goroutine

**Reference:** IC §3.1 (heartbeat multiaddr update), IC §12.2 (DHT republication
coordinated with heartbeat)

#### Session 6.3.1 — Implement heartbeat goroutine

**Task:** Create `internal/p2p/heartbeat.go`. Implement the heartbeat loop per IC §3.1
pseudo-code:
- Ticker interval: `profile.HeartbeatInterval` (NOT hardcoded 4h)
- Token refresh check: if `tokenExpiresIn() < 24*time.Hour`, call `refreshToken()` (refer **Session 11.4.3** to call the token refresh endpoint)
- Call `sendHeartbeat()` on each tick
- Cold-start handling: if token is expired and grace period has passed, prompt
  re-registration (IC §3.1)

Also integrate DHT republication per IC §12.2: a separate ticker at
`profile.DHTRepublishInterval` triggers `PutProviderRecord` for all locally-cached
`dht_key` values (IC §12.2 note: the daemon caches dht_keys from upload receipts;
it does NOT recompute them).

---

## Milestone 7 — Audit System (`internal/audit`)

**Deliverable:** Challenge generation, response validation, two-phase crash-safe receipt
write, cluster secret cache, and JIT detection. The audit package does NOT import
`scoring`, `repair`, or `payment` (IC §9 constraint).

**Reference:** IC §5.5 (full package contract), IC §3.2 (Ed25519 signing conventions),
IC §4.2 (audit challenge protocol wire format — this is what the microservice sends;
the provider-side handler is in M8 provider daemon), DM §4.7 (audit_receipts schema),
IC §8 (secrets manager contract), MVP §8.2 (file inventory for internal/audit)

---

### Phase 7.1 — Challenge Nonce Generation

**Reference:** IC §5.5 (`ChallengeNonce`), DM §3 Invariant 5 (33 bytes always),
IC §11 (forbidden: `challenge_nonce` as 32-byte field)

#### Session 7.1.1 — Implement `ChallengeNonce()`

**Task:** In `internal/audit/challenge.go`, implement `ChallengeNonce(serverSecretVN
[]byte, versionByte uint8, chunkID [32]byte, serverTsMs int64) [33]byte` per IC §5.5.
Formula: `nonce = version_byte || HMAC-SHA256(serverSecretVN, chunkID || serverTsMs)`.
Return type is `[33]byte` — a fixed-size array, not a slice — so any code attempting to
use it as a 32-byte value fails at compile time.

Add a CI grep check to `scripts/ci/grep_checks.sh`: fail if `ChallengeNonce` is called
anywhere with a result cast to `[32]byte` (catches accidental truncation).

---

### Phase 7.2 — Response Validation

**Reference:** IC §5.5 (`ValidateResponse` and its explicit limitation note)

#### Session 7.2.1 — Implement `ValidateResponse()`

**Task:** In `internal/audit/validate.go`, implement `ValidateResponse` per IC §5.5.
What this function DOES verify:
1. `len(challengeNonce) == 33`
2. `challengeNonce[0]` identifies a currently-valid secret version
3. `providerSig` is a valid Ed25519 signature

What this function DOES NOT verify (add as a code comment per IC §5.5):
```go
// LIMITATION: The microservice cannot verify that responseHash == SHA-256(chunkData ||
// challengeNonce) because it never holds chunkData. Correctness depends on economic
// deterrence and JIT detection (ADR-014 Defence 3). This is a stated design property.
```

Return `ErrInvalidSignature` or `ErrNonceLength` from IC §5.5 sentinel errors.

---

### Phase 7.3 — Two-Phase Receipt Write

**Reference:** IC §5.5 (`WriteReceiptPhase1`, `WriteReceiptPhase2`, `ErrReceiptAlreadyFinal`),
DM §4.7 (two-phase write semantics), DM §6 (RSP enforces the phase2 update constraint)

#### Session 7.3.1 — Implement `WriteReceiptPhase1()`

**Task:** In `internal/audit/receipt.go`, implement Phase 1 per IC §5.5: INSERT a row
with `audit_result = NULL` and `provider_sig` populated. The INSERT must be WAL-flushed
(use `SELECT pg_wal_flush()` in the same transaction or rely on `synchronous_commit = on`).
Return the assigned `receipt_id` (UUIDv7 generated at the application layer — IC §5.5).

#### Session 7.3.2 — Implement `WriteReceiptPhase2()`

**Task:** Phase 2 per IC §5.5: UPDATE the row identified by `receiptID`, setting
`audit_result`, `service_sig`, `service_countersign_ts` atomically. The `WHERE` clause:
`WHERE receipt_id = $1 AND audit_result IS NULL AND abandoned_at IS NULL`
(matches the RSP in DM §6). Return `ErrReceiptAlreadyFinal` if the row already has a
non-NULL `audit_result` (idempotent retry: IC §5.5 post-condition "treat as success").

---

### Phase 7.4 — Cluster Secret Cache

**Reference:** IC §8 (full secrets manager contract), IC §5.5 (`SecretsManagerClient`
interface, `ErrSecretExpired`), MVP §8.2 (`secret.go`, `secrets_iface.go`)

#### Session 7.4.1 — Implement `SecretsManagerClient` interface and cache

**Task:** Create `internal/audit/secrets_iface.go` with the `SecretsManagerClient`
interface from IC §8. Create `internal/audit/secret.go` with `ClusterSecretCache`:

- 5-minute TTL cache of the cluster audit secret
- Reads `server_secret_vN` and `server_secret_v{N+1}` during rotation overlap window
- Fail-closed on startup: if secrets manager is unreachable, return an error and the
  caller (microservice startup) must refuse to start (IC §8: "fail-closed")
- During operation: return `ErrSecretExpired` after TTL expiry if manager is unreachable;
  caller must back off and not issue challenges (IC §8)

**Local dev / demo mode:** If `RequireSecretsManager == false`, read from
`VYOMANAUT_CLUSTER_MASTER_SEED` environment variable (IC §8). The presence of this env
var in prod mode is caught by the guard in M1 Session 1.3.2.

---

### Phase 7.5 — JIT Detection

**Reference:** IC §4.2 (JIT detection note), DM §4.7 (`jit_flag` column),
MVP §8.2 (`jit.go`)

#### Session 7.5.1 — Implement JIT threshold computation

**Task:** Create `internal/audit/jit.go`. Implement the JIT flag evaluation:
`jit_flag = (response_latency_ms < (chunk_size_kb / p95_throughput_kbps) × 0.3)`.

**Important:** `p95_throughput_kbps` may be NULL for new providers (DM §4.2). When NULL,
skip JIT detection (no flag set). Compute `chunk_size_kb = 256` (always, since
`ShardSize = 262144 = 256 KB` from Invariant 7). Note per IC §4.2: this threshold
"is distinct from the RTO" — document this separation in a code comment.

---

## Milestone 8 — Scoring System (`internal/scoring`)

**Deliverable:** Three-window reliability score, consecutive pass counter and
VETTING→ACTIVE transition, EWMA RTO tracking. Read-only against the database.
Must NOT import `repair` or `payment` (IC §9).

**Reference:** IC §5.6 (full package contract), DM §7 (`mv_provider_scores` view),
MVP §8.2 (file inventory)

---

### Phase 8.1 — Score Retrieval

**Reference:** IC §5.6 (`GetScore`, `ProviderScore` struct)

#### Session 8.1.1 — Implement `GetScore()`

**Task:** In `internal/scoring/score.go`, implement `GetScore()` querying the
`mv_provider_scores` materialised view. The view may be up to 60 seconds stale — this
is acceptable (IC §5.6). Compute `DualWindowFlag = (score30d - score7d > 0.20)` in
application code, not in SQL (easier to unit-test). Return `ErrProviderNotFound` if
the provider does not appear in the view.

Add `GetScoreFromPrimary()` (MVP §8.2: "monthly release multiplier must use primary") —
this version forces a read from the primary replica via connection hint (implementation
detail: pass a different `*sql.DB` handle pointing to the primary).

---

### Phase 8.2 — Consecutive Pass Counter

**Reference:** IC §5.6 (`IncrementConsecutivePasses`, `ResetConsecutivePasses`),
DM §4.2 (`consecutive_audit_passes`), MVP §3.4 (`VettingMinPasses` from profile)

#### Session 8.2.1 — Implement `IncrementConsecutivePasses()` with VETTING→ACTIVE transition

**Task:** In `internal/scoring/passes.go`, implement per IC §5.6. Critical: the
transition check is `profile.VettingMinPasses` (read from NetworkProfile — 80 in prod,
5 in demo), NOT a hardcoded 80. The VETTING→ACTIVE transition also requires the minimum
duration check: `first_chunk_assignment_at + profile.VettingMinDuration <= NOW()`
(IC §5.6 cross-reference to FR-026, MVP §5.4).

Use `SELECT FOR UPDATE` within a transaction to prevent concurrent increments from
both triggering the transition (IC §5.6 goroutine-safe note).

Return `ErrProviderNotVetting` if the provider is not in VETTING status (IC §5.6).

**Note on stale address timeouts (DM §4.7):** `ResetConsecutivePasses` must NOT be
called when `audit_result = 'TIMEOUT' AND address_was_stale = TRUE` — this was added
to `audit_receipts` specifically to prevent false resets on DHT-fallback timeouts.
The scoring package must check this flag before calling `ResetConsecutivePasses`.

---

### Phase 8.3 — EWMA RTO Tracking

**Reference:** IC §4.2 (RTO formula), DM §4.2 (`avg_rtt_ms`, `var_rtt_ms`,
`rto_sample_count`, `p95_throughput_kbps`), MVP §8.2 (`rto.go`)

#### Session 8.3.1 — Implement EWMA update functions

**Task:** In `internal/scoring/rto.go`, implement:

- EWMA update for `avg_rtt_ms` and `var_rtt_ms` after each audit response
- EWMA update for `p95_throughput_kbps` (the 95th-percentile upload throughput)
- `rto_sample_count` increment to track when to switch from pool-median RTO to
  per-provider formula (threshold: `rto_sample_count >= 5` per IC §4.2)

Pool-median RTO fallback: query `AVG(avg_rtt_ms + 4 * var_rtt_ms)` across all ACTIVE
providers when `rto_sample_count < 5`. Cache this pool median for 5 minutes.

**`p95_throughput_kbps` and `avg_rtt_ms` NULL handling (DM §4.2):** Both default to NULL
for new providers. The EWMA update initialises them on the first sample; never write 0
as the initial value (DM §9 checklist: "default to NULL, not 0/2000").

---

## Milestone 9 — Repair System (`internal/repair`)

**Deliverable:** Departure detector, repair job queue with priority ordering, repair
executor, and vetting chunk exclusion. Must NOT import `payment` or `p2p` (IC §9).

**Reference:** IC §5.7 (full package contract), DM §4.10 (`repair_jobs` schema),
DM §3 Invariant 6 (no repair for synthetic chunks)

---

### Phase 9.1 — Repair Job Queue

**Reference:** IC §5.7 (`EnqueueJob`, `DequeueNextJob`, `IsVettingChunk`,
`DeleteVettingChunksOnDeparture`, `MarkJobComplete`)

#### Session 9.1.1 — Implement `EnqueueJob()`

**Task:** In `internal/repair/queue.go`, implement `EnqueueJob()` per IC §5.7. The
priority is derived automatically from `triggerType` (enforcing the
`repair_jobs_priority_matches_trigger` DB CHECK via application logic before INSERT).

**Invariant 6 enforcement (DM §3, IC §5.7 pre-condition):** Calling `EnqueueJob` for
a synthetic vetting chunk panics in debug builds. The caller (departure handler) must
call `IsVettingChunk()` first. Add the check as a debug-mode guard within `EnqueueJob`
as a second line of defence (IC §5.7: "panics in debug builds").

Also enforce the `availableShardCount` range from the active `NetworkProfile` (16–56 in
prod, 3–5 in demo) — pass the profile to the constructor.

#### Session 9.1.2 — Implement `DequeueNextJob()`

**Task:** Use `SELECT ... FOR UPDATE SKIP LOCKED` to atomically dequeue and mark as
`IN_PROGRESS`. Priority ordering: `ORDER BY priority ASC, created_at ASC`. The ENUM
ordering (EMERGENCY < PERMANENT_DEPARTURE < PRE_WARNING alphabetically) determines
priority — verify this matches the intended drain order (DM §4.10 comment: "verify
ENUM order").

#### Session 9.1.3 — Implement `IsVettingChunk()` and `DeleteVettingChunksOnDeparture()`

**Task:** `IsVettingChunk()` per IC §5.7: returns true iff `chunk_assignments` row
for `(chunkID, providerID)` has `is_vetting_chunk = TRUE`.

`DeleteVettingChunksOnDeparture()` per IC §5.7: bulk soft-delete all synthetic
chunk_assignments for a departing vetting provider (`status = 'DELETED'`,
`deleted_at = NOW()`). Zero repair_jobs are created (Invariant 6, FR-065).

---

### Phase 9.2 — Repair Executor

**Reference:** IC §4.4.1 (repair download stream), IC §4.4.2 (repair upload stream),
IC §5.7 (`RepairJob`, `RepairPromotionTimeout`)

#### Session 9.2.1 — Implement repair download client

**Task:** In `internal/repair/executor.go`, implement the microservice-side client for
the `/vyomanaut/repair-download/1.0.0` protocol (IC §4.4.1). The repair scheduler:
1. Contacts `DataShards` (=16 in prod) surviving shard holders
2. For each: opens a stream, sends `RepairDownloadRequest` with `repair_auth_sig`
   (Ed25519 over `SHA-256(chunk_id || request_ts_ms || microservice_peer_id)` per IC §4.4.1)
3. Receives `chunk_data` (262144 bytes)
4. 0-RTT must be PROHIBITED for this stream (IC §4.4.1)
5. Timeout: 10,000ms (IC §4.4.1)

After download: RS decode via `internal/erasure`, generate missing parity shards,
upload via the standard `/vyomanaut/chunk-upload/1.0.0` protocol (IC §4.4.2).

#### Session 9.2.2 — Implement `RepairPromotionTimeout()`

**Task:** Implement the simple accessor per IC §5.7: `return profile.RepairPromotionTimeout`.
The scheduler must call this rather than reading a constant (IC §5.7, ADR-031).

---

### Phase 9.3 — Departure Detector

**Reference:** IC §3.1 (heartbeat effect on departure detection), DM §4.2
(`departed_at`, `providers.status = 'DEPARTED'`)

#### Session 9.3.1 — Implement departure detector loop

**Task:** In `internal/repair/departure.go`, implement the departure detector that
periodically runs:
```sql
SELECT provider_id FROM providers
WHERE status = 'ACTIVE'
  AND last_heartbeat_ts < NOW() - $1  -- profile.DepartureThreshold
```
For each departed provider:
1. Set `status = 'DEPARTED'`, `frozen = TRUE`, `departed_at = NOW()`
2. Call `IsVettingChunk()` for each assigned chunk — if true, call
   `DeleteVettingChunksOnDeparture()` instead of `EnqueueJob()`
3. Call `payment.Penalise()` (the departure handler wires payment — but repair package
   cannot import payment; this call is in the microservice entrypoint that orchestrates both)
4. Call `EnqueueJob()` for each real shard (Invariant 3: no physical deletion)

**IC §6 DML constraint:** `UPDATE providers SET status = 'DEPARTED'` is only permitted by
`vyomanaut_app` (IC §6 `providers` table row).

---

## Milestone 10 — Payment System (`internal/payment`)

**Deliverable:** `PaymentProvider` interface, mock provider, Razorpay live implementation,
append-only escrow ledger, release computation, seizure. Zero float arithmetic. Must NOT
import `repair` or `p2p` (IC §9). `TestNoFloatArithmetic` must pass (CI check 6).

**Reference:** IC §5.8 (full package contract, INVARIANT block), DM §3 Invariant 4
(all amounts in integer paise), DM §4.8 (escrow_events schema), IC §7 (Razorpay webhook
contracts), IC §11 (float arithmetic forbidden in internal/payment/)

---

### Phase 10.1 — PaymentProvider Interface & Mock

**Reference:** IC §5.8 (`PaymentProvider` interface)

#### Session 10.1.1 — Define `PaymentProvider` interface

**Task:** Create `internal/payment/provider.go` with the `PaymentProvider` interface
from IC §5.8. All `amountPaise` parameters must be typed as `int64` — never `float64`
or `float32` (Invariant 4).

#### Session 10.1.2 — Implement mock payment provider

**Task:** Create `internal/payment/mock.go`. The mock implements `PaymentProvider` fully
(MVP §6.1 CR-10: "The mock must implement the interface fully, not bypass it"). Mock
behaviour:
- `InitiateEscrow`: stores in-memory, returns a fake VPA
- `ReleaseEscrow`: logs the payout, deduplicates on `idempotencyKey`
- `Penalise`: marks the provider as seized in-memory
- `GetBalance`: computes from in-memory events

**Idempotency in mock:** The `escrow_events.idempotency_key` UNIQUE constraint at the DB
layer enforces idempotency for the real provider; the mock must also enforce it
(MVP §7.7: "Idempotency is DB-enforced regardless of payment provider implementation").

---

### Phase 10.2 — Escrow Ledger

**Reference:** IC §5.8 (`InsertEscrowEvent`, `EscrowEventType`, `ErrDuplicateIdempotencyKey`),
DM §4.8

#### Session 10.2.1 — Implement `InsertEscrowEvent()`

**Task:** Create `internal/payment/ledger.go`. `InsertEscrowEvent()` is the only
permitted write to `escrow_events` (Invariant 2). The function must:
- Accept `amountPaise int64` — never accept float via a type check or debug panic
- Handle `ErrDuplicateIdempotencyKey` on UNIQUE constraint violation (idempotent retry)

Define `EscrowEventType` constants per IC §5.8: `EscrowDeposit`, `EscrowRelease`,
`EscrowSeizure`, and `EscrowReversal` (note: IC §5.8 has a duplicate `EscrowSeizure`
constant name for REVERSAL — this is an error in the document. Use `EscrowReversal`
for the `"REVERSAL"` event type per DM §4.8 and DM §7 which adds REVERSAL).

**Document the IC §5.8 inconsistency:** Add a code comment: "IC §5.8 contains a
duplicate constant name for REVERSAL (shows EscrowSeizure twice). This implementation
uses EscrowReversal for the REVERSAL event type per DM §4.8. The inconsistency in
IC §5.8 requires a correction PR."

---

### Phase 10.3 — Razorpay Implementation

**Reference:** IC §7 (all three webhook contracts), IC §3.3 (RAZORPAY_UNAVAILABLE error code)

#### Session 10.3.1 — Implement Razorpay webhook handlers

**Task:** Create `internal/payment/razorpay.go`. Implement handlers for all three
Razorpay webhooks from IC §7:

**`virtual_account.payment.captured` (IC §7.1):**
- Verify `X-Razorpay-Signature` before any DB write
- `amount_paise` is ALREADY in paise from Razorpay — do NOT multiply (IC §7.1: "already in paise; do not multiply")
- Use the SQL in IC §7.1 with `ON CONFLICT DO NOTHING`

**`payout.reversed` (IC §7.2):**
- Insert a `REVERSAL` event (not a negative RELEASE — IC §7.2)
- `idempotency_key = SHA-256('reversal' || original_idempotency_key)` (DM §7, IC §7.2)

**`account.created` (IC §7.3):**
- Set `razorpay_cooling_until = NOW() + profile.RazorpayCoolingPeriod`
  (profile-variable, NOT hardcoded 24h — MVP §5.4)
- The `AND razorpay_linked_account_id IS NULL` idempotency guard (IC §7.3)

**UPI Collect prohibition (IC §11):** Any call to the Razorpay Collect API endpoint is
forbidden. All deposit flows use UPI Intent only (deprecated 28 February 2026).

---

### Phase 10.4 — Release Computation & Seizure

**Reference:** IC §5.8 (`ReleaseEscrow`, `Penalise`), MVP §5.4 (release computation
cycle: calendar date in prod, ticker in demo), DM §4.2 (`frozen` column)

#### Session 10.4.1 — Implement monthly release computation

**Task:** Create `internal/payment/release.go`. The release computation:
1. For each provider with `release_computed = FALSE` for the current audit period
2. Compute the release multiplier from the score (score-to-multiplier table — ⚠️ AWAITING
   `requirements.md` for the exact multiplier table; use a placeholder linear function
   and mark `// TODO: replace with exact multiplier table from requirements.md §...`)
3. Call `ReleaseEscrow()` with idempotency key `SHA-256(provider_id || audit_period_id)`
4. Set `release_computed = TRUE`

In demo mode, the computation fires on `profile.ReleaseComputationInterval` ticker
(MVP §5.4). In prod mode, it fires on the 23rd of each month calendar trigger
(`profile.ReleaseComputationInterval == 0` means calendar-driven — MVP §5.4).

#### Session 10.4.2 — Implement `TestNoFloatArithmetic` (CI check 6)

**Task:** Create `internal/payment/payment_test.go` with `TestNoFloatArithmetic` (CI
check 6, MVP §8.4). This test uses `go/ast` to parse every `.go` file in
`internal/payment/` and assert that no `float64`, `float32`, `FLOAT`, `DECIMAL`, or
`NUMERIC` identifier appears in any expression. Disabling or weakening this test is
itself a prohibited change (IC §11).

---

## Milestone 11 — REST API Layer

**Status:** The error envelope, readiness gate, and Razorpay webhook handler can be
implemented from the documents in context. All other endpoint schemas require
`openapi.yaml`. Stub handlers that return `501 Not Implemented` must be committed for
every endpoint referenced in IC and DM so that the routing tree is established.

**Reference:** OAS (all sections) IC §3 (all REST contracts), IC §3.3 (error envelope), IC §3.4 (readiness
gate), IC §7 (webhook contracts implemented in M10)

---

### Phase 11.1 — Error Envelope

**Reference:** OAS `components/schemas/Error.properties`, IC §3.3 (standard error body, HTTP status code table)

#### Session 11.1.1 — Implement standard error response

**Task:** Create `internal/api/errors.go` (new package `internal/api` for HTTP-layer
concerns). Implement a `WriteError(w http.ResponseWriter, statusCode int, errorCode string,
message string, retryAfter *int)` function that writes the exact JSON body from IC §3.3:

```json
{
  "error_code": "...",
  "message": "...",
  "request_id": "...",
  "retry_after": null
}
```

The OAS confirms `retry_after` is `integer` (seconds), `nullable: true`, and is non-null for HTTP 429 and HTTP 503 responses with a known backoff (OAS `components/schemas/Error.properties.retry_after`). Add `field` (string, for validation errors) and `details` (object) as additional optional fields on the error struct per OAS

The `request_id` must be a UUIDv7 matching the `X-Request-ID` response header (IC §3.3).
Implement all error codes from the IC §3.3 table as typed constants.

**`INSUFFICIENT_ASN_DIVERSITY` special case (IC §3.3):** the `available_asns` field is a sibling of `error_code` inside the standard Error schema, not a wrapper type. Implement as an optional field on the shared error struct, populated only when `error_code == INSUFFICIENT_ASN_DIVERSITY`. (Resolved by OAS `components/schemas/Error.properties.available_asns`)

---

### Phase 11.2 — Readiness Gate Endpoint

**Reference:** OAS `components/schemas/ReadinessResponse`, IC §3.4 (readiness gate contract, exact response body)

#### Session 11.2.1 — Implement `GET /api/v1/admin/readiness`

**Task:** In `internal/api/readiness.go`, implement the readiness gate evaluator and
HTTP handler per IC §3.4. All seven conditions must be evaluated simultaneously:

1. Active vetted providers: `COUNT(*) WHERE status IN ('VETTING','ACTIVE') >= profile.MinActiveProviders`
2. Distinct ASNs: `COUNT(DISTINCT asn) >= profile.MinDistinctASNs`
3. Distinct metro regions: `COUNT(DISTINCT region) >= profile.MinMetroRegions`
4. Quorum: depends on `profile.RequireQuorum` — ⚠️ AWAITING `architecture.md` for
   gossip cluster details; stub as `true` when `profile.RequireQuorum == false`
5. Razorpay cooled accounts: **LIVE QUERY** every 60-second cycle — must NOT cache
   (IC §3.4: "MUST NOT cache this value")
6. Relay nodes: `COUNT(relay heartbeat entries) >= profile.MinRelayNodes`
7. Cluster audit secret: `clusterSecretCache.IsLoaded()`

Response body must match IC §3.4 exactly. The `"threshold"` field is populated from
`NetworkProfile.Min*` fields, not hardcoded constants (IC §3.4: "must not assume the
value is always 56").

**ReadinessResponse field set from OAS `components/schemas/ReadinessResponse`:**

- `all_conditions_met` (bool)
- `evaluated_at` (date-time)
- `mode` (`"demo"` | `"prod"`) — must reflect active `NetworkProfile`
- `conditions` object with exactly these seven keys, each a `ReadinessCondition`:
  - `active_vetted_providers` — threshold = `NetworkProfile.MinActiveProviders`
  - `distinct_asns` — threshold = `NetworkProfile.MinDistinctASNs`
  - `distinct_metro_regions` — threshold = `NetworkProfile.MinMetroRegions`
  - `microservice_quorum` — threshold: prod=3, demo=1 (`RequireQuorum=false`)
  - `razorpay_accounts_ready` — threshold = `NetworkProfile.MinCooledAccounts`; **LIVE QUERY always** (OAS description: "MUST NOT cache this value")
  - `relay_nodes_deployed` — threshold = `NetworkProfile.MinRelayNodes`
  - `cluster_audit_secret_loaded` — bool from `ClusterSecretCache.IsLoaded()`

Each `ReadinessCondition` carries: `name` (string), `satisfied` (bool), `current_value` (int), `required_value` (int), and `demo_value` (int, nullable — non-null only in demo mode) (OAS `components/schemas/ReadinessCondition`).

---

### Phase 11.3 — Endpoint Routing Stubs

**Reference:** IC §2 (component communication map — lists all permitted endpoints),
IC §3 (REST contract cross-references)

#### Session 11.3.1 — Establish HTTP routing tree with stubs

**Task:** Create `internal/api/router.go` defining the full routing tree. For each
endpoint referenced in IC §2 and IC §3 cross-reference table, create a stub handler
returning `501 Not Implemented` with error_code `INTERNAL_ERROR`. The purpose is to
establish the URL structure so that `openapi.yaml` can be validated against it once
available.

| Method | Path | OAS operationId | Auth |
| --- | --- | --- | --- |
| GET | `/.well-known/jwks.json` | `getJwks` | public |
| POST | `/api/v1/auth/otp/send` | `sendOtp` | public |
| POST | `/api/v1/auth/otp/verify` | `verifyOtp` | public |
| POST | `/api/v1/owner/register` | `registerOwner` | BearerAuth |
| POST | `/api/v1/owner/deposit` | `initiateDeposit` | BearerAuth |
| GET | `/api/v1/owner/{owner_id}/balance` | `getOwnerBalance` | BearerAuth |
| GET | `/api/v1/owner/{owner_id}/files` | `listOwnerFiles` | BearerAuth |
| GET | `/api/v1/owner/{owner_id}/escrow` | `getOwnerEscrowHistory` | BearerAuth |
| POST | `/api/v1/owner/withdraw` | `withdrawOwnerEscrow` | BearerAuth |
| POST | `/api/v1/provider/register` | `registerProvider` | BearerAuth |
| POST | `/api/v1/provider/heartbeat` | `providerHeartbeat` | BearerAuth |
| POST | `/api/v1/provider/token/refresh` | `refreshProviderToken` | BearerAuth (grace) |
| GET | `/api/v1/provider/{provider_id}/status` | `getProviderStatus` | BearerAuth |
| GET | `/api/v1/provider/receipts` | `listProviderReceipts` | BearerAuth |
| POST | `/api/v1/provider/downtime` | `announceDowntime` | BearerAuth |
| GET | `/api/v1/provider/downtime` | `getActiveDowntime` | BearerAuth |
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
| POST | `/webhooks/razorpay` | *(IC §7, no OAS path)* | signature |

**AdminApiKey middleware note (OAS `components/securitySchemes/AdminApiKey`):** `X-Admin-API-Key` header; must be ≥ 32 random bytes hex-encoded. Add a middleware that verifies this header on all `/api/v1/admin/*` and `/api/v1/audit/challenge` routes.

---

### Phase 11.4 — Authentication Endpoints

#### Session 11.4.1 — OTP Send (`POST /api/v1/auth/otp/send`)

**Reference:** OAS `paths./api/v1/auth/otp/send`, `OtpSendRequest`, `OtpSendResponse`

Implement rate limiting: 3 OTP attempts per phone number per 10 minutes. On breach return HTTP 429 with `Retry-After: 600` and `error_code: OTP_RATE_LIMITED`. Response body: `expires_at` (OTP TTL, typically 10 minutes from issuance). Authentication: public endpoint — no JWT required.

#### Session 11.4.2 — OTP Verify (`POST /api/v1/auth/otp/verify`)

**Reference:** OAS `paths./api/v1/auth/otp/verify`, `OtpVerifyRequest`, `OtpVerifyResponse`

On success: issue Ed25519-signed JWT. If phone not previously registered (`is_new_entity: true`): issue 1-hour registration token; `role` field in response is `null`. If already registered (`is_new_entity: false`): issue 24-hour (owner) or 7-day (provider) JWT; populate `entity_id` and `role`.

#### Session 11.4.3 — Provider Token Refresh (`POST /api/v1/provider/token/refresh`)

**Reference:** OAS `paths./api/v1/provider/token/refresh`, operationId `refreshProviderToken`

Two-factor auth: (1) current JWT accepted if valid **or** expired within the last 1 hour (grace window); (2) Ed25519 body signature over `SHA-256(provider_id ‖ timestamp_string)`. Rate limit: one successful refresh per 30 minutes per `provider_id` (HTTP 429 with `Retry-After` on breach, `error_code: TOKEN_REFRESH_RATE_LIMITED`). On success: issue new 7-day JWT; return `token` and `expires_at`.

This session also requires for **Session 6.3.1** (heartbeat goroutine) to call this endpoint when `tokenExpiresIn() < 24*time.Hour`, using the daemon's registered Ed25519 key to produce the body signature.

---

### Phase 11.5 — Owner Endpoints

#### Session 11.5.1 — Owner Register (`POST /api/v1/owner/register`)

**Reference:** OAS `paths./api/v1/owner/register`, `OwnerRegisterRequest`, `OwnerRegisterResponse`

Verify `owner_sig`: Ed25519 over `{"ed25519_public_key": "<value>"}` (sorted keys, no trailing whitespace). Store `ed25519_public_key` against `owner_id`. Return 201 with new 24-hour JWT (`role=owner`, `sub=owner_id`). Return 409 `PHONE_ALREADY_REGISTERED` if phone already has an owner account.

#### Session 11.5.2 — Deposit Initiate (`POST /api/v1/owner/deposit`)

**Reference:** OAS `paths./api/v1/owner/deposit`, `DepositInitiateRequest`, `DepositInitiateResponse`

Triggers Razorpay Smart Collect 2.0 VPA creation. Returns `vpa`, `qr_code_url`, `expires_at`. No `escrow_events` row is written here — the row is written only on receipt of the `virtual_account.payment.captured` webhook (M10 Phase 10.3.1). UPI Collect MUST NOT be used (OAS description, NFR-029).

#### Session 11.5.3 — Owner Balance (`GET /api/v1/owner/{owner_id}/balance`)

**Reference:** OAS `paths./api/v1/owner/{owner_id}/balance`, `OwnerBalance`

Read from `mv_owner_escrow_balance` (up to 60s stale). Return `balance_paise`, `reserved_next_30d_paise`, `available_paise` (= balance − reserved). All values `int64`, minimum 0.

#### Session 11.5.4 — Owner File List (`GET /api/v1/owner/{owner_id}/files`)

**Reference:** OAS `paths./api/v1/owner/{owner_id}/files`, `FileListItem`

Query params: `status` (default `ACTIVE`), `limit` (1–100, default 50), `cursor`. Availability field derived from `mv_segment_shard_counts`: `OK` ≥ 24 shards, `DEGRADED` < 24, `CRITICAL` < 16 (OAS `FileListItem.properties.availability`). Return `display_name_ciphertext/nonce/tag` when present (FR-019).

#### Session 11.5.5 — Owner Escrow History (`GET /api/v1/owner/{owner_id}/escrow`)

**Reference:** OAS `paths./api/v1/owner/{owner_id}/escrow`, `OwnerEscrowTransaction`

Paginated `owner_escrow_events` log. Response envelope includes balance summary fields (`balance_paise`, `reserved_next_30d_paise`, `available_paise`) alongside the `events` array and `next_cursor`.

#### Session 11.5.6 — Owner Withdraw (`POST /api/v1/owner/withdraw`)

**Reference:** OAS `paths./api/v1/owner/withdraw`, `WithdrawRequest`, `WithdrawResponse`

`idempotency_key` format: `SHA-256(owner_id ‖ withdrawal_request_id)`, hex-encoded (OAS `WithdrawRequest.properties.idempotency_key`). Block withdrawal while any file upload is in-flight (FR-059). On success: return Razorpay `payout_id` and initial status (`QUEUED`). Final `REVERSED` status arrives via the `payout.reversed` webhook (M10 Phase 10.3.1).

---

### Phase 11.6 — Provider REST Endpoints

#### Session 11.6.1 — Provider Register (`POST /api/v1/provider/register`)

**Reference:** OAS `paths./api/v1/provider/register`, `ProviderRegisterRequest`, `ProviderRegisterResponse`

Verify `provider_sig` over canonical JSON of all other fields (sorted keys, `provider_sig` omitted). Store `ed25519_public_key`, `declared_storage_gb`, `city`, `region`, `asn`, `initial_multiaddrs`. In demo mode: accept `demo_asn` field (pattern `^SIM-AS\d+$`); if omitted, auto-assign next available from pool (OAS description, FR-068). In production: silently ignore `demo_asn`. Set `razorpay_cooling_until = NOW() + profile.RazorpayCoolingPeriod`. Return status `PENDING_ONBOARDING` and 7-day JWT.

#### Session 11.6.2 — Provider Heartbeat (`POST /api/v1/provider/heartbeat`)

**Reference:** OAS `paths./api/v1/provider/heartbeat`, `HeartbeatRequest`, `HeartbeatResponse`

Verify `provider_sig` over canonical JSON body (sorted keys, `provider_sig` omitted). Reject timestamps with skew > ±5 minutes. Update `providers.current_multiaddrs` and `last_heartbeat_ts`. Return `received_at` and `microservice_sig` (Ed25519 countersignature over `{"received_at":"<value>","provider_id":"<value>"}` sorted keys). Daemon stores `microservice_sig` as proof of acknowledgement.

#### Session 11.6.3 — Provider Status (`GET /api/v1/provider/{provider_id}/status`)

**Reference:** OAS `paths./api/v1/provider/{provider_id}/status`, `ProviderStatusResponse`

Join `providers` table with `mv_provider_scores` (up to 60s stale). Include vetting fields when `status = 'VETTING'`: `vetting_chunks_assigned`, `vetting_chunk_cap` (= `floor(declared_storage_gb × 400)`), `vetting_eligible_at` (= `first_chunk_assignment_at + 120 days`), `vetting_gc_pending`. Include `network_mode` field (`"demo"` | `"prod"`) from active profile (OAS `ProviderStatusResponse.properties.network_mode`).

#### Session 11.6.4 — Provider Audit Receipts (`GET /api/v1/provider/receipts`)

**Reference:** OAS `paths./api/v1/provider/receipts`, `AuditReceiptListItem`

Filter params: `chunk_id`, `from`, `to`, `result` (PASS/FAIL/TIMEOUT), `limit` (1–200, default 100), `cursor`. Available even when `status = 'DEPARTED'` (FR-058). Response includes both `provider_sig` and `service_sig` per receipt. `file_id` is nullable on receipts for vetting chunks.

#### Session 11.6.5 — Provider Downtime (`POST /api/v1/provider/downtime` and `GET /api/v1/provider/downtime`)

**Reference:** OAS `paths./api/v1/provider/downtime`, `ProviderDowntimeRequest`, `ProviderDowntimeResponse`

POST: `promised_return_at` must be between 0 and 72 hours from now; past timestamps rejected HTTP 400. Verify `provider_sig`. Return 409 `DOWNTIME_ALREADY_ACTIVE` if a window is already open. Return 409 `PROVIDER_DEPARTED` if status is DEPARTED. `penalty_fires_at` = `promised_return_at`.

GET: Returns `{active: bool, promised_return_at: date-time|null, penalty_fires_at: date-time|null}`.

#### Session 11.6.6 — Provider Depart (`POST /api/v1/provider/depart`)

**Reference:** OAS `paths./api/v1/provider/depart`, `ProviderDepartRequest`, `ProviderDepartResponse`

Verify `provider_sig`. Accept optional `depart_at` (if omitted or in past: immediate). Return `repair_jobs_queued` and `escrow_release_paise` (proportional to fraction of 30-day window completed). This endpoint wires M9 `repair.EnqueueJob()` and M10 `payment.ReleaseEscrow()` in the microservice entrypoint — consistent with the import constraint that prevents `repair` importing `payment` (IC §9).

---

### Phase 11.7 — Upload and File Endpoints

#### Session 11.7.1 — Upload Assign (`POST /api/v1/upload/assign`)

**Reference:** OAS `paths./api/v1/upload/assign`, `UploadAssignRequest`, `UploadAssignResponse`, `SegmentAssignment`, `ShardAssignment`

Checks (in order):

1. Readiness gate — HTTP 503 `NETWORK_NOT_READY` with `retry_after: 60` and `readiness_url` field if not satisfied.
2. Escrow balance ≥ `cost_for_30_days(original_size_bytes)` — HTTP 409 `INSUFFICIENT_ESCROW_BALANCE` if not.
3. ASN cap: no single ASN > 20% of 56 shards per segment — HTTP 503 `INSUFFICIENT_ASN_DIVERSITY` with `available_asns` field and `retry_after: 300` if unsatisfiable (OAS `paths./api/v1/upload/assign.post.responses.503`).

Response includes `monthly_cost_paise`, `required_escrow_paise`, and per-shard `capability_token` (72-byte hex, 144 chars) in each `ShardAssignment`. Idempotent on `file_id`. `shard_index` 0–15 = data shards, 16–55 = parity shards (OAS `ShardAssignment.properties.shard_index`).

#### Session 11.7.2 — File Register (`POST /api/v1/file/register`)

**Reference:** OAS `paths./api/v1/file/register`, `FileRegisterRequest`, `FileRegisterResponse`

Verify `owner_sig` over canonical JSON of all other fields. Store `pointer_ciphertext`, `pointer_nonce`, `pointer_tag`, `original_size_bytes`, `schema_version` (must be 1). Optionally store `display_name_ciphertext/nonce/tag` (FR-019). Return 404 if `file_id` was not created by a prior assign call. Return 409 `FILE_ALREADY_DELETED` if file exists in ACTIVE status (re-registration not permitted).

#### Session 11.7.3 — Pointer File Retrieval (`GET /api/v1/file/{file_id}/pointer`)

**Reference:** OAS `paths./api/v1/file/{file_id}/pointer`, `PointerFileResponse`

Return `pointer_ciphertext`, `pointer_nonce`, `pointer_tag`, `schema_version`, `original_size_bytes`, and optional `display_name_*` fields. Client decrypts locally; microservice never decrypts.

#### Session 11.7.4 — File Delete (`DELETE /api/v1/file/{file_id}`)

**Reference:** OAS `paths./api/v1/file/{file_id}.delete`

Mark all `chunk_assignments` as `PENDING_DELETION`. Notify reachable providers immediately. Return `assignments_marked`, `providers_notified`, `providers_pending`, `status: "DELETED"`. Return 409 `FILE_ALREADY_DELETED` if already deleted (OAS `paths./api/v1/file/{file_id}.delete.responses.409`).

---

### Phase 11.8 — Pricing Endpoints

#### Session 11.8.1 — Storage Pricing Estimate (`GET /api/v1/pricing/estimate`)

**Reference:** OAS `paths./api/v1/pricing/estimate`, `PricingEstimateResponse`

Public endpoint (no auth). Optional `file_size_bytes` query param; default to 1 GB if omitted. Return `storage_rate_paise_per_gb_per_month`, `monthly_cost_paise`, `annual_cost_paise` (= monthly × 12), `min_escrow_balance_paise` (= 30 days cost). All values `int64` paise.

#### Session 11.8.2 — Provider Earnings Estimate (`GET /api/v1/pricing/provider-estimate`)

**Reference:** OAS `paths./api/v1/pricing/provider-estimate`

Public endpoint. Required params: `storage_gb` (int, 10–100000) and `uptime_target_pct` (float, 0–100). Return `gross_monthly_paise` (= `storage_gb × rate × uptime_pct/100`), `estimated_escrow_hold_vetting_paise` (30% hold during vetting, ADR-024), `estimated_net_paise_vetting` (= gross × 0.50 release cap, FR-051), `estimated_net_paise_post_vetting` (= gross × release_multiplier at ≥0.95 score).

---

### Phase 11.9 — Audit Admin Endpoints

#### Session 11.9.1 — Manual Audit Challenge (`POST /api/v1/audit/challenge`)

**Reference:** OAS `paths./api/v1/audit/challenge`, `AuditChallengeDispatchRequest`, `AuditChallengeDispatchResponse`

AdminApiKey auth. Dispatches an immediate out-of-cycle challenge to `(provider_id, chunk_id)`. Does not bypass the 24-hour deduplication window. Returns HTTP 202 with `challenge_nonce` (33-byte / 66 hex chars), `server_challenge_ts`, and `deadline_ms` (= `ceil((256 / p95_throughput_kbps) × 1500)`). Returns 403 if provider is `DEPARTED`.

---

### Phase 11.10 — Admin Endpoints

#### Session 11.10.1 — Admin Providers List (`GET /api/v1/admin/providers`)

**Reference:** OAS `paths./api/v1/admin/providers`, `AdminProvidersResponse`, `AdminProviderItem`

Filter params: `status`, `asn`, `region`, `multiaddr_stale` (bool), `accelerated_reaudit` (bool), `vetting_gc_pending` (bool — filter to ACTIVE providers with incomplete GC, ADR-030), `limit` (1–200, default 100), `cursor`. Include `vetting_chunks_assigned`, `vetting_chunk_cap`, `vetting_gc_pending` fields for VETTING providers (OAS `AdminProviderItem`).

#### Session 11.10.2 — Repair Queue (`GET /api/v1/admin/repair/queue`)

**Reference:** OAS `paths./api/v1/admin/repair/queue`, `RepairQueueResponse`, `RepairJobItem`

Filter params: `status` (default QUEUED), `priority`, `limit` (1–100, default 50), `cursor`. Return counts split by priority: `emergency_queued`, `permanent_departure_queued`, `pre_warning_queued`, `total_queued`. Alert threshold: `total_queued > 1000`; `emergency_queued > 0` fires immediately (OAS `RepairQueueResponse.properties.emergency_queued` description, NFR-027).

#### Session 11.10.3 — Manual Repair Trigger (`POST /api/v1/admin/repair/trigger`)

**Reference:** OAS `paths./api/v1/admin/repair/trigger`

Required body fields: `chunk_id`, `segment_id`, `trigger_type`. Optional: `provider_id` (nullable for threshold-triggered repairs). Return 202 with the created `RepairJobItem`. Wires directly to M9 `repair.EnqueueJob()`.

#### Session 11.10.4 — Audit Statistics (`GET /api/v1/admin/audit/stats`)

**Reference:** OAS `paths./api/v1/admin/audit/stats`, `AuditStatsResponse`

Query params: `from` (default NOW()-1h), `to` (default NOW()), `provider_id` (optional). Backed by `audit_receipts WHERE abandoned_at IS NULL`. Return `challenges_issued`, breakdown `{pass, fail, timeout, pending}`, `pass_rate` (= pass/(pass+fail+timeout)), `timeout_rate`, `content_hash_failures` (0x02 status codes), `jit_flags_raised`. Alert: `timeout_rate > 0.05` (NFR-027).

#### Session 11.10.5 — Vetting Status (`GET /api/v1/admin/vetting/status`)

**Reference:** OAS `paths./api/v1/admin/vetting/status`, `VettingStatusSummary`

Query param: `include_gc_pending_only` (bool, default false). Return `total_vetting_providers`, `total_synthetic_chunks_active`, `total_synthetic_chunks_pending_gc`, and per-provider `vetting_summary` (OAS `VettingStatusSummary`: `chunks_assigned`, `chunk_cap`, `cap_utilisation_pct`, `chunks_pending_gc`).

#### Session 11.10.6 — Vetting GC Retry (`POST /api/v1/admin/vetting/gc/retry`)

**Reference:** OAS `paths./api/v1/admin/vetting/gc/retry`

Required body: `provider_id` (must be ACTIVE with `chunks_pending_gc > 0`). Attempt libp2p connection and GC instruction delivery immediately. Return `delivery_attempted` (false if provider unreachable), `chunks_pending_gc_before`, `chunks_pending_gc_after`. If provider offline: return 200 with `delivery_attempted: false`; background retry continues. Wires to M14 `vettingchunk.DeliverGCInstruction()`.

---

## Milestone 12 — Coordination Microservice (`cmd/microservice`)

**Deliverable:** Fully wired microservice that starts up, passes guard rails, loads the
cluster secret, passes the readiness gate (in demo mode), and handles the provider
heartbeat. Full REST API implementation awaits `openapi.yaml`.

**Reference:** MVP §2.3 (startup guard rails), IC §8 (secrets manager contract),
IC §3.4 (readiness gate), IC §9 (import graph: microservice entrypoint wires audit,
scoring, repair, payment), IC §2 (component communication map)

---

### Phase 12.1 — Microservice Startup

**Reference:** MVP §5.3 (profile selection), IC §8 (startup fail-closed on secret),
MVP §2.3 (guard rails)

#### Session 12.1.1 — Wire microservice `main()`

**Task:** Replace the stub in `cmd/microservice/main.go` with full startup wiring:

1. Call `config.SelectProfile()` — print startup banner
2. Call `config.ValidateStartupGuards(profile)` — halt on violation
3. Initialise `ClusterSecretCache` — fail-closed if `RequireSecretsManager == true` and secrets manager is unreachable (IC §8)
4. Initialise database connection pool (PostgreSQL)
5. Drop and recreate `mv_provider_scores` view using `profile.ScoreWindow*` values (DM §7, MVP §5.5 — this is an application-layer step at startup, not a migration)
6. Start the readiness gate evaluator goroutine (60-second cycle per IC §3.4)
7. Start the HTTP server with the routing tree from Phase 11.3
8. Start the audit challenge scheduler goroutine (interval: `profile.PollingInterval`)
9. Start the departure detector goroutine
10. Start the release computation goroutine (ticker or calendar per profile)

**The authoritative JWT claim set from OAS `components/securitySchemes/BearerAuth`:**

- `sub` = entity UUID
- `role` = `"owner"` | `"provider"`
- `iss` = `"vyomanaut-microservice-v1"`
- `exp` = Unix timestamp

TTL: 24 hours for owners, 7 days for provider daemons (OAS `OtpVerifyResponse.properties.token` description).

Registration token TTL (short-lived, single-use): 1 hour, valid only for the matching register call (OAS `OtpVerifyResponse.properties.token` description, `is_new_entity = true` case).

**⚠️ AWAITING architecture.md** for: gossip cluster initialisation, replica synchronisation.
Stub with `if profile.RequireQuorum { log.Fatal("quorum: awaiting architecture.md") }`.

#### Session 12.1.2 — Audit challenge dispatch loop

**Task:** In `cmd/microservice/`, implement the audit challenge dispatch logic.
For each active chunk in `active_chunk_assignments`:
1. Generate nonce via `audit.ChallengeNonce()`
2. Determine provider multiaddrs (heartbeat record primary, DHT fallback if `multiaddr_stale = true`)
3. Open libp2p stream to provider via `/vyomanaut/audit-challenge/1.0.0`
4. Send `ChallengeRequest` frame per IC §4.2 field table (73-byte payload)
5. Apply per-provider RTO timeout (IC §4.2)
6. Call `audit.WriteReceiptPhase1()` before dispatching
7. On response: call `audit.ValidateResponse()`, then `audit.WriteReceiptPhase2()`
8. Call `scoring.IncrementConsecutivePasses()` or `scoring.ResetConsecutivePasses()`
   depending on result, with the `address_was_stale` check
9. Update EWMA metrics via `scoring.UpdateRTO()`

**Concurrency (IC §4.2):** The microservice may open multiple concurrent challenge streams
to a single provider. Use a goroutine per chunk assignment, bounded by a semaphore.

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

**⚠️ PARTIAL MILESTONE — AWAITING `architecture.md`**

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

**⚠️ AWAITING `architecture.md`**

#### Session 17.2.1 — STUB: Gossip cluster initialisation

**Task:** Stub `internal/cluster/gossip.go` with `// TODO: implement per architecture.md
§{section} — requires gossip membership, quorum (3,2,2) configuration, and replica sync
for cluster audit secret`. The stub must compile and return `ErrNotImplemented`. The
readiness gate `quorum_healthy` condition must return `false` until this is implemented.

#### Session 17.2.2 — STUB: Relay node deployment configuration

**Task:** Stub `deployments/production/relay/` with a placeholder `README.md`:
"Relay node configuration awaits architecture.md §{section} — relay nodes are
Vyomanaut-operated, co-located in Indian cloud regions, each supporting 128 concurrent
reservations (IC §4.3 capacity.md §5.2)." The 50ms RTT overhead constraint (IC §4.3
NFR-006) is a validation gate in M18.

---

## Milestone 18 — Launch Readiness

**⚠️ PARTIAL MILESTONE — AWAITING `requirements.md`**

**Status:** Runbooks and benchmark scripts are structurally defined by document references.
FR/NFR completion gates require `requirements.md`.

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

Each script must output pass/fail against a defined threshold. These are launch gate
criteria per `requirements.md §7.4` (⚠️ exact thresholds await `requirements.md`; use
placeholder thresholds from IC pre-condition comments as temporary values).

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

**⚠️ AWAITING `requirements.md`:** Additional security NFR items from the requirements
document must be added to this checklist when shared.

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

## Appendix A — Document Gap Register

The following table records every implementation decision deferred pending missing
documents. Each entry must be resolved before the corresponding session is closed.

| Gap ID | Session | Missing Document | Blocked Decision |
| --- | --- | --- | --- |
| GAP-001 | M8 Phase 8.2 | architecture.md | Gossip cluster quorum check in readiness gate |
| GAP-002 | M10 Phase 10.4.1 | requirements.md | Exact score-to-release-multiplier table |
| GAP-004 | M12 Phase 12.1.1 | architecture.md | Gossip cluster initialisation (replicas 3,2,2) |
| GAP-005 | M17 Phase 17.2.1 | architecture.md | HA microservice replica sync and relay deployment |
| GAP-006 | M18 Phase 18.2.1 | requirements.md | Exact benchmark thresholds (requirements.md §7.4) |
| GAP-007 | M18 Phase 18.3.1 | requirements.md | Additional security NFR checklist items |

When a missing document is shared, search this register for the corresponding gap IDs,
implement the blocked decisions, and update the session log.

---

## Appendix B — Invariant Enforcement Traceability

Every design invariant from DM §3 and IC must be enforced at the layer shown below.
A PR that breaks any row here must be rejected.

| Invariant | Source | Enforced at | Milestone |
|-----------|--------|-------------|-----------|
| 1 — Append-only audit log | DM §3 | DB RSP (DM §6); `WriteReceiptPhase2` WHERE clause | M4, M7 |
| 2 — Append-only escrow ledger | DM §3 | DB RSP (DM §6); `InsertEscrowEvent` only write | M4, M10 |
| 3 — No physical provider deletion | DM §3 | IC §6 DML table; departure sets `status='DEPARTED'` | M4, M9 |
| 4 — All amounts integer paise | DM §3 | DB BIGINT constraint; `TestNoFloatArithmetic`; debug panic | M4, M10 |
| 5 — Challenge nonce 33 bytes | DM §3 | DB CHECK constraint; `[33]byte` return type; CI grep | M4, M7 |
| 6 — No real shard on vetting / no repair for synthetic | DM §3 | DB CHECK constraint; `EnqueueJob` debug panic; `IsVettingChunk` pre-check | M4, M9, M14 |
| 7 — ShardSize compile-time constant | DM §3 | `const ShardSize = 262144`; `TestProfileShardSizeIsConstant` | M1, M3 |
| Single-writer vLog | IC §5.3 | Channel to writer goroutine; `TestSingleWriterGoroutine` | M5 |
| 0-RTT prohibited on audit/gc streams | IC §4 | Host middleware on protocol ID suffix; `DisableEarlyData: true` | M6 |
| No JSON signing inputs | IC §3.2 | `SignBytes` comment; code review rule | M2 |
| DHT HMAC key validator | IC §12 | Custom namespace validator; `TestDHTKeyValidatorPersists` | M6 |
| No float in payment | IC §11 | `TestNoFloatArithmetic` CI check; debug panic in `InsertEscrowEvent` | M10 |

---

## Appendix C — Known Document Inconsistencies

The following inconsistencies were found during BUILD file derivation and must be
resolved via PR before the relevant session is closed.

| ID | Document | Location | Inconsistency | Resolution |
|----|----------|----------|---------------|------------|
| INC-001 | IC §3.2 | Heartbeat section | Refers to `DeriveDKSKeystoreEncKey()` but IC §5.1 exports `DeriveKeystoreEncKey` | Use `DeriveKeystoreEncKey`; file PR to fix IC §3.2 |
| INC-002 | IC §5.8 | `EscrowEventType` constants | `EscrowSeizure` constant name appears twice (for both SEIZURE and REVERSAL) | Use `EscrowReversal` for REVERSAL; file PR to fix IC §5.8 |
| INC-003 | IC §4.1 Step 3 | Capability token expiry check | Step 3 says "reject with `0x03`" then says "Use `0x07 CAPABILITY_EXPIRED`" — contradictory | Use `0x07` for expiry per the explicit note; `0x03` is for signature failure |
| INC-004 | IC §11 | Package import constraints section header | Section says "See §11" while referencing itself (should reference §9) | The import constraints are in §9; the §11 reference in the prohibition line is a typo |
| INC-005 | openapi.yaml | `OtpVerifyRequest.properties` | Field named `otp_error_code` instead of `otp_code` | Implement as `otp_code`; file correction PR against OAS |
| INC-006 | openapi.yaml | `AuditReceiptListItem.properties.file_id` | Two `description` keys on the same property (duplicate YAML key — second wins; first is dropped by parsers) | Use the second description (nullable for vetting chunks, per ADR-030); file correction PR |

---

*End of BUILD.md*  
*Repository: https://github.com/masamasaowl/Vyomanaut_Research*  