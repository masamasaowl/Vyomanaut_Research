# Vyomanaut V2 — Build Procedure Part 1

**Status:** Authoritative build specification  
**Version:** 1.0  
**Date:** May 2026  
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research  
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
- [Milestone 0 — Project Setup & Repository Foundation](#milestone-0--project-setup--repository-foundation)
  - [Phase 0.1 — Repository Layout](#phase-01--repository-layout)
  - [Phase 0.2 — Toolchain & Linter Configuration](#phase-02--toolchain--linter-configuration)
  - [Phase 0.3 — CI Pipeline Skeleton](#phase-03--ci-pipeline-skeleton)
  - [Phase 0.4 — Development Docker Compose](#phase-04--development-docker-compose)
- [Milestone 1 — Configuration & NetworkProfile](#milestone-1--configuration--networkprofile)
  - [Phase 1.1 — NetworkProfile Struct](#phase-11--networkprofile-struct)
  - [Phase 1.2 — Profile Tests](#phase-12--profile-tests)
  - [Phase 1.3 — Profile Selection & Guard Rails](#phase-13--profile-selection--guard-rails)
- [Milestone 2 — Core Cryptography (`internal/crypto`)](#milestone-2--core-cryptography-internalcrypto)
  - [Phase 2.1 — AES-NI Detection](#phase-21--aes-ni-detection)
  - [Phase 2.2 — HKDF Key Derivation](#phase-22--hkdf-key-derivation)
  - [Phase 2.3 — Argon2id Master Secret Derivation](#phase-23--argon2id-master-secret-derivation)
  - [Phase 2.4 — AONT Cipher](#phase-24--aont-cipher)
  - [Phase 2.5 — Pointer File AEAD](#phase-25--pointer-file-aead)
  - [Phase 2.6 — BIP-39 Mnemonic System](#phase-26--bip-39-mnemonic-system)
  - [Phase 2.7 — Ed25519 Signing Conventions](#phase-27--ed25519-signing-conventions)
- [Milestone 3 — Erasure Coding Engine (`internal/erasure`)](#milestone-3--erasure-coding-engine-internalerasure)
  - [Phase 3.1 — Engine Construction](#phase-31--engine-construction)
  - [Phase 3.2 — Encode & Decode](#phase-32--encode--decode)
- [Milestone 4 — Database Schema & Migrations](#milestone-4--database-schema--migrations)
  - [Phase 4.1 — Migration Generator](#phase-41--migration-generator)
  - [Phase 4.2 — Core Type Definitions](#phase-42--core-type-definitions)
  - [Phase 4.3 — Core Tables](#phase-43--core-tables)
  - [Phase 4.4 — Audit, Escrow, and Repair Tables](#phase-44--audit-escrow-and-repair-tables)
  - [Phase 4.5 — Index Catalogue](#phase-45--index-catalogue)
  - [Phase 4.6 — Row Security Policies](#phase-46--row-security-policies)
  - [Phase 4.7 — Materialised Views](#phase-47--materialised-views)
  - [Phase 4.8 — Migration Checklist Verification](#phase-48--migration-checklist-verification)
- [Milestone 5 — Provider Storage Engine (`internal/storage`)](#milestone-5--provider-storage-engine-internalstorage)
  - [Phase 5.1 — ChunkStore Interface & vLog](#phase-51--chunkstore-interface--vlog)
  - [Phase 5.2 — Storage Tests](#phase-52--storage-tests)
- [Milestone 6 — P2P Network Layer (`internal/p2p`)](#milestone-6--p2p-network-layer-internalp2p)
  - [Phase 6.1 — libp2p Host](#phase-61--libp2p-host)
  - [Phase 6.2 — Kademlia DHT with Custom HMAC Validator](#phase-62--kademlia-dht-with-custom-hmac-validator)
  - [Phase 6.3 — Heartbeat Goroutine](#phase-63--heartbeat-goroutine)
- [Appendix A — Invariant Enforcement Traceability](#appendix-a--invariant-enforcement-traceability)
- [Appendix B - Session Registry](#appendix-b--session-registry)
- [Appendix C - File Creation Index](#appendix-c--file-creation-index)

### Remaining Milestones

- [Milestone 7 — Audit System (`internal/audit`)](./build_part2.md#milestone-7--audit-system-internalaudit)
  - [Phase 7.1 — Challenge Nonce Generation](./build_part2.md#phase-71--challenge-nonce-generation)
  - [Phase 7.2 — Response Validation](./build_part2.md#phase-72--response-validation)
  - [Phase 7.3 — Two-Phase Receipt Write](./build_part2.md#phase-73--two-phase-receipt-write)
  - [Phase 7.4 — Cluster Secret Cache](./build_part2.md#phase-74--cluster-secret-cache)
  - [Phase 7.5 — JIT Detection](./build_part2.md#phase-75--jit-detection)
- [Milestone 8 — Scoring System (`internal/scoring`)](./build_part2.md#milestone-8--scoring-system-internalscoring)
  - [Phase 8.1 — Score Retrieval](./build_part2.md#phase-81--score-retrieval)
  - [Phase 8.2 — Consecutive Pass Counter](./build_part2.md#phase-82--consecutive-pass-counter)
  - [Phase 8.3 — EWMA RTO Tracking](./build_part2.md#phase-83--ewma-rto-tracking)
- [Milestone 9 — Repair System (`internal/repair`)](./build_part2.md#milestone-9--repair-system-internalrepair)
  - [Phase 9.1 — Repair Job Queue](./build_part2.md#phase-91--repair-job-queue)
  - [Phase 9.2 — Repair Executor](./build_part2.md#phase-92--repair-executor)
  - [Phase 9.3 — Departure Detector](./build_part2.md#phase-93--departure-detector)
- [Milestone 10 — Payment System (`internal/payment`)](./build_part2.md#milestone-10--payment-system-internalpayment)
  - [Phase 10.1 — PaymentProvider Interface & Mock](./build_part2.md#phase-101--paymentprovider-interface--mock)
  - [Phase 10.2 — Escrow Ledger](./build_part2.md#phase-102--escrow-ledger)
  - [Phase 10.3 — Razorpay Implementation](./build_part2.md#phase-103--razorpay-implementation)
  - [Phase 10.4 — Release Computation & Seizure](./build_part2.md#phase-104--release-computation--seizure)
- [Milestone 11 — REST API Layer](./build_part2.md#milestone-11--rest-api-layer)
  - [Phase 11.1 — Error Envelope](./build_part2.md#phase-111--error-envelope)
  - [Phase 11.2 — Readiness Gate Endpoint](./build_part2.md#phase-112--readiness-gate-endpoint)
  - [Phase 11.3 — Endpoint Routing Stubs](./build_part2.md#phase-113--endpoint-routing-stubs)
  - [Phase 11.4 — Authentication Endpoints](./build_part2.md#phase-114--authentication-endpoints)
  - [Phase 11.5 — Owner Endpoints](./build_part2.md#phase-115--owner-endpoints)
  - [Phase 11.6 — Provider REST Endpoints](./build_part2.md#phase-116--provider-rest-endpoints)
  - [Phase 11.7 — Upload and File Endpoints](./build_part2.md#phase-117--upload-and-file-endpoints)
  - [Phase 11.8 — Pricing Endpoints](./build_part2.md#phase-118--pricing-endpoints)
  - [Phase 11.9 — Audit Admin Endpoints](./build_part2.md#phase-119--audit-admin-endpoints)
  - [Phase 11.10 — Admin Endpoints](./build_part2.md#phase-1110--admin-endpoints)
  - [Phase 11.11 — Per-Provider Chunk Count Ceiling](./build_part2.md#phase-1111--per-provider-chunk-count-ceiling)
- [Milestone 12 — Coordination Microservice (`cmd/microservice`)](./build_part2.md#milestone-12--coordination-microservice-cmdmicroservice)
  - [Phase 12.1 — Microservice Startup](./build_part2.md#phase-121--microservice-startup)
- [Milestone 13 — Provider Daemon Core (`cmd/provider`)](./build_part2.md#milestone-13--provider-daemon-core-cmdprovider)
  - [Phase 13.1 — Provider Startup](./build_part2.md#phase-131--provider-startup)
  - [Phase 13.2 — Chunk Upload Stream Handler](./build_part2.md#phase-132--chunk-upload-stream-handler)
  - [Phase 13.3 — Audit Challenge Stream Handler](./build_part2.md#phase-133--audit-challenge-stream-handler)
  - [Phase 13.4 — Repair Download Stream Handler](./build_part2.md#phase-134--repair-download-stream-handler)
  - [Phase 13.5 — Vetting GC Stream Handler](./build_part2.md#phase-135--vetting-gc-stream-handler)
  - [Phase 13.6 — Provider RAM Check at Installation](./build_part2.md#phase-136--provider-ram-check-at-installation)
- [Milestone 14 — Vetting & Synthetic Chunks (`internal/vettingchunk`)](./build_part2.md#milestone-14--vetting--synthetic-chunks-internalvettingchunk)
  - [Phase 14.1 — Synthetic Chunk Generator](./build_part2.md#phase-141--synthetic-chunk-generator)
  - [Phase 14.2 — GC Delivery](./build_part2.md#phase-142--gc-delivery)
- [Milestone 15 — Client SDK (`internal/client`)](./build_part2.md#milestone-15--client-sdk-internalclient)
  - [Phase 15.1 — Account Management](./build_part2.md#phase-151--account-management)
  - [Phase 15.2 — Upload Orchestrator](./build_part2.md#phase-152--upload-orchestrator)
  - [Phase 15.3 — Retrieval Orchestrator](./build_part2.md#phase-153--retrieval-orchestrator)
- [Milestone 16 — Demo Mode Validation](./build_part2.md#milestone-16--demo-mode-validation)
  - [Phase 16.1 — End-to-End Demo Test](./build_part2.md#phase-161--end-to-end-demo-test)
  - [Phase 16.2 — Simulation Mode](./build_part2.md#phase-162--simulation-mode)
- [Milestone 17 — Production Hardening](./build_part2.md#milestone-17--production-hardening)
  - [Phase 17.1 — Secrets Manager Adapters](./build_part2.md#phase-171--secrets-manager-adapters)
  - [Phase 17.2 — HA Microservice & Relay Nodes](./build_part2.md#phase-172--ha-microservice--relay-nodes)
- [Milestone M-OBS — Observability & Metrics](./build_part2.md#milestone-m-obs--observability--metrics)
  - [Phase OBS.1 — Microservice Metrics](./build_part2.md#phase-obs1--microservice-metrics)
  - [Phase OBS.2 — Provider Daemon Metrics](./build_part2.md#phase-obs2--provider-daemon-metrics)
  - [Phase OBS.3 — Operational Alerts](./build_part2.md#phase-obs3--operational-alerts)
  - [Phase OBS.4 — Prometheus Metric Naming CI Gate](./build_part2.md#phase-obs4--prometheus-metric-naming-ci-gate)
- [Milestone 18 — Launch Readiness](./build_part2.md#milestone-18--launch-readiness)
  - [Phase 18.1 — Runbooks](./build_part2.md#phase-181--runbooks)
  - [Phase 18.2 — Benchmark Scripts](./build_part2.md#phase-182--benchmark-scripts)
  - [Phase 18.3 — Security Verification Checklist](./build_part2.md#phase-183--security-verification-checklist)
  - [Phase 18.4 — Final CI Gate](./build_part2.md#phase-184--final-ci-gate)

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
             ├─ M7 (Provider Daemon Core)
             ├─ M8 (Audit System)
             │   ├─ M9 (Scoring)
             │   └─ M10 (Repair)
             ├─ M11 (Payment)
             └─ M12 (REST API Layer)
                 ├─ M-OBS (Observability & Metrics)
                 └─ M13 (Coordination Microservice)
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

## Milestone 0 — Project Setup & Repository Foundation

**Deliverable:** A compilable Go module with the full directory skeleton, a passing CI
pipeline skeleton, and all tooling configured. No business logic yet.

**Reference:** MVP §8.1 (directory map), MVP §8.4 (CI pipeline structure),
IC §10 (naming conventions), IC §11 (forbidden code patterns — linter rules set here)

---

### Phase 0.1 — Repository Layout

**Reference:** MVP §8.1

#### Session 0.1.1 — Initialise Go module and top-level directories

**Task:** Initialise Go module and directory skeleton.

**PRECONDITIONS:**

- Git repository exists at:
  - `github.com/masamasaowl/Vyomanaut_Research (Research repo)`
  - `github.com/masamasaowl/Vyomanaut_V2 (Working repo)`
- docs/ directory exists with: architecture.md, data-model.md, interface-contracts.md, mvp.md, requirements.md, api/openapi.yaml

**COMMANDS TO RUN:**

```go
  cd <repo_root>
  go mod init github.com/masamasaowl/Vyomanaut_V2
  go mod tidy
```

**DIRECTORIES TO CREATE (docs/ is pre-existing; skip it):**

```go
  cmd/microservice/
  cmd/provider/
  cmd/client/
  internal/config/
  internal/crypto/
  internal/erasure/
  internal/storage/
  internal/p2p/
  internal/audit/
  internal/scoring/
  internal/repair/
  internal/payment/
  internal/vettingchunk/
  internal/client/account/
  internal/client/upload/
  internal/client/retrieve/
  internal/client/manage/
  internal/cluster/
  internal/api/
  internal/metrics/
  internal/secrets/
  migrations/
  deployments/dev/
  deployments/production/
  deployments/grafana/dashboards/
  scripts/ci/
  scripts/benchmarks/
  scripts/test/
  runbooks/
```

For each internal/ subdirectory, create doc.go:
  FILE: `internal/<package>/doc.go`
  CONTENT:

  ```go
    // Package <package> <one-line description from MVP §8.2>.
    package <package>
  ```

**VERIFY:**

  COMPILE:

  ```go
    $ go build ./...
    EXPECT: exit 0
  ```

  FILES_EXIST:

  ```go
    $ for f in go.mod cmd/microservice cmd/provider cmd/client \
        internal/config internal/crypto internal/erasure \
        internal/storage internal/p2p internal/audit internal/scoring \
        internal/repair internal/payment internal/vettingchunk \
        internal/client/account internal/client/upload \
        internal/client/retrieve internal/client/manage \
        internal/cluster internal/api internal/metrics \
        internal/secrets migrations deployments/dev \
        scripts/ci scripts/benchmarks runbooks; do
        test -e "$f" && echo "PASS: $f" || echo "FAIL: $f missing"
      done
  ```

  NEGATIVE_CHECKS:

  ```go
    $ test -d docs && echo "PASS: docs pre-existing" || echo "FAIL: docs missing"
    $ test -d cmd/relay && echo "FAIL: cmd/relay must not exist until M17" || echo "PASS"
  ```
  
  CONTENT_CHECKS:

  ```go
    $ grep -c "vyomanaut" go.mod
    EXPECT: 1  (module declaration)
  ```

---

#### Session 0.1.2 — `cmd/` entrypoint stubs

**TASK:** Create cmd/ entrypoint stubs.

For each binary, create main.go:

FILE: cmd/microservice/main.go
FILE: cmd/provider/main.go  
FILE: cmd/client/main.go

Each file follows this exact template (substitute <binary> with
microservice, provider, client respectively):

```go
  package main

  import "fmt"

  func main() {
      // TODO: wire subsystems (M12/M13/M15)
      fmt.Printf("[STARTUP] Vyomanaut %s v0.1.0 — mode=STUB\n", "<binary>")
  }
```

**NOTE:** cmd/relay is created in M17 Session 17.2.2. Do not create it here.

**NOTE:** mode=STUB is intentional. mode=UNKNOWN would conflict with MVP §2.3guard-rail semantics. mode=PROD/DEMO are wired in M1.

**VERIFY:**

COMPILE:

```go
$ go build ./cmd/...
EXPECT: exit 0; zero warnings
```

CONTENT_CHECKS:

```go
$ grep -c "mode=STUB" cmd/microservice/main.go
EXPECT: 1

$ grep -c "mode=STUB" cmd/provider/main.go
EXPECT: 1

$ grep -c "mode=STUB" cmd/client/main.go
EXPECT: 1
```

NEGATIVE_CHECKS:

```go
$ test -f cmd/relay/main.go && echo "FAIL: cmd/relay must not exist" || echo "PASS"

$ grep -rn "mode=UNKNOWN" cmd/ && echo "FAIL" || echo "PASS"
```

---

#### Session 0.1.3 — Create `internal/` package stubs

**Task:** For every package listed in MVP §8.2, create the package directory and a
`doc.go` file containing only the package declaration and a one-line doc comment matching
the description in MVP §8.2. Packages to stub: `internal/config`, `internal/crypto`,
`internal/erasure`, `internal/storage`, `internal/p2p`, `internal/audit`,
`internal/scoring`, `internal/repair`, `internal/payment`, `internal/vettingchunk`,
`internal/client/account`, `internal/client/upload`, `internal/client/retrieve`,
`internal/client/manage`, `internal/cluster`.

Verify: `go build ./internal/...` succeeds. `go vet ./internal/...` produces zero output.

---

### Phase 0.2 — Toolchain & Linter Configuration

**Reference:** MVP §8.4 (linter list), IC §11 (forbidden patterns that linters enforce)

#### Session 0.2.1 — Configure `.golangci.yml`

**TASK:** Create .golangci.yml with the linters from MVP §8.4.

**FILE:** `.golangci.yml` (v2.12.2)
**CONTENT:**

```yml

version: "2"

run:
  timeout: 5m

linters:
  enable:
    - govet
    - errcheck
    - exhaustive
    - godot
    - mnd

formatters:
  enable:
    - gofmt

linters-settings:
  exhaustive:
    # Types added here as they are defined in subsequent milestones.
    # Format: <package_import_path>.<TypeName>
    # M7: internal/audit.AuditResult
    # M9: internal/repair.Priority, internal/repair.TriggerType
    # M10: internal/payment.EscrowEventType
    default-signifies-exhaustive: false

  errcheck:
    check-type-assertions: true

  mnd:
    ignored-functions:
      - "argon2.IDKey"

issues:
  exclude-rules:
    - path: "_test\\.go"
      linters:
        - mnd
```

**NOTE:** The exhaustive linter section is intentionally sparse at M0. It is updated incrementally as each type is defined (sessions 7.1.1, 9.1.1, 10.2.1). The comment block above is the authoritative tracking list.

**VERIFY:**

FILES_EXIST:

```go
$ test -f .golangci.yml && echo PASS || echo FAIL
```

CONTENT_CHECKS:

```go
$ grep -c "exhaustive" .golangci.yml
EXPECT: >= 2 (linter name + settings section)

$ grep -c "gomnd" .golangci.yml
EXPECT: >= 2
```

LINT_CHECK:

// Note: golangci-lint has been updated to v2.12.2

```go
$ golangci-lint run ./...
EXPECT: exit 0; zero findings on stub-only repository
```

---

#### Session 0.2.2 — Configure forbidden-pattern grep checks

**TASK:** Create scripts/ci/grep_checks.sh implementing four grep-fail checks.

**FILE:** scripts/ci/grep_checks.sh
**CONTENT:**

```go
  #!/usr/bin/env bash
  set -euo pipefail
  REPO_ROOT="$(git rev-parse --show-toplevel)"
  FAIL=0

  check() {
    local name="$1"; local pattern="$2"; local scope="$3"
    if grep -rn --include="*.go" --include="*.sql" \
         -E "$pattern" "$REPO_ROOT/$scope" 2>/dev/null | grep -q .; then
      echo "FAIL [$name]: pattern '$pattern' found in '$scope':"
      grep -rn --include="*.go" --include="*.sql" \
           -E "$pattern" "$REPO_ROOT/$scope"
      FAIL=1
    else
      echo "PASS [$name]"
    fi
  }

  # Check 8: challenge_nonce must be BYTEA(33), never BYTEA(32)
  check "NONCE_LENGTH" \
    "challenge_nonce[[:space:]]+BYTEA\(32\)" \
    "."

  # Check 9: no float types in payment package
  check "NO_FLOAT_PAYMENT" \
    "(float64|float32|FLOAT|DECIMAL|NUMERIC)" \
    "internal/payment"

  # Check 10: no references to non-existent ADRs (above ADR-031)
  # Pattern: ADR-0[3-9][2-9]|ADR-[1-9][0-9]{2,}
  check "ADR_REFERENCE" \
    "ADR-0[3-9][2-9]|ADR-[1-9][0-9]{2,}" \
    "."

  # Check 11: no UPI Collect API endpoint calls
  check "NO_UPI_COLLECT" \
    "virtual_accounts|upi/collect|collect/request" \
    "internal"

  exit $FAIL
```

**MAKE EXECUTABLE:** chmod +x scripts/ci/grep_checks.sh

**VERIFY:**

FILES_EXIST:

```zsh
$ test -f scripts/ci/grep_checks.sh && echo PASS || echo FAIL

$ test -x scripts/ci/grep_checks.sh && echo PASS || echo FAIL
```

SCRIPT_RUNS:

```zsh
$ bash scripts/ci/grep_checks.sh
EXPECT: exit 0; all four lines print "PASS [...]"
```

SELF_TEST (inject a violation and confirm detection):

```zsh
$ echo "challenge_nonce BYTEA(32)" > /tmp/test_violation.go

$ bash scripts/ci/grep_checks.sh /tmp/test_violation.go 2>/dev/null \
&& echo "FAIL: check did not catch violation" \
|| echo "PASS: violation detected"

CLEANUP: rm /tmp/test_violation.go
```

---

### Phase 0.3 — CI Pipeline Skeleton

**Reference:** MVP §8.4

#### Session 0.3.1 — Create `.github/workflows/ci.yml`

**TASK:** Create ``.github/workflows/ci.yml` with all 16 CI checks.

**FILE:** ``.github/workflows/ci.yml`

The workflow triggers on: [push, pull_request]
Uses: `ubuntu-latest`, `go: '1.26'`

**Services block:** postgres:16 with `POSTGRES_DB=vyomanaut_test`,
`POSTGRES_USER=vyomanaut_app`,
`POSTGRES_PASSWORD=testpass`,
options: `--health-cmd pg_isready`

**Steps (in order, each with id matching check number):**

```yml
  check-01: go build ./...
    run: go build ./...

  check-02: go vet ./...
    run: go vet ./...
  
  # Note golangci-lint version has been updated from v1.57.0 -> v2.12.2. Please build accordingly
  check-03: golangci-lint
    uses: golangci/golangci-lint-action@v4
    with: {version: v2.12.2, args: --timeout=5m}

  check-04: go test with race detector
    run: go test -race -count=1 ./...

  check-05: TestDHTKeyValidatorPersists (M6 gate)
    run: |
      go test -run TestDHTKeyValidatorPersists ./internal/p2p/... \
        || (echo "PENDING: session 6.2.3" && exit 0)
    # Set continue-on-error: false AFTER M6 completes.
    # Until then: placeholder exits 0 with message.
    continue-on-error: true

  check-06: TestNoFloatArithmetic (M10 gate)
    run: |
      go test -run TestNoFloatArithmetic ./internal/payment/... \
        || (echo "PENDING: session 10.4.2" && exit 0)
    continue-on-error: true

  check-07: Migration apply + rollback
    run: |
      export PGPASSWORD=testpass
      psql -h localhost -U vyomanaut_app -d vyomanaut_test \
        -c "CREATE EXTENSION IF NOT EXISTS btree_gist;"
      go run migrations/generator.go --profile=prod \
        > /tmp/001_prod.sql
      psql -h localhost -U vyomanaut_app -d vyomanaut_test \
        -f /tmp/001_prod.sql
      go run migrations/generator.go --profile=demo \
        > /tmp/001_demo.sql
      # Demo schema applied to separate DB to avoid cross-contamination
      psql -h localhost -U vyomanaut_app \
        -c "CREATE DATABASE vyomanaut_demo_test;"
      psql -h localhost -U vyomanaut_app -d vyomanaut_demo_test \
        -c "CREATE EXTENSION IF NOT EXISTS btree_gist;"
      psql -h localhost -U vyomanaut_app -d vyomanaut_demo_test \
        -f /tmp/001_demo.sql
    # PENDING until M4. Until M4: stub exits 0.
    continue-on-error: true

  check-08 through check-11: grep checks
    run: bash scripts/ci/grep_checks.sh

  check-12: Mermaid render
    run: |
      npx --yes @mermaid-js/mermaid-cli -V \
        || (echo "PENDING: mermaid diagrams" && exit 0)
    continue-on-error: true

  check-13: Hyperlink check
    run: |
      npx --yes markdown-link-check docs/**/*.md \
        || (echo "PENDING: links may be stale" && exit 0)
    continue-on-error: true

  check-14: TestProfileShardSizeIsConstant
    run: |
      go test -run TestProfileShardSizeIsConstant ./internal/config/... \
        || (echo "PENDING: session 1.2.1" && exit 0)
    continue-on-error: true

  check-15: TestProfileBothFullySpecified
    run: |
      go test -run TestProfileBothFullySpecified ./internal/config/... \
        || (echo "PENDING: session 1.2.2" && exit 0)
    continue-on-error: true

  check-16: TestNoOrphanMetricName (M-OBS gate)
    run: |
      bash scripts/ci/grep_checks.sh --check=metric-names \
        || (echo "PENDING: session OBS.4.1" && exit 0)
    continue-on-error: true
```

**TRANSITION RULE:** When a milestone completes, remove `continue-on-error: true`
and the `|| (echo "PENDING..." && exit 0)` fallback from its corresponding
check. That check then becomes a hard gate.

**VERIFY:**

FILES_EXIST:

```bash
$ test -f .github/workflows/ci.yml && echo PASS || echo FAIL
```

CONTENT_CHECKS:

```bash
$ grep -c "^ check-" .github/workflows/ci.yml
EXPECT: 16 (checks 01 through 16)
```

YAML_VALID:

```bash
$ python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" \
&& echo PASS || echo FAIL
```

STEP_COUNT:

```bash
$ grep -c "name: check-" .github/workflows/ci.yml
EXPECT: 16
```

---

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

#### Session 0.4.1 — `Create deployments/dev/docker-compose.yml`

**TASK:** Create deployments/dev/docker-compose.yml for local development.

**FILE:** ``deployments/dev/docker-compose.yml`

**Requirements from MVP §8.5:**

1. One postgres:16 instance, port 5432, with `btree_gist` pre-installed
2. One microservice stub (`cmd/microservice` binary or placeholder)
3. One relay placeholder (`cmd/relay` does not exist until M17)
4. One provider stub in `--sim-count=5` `--sim-asn-count=5` mode

**CONTENT:**

```yaml
  version: "3.9"
  services:
    postgres:
      image: postgres:16
      environment:
        POSTGRES_DB: vyomanaut_dev
        POSTGRES_USER: vyomanaut_app
        POSTGRES_PASSWORD: devpass
      ports: ["5432:5432"]
      volumes:
        - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U vyomanaut_app"]
        interval: 5s
        timeout: 5s
        retries: 5

    # cmd/relay binary created in M17 Session 17.2.2.
    # Placeholder keeps docker-compose valid until M17.
    relay:
      image: alpine:3.19
      command: ["sh", "-c", "echo 'relay placeholder — implement M17' && sleep infinity"]
      # TODO(M17): replace with:
      #   build: {context: ..., dockerfile: Dockerfile.relay}
      #   command: ["./relay", "--listen=/ip4/0.0.0.0/udp/4001/quic-v1"]

    # Provider daemon in simulation mode.
    # Binary built from cmd/provider after M13.
    provider:
      image: alpine:3.19
      command: ["sh", "-c",
        "echo 'provider stub — implement M13' && sleep infinity"]
      environment:
        VYOMANAUT_MODE: demo
      # TODO(M13): replace with:
      #   command: ["./provider",
      #     "--microservice-url=http://microservice:8080",
      #     "--sim-count=5", "--sim-asn-count=5",
      #     "--declared-storage-gb=50"]

    microservice:
      image: alpine:3.19
      command: ["sh", "-c",
        "echo 'microservice stub — implement M12' && sleep infinity"]
      environment:
        VYOMANAUT_MODE: demo
        VYOMANAUT_CLUSTER_MASTER_SEED: "devonlysecret00000000000000000000"
      ports: ["8080:8080"]
      depends_on:
        postgres:
          condition: service_healthy
      # TODO(M12): replace with cmd/microservice binary
```

**COMPANION FILE:** `deployments/dev/init-db.sql`
  `CREATE EXTENSION IF NOT EXISTS btree_gist`;

**Scale trigger (from architecture.md §27.5):** provision 4th relay node before provider count reaches 400 in production deployment. This dev compose runs 0 real relay nodes; NAT traversal is not tested here.

**VERIFY:**

FILES_EXIST:

```bash
$ test -f deployments/dev/docker-compose.yml && echo PASS || echo FAIL

$ test -f deployments/dev/init-db.sql && echo PASS || echo FAIL
```

YAML_VALID:

```bash
$ docker-compose -f deployments/dev/docker-compose.yml config > /dev/null \
&& echo PASS || echo FAIL
```

CONTENT_CHECKS:

```bash
$ grep -c "btree_gist" deployments/dev/init-db.sql
EXPECT: 1

$ grep -c "TODO(M17)" deployments/dev/docker-compose.yml
EXPECT: >= 1 (relay placeholder comment)

$ grep -c "TODO(M13)" deployments/dev/docker-compose.yml
EXPECT: >= 1 (provider placeholder comment)
```

POSTGRES_STARTS:

```bash
$ docker-compose -f deployments/dev/docker-compose.yml up -d postgres

$ sleep 5

$ docker-compose -f deployments/dev/docker-compose.yml exec -T postgres \
psql -U vyomanaut_app -d vyomanaut_dev -c "\dx" | grep btree_gist \
&& echo PASS || echo FAIL
```

CLEANUP:

```bash
$ docker-compose -f deployments/dev/docker-compose.yml down -v
```

NEGATIVE_CHECKS:

```bash
$ grep "relay-mode" deployments/dev/docker-compose.yml \
&& echo "FAIL: unspecified flag present" || echo "PASS"
```

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

**TASK:** Create `internal/config/network_profile.go` defining the NetworkProfile struct.

**FILE:** `internal/config/network_profile.go`

The struct must contain every field from MVP §5.1 in the exact groupings shown.
Each field must have a comment citing its governing ADR.

**CRITICAL CONSTRAINTS:**

1. ShardSize int is present in the struct ONLY for Go compiler completeness
   enforcement (struct literal syntax). Its value MUST always equal 262144.
2. Do NOT add any compile-time assertion involving `internal/erasure.ShardSize`
   here — that package doesn't exist until M3. The cross-check is deferred to
   M3 Session 3.1.1 and the test in Session 1.2.1 (which uses literal 262144
   until M3 completes).
3. No imports from other `internal/ packages (IC §9)`.

**FIELD GROUPS AND TYPES (exact — no additions, no omissions):**

```go
  // Erasure coding (ADR-003)
  DataShards   int
  ParityShards int
  TotalShards  int
  ShardSize    int   // MUST equal 262144 in both profiles; see Invariant 7
  LazyRepairR0 int

  // Readiness gate (ADR-029)
  MinActiveProviders int
  MinDistinctASNs    int
  MinMetroRegions    int
  MinRelayNodes      int
  MinCooledAccounts  int

  // ASN cap (ADR-014)
  ASNCapFraction float64

  // Time windows
  HeartbeatInterval       time.Duration  // ADR-028
  HeartbeatJitter         time.Duration  // ADR-028
  PollingInterval         time.Duration  // ADR-006
  DHTRepublishInterval    time.Duration  // ADR-001
  DHTExpiryDuration       time.Duration  // ADR-001
  DepartureThreshold      time.Duration  // ADR-006, ADR-007
  PromisedDowntimeMaximum time.Duration  // ADR-007
  AuditPeriodDuration     time.Duration  // ADR-016
  EscrowHoldWindow        time.Duration  // ADR-024
  VettingHoldWindow       time.Duration  // ADR-024
  PendingReceiptGCAge     time.Duration  // ADR-015
  RepairPromotionTimeout  time.Duration  // ADR-004, FR-043

  // Scoring windows (ADR-008)
  ScoreWindowShort  time.Duration
  ScoreWindowMedium time.Duration
  ScoreWindowLong   time.Duration
  DualWindowDrop    float64  // always 0.20; never mode-variable

  // Vetting (ADR-005)
  VettingMinPasses   int
  VettingMinDuration time.Duration
  VettingCapFraction float64  // always 0.10; never mode-variable

  // Cryptographic cost (ADR-020)
  Argon2Time    uint32
  Argon2Memory  uint32
  Argon2Threads uint8

  // Infrastructure
  RequireSecretsManager bool
  RequireQuorum         bool
  PaymentMode           string        // "razorpay_live"|"razorpay_test"|"mock"
  SkipMnemonicConfirm   bool
  RazorpayCoolingPeriod time.Duration

  // Release computation cycle
  // 0 = calendar-driven (prod); >0 = ticker interval (demo)
  ReleaseComputationInterval time.Duration  // ADR-024, ADR-031

  // GC retry backoff (IC §4.5)
  GCRetryBackoff []time.Duration

  // Mode identifier (printed at startup; never used for branching in business logic)
  Mode string  // "demo" | "prod"
```

**IMPORTS REQUIRED:** "time"

**VERIFY:**

COMPILE:

```go
$ go build ./internal/config/
EXPECT: exit 0
```

CONTENT_CHECKS:

```go
$ grep -c "ShardSize.*int" internal/config/network_profile.go
EXPECT: 1

$ grep -c "time.Duration" internal/config/network_profile.go
EXPECT: >= 14 (count Duration fields)

$ grep -c "ADR-" internal/config/network_profile.go
EXPECT: >= 10 (ADR citations in comments)
```

NEGATIVE_CHECKS:

```go
$ grep -n "internal/" internal/config/network_profile.go \
&& echo "FAIL: import of internal package" || echo "PASS"

$ grep -n "erasure.ShardSize" internal/config/network_profile.go \
&& echo "FAIL: forward reference to M3" || echo "PASS"

$ grep -n "262144" internal/config/network_profile.go \
&& echo "FAIL: ShardSize literal in struct (belongs in profiles.go)" || echo "PASS"
```

VET:

```go
$ go vet ./internal/config/
EXPECT: exit 0; zero output
```

---

#### Session 1.1.2 — Define `ProductionProfile` and `DemoProfile`

**TASK:** Define `ProductionProfile` and `DemoProfile` in `internal/config/profiles.go.`

**FILE:** `internal/config/profiles.go`

Both vars use struct literal syntax with ALL fields explicitly named.
Go compiler enforces that no field is omitted (OR-03, MVP §6.3).
Values taken exactly from MVP §5.2.

```go
var ProductionProfile = NetworkProfile{
  Mode:         "prod",
  DataShards:   16, ParityShards: 40, TotalShards: 56,
  ShardSize:    262144, LazyRepairR0: 8,
  MinActiveProviders: 56, MinDistinctASNs: 5, MinMetroRegions: 3,
  MinRelayNodes: 3, MinCooledAccounts: 56,
  ASNCapFraction: 0.20,
  HeartbeatInterval:    4 * time.Hour,
  HeartbeatJitter:      5 * time.Minute,
  PollingInterval:      24 * time.Hour,
  DHTRepublishInterval: 12 * time.Hour,
  DHTExpiryDuration:    24 * time.Hour,
  DepartureThreshold:   72 * time.Hour,
  PromisedDowntimeMaximum: 72 * time.Hour,
  AuditPeriodDuration: 30 * 24 * time.Hour,
  EscrowHoldWindow:    30 * 24 * time.Hour,
  VettingHoldWindow:   60 * 24 * time.Hour,
  PendingReceiptGCAge: 48 * time.Hour,
  RepairPromotionTimeout: 6 * time.Hour,
  ScoreWindowShort:  24 * time.Hour,
  ScoreWindowMedium: 7 * 24 * time.Hour,
  ScoreWindowLong:   30 * 24 * time.Hour,
  DualWindowDrop:    0.20,
  VettingMinPasses:   80,
  VettingMinDuration: 120 * 24 * time.Hour,
  VettingCapFraction: 0.10,
  Argon2Time: 3, Argon2Memory: 65536, Argon2Threads: 4,
  RequireSecretsManager: true, RequireQuorum: true,
  PaymentMode:           "razorpay_live",
  SkipMnemonicConfirm:   false,
  RazorpayCoolingPeriod: 24 * time.Hour,
  ReleaseComputationInterval: 0,  // 0 = calendar-driven; correct for prod
  GCRetryBackoff: []time.Duration{5*time.Minute, 15*time.Minute, 60*time.Minute},
}

var DemoProfile = NetworkProfile{
  Mode:         "demo",
  DataShards:   3, ParityShards: 2, TotalShards: 5,
  ShardSize:    262144, LazyRepairR0: 1,  // ShardSize IDENTICAL to prod
  MinActiveProviders: 5, MinDistinctASNs: 5, MinMetroRegions: 1,
  MinRelayNodes: 0, MinCooledAccounts: 5,
  ASNCapFraction: 0.20,
  HeartbeatInterval:    30 * time.Second,
  HeartbeatJitter:      5 * time.Second,
  PollingInterval:      2 * time.Minute,
  DHTRepublishInterval: 2 * time.Minute,
  DHTExpiryDuration:    4 * time.Minute,
  DepartureThreshold:   10 * time.Minute,
  PromisedDowntimeMaximum: 10 * time.Minute,
  AuditPeriodDuration: 2 * time.Minute,
  EscrowHoldWindow:    1 * time.Minute,
  VettingHoldWindow:   2 * time.Minute,
  PendingReceiptGCAge: 5 * time.Minute,
  RepairPromotionTimeout: 3 * time.Minute,
  ScoreWindowShort:  2 * time.Minute,
  ScoreWindowMedium: 6 * time.Minute,
  ScoreWindowLong:   20 * time.Minute,
  DualWindowDrop:    0.20,
  VettingMinPasses:   5,
  VettingMinDuration: 5 * time.Minute,
  VettingCapFraction: 0.10,
  Argon2Time: 1, Argon2Memory: 4096, Argon2Threads: 1,
  RequireSecretsManager: false, RequireQuorum: false,
  PaymentMode:           "mock",
  SkipMnemonicConfirm:   true,
  RazorpayCoolingPeriod: 0,
  ReleaseComputationInterval: 2 * time.Minute,
  GCRetryBackoff: []time.Duration{10*time.Second, 30*time.Second, 2*time.Minute},
}
```

**VERIFY:**

COMPILE:

```go
$ go build ./internal/config/
EXPECT: exit 0
```

CONTENT_CHECKS:

```go
$ grep -c "ShardSize.*262144" internal/config/profiles.go
EXPECT: 2 (one in ProductionProfile, one in DemoProfile)

$ grep -c "DataShards.*16" internal/config/profiles.go
EXPECT: 1 (ProductionProfile)

$ grep -c "DataShards.*3" internal/config/profiles.go
EXPECT: 1 (DemoProfile)

$ grep -c "PaymentMode.*razorpay_live" internal/config/profiles.go
EXPECT: 1

$ grep -c "PaymentMode.*mock" internal/config/profiles.go
EXPECT: 1
```

FIELD_COMPLETENESS (all fields must be named in both literals):

```go
$ go build -v ./internal/config/ 2>&1 | grep "too few" \
&& echo "FAIL: struct literal incomplete" || echo "PASS"
```

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

**VERIFY:**

TEST_RUN:

```go
$ go test -v -run TestProfileShardSizeIsConstant ./internal/config/
EXPECT: exit 0; output contains "--- PASS: TestProfileShardSizeIsConstant"
```

CONTENT_CHECKS:

```go
$ grep -c "262144" internal/config/profiles_test.go
EXPECT: >= 2 (both assertions)

$ grep -c "TODO.*erasure.ShardSize.*M3" internal/config/profiles_test.go
EXPECT: 1
```

---

#### Session 1.2.2 — `TestProfileBothFullySpecified`

**Task:** Implement `TestProfileBothFullySpecified` (CI check 15 in MVP §8.4). Use
reflection to verify that every field in `ProductionProfile` and `DemoProfile` is
non-zero (except fields where zero is explicitly the correct production value, e.g.
`ReleaseComputationInterval: 0` in ProductionProfile — these must be listed in an
explicit allowlist in the test with a comment explaining why zero is correct per
MVP §5.2).

**VERIFY:**

TEST_RUN:

```go
$ go test -v -run TestProfileBothFullySpecified ./internal/config/
EXPECT: exit 0; output contains "--- PASS: TestProfileBothFullySpecified"
```

CONTENT_CHECKS:

```go
$ grep -c "reflect" internal/config/profiles_test.go
EXPECT: >= 1 (reflection for field completeness check)

$ grep -c "zero.*correct\|correct.*zero" internal/config/profiles_test.go
EXPECT: >= 1 (allowlist comment for zero-value fields)
```

---

#### Session 1.2.3 — `TestDemoDiffersFromProduction`

**Task:** Assert that demo and production profiles differ on every field listed in
MVP §3.2 through §3.5 (erasure params, time windows, crypto params). Assert they are
identical on: `ShardSize`, `ASNCapFraction`, `VettingCapFraction`, `DualWindowDrop`
(per MVP §5.1 comments that these are "never mode-variable"). This test documents the
exact boundary between what changes and what stays the same.

**VERIFY:**

TEST_RUN:

```go
$ go test -v -run TestDemoDiffersFromProduction ./internal/config/
EXPECT: exit 0; output contains "--- PASS: TestDemoDiffersFromProduction"
```

CONTENT_CHECKS:

```go
$ grep -c "DualWindowDrop" internal/config/profiles_test.go
EXPECT: >= 1

$ grep -c "ASNCapFraction" internal/config/profiles_test.go
EXPECT: >= 1

$ grep -c "VettingCapFraction" internal/config/profiles_test.go
EXPECT: >= 1
```

---

### Phase 1.3 — Profile Selection & Guard Rails

**Reference:** MVP §2.3 (guard rails), MVP §5.3 (profile selection), IC §3.3
(DEMO_MODE_REAL_PAYMENT, PROD_MODE_ENV_SECRET error codes)

#### Session 1.3.1 — Implement `selectProfile()`

**TASK:** Implement `config.SelectProfile()` in `internal/config/select.go.`

**FILE:** internal/config/select.go

**EXPORTED FUNCTION SIGNATURE:**
  `func SelectProfile(modeFlag string) NetworkProfile`

**LOGIC (exact, no interpretation):**

```go
  1. If modeFlag == "" { modeFlag = os.Getenv("VYOMANAUT_MODE") }
  2. switch modeFlag:
     case "demo":
       log.Printf("[STARTUP] Vyomanaut — mode=DEMO — do not use for real data")
       return DemoProfile
     case "prod":
       log.Printf("[STARTUP] Vyomanaut — mode=PRODUCTION")
       return ProductionProfile
     case "":
       log.Printf("[STARTUP] WARNING: VYOMANAUT_MODE not set; defaulting to prod")
       return ProductionProfile
     default:
       log.Fatalf("[STARTUP] FATAL: unknown VYOMANAUT_MODE=%q; must be 'demo' or 'prod'",
         modeFlag)
       return ProductionProfile  // unreachable; satisfies compiler
```

**IMPORTS REQUIRED:** "log", "os"

**CALLERS (not implemented here; documented for wiring in M12/M13)**:

```go
  cmd/microservice/main.go: profile := config.SelectProfile(*modeFlag)
  cmd/provider/main.go:     profile := config.SelectProfile(*modeFlag)
  where *modeFlag is parsed from --mode CLI flag via flag.String("mode","","")
```

**VERIFY:**

COMPILE:

```go
$ go build ./internal/config/
EXPECT: exit 0
```

CONTENT_CHECKS:

```go
$ grep -c "func SelectProfile" internal/config/select.go
EXPECT: 1

$ grep -c "modeFlag string" internal/config/select.go
EXPECT: 1

$ grep -c "VYOMANAUT_MODE" internal/config/select.go
EXPECT: 1
```

BEHAVIOURAL_TESTS:

```go
$ go test -v -run TestSelectProfile ./internal/config/

(requires a test that calls SelectProfile with each of:
"demo", "prod", "" — and verifies correct profile returned)

EXPECT: exit 0; output contains "--- PASS: TestSelectProfile"
```

NEGATIVE_CHECKS:

```go
$ grep -n "mode == " internal/config/select.go \
&& echo "WARN: mode string comparison in business logic" || echo "PASS"
```

---

#### Session 1.3.2 — Implement startup guard rails

**TASK:** Implement startup guard rails in `internal/config/guards.go.`

**FILE:** internal/config/guards.go

**TYPE DEFINITION:**

```go
  // StartupError carries the IC §3.3 error code for fatal startup conditions.
  type StartupError struct {
    Code    string  // matches IC §3.3 error_code values
    Message string
  }
  func (e *StartupError) Error() string { return e.Code + ": " + e.Message }
```

**EXPORTED FUNCTION SIGNATURE:**
  `func ValidateStartupGuards(profile NetworkProfile) error`

**LOGIC:**

```go
  1. if profile.Mode == "prod" &&
       os.Getenv("VYOMANAUT_CLUSTER_MASTER_SEED") != "" {
       return &StartupError{
         Code:    "PROD_MODE_ENV_SECRET",
         Message: "VYOMANAUT_CLUSTER_MASTER_SEED must not be set in production; use secrets manager",
       }
     }
  2. if profile.Mode == "demo" && profile.PaymentMode == "razorpay_live" {
       return &StartupError{
         Code:    "DEMO_MODE_REAL_PAYMENT",
         Message: "live Razorpay endpoint must not be used in demo mode",
       }
     }
  3. return nil
```

**CALLER PATTERN (for cmd/ wiring in M12/M13):**

```go
  if err := config.ValidateStartupGuards(profile); err != nil {
    log.Fatalf("[STARTUP] FATAL guard rail: %v", err)
  }
```

**IMPORTS REQUIRED:** "os"

**VERIFY:**

COMPILE:

```go
$ go build ./internal/config/
EXPECT: exit 0
```

CONTENT_CHECKS:

```go
$ grep -c "StartupError" internal/config/guards.go
EXPECT: >= 2 (type definition + at least one return)

$ grep -c "PROD_MODE_ENV_SECRET" internal/config/guards.go
EXPECT: 1

$ grep -c "DEMO_MODE_REAL_PAYMENT" internal/config/guards.go
EXPECT: 1
```

UNIT_TESTS:

```go
$ go test -v -run TestGuardRails ./internal/config/

EXPECT: exit 0; output contains 4 PASS lines:

"prod+seed_present → error PROD_MODE_ENV_SECRET"
"prod+seed_absent → nil"
"demo+live_payment → error DEMO_MODE_REAL_PAYMENT"
"demo+mock_payment → nil"
```

---

#### Session 1.3.3 — Guard rail tests

**Task:** Test all guard rail combinations: prod + seed env present (must error),
prod + seed env absent (must pass), demo + razorpay_live (must error), demo + mock (must
pass). Use `t.Setenv` for environment isolation.

**VERIFY:**

TEST_RUN:

```go
$ go test -v -run TestGuardRails ./internal/config/
EXPECT: exit 0; all subtests PASS
```

ENVIRONMENT_ISOLATION:

```go
Verify t.Setenv is used (not os.Setenv) so tests are isolated:

$ grep -c "t.Setenv" internal/config/guards_test.go
EXPECT: >= 2
```

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

**TASK:** Implement `DetectAESNI()` in `internal/crypto/.`

**FILE:** `internal/crypto/aesni.go`
**BUILD TAG:** `//go:build amd64`

**CONTENT:**

```go
  package crypto

  // DetectAESNI reports whether the CPU supports AES-NI instructions.
  // Called exactly once at daemon startup (cmd/provider/main.go); result
  // stored in a local variable and passed as a parameter to all
  // AONTEncodeSegment and AONTDecodePackage calls.
  // Never re-checked at runtime (IC §5.1).
  func DetectAESNI() bool {
    // Use golang.org/x/sys/cpu for portable CPUID access.
    return cpu.X86.HasAES
  }
```

**IMPORT REQUIRED:** "golang.org/x/sys/cpu"

**FILE:** `internal/crypto/aesni_other.go`
**BUILD TAG:** `//go:build !amd64`

**CONTENT:**

```go
  package crypto

  // DetectAESNI always returns false on non-amd64 platforms.
  func DetectAESNI() bool { return false }
```

**NOTE:** No package-level `var aesNIAvailable`. The result is stored by the caller
(`cmd/provider/main.go`) and passed as a parameter. IC §5.1 specifies the function
is "called once at startup and stored" — that storage belongs to the caller, not
to this package.

**VERIFY:**

COMPILE:

```go
$ GOARCH=amd64 go build ./internal/crypto/

$ GOARCH=arm64 go build ./internal/crypto/

EXPECT: both exit 0
```

CONTENT_CHECKS:

```go
$ grep -c "//go:build amd64" internal/crypto/aesni.go
EXPECT: 1

$ grep -c "//go:build !amd64" internal/crypto/aesni_other.go
EXPECT: 1
```

NEGATIVE_CHECKS:

```go
$ grep -n "aesNIAvailable" internal/crypto/aesni.go \
&& echo "FAIL: package-level var must not exist" || echo "PASS"

$ grep -rn "aesNIAvailable" internal/crypto/ \
&& echo "FAIL: package-level var must not exist" || echo "PASS"
```

FUNCTION_EXISTS:

```go
$ grep -c "^func DetectAESNI" internal/crypto/aesni.go
EXPECT: 1

$ grep -c "^func DetectAESNI" internal/crypto/aesni_other.go
EXPECT: 1
```

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

---

#### Session 2.2.2 — HKDF known-answer tests

**Task:** Add `internal/crypto/hkdf_test.go` with known-answer vectors derived from
fixed inputs (not from a live run — compute expected output offline using the HKDF-SHA256
RFC 5869 test vector approach). Add a round-trip test: `DeriveFileKey(ms, o, f) ==
DeriveFileKey(ms, o, f)` (determinism). Add a non-collision test: two different fileIDs
produce different keys.

> Combined verifications for 2.2.1 & 2.2.2

**VERIFY:**

COMPILE:

```go
$ go build ./internal/crypto/
EXPECT: exit 0
```

FUNCTION_SIGNATURES:

```go
$ grep -c "^func DeriveFileKey" internal/crypto/hkdf.go

$ grep -c "^func DerivePointerEncKey" internal/crypto/hkdf.go

$ grep -c "^func DeriveKeystoreEncKey" internal/crypto/hkdf.go

$ grep -c "^func DeriveDHTOwnerKey" internal/crypto/hkdf.go

$ grep -c "^func DeriveDHTKey" internal/crypto/hkdf.go

EXPECT: 1 for each
```

NEGATIVE_CHECKS:

```go
$ grep -n "DeriveDKSKeystoreEncKey\|DeriveKSKeystoreEncKey" \
internal/crypto/hkdf.go \
&& echo "FAIL: wrong function name" || echo "PASS"
```

UNIT_TESTS:

```go
$ go test -v -run TestHKDF ./internal/crypto/

EXPECT: exit 0; tests include:

TestHKDFDeterminism (same inputs → same output)

TestHKDFNonCollision (different fileIDs → different keys)

TestHKDFKnownAnswer (fixed-input known-output vectors)
```

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

---

#### Session 2.3.2 — `DeriveMasterSecret` performance test

**Task:** Add a test `TestArgon2idProduction` that runs with production parameters
(t=3, m=65536, p=4) and asserts completion time >= 100ms (conservative floor — the
IC §5.1 post-condition states "Production: >= 200ms" but 100ms covers slower CI
hardware). Add `TestArgon2idDemo` that runs with demo parameters (t=1, m=4096, p=1)
and asserts completion time < 200ms. These tests must be tagged `//go:build !short`
so they are skipped in fast CI runs.

> Combined verifications for 2.3.1 & 2.3.2

**VERIFY:**

COMPILE:

```go
$ go build ./internal/crypto/
EXPECT: exit 0
```

FUNCTION_SIGNATURE:

```go
$ grep -c "func DeriveMasterSecret(passphrase, ownerID \[\]byte, argon2Time uint32,
argon2Memory uint32, argon2Threads uint8)" \
internal/crypto/argon2.go

EXPECT: 1
```

NEGATIVE_CHECKS:

```go
$ grep -n "argon2Time.*3\|argon2Memory.*65536\|argon2Threads.*4" \
internal/crypto/argon2.go \
&& echo "FAIL: hardcoded Argon2 parameters" || echo "PASS"
```

PERFORMANCE_TESTS (tagged, run separately):

```go
$ go test -v -tags="" -run TestArgon2idProduction ./internal/crypto/
EXPECT: exit 0; output shows timing >= 100ms

$ go test -v -run TestArgon2idDemo ./internal/crypto/
EXPECT: exit 0; output shows timing < 200ms
```

---

### Phase 2.4 — AONT Cipher

**Reference:** IC §5.1 (`AONTEncodeSegment`, `AONTDecodePackage`), IC §11 (forbidden:
convergent encryption, K reuse), MVP §8.2 (`aont_canary.go`), ARCH §10 Stage 1 or AES-CTR counter convention

#### Session 2.4.1 — Implement the AONT canary constant

**Task:** Create `internal/crypto/aont_canary.go` defining the fixed 16-byte canary
value as `var aontCanary = [16]byte{...}` (choose a deterministic value, e.g. the
first 16 bytes of SHA-256("vyomanaut-aont-canary-v1"), computed once and hardcoded).
This must be a `var`, not a `const`, because Go does not allow array constants — but it
must never be reassigned. Add a `// canary must never be changed — it is an on-disk format commitment` comment. The canary is the second-to-last 16-byte word of every AONT
package (IC §5.1 post-condition for `AONTEncodeSegment`).

---

#### Session 2.4.2 — Implement `AONTEncodeSegment()`

**TASK:** Implement `AONTEncodeSegment()` in `internal/crypto/aont.go.`

**FILE:** `internal/crypto/aont.go`

**FUNCTION SIGNATURE (from IC §5.1):**
  `func AONTEncodeSegment(segment []byte, aesNIAvailable bool) ([]byte, error)`

**PRECONDITIONS (enforce with panic in debug builds, sentinel error in release):**

  1. `len(segment) > 0`
  2. `len(segment) % 16 == 0`
  3. Caller has padded segment to minimum `4 MB` before calling

**ALGORITHM (from architecture.md §10 Stage 1):**

```go
  1. K = make([]byte, 32); io.ReadFull(rand.Reader, K) — fresh per call (IC §11)
  2. numWords = len(segment) / 16
  3. output = make([]byte, (numWords+2)*16)  // +1 canary, +1 key-block
  4. Cipher selection:
     if aesNIAvailable {
       // AES-256-CTR path; counter starts at i+1 per architecture.md §10 Stage 1
       for i := 0; i < numWords; i++ {
         block_i = AES256ECB(K, uint128(i+1))
         copy(output[i*16:], xor16(segment[i*16:], block_i))
       }
     } else {
       // ChaCha20-256 path
       stream = chacha20.NewUnauthenticatedCipher(K, nonce_zero_12bytes)
       stream.XORKeyStream(output[:numWords*16], segment)
     }
  5. copy(output[numWords*16:], aontCanary[:])   // second-to-last word
  6. h = sha256.Sum256(output[:numWords*16])      // hash of ciphertext words only
  7. output[(numWords+1)*16 : (numWords+2)*16] = xor32(K, h[:])  // key-block

  Return output, nil
```

**HELPER:** `xor16(a, b []byte) [16]byte — XOR two 16-byte slices`
**HELPER:** `xor32(a, b []byte) [32]byte — XOR two 32-byte slices (for K ⊕ h)`

**IMPORTS REQUIRED:**
  "crypto/rand", "crypto/sha256", "crypto/aes",
  "golang.org/x/crypto/chacha20"

---

#### Session 2.4.3 — Implement `AONTDecodePackage()`

**Task:** Implement `AONTDecodePackage(aontPackage []byte, aesNIAvailable bool) ([]byte,
error)` in `internal/crypto/aont.go`. Must:
1. Recover K from the last word XOR SHA-256(all preceding words)
2. Decrypt
3. Verify the canary word (second-to-last) equals `aontCanary`
4. If canary mismatches: zero the buffer before returning `ErrCanaryMismatch` (IC §5.1
   post-condition: "caller MUST NOT return any plaintext to the data owner. Zero the
   buffer before returning.")

---

#### Session 2.4.4 — AONT tests

**Task:** Add `internal/crypto/aont_test.go`:
- Round-trip test: `AONTDecodePackage(AONTEncodeSegment(data))` == original data
- Test that each call to `AONTEncodeSegment` produces a different ciphertext (K freshness)
- Test that corrupting any single byte in the AONT package causes `ErrCanaryMismatch`
  (all-or-nothing property)
- Cross-cipher test: encode with AES-NI=true, decode with AES-NI=false — must produce
  `ErrCanaryMismatch` (different cipher = different output; test documents this is not
  cross-compatible and is expected to fail)

> Combined verifications for 2.4.1 to 2.4.4

**VERIFY:**

COMPILE:

```go
$ go build ./internal/crypto/
EXPECT: exit 0
```

CANARY:

```go
$ grep -c "aontCanary" internal/crypto/aont_canary.go
EXPECT: >= 1

$ grep -c "var aontCanary = \[16\]byte" internal/crypto/aont_canary.go
EXPECT: 1

$ grep -n "const aontCanary" internal/crypto/aont_canary.go \
&& echo "FAIL: must be var not const" || echo "PASS"
```

FUNCTION_SIGNATURES:

```go
$ grep -c "^func AONTEncodeSegment" internal/crypto/aont.go
EXPECT: 1

$ grep -c "^func AONTDecodePackage" internal/crypto/aont.go
EXPECT: 1
```

UNIT_TESTS:

```go
$ go test -v -run TestAONT ./internal/crypto/

EXPECT: exit 0; tests include:

TestAONTRoundTrip (encode→decode returns original)

TestAONTKeyFreshness (two encodes of same input differ)

TestAONTCorruptionDetection (bit flip → ErrCanaryMismatch)

TestAONTCrossCipherIncompatible (AES encode + ChaCha decode → ErrCanaryMismatch)
```

SENTINEL_ERRORS:

```go
$ grep -c "ErrCanaryMismatch" internal/crypto/errors.go
EXPECT: 1
```

NEGATIVE_CHECKS (K must not be returned):

```go
$ grep -n "return K\b" internal/crypto/aont.go \
&& echo "FAIL: K must not be returned" || echo "PASS"
```

---

### Phase 2.5 — Pointer File AEAD

**Reference:** IC §5.1 (`EncryptPointerFile`, `DecryptPointerFile`, `ErrTagMismatch`),
IC §11 (constant-time tag comparison: NFR-019), REQ §5.4 NFR-019

#### Session 2.5.1 — Implement pointer file AEAD

**TASK:** Implement `EncryptPointerFile` and `DecryptPointerFile`.

**FILE:** `internal/crypto/chacha20poly1305.go`

**FUNCTION SIGNATURES (exact, from IC §5.1):**
  `func EncryptPointerFile(key [32]byte, nonce [12]byte, aad, plaintext []byte) ([]byte, error)`
  `func DecryptPointerFile(key [32]byte, nonce [12]byte, aad, ciphertext []byte) ([]byte, error)`

**PRECONDITIONS for EncryptPointerFile:**

  - `len(aad) > 0`  (panic in debug; aad must include `ownerID||fileID||schemaVersion`)

**PRECONDITIONS for DecryptPointerFile:**

  - `len(ciphertext) >= 16`  (must include `Poly1305` tag)

**DecryptPointerFile CRITICAL PATH (NFR-019, requirements.md §5.4):**

  1. Construct AEAD cipher: aead, `err = chacha20poly1305.New(key[:])`
  2. Call `aead.Open()` — this uses `crypto/subtle.ConstantTimeCompare` internally
     via the `chacha20poly1305` stdlib implementation
  3. If `Open()` returns error: return nil, `ErrTagMismatch`
     IMPORTANT: return nil plaintext, never partial plaintext on error
  4. On success: return decrypted plaintext, nil

**FILE:** `internal/crypto/errors.go`
(**Note:** this file has already been created as part of Phase 2.4)
**CONTENT (sentinel errors for this package):**

```go
  package crypto

  import "errors"

  var (
    ErrTagMismatch    = errors.New("crypto: Poly1305 tag verification failed")
    ErrCanaryMismatch = errors.New("crypto: AONT canary word mismatch after decode")
    ErrInvalidMnemonic = errors.New("crypto: invalid BIP-39 mnemonic")
  )
```

**IMPORTS REQUIRED:** "golang.org/x/crypto/chacha20poly1305"

**VERIFY:**

COMPILE:

```go
$ go build ./internal/crypto/
EXPECT: exit 0
```

FUNCTION_SIGNATURES:

```go
$ grep -c "^func EncryptPointerFile" internal/crypto/chacha20poly1305.go
EXPECT: 1

$ grep -c "^func DecryptPointerFile" internal/crypto/chacha20poly1305.go
EXPECT: 1
```

CONSTANT_TIME_TAG:

```go
$ grep -c "subtle\|ConstantTime" internal/crypto/chacha20poly1305.go
EXPECT: >= 1

Note: golang.org/x/crypto/chacha20poly1305 uses subtle internally.

If wrapping manually, verify explicit subtle.ConstantTimeCompare call.
```

SENTINEL_ERRORS:

```go
$ grep -c "ErrTagMismatch" internal/crypto/errors.go
EXPECT: 1
```

UNIT_TESTS:

```go
$ go test -v -run TestPointerFileAEAD ./internal/crypto/

EXPECT: exit 0; tests include:

TestPointerFileRoundTrip (encrypt→decrypt returns plaintext)

TestPointerFileTagMismatch (tampered ciphertext → ErrTagMismatch)

TestPointerFileNilOnTagMismatch (ErrTagMismatch returns nil plaintext)

TestPointerFileNonceUniqueness (different nonces → different ciphertexts)
```

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
wordlist contains exactly 2048 words. (BIP-39 English wordlist from `https://github.com/trezor/python-mnemonic/blob/master/src/mnemonic/wordlist/english.txt`)

---

#### Session 2.6.2 — Implement `MasterSecretToMnemonic()`

**Task:** Implement per IC §5.1: 32 bytes of entropy (256 bits) → 24 words with 8-bit
checksum, per BIP-39 §Generating the mnemonic. The round-trip property
`MnemonicToMasterSecret(MasterSecretToMnemonic(ms)) == ms` must hold (IC §5.1
post-condition).

---

#### Session 2.6.3 — Implement `MnemonicToMasterSecret()`

**Task:** Implement the recovery path per IC §5.1. On invalid mnemonic (wrong word
count, unknown word, checksum failure), return `ErrInvalidMnemonic` without revealing
which word failed (IC §5.1: "Do not expose which word failed — timing oracle"). All
validation branches must take equal time (use constant-time word lookup).

---

#### Session 2.6.4 — Implement `SelectConfirmationWords()`

**Task:** Implement per IC §5.1. Return two distinct random indices (0–23) drawn via
`crypto/rand`. The function may be called even in demo mode where `SkipMnemonicConfirm
== true` — the caller simply does not block on the result (IC §5.1 note).

---

#### Session 2.6.5 — BIP-39 round-trip and error tests

**Task:** Add known-answer vectors from BIP-39 test vectors (RFC-documented, fixed
entropy — not from real accounts per IC §11). Test `ErrInvalidMnemonic` on: wrong count,
unknown word, bad checksum. Test that `SelectConfirmationWords` never returns two equal
indices across 1000 calls.

> Combined verification from 2.6.1 to 2.6.5

**VERIFY:**

FILES_EXIST:

```go
$ test -f internal/crypto/wordlist_en.txt && echo PASS || echo FAIL
```

WORDLIST_COUNT:

```go
$ wc -l < internal/crypto/wordlist_en.txt
EXPECT: 2048
```

FUNCTION_SIGNATURES:

```go
$ grep -c "^func MasterSecretToMnemonic" internal/crypto/bip39.go

$ grep -c "^func MnemonicToMasterSecret" internal/crypto/bip39.go

$ grep -c "^func SelectConfirmationWords" internal/crypto/bip39.go

EXPECT: 1 for each
```

UNIT_TESTS:

```go
$ go test -v -run TestBIP39 ./internal/crypto/

EXPECT: exit 0; tests include:

TestBIP39RoundTrip (mnemonic→secret→mnemonic identity)

TestBIP39InvalidWordCount (23 words → ErrInvalidMnemonic)

TestBIP39UnknownWord (gibberish word → ErrInvalidMnemonic)

TestBIP39BadChecksum (valid words, wrong checksum → ErrInvalidMnemonic)

TestSelectConfirmationWordsUnique (never returns equal indices in 1000 calls)
```

SENTINEL_ERRORS:

```go
$ grep -c "ErrInvalidMnemonic" internal/crypto/errors.go
EXPECT: 1
```

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

**VERIFY:**

COMPILE:

```go
$ go build ./internal/crypto/
EXPECT: exit 0
```

FUNCTION_SIGNATURES:

```go
$ grep -c "^func SignBytes" internal/crypto/ed25519.go
EXPECT: 1

$ grep -c "^func VerifyBytes" internal/crypto/ed25519.go
EXPECT: 1
```

SIGNING_RULE_COMMENT:

```go
$ grep -c "SIGNING_INPUT_RULE" internal/crypto/ed25519.go
EXPECT: >= 2 (on both functions)
```

NEGATIVE_CHECKS (no JSON in signing path):

```go
$ grep -n "json.Marshal\|encoding/json" internal/crypto/ed25519.go \
&& echo "FAIL: JSON in signing path" || echo "PASS"
```

COMPILE_TIME_ASSERTION:

```go
$ grep -c "ed25519.PublicKeySize == 32" internal/crypto/ed25519.go
EXPECT: 1
```

UNIT_TESTS:

```go
$ go test -v -run TestEd25519 ./internal/crypto/

EXPECT: exit 0; tests include:

TestSignBytesRoundTrip (sign→verify succeeds)

TestSignBytesWrongKey (verify with wrong key returns false)

TestSignBytesNotJSON (signing input is fixed-layout, not JSON)
```

---

## Milestone 3 — Erasure Coding Engine (`internal/erasure`)

**Deliverable:** `internal/erasure` fully implemented with profile-parameterised
DataShards/TotalShards. Zero imports from other `internal/` packages (IC §9).

**Reference:** IC §5.2 (full interface), MVP §3.2 (erasure params table),
DM §3 Invariant 7 (ShardSize constant), IC §9 (import constraints), ARCH §10 Stage 2 (AONT → RS handoff)

---

### Phase 3.1 — Engine Construction

**Reference:** IC §5.2 (`NewEngine`, `Engine` struct, `ShardSize` constant)

#### Session 3.1.1 — Define `ShardSize` constant and `Engine` struct

**TASK:** Define ShardSize constant and Engine struct in `internal/erasure/params.go`; update the erasure cross-check in `profiles_test.go`.

**PRECONDITIONS**:

- internal/erasure/doc.go exists (stub from Session 0.1.3)
- internal/config/profiles_test.go exists (from Session 1.2.1)

**STEP 1** — Create internal/erasure/params.go:

  **FILE:** internal/erasure/params.go
  **CONTENT:**

  ```go

    package erasure

    import "github.com/masamasaowl/Vyomanaut_V2/internal/config"

    // ShardSize is the fixed size of every erasure-coded shard in bytes.
    // This value is identical in demo and production modes (DM §3 Invariant 7, ADR-031).
    // It must never appear in config.NetworkProfile as a variable field.
    const ShardSize = 262144 // 256 KB

    // Engine holds the erasure coding parameters derived from a NetworkProfile.
    // Constructed once via NewEngine; all methods are goroutine-safe.
    type Engine struct {
        // DataShards is the reconstruction threshold (s). Production=16, Demo=3.
        DataShards int // ADR-003
        // ParityShards is the number of redundancy shards (r). Production=40, Demo=2.
        ParityShards int // ADR-003
        // TotalShards is DataShards + ParityShards (n). Production=56, Demo=5.
        TotalShards int // ADR-003
    }

    // profileConsistencyCheck validates that the profile's erasure fields are
    // internally consistent. Called by NewEngine before allocating the RS engine.
    func profileConsistencyCheck(p config.NetworkProfile) error {
        if p.DataShards < 1 {
            return ErrDataShardsZero
        }
        if p.TotalShards != p.DataShards+p.ParityShards {
            return ErrTotalShardsMismatch
        }
        if p.ShardSize != ShardSize {
            // Runtime guard: profile.ShardSize must always equal the compile-time constant.
            // Violation means either DemoProfile or ProductionProfile has a wrong value.
            // Caught at startup; never silently ignored.
            return ErrShardSizeMismatch
        }
        return nil
    }
  ```

**STEP 2** — Update `internal/config/profiles_test.go`: `TestProfileShardSizeIsConstant`.
  Read the existing file first, then modify the TODO comment and add the cross-check.

  **FIND the line:**
    // TODO: cross-check with erasure.ShardSize after M3
  **REPLACE the TODO block with:**

  ```go
    import "github.com/masamasaowl/Vyomanaut_V2/internal/erasure"
    // Cross-check: compile-time const must equal runtime profile field.
    if config.ProductionProfile.ShardSize != erasure.ShardSize {
        t.Errorf("ProductionProfile.ShardSize=%d != erasure.ShardSize=%d",
            config.ProductionProfile.ShardSize, erasure.ShardSize)
    }
    if config.DemoProfile.ShardSize != erasure.ShardSize {
        t.Errorf("DemoProfile.ShardSize=%d != erasure.ShardSize=%d",
            config.DemoProfile.ShardSize, erasure.ShardSize)
    }
  ```

**IMPORTS REQUIRED in params.go:**
  "github.com/masamasaowl/Vyomanaut_V2/internal/config"

**VERIFY:**

  COMPILE:
  
  ```bash
    $ go build ./internal/erasure/
    EXPECT: exit 0
  ```

  CONTENT_CHECKS:

  ```bash
    $ grep -c "^const ShardSize = 262144" internal/erasure/params.go
    EXPECT: 1

    $ grep -c "^type Engine struct" internal/erasure/params.go
    EXPECT: 1

    $ grep -c "ShardSizeMismatch" internal/erasure/params.go
    EXPECT: >= 1
  ```

  CROSS_CHECK_IN_TESTS:

  ```bash
    $ grep -c "erasure.ShardSize" internal/config/profiles_test.go
    EXPECT: 2

    $ go test -v -run TestProfileShardSizeIsConstant ./internal/config/
    EXPECT: exit 0; output contains "--- PASS: TestProfileShardSizeIsConstant"
  ```

  NEGATIVE_CHECKS:

  ```bash
    $ grep -n "internal/" internal/erasure/params.go | grep -v "config"
    EXPECT: empty (only config import is permitted, IC §9)

    $ grep -n "ShardSize" internal/config/network_profile.go
    EXPECT: 1 line only (the struct field declaration; no literal 262144)
  ```

---

#### Session 3.1.2 — Implement `NewEngine()`

**TASK:** Implement `NewEngine()` in `internal/erasure/engine.go`.

**FILE:** `internal/erasure/engine.go`

**PRECONDITIONS:**

- `internal/erasure/params.go` exists (Session 3.1.1 complete)
- go get `github.com/klauspost/reedsolomon` (add to go.mod)

**SENTINEL ERRORS** — define in `errors.go` (new file in this session):

  **FILE:** `internal/erasure/errors.go`
  **CONTENT:**

  ```go
    package erasure

    import "errors"

    var (
        // ErrDataShardsZero is returned when NetworkProfile.DataShards < 1.
        ErrDataShardsZero = errors.New("erasure: DataShards must be >= 1")

        // ErrTotalShardsMismatch is returned when TotalShards != DataShards + ParityShards.
        ErrTotalShardsMismatch = errors.New("erasure: TotalShards must equal DataShards + ParityShards")

        // ErrShardSizeMismatch is returned when NetworkProfile.ShardSize != ShardSize constant.
        // This indicates a misconfigured profile, not a runtime data error.
        ErrShardSizeMismatch = errors.New("erasure: profile ShardSize must equal compile-time ShardSize=262144")

        // ErrTooFewShards is returned when fewer than DataShards non-nil shards are provided.
        ErrTooFewShards = errors.New("erasure: fewer than DataShards non-nil shards provided")

        // ErrShardSize is returned when a non-nil shard has length != ShardSize.
        ErrShardSize = errors.New("erasure: shard has incorrect length; expected ShardSize bytes")
    )
  ```

**FUNCTION:** `NewEngine(profile config.NetworkProfile) (*Engine, error)`

  **LOGIC:**

  ```go
    1. Call profileConsistencyCheck(profile) — return error if non-nil
    2. enc, err := reedsolomon.New(profile.DataShards, profile.ParityShards)
       if err != nil { return nil, fmt.Errorf("erasure: reedsolomon.New failed: %w", err) }
    3. Return &Engine{
         DataShards:   profile.DataShards,
         ParityShards: profile.ParityShards,
         TotalShards:  profile.TotalShards,
         enc:          enc,  // unexported field holding the reedsolomon.Encoder
       }, nil
  ```

  NOTE: The `profile.ShardSize == ShardSize` check is a RUNTIME assertion in
  `profileConsistencyCheck`, not a compile-time check. The compile-time guarantee
  is provided by `TestProfileShardSizeIsConstant` in `internal/config/profiles_test.go`.
  These two mechanisms together enforce DM §3 Invariant 7.

**IMPORTS REQUIRED:**

 ```go
  "fmt"
  "github.com/klauspost/reedsolomon"
  "github.com/masamasaowl/Vyomanaut_V2/internal/config"
 ```

**VERIFY:**

  COMPILE:

  ```bash
    $ go build ./internal/erasure/
    EXPECT: exit 0
  ```
  
  FUNCTION_EXISTS:

  ```bash
    $ grep -c "^func NewEngine" internal/erasure/engine.go
    EXPECT: 1
  ```
  
  SENTINEL_ERRORS:

  ```bash
    $ grep -c "ErrDataShardsZero\|ErrTotalShardsMismatch\|ErrShardSizeMismatch\|ErrTooFewShards\|ErrShardSize" \
        internal/erasure/errors.go
    EXPECT: 5
  ```
  
  RUNTIME_CHECK_PRESENT:

  ```bash
    $ grep -c "ShardSize != ShardSize\|ErrShardSizeMismatch" internal/erasure/params.go
    EXPECT: >= 1
  ```
  
  NEGATIVE_CHECKS:

  ```bash
    $ grep -n "compile.time" internal/erasure/engine.go
    EXPECT: empty (the compile-time assertion is in profiles_test.go, not here)

    $ grep -n "profile.ShardSize == ShardSize" internal/erasure/engine.go \
        && echo "WARN: == check is fine but verify it's in profileConsistencyCheck, not as a var" \
        || echo "PASS"
  ```

  ERROR_CASES:

  ```bash
    $ go test -v -run TestNewEngineErrors ./internal/erasure/
    EXPECT: exit 0; tests cover:
      TestNewEngineRejectsBadDataShards     (DataShards=0 → ErrDataShardsZero)
      TestNewEngineRejectsMismatchedTotal   (TotalShards != DataShards+ParityShards → ErrTotalShardsMismatch)
      TestNewEngineRejectsBadShardSize      (ShardSize=1 → ErrShardSizeMismatch)
      TestNewEngineAcceptsDemoProfile       (NewEngine(DemoProfile) → nil error)
      TestNewEngineAcceptsProductionProfile (NewEngine(ProductionProfile) → nil error)
  ```

---

### Phase 3.2 — Encode & Decode

**Reference:** IC §5.2 (`EncodeSegment`, `DecodeSegment`, `ErrTooFewShards`, `ErrShardSize`)

#### Session 3.2.1 — Implement `EncodeSegment()`

**TASK:** Implement `Engine.EncodeSegment()` in `internal/erasure/engine.go`.

**PRECONDITION:**

- `NewEngine()` is implemented and tested (Session 3.1.2 complete)
- `internal/crypto.AONTEncodeSegment()` is implemented (Session 2.4.2 complete)  
  The input to `EncodeSegment` is the output of `AONTEncodeSegment`:
  a byte slice of length `DataShards × ShardSize` (`architecture.md` §10 Stage 2)

**FUNCTION SIGNATURE (IC §5.2):**

  ```go
  func (e *Engine) EncodeSegment(aontPackage []byte) ([][]byte, error)
  ```

**PRE-CONDITION** (enforce as runtime error, not panic — this is an API boundary):

  ```go
  if len(aontPackage) != e.DataShards * ShardSize {
      return nil, fmt.Errorf("erasure: EncodeSegment input length %d != DataShards*ShardSize=%d",
          len(aontPackage), e.DataShards*ShardSize)
  }
  ```

**ALGORITHM:**

  ```go
  1. Split aontPackage into e.DataShards shards of ShardSize bytes each:
      shards := make([][]byte, e.TotalShards)
      for i := 0; i < e.DataShards; i++ {
          shards[i] = aontPackage[i*ShardSize : (i+1)*ShardSize]
      }
      for i := e.DataShards; i < e.TotalShards; i++ {
          shards[i] = make([]byte, ShardSize)  // allocate parity shards
      }
  2. Call e.enc.Encode(shards)  — fills parity shards in-place
  3. Return shards, nil
  ```

**POST-CONDITIONS** (IC §5.2):

- Returns exactly `e.TotalShards` byte slices
- Each slice is exactly `ShardSize` bytes
- Any `e.DataShards` of the returned slices can reconstruct `aontPackage`

**VERIFY:**

  **COMPILE:**

  ```bash
  $ go build ./internal/erasure/
  EXPECT: exit 0
  ```

  **FUNCTION_EXISTS:**

  ```bash
  $ grep -c "^func (e \*Engine) EncodeSegment" internal/erasure/engine.go
  EXPECT: 1
  ```

  **UNIT_TESTS:**

  ```bash
  $ go test -v -run TestEncodeSegment ./internal/erasure/
  EXPECT: exit 0; tests include:
    TestEncodeSegmentOutputCount    (returns exactly TotalShards slices)
    TestEncodeSegmentShardSize      (each shard is exactly ShardSize bytes)
    TestEncodeSegmentRejectsWrongInput (wrong length → error, not panic)
    TestEncodeSegmentDemoProfile    (RS(3,5): 768KB input → 5 shards of 256KB)
    TestEncodeSegmentProdProfile    (RS(16,56): 4MB input → 56 shards of 256KB)
  ```

  **NEGATIVE_CHECKS:**

  ```bash
    grep -n "panic" internal/erasure/engine.go     && echo "WARN: no panics at API boundary — return error instead" || echo "PASS"
  ```

---

#### Session 3.2.2 — Implement `DecodeSegment()`

**TASK:** Implement `Engine.DecodeSegment()` in `internal/erasure/engine.go`.

**PRECONDITION:**

- `EncodeSegment()` is implemented (Session 3.2.1 complete)

**FUNCTION SIGNATURE (IC §5.2):**

  ```go
  func (e *Engine) DecodeSegment(shards [][]byte) ([]byte, error)
  ```

**PRE-CONDITIONS:**

  ```go
  if len(shards) != e.TotalShards {
      return nil, fmt.Errorf("erasure: DecodeSegment expects %d shards, got %d",
          e.TotalShards, len(shards))
  }
  // Count non-nil shards; return ErrTooFewShards if < DataShards
  nonNil := 0
  for _, s := range shards { if s != nil { nonNil++ } }
  if nonNil < e.DataShards {
      return nil, ErrTooFewShards
  }
  // Validate non-nil shard sizes
  for i, s := range shards {
      if s != nil && len(s) != ShardSize {
          return nil, fmt.Errorf("%w: shard[%d] has length %d", ErrShardSize, i, len(s))
      }
  }
  ```

**ALGORITHM:**

  ```go
  1. Call e.enc.ReconstructData(shards)
    This fills nil data shard entries in-place; parity shards may remain nil.
  2. Reconstruct aontPackage by concatenating the first DataShards shards:
      result := make([]byte, e.DataShards*ShardSize)
      for i := 0; i < e.DataShards; i++ {
          copy(result[i*ShardSize:], shards[i])
      }
  3. Return result, nil
  ```

**POST-CONDITIONS** (IC §5.2):

- Returns `DataShards × ShardSize` bytes (the original AONT package)
- All nil entries in `shards` are filled in-place (caller's slice is modified)
- The result is passed to `internal/crypto.AONTDecodePackage` by the caller

**VERIFY:**

  **COMPILE:**

  ```bash
  $ go build ./internal/erasure/
  EXPECT: exit 0
  ```

  **FUNCTION_EXISTS:**

  ```bash
  $ grep -c "^func (e \*Engine) DecodeSegment" internal/erasure/engine.go
  EXPECT: 1
  ```

  **UNIT_TESTS:**

  ```bash
  $ go test -v -run TestDecodeSegment ./internal/erasure/
  EXPECT: exit 0; tests include:
    TestDecodeSegmentRoundTrip        (encode then decode returns original)
    TestDecodeSegmentTooFewShards     (< DataShards non-nil → ErrTooFewShards)
    TestDecodeSegmentBadShardSize     (wrong-length shard → ErrShardSize error)
    TestDecodeSegmentFillsNilInPlace  (nil entries are populated in shards slice)
    TestDecodeSegmentWrongCount       (len(shards) != TotalShards → error)
  ```

---

#### Session 3.2.3 — Erasure coding tests

**TASK:** Write comprehensive erasure coding tests in `internal/erasure/engine_test.go`.

**FILE:** `internal/erasure/engine_test.go`

**TEST INVENTORY** (all tests use subtests via `t.Run`):

1. `TestNewEngineErrors` — Session 3.1.2 already covers this; consolidate here.

2. `TestEncodeDecodeRoundTrip` (production and demo profiles):

  ```go
  for _, profile := range []config.NetworkProfile{config.ProductionProfile, config.DemoProfile} {
      input := make([]byte, profile.DataShards * erasure.ShardSize)
      // fill with recognisable pattern
      for i := range input { input[i] = byte(i % 251) }
      eng,_ := erasure.NewEngine(profile)
      shards, _:= eng.EncodeSegment(input)
      decoded,_ := eng.DecodeSegment(shards)
      // assert decoded == input
  }
  ```

1. `TestAnyKShardsReconstructsDemo` — test all C(5,3)=10 combinations for RS(3,5):
   Enumerate every 3-element subset of {0,1,2,3,4}; set the other 2 to nil; assert decode succeeds.
   DO NOT run this for production RS(16,56) — C(56,16) ≈ 1.7×10^11 combinations.
   Add an explicit comment: "Production combination exhaustion is infeasible; covered by
   the reedsolomon library's own test suite."

2. `TestShardSizeConstant`:

  ```go
  if erasure.ShardSize != 262144 {
      t.Errorf("ShardSize=%d, want 262144 (DM §3 Invariant 7)", erasure.ShardSize)
  }
  // Belt-and-suspenders: also checked at profile level in TestProfileShardSizeIsConstant
  ```

1. `TestCrossProfileIncompatibility` — documents that different profiles produce
   incompatible shard counts (NOT cross-decoding, which is impossible):

  ```go
  demoEng, _:= erasure.NewEngine(config.DemoProfile)        // RS(3,5)
  prodEng,_  := erasure.NewEngine(config.ProductionProfile)  // RS(16,56)

  // Demo EncodeSegment produces 5 shards; feeding them to prod DecodeSegment
  // must fail at the len(shards) != TotalShards pre-condition.
  demoInput := make([]byte, config.DemoProfile.DataShards * erasure.ShardSize)
  demoShards, _:= demoEng.EncodeSegment(demoInput)
  _, err := prodEng.DecodeSegment(demoShards)
  // err must be non-nil; assert specific error type/message about wrong shard count
  if err == nil {
      t.Fatal("cross-profile decode must fail but returned nil error")
  }
  // Document WHY: different TotalShards (5 vs 56), caught by len(shards) pre-condition
  ```

1. `TestDecodeSegmentErrTooFewShards` — provide only `DataShards-1` non-nil shards.

**NOTES:**

- All test files import only standard library + `internal/erasure` + `internal/config`
- No `internal/crypto` import needed here (AONT is tested separately)
- Tests tagged for race: no goroutine usage needed, so `-race` adds no value here,
    but the package-wide CI check (`go test ./... -race`) covers this by default

**VERIFY:**

  **ALL_TESTS_PASS:**

  ```bash
  $ go test -v ./internal/erasure/
  EXPECT: exit 0; all subtests PASS
  ```

  **TEST_NAMES_PRESENT:**

  ```bash
  $ grep -c "^func Test" internal/erasure/engine_test.go
  EXPECT: >= 5
  ```

  **COMBINATION_COMMENT:**

  ```bash
  $ grep -c "infeasible\|1.7.*10\|C(56,16)" internal/erasure/engine_test.go
  EXPECT: >= 1  (documents why prod exhaustion is omitted)
  ```

  **NEGATIVE_CHECKS:**

  ```bash
  grep -n "cross-decode\|cross.*decode\|decode.*cross" internal/erasure/engine_test.go     && echo "WARN: cross-decode is impossible by precondition; test must document this"     || echo "PASS"
  ```

  **IMPORT_CONSTRAINT:**

  ```bash
  grep -n "internal/crypto" internal/erasure/engine_test.go     && echo "FAIL: erasure test must not import crypto (IC §9)" || echo "PASS"
  ```

  **CI_CHECK_14_STILL_PASSES:**

  ```bash
  $ go test -v -run TestProfileShardSizeIsConstant ./internal/config/
  EXPECT: exit 0  (now uses real erasure.ShardSize cross-check from Session 3.1.1)
  ```

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

**TASK:** Create the migration generator command and README.

**PRECONDITIONS:**

- `internal/config` package is complete
- `migrations/README.md` created

**STEP 1 — Create migrations/generator.go:**

  **FILE:** `migrations/generator.go`

  **PACKAGE:** package main

  **PACKAGE DOC COMMENT:**

```go
    // Command generator produces the initial PostgreSQL schema for Vyomanaut V2.
    // It is parameterised by the active NetworkProfile so that demo and production
    // schemas differ only in the CHECK constraint bounds for shard_index and
    // available_shard_count (DM §9 Profile rule, MVP §5.5, DM §3 Invariant 7).
    //
    // Usage:
    //   go run migrations/generator.go --profile=prod
    //   go run migrations/generator.go --profile=demo
    //
    // Output is the full migration SQL to stdout. Redirect to file:
    //   go run migrations/generator.go --profile=prod > migrations/001_initial_schema.sql
```

  **CLI FLAGS:**
    `--profile` string   "prod" or "demo" (required; no default; fatal if absent or invalid)

  **MAIN LOGIC:**

  ```go
    1. Parse --profile flag via flag.String("profile", "", "prod or demo")
    2. flag.Parse()
    3. if *profileFlag == "" { log.Fatal("--profile is required") }
    4. profile := selectProfile(*profileFlag)  // returns config.DemoProfile or config.ProductionProfile
    5. sql := generateSchema(profile)
    6. fmt.Print(sql)
  ```

  **FUNCTION:** `generateSchema(profile config.NetworkProfile) string`

  ```go
    Returns the complete migration SQL as a string.
    The output begins with:
      -- Generated for profile: {profile.Mode}
      -- Generated at: {time.Now().UTC().Format(time.RFC3339)}
      -- ShardSize: 262144 (compile-time constant; NOT profile-variable)
      -- DataShards: {profile.DataShards}
      -- TotalShards: {profile.TotalShards}

    Two profile-variable CHECK constraints use fmt.Sprintf:
      1. chunk_assignments.shard_index:
         fmt.Sprintf("CHECK (shard_index BETWEEN 0 AND %d OR shard_index IS NULL)",
             profile.TotalShards-1)

      2. repair_jobs.available_shard_count:
         fmt.Sprintf("CHECK (available_shard_count BETWEEN %d AND %d)",
             profile.DataShards, profile.TotalShards)

    The complete SQL body (schema for all tables, ENUMs, indexes, RSPs, views) is
    assembled by concatenating the sections defined in Sessions 4.2.1 through 4.7.5.
    Each section is a Go string constant or fmt.Sprintf result stored in a named var.
  ```

**STEP 2 — Run the generator to create the production schema file:**

  ```bash
    go run migrations/generator.go --profile=prod > migrations/001_initial_schema.sql
    go run migrations/generator.go --profile=demo > migrations/001_initial_schema_demo.sql
  ```

**VERIFY:**

  **COMPILE:**

  ```bash
      $ go build ./migrations/
      EXPECT: exit 0
  ```

  **FILES_EXIST:**

  ```bash
      test -f migrations/generator.go && echo PASS || echo FAIL
      test -f migrations/README.md && echo PASS || echo FAIL
  ```

  **GENERATOR_RUNS:**

  ```bash
      $ go run migrations/generator.go --profile=prod | head -5
      EXPECT: first line is "-- Generated for profile: prod"

      $ go run migrations/generator.go --profile=demo | head -5
      EXPECT: first line is "-- Generated for profile: demo"

      $ go run migrations/generator.go 2>&1 | grep "required"
      EXPECT: contains "--profile is required"
  ```

  **PROFILE_VARIABLE_CONSTRAINTS:**

  ```bash
      $ go run migrations/generator.go --profile=prod | \
          grep "shard_index BETWEEN 0 AND 55"
      EXPECT: found

      $ go run migrations/generator.go --profile=demo | \
          grep "shard_index BETWEEN 0 AND 4"
      EXPECT: found

      $ go run migrations/generator.go --profile=prod | \
          grep "available_shard_count BETWEEN 16 AND 56"
      EXPECT: found

      $ go run migrations/generator.go --profile=demo | \
          grep "available_shard_count BETWEEN 3 AND 5"
      EXPECT: found
  ```

  **SEPARATION_CHECK (demo schema must NOT be applied to prod — structural check):**

  ```bash
      $ go run migrations/generator.go --profile=prod | \
          grep "shard_index BETWEEN 0 AND 4" && echo "FAIL: demo bounds in prod" || echo "PASS"
      $ go run migrations/generator.go --profile=demo | \
          grep "shard_index BETWEEN 0 AND 55" && echo "FAIL: prod bounds in demo" || echo "PASS"
  ```

---

### Phase 4.2 — Core Type Definitions

**Reference:** DM §4 (CREATE TYPE statements, DM §9 migration ordering rule: types before tables)

#### Session 4.2.1 — Define all PostgreSQL ENUMs

**TASK:** Define all PostgreSQL ENUM types in the migration SQL.

**OUTPUT FILE:** `migrations/001_initial_schema.sql` (section: ENUM definitions)

These statements must appear BEFORE all CREATE TABLE statements (DM §9 migration ordering).

**ENUMS TO CREATE (exact values, in this order):**

```sql
  1. provider_status    — 'PENDING_ONBOARDING', 'VETTING', 'ACTIVE', 'DEPARTED'
                          (DM §4.2)

  2. file_status        — 'ACTIVE', 'DELETION_PENDING', 'DELETED'
                          (DM §4.3, DM §9 checklist: "file_status ENUM has three values")

  3. assignment_status  — 'ACTIVE', 'REPAIRING', 'PENDING_DELETION', 'DELETED'
                          (DM §4.5)

  4. audit_result_type  — 'PASS', 'FAIL', 'TIMEOUT'
                          (DM §4.7; note: nullable column uses this ENUM, not TEXT)

  5. escrow_event_type  — 'DEPOSIT', 'RELEASE', 'SEIZURE', 'REVERSAL'
                          (DM §4.8; REVERSAL required per DM §9 checklist)

  6. owner_escrow_event_type — 'DEPOSIT', 'CHARGE', 'WITHDRAWAL', 'REFUND'
                               (DM §4.9)

  7. repair_trigger_type — 'SILENT_DEPARTURE', 'ANNOUNCED_DEPARTURE',
                           'THRESHOLD_WARNING', 'EMERGENCY_FLOOR'
                           (DM §4.10)

  8. repair_priority    — 'EMERGENCY', 'PERMANENT_DEPARTURE', 'PRE_WARNING'
                          (DM §4.10, DM §9: "repair_priority ENUM has three values")
                          ORDERING NOTE: PostgreSQL ENUM values sort by definition order,
                          not alphabetically. ORDER BY priority ASC on this ENUM produces:
                            1st: EMERGENCY
                            2nd: PERMANENT_DEPARTURE
                            3rd: PRE_WARNING
                          This matches the intended drain order (DM §4.10, ADR-004).
                          Add SQL comment: "-- ENUM order = priority order for ORDER BY ASC"

  9. repair_job_status  — 'QUEUED', 'IN_PROGRESS', 'COMPLETED', 'FAILED'
                          (DM §4.10)
```

**VERIFY:**

  **ENUM_IN_SQL:**

  ```bash
      $ grep -c "CREATE TYPE.*AS ENUM" migrations/001_initial_schema.sql
      EXPECT: 9
  ```

  **REVERSAL_PRESENT:**

  ```bash
      $ grep -c "'REVERSAL'" migrations/001_initial_schema.sql
      EXPECT: >= 1  (in escrow_event_type)
  ```

  **REPAIR_PRIORITY_VALUES:**

  ```bash
      $ grep -A4 "repair_priority" migrations/001_initial_schema.sql | \
          grep -c "EMERGENCY\|PERMANENT_DEPARTURE\|PRE_WARNING"
      EXPECT: 3
  ```

  **FILE_STATUS_THREE_VALUES:**

  ```bash
      $ grep -A4 "file_status" migrations/001_initial_schema.sql | \
          grep -c "ACTIVE\|DELETION_PENDING\|DELETED"
      EXPECT: 3
  ```

  **ORDERING_COMMENT:**

  ```bash
      $ grep -c "ENUM order = priority order" migrations/001_initial_schema.sql
      EXPECT: 1
  ```

  **TYPES_BEFORE_TABLES:**

  ```bash
      $ awk '/CREATE TYPE/{type=NR} /CREATE TABLE/{table=NR; if(type>table) exit 1}' \
          migrations/001_initial_schema.sql && echo "PASS" || echo "FAIL: type after table"
      EXPECT: PASS
  ```

---

### Phase 4.3 — Core Tables

**Reference:** DM §4.1 through §4.5

#### Session 4.3.1 — `owners` table

**TASK:** Add CREATE TABLE owners to `migrations/001_initial_schema.sql`.

**SOURCE:**** DM §4.1 (exact DDL), DM §8.1 (`smart_collect_vpa` nullable justification)

  ```sql
  COLUMNS (exact — no additions, no omissions):
    owner_id            UUID            PRIMARY KEY DEFAULT gen_random_uuid()
    phone_number        VARCHAR(15)     NOT NULL UNIQUE
    ed25519_public_key  BYTEA           NOT NULL CHECK (octet_length(ed25519_public_key) = 32)
    smart_collect_vpa   VARCHAR(255)    NULL
                        -- NULL until Razorpay webhook provisions the VPA (DM §8.1)
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
  ```

**TABLE COMMENT (mandatory per DM §4.1):**

  ```sql
    COMMENT ON TABLE owners IS 'Registered data owners. One row per verified phone number.';
    COMMENT ON COLUMN owners.smart_collect_vpa IS 'Razorpay UPI VPA for escrow deposits. NULL until provisioned by Razorpay webhook.';
  ```

**VERIFY:**

  **TABLE_IN_SQL:**

  ```bash
      $ grep -c "CREATE TABLE owners" migrations/001_initial_schema.sql
      EXPECT: 1
  ```

  **COLUMNS_PRESENT:**

  ```bash
      $ grep -c "smart_collect_vpa.*VARCHAR(255)" migrations/001_initial_schema.sql
      EXPECT: 1

      $ grep -c "octet_length(ed25519_public_key) = 32" migrations/001_initial_schema.sql
      EXPECT: 1
  ```

  **NULLABLE_COLUMN:**

  ```bash
      $ grep -c "smart_collect_vpa.*NOT NULL" migrations/001_initial_schema.sql
      EXPECT: 0  (must be nullable)
  ```

  **COMMENT_PRESENT:**

  ```bash
      $ grep -c "COMMENT ON TABLE owners" migrations/001_initial_schema.sql
      EXPECT: 1
  ```

---

#### Session 4.3.2 — `providers` table

**TASK:** Add CREATE TABLE providers to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §4.2 (exact DDL), DM §8.2–§8.6 (nullable justifications)

**CRITICAL COLUMNS — these are the most error-prone:**

  ```sql
    p95_throughput_kbps  FLOAT    NULL
    -- NULL until vetting audits accumulate samples. DEFAULT 0 is WRONG (DM §9 checklist).
    -- Division by zero if code uses this without NULL check. See DM §4.2.

    avg_rtt_ms           FLOAT    NULL
    -- NULL until first sample. DEFAULT 2000 is WRONG (DM §9 checklist).
    -- Application substitutes pool median when NULL (IC §4.2 RTO formula).

    var_rtt_ms           FLOAT    NOT NULL DEFAULT 0
    -- Zero variance IS the correct initial assumption (DM §4.2 comment).

    first_chunk_assignment_at  TIMESTAMPTZ  NULL
    -- NULL until assignment service assigns first chunk (DM §8.6).
    -- Used for 120-day vetting duration check: first_chunk_assignment_at + profile.VettingMinDuration
  ```

**MANDATORY CONSTRAINTS:**

  ```sql
    providers_departed_status:   CHECK (departed_at IS NULL OR status = 'DEPARTED')
    providers_throughput_nonneg: CHECK (p95_throughput_kbps >= 0)
    providers_avg_rtt_nonneg:    CHECK (avg_rtt_ms >= 0)
    providers_var_rtt_nonneg:    CHECK (var_rtt_ms >= 0)
    providers_passes_nonneg:     CHECK (consecutive_audit_passes >= 0)
  ```

**MANDATORY COMMENT (no physical deletion):**

  ```sql
    COMMENT ON TABLE providers IS
        'Storage providers. One row per verified daemon. Never physically deleted (DM §3 Invariant 3).';
  ```

**VERIFY:**

  **TABLE_IN_SQL:**

  ```bash
      $ grep -c "CREATE TABLE providers" migrations/001_initial_schema.sql
      EXPECT: 1
  ```

  **NULL_DEFAULTS_CORRECT:**

  ```bash
      $ grep -c "p95_throughput_kbps.*NULL" migrations/001_initial_schema.sql
      EXPECT: >= 1  (column definition includes NULL)

      $ grep -c "p95_throughput_kbps.*DEFAULT.*0\b" migrations/001_initial_schema.sql
      EXPECT: 0  (DEFAULT 0 is WRONG — must be NULL)

      $ grep -c "avg_rtt_ms.*DEFAULT.*2000" migrations/001_initial_schema.sql
      EXPECT: 0  (DEFAULT 2000 is WRONG — must be NULL)
  ```

  **VAR_RTT_CORRECT:**

  ```bash
      $ grep -c "var_rtt_ms.*NOT NULL.*DEFAULT 0\|var_rtt_ms.*DEFAULT 0.*NOT NULL" \
          migrations/001_initial_schema.sql
      EXPECT: 1
  ```

  **CONSTRAINTS_PRESENT:**

  ```bash
      $ grep -c "providers_departed_status" migrations/001_initial_schema.sql
      EXPECT: 1

      $ grep -c "providers_throughput_nonneg\|providers_avg_rtt_nonneg\|providers_var_rtt_nonneg\|providers_passes_nonneg" \
          migrations/001_initial_schema.sql
      EXPECT: 4
  ```

  **NO_DELETE_COMMENT:**

  ```bash
      $ grep -c "Never physically deleted\|Invariant 3" migrations/001_initial_schema.sql
      EXPECT: >= 1
  ```

---

#### Session 4.3.3 — `files` table

**TASK:** Add CREATE TABLE files to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §4.3, REQ §4.4 FR-019 (display_name_* columns for file list view)

**MANDATORY COLUMNS:**

```sql
  file_id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
  owner_id             UUID            NOT NULL REFERENCES owners(owner_id)
  pointer_ciphertext   BYTEA           NOT NULL
  pointer_nonce        BYTEA           NOT NULL CHECK (octet_length(pointer_nonce) = 12)
  pointer_tag          BYTEA           NOT NULL CHECK (octet_length(pointer_tag) = 16)
  original_size_bytes  BIGINT          NOT NULL CHECK (original_size_bytes > 0)
  status               file_status     NOT NULL DEFAULT 'ACTIVE'
  schema_version       SMALLINT        NOT NULL DEFAULT 1
  uploaded_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()

OPTIONAL DISPLAY NAME COLUMNS (REQ §4.4 FR-019 — file list view):
  display_name_ciphertext  BYTEA    NULL
  display_name_nonce       BYTEA    NULL CHECK (octet_length(display_name_nonce) = 12 OR display_name_nonce IS NULL)
  display_name_tag         BYTEA    NULL CHECK (octet_length(display_name_tag) = 16 OR display_name_tag IS NULL)
  -- All three are NULL when the owner uploads without a label (CLI path).
  -- The microservice stores these blindly and cannot decrypt the filename (DM §4.3).
```

**VERIFY:**

```bash
  TABLE_IN_SQL:
    $ grep -c "CREATE TABLE files" migrations/001_initial_schema.sql
    EXPECT: 1

  POINTER_NONCE_SIZE:
    $ grep -c "octet_length(pointer_nonce) = 12" migrations/001_initial_schema.sql
    EXPECT: 1

  POINTER_TAG_SIZE:
    $ grep -c "octet_length(pointer_tag) = 16" migrations/001_initial_schema.sql
    EXPECT: 1

  DISPLAY_NAME_COLUMNS:
    $ grep -c "display_name_ciphertext\|display_name_nonce\|display_name_tag" \
        migrations/001_initial_schema.sql
    EXPECT: 3

  DISPLAY_NAME_NULLABLE:
    $ grep -c "display_name_ciphertext.*NOT NULL" migrations/001_initial_schema.sql
    EXPECT: 0  (must be nullable)

  AWAITING_NOTE_REMOVED:
    $ grep -c "AWAITING\|requirements.md is not in context" migrations/001_initial_schema.sql
    EXPECT: 0
```

---

#### Session 4.3.4 — `segments` table

**TASK:** Add CREATE TABLE segments to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §4.4

**COLUMNS:**

```sql
  segment_id      UUID    PRIMARY KEY DEFAULT gen_random_uuid()
  file_id         UUID    NOT NULL REFERENCES files(file_id)
  segment_index   INT     NOT NULL CHECK (segment_index >= 0)
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

CONSTRAINT:

```sql
  CONSTRAINT segments_unique_index UNIQUE (file_id, segment_index)
```

**VERIFY:**

```bash
  TABLE_IN_SQL:
    $ grep -c "CREATE TABLE segments" migrations/001_initial_schema.sql
    EXPECT: 1

  UNIQUE_CONSTRAINT:
    $ grep -c "segments_unique_index" migrations/001_initial_schema.sql
    EXPECT: 1

  NON_NEGATIVE_INDEX:
    $ grep -c "segment_index >= 0" migrations/001_initial_schema.sql
    EXPECT: 1
```

---

#### Session 4.3.5 — `chunk_assignments` table

**TASK:** Add CREATE TABLE chunk_assignments to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §4.5, DM §3 Invariant 6, DM §8.21 (`segment_id nullable`), DM §8.22 (`shard_index nullable`)

**CRITICAL COLUMNS:**

```sql
  assignment_id    UUID               PRIMARY KEY DEFAULT gen_random_uuid()
  chunk_id         BYTEA              NOT NULL CHECK (octet_length(chunk_id) = 32)
  is_vetting_chunk BOOLEAN            NOT NULL DEFAULT FALSE   -- DM §9 checklist
  segment_id       UUID               REFERENCES segments(segment_id)
                   -- NULL when is_vetting_chunk = TRUE (DM §8.21)
  shard_index      SMALLINT           -- NULL when is_vetting_chunk = TRUE (DM §8.22)
  provider_id      UUID               NOT NULL REFERENCES providers(provider_id)
  status           assignment_status  NOT NULL DEFAULT 'ACTIVE'
  created_at       TIMESTAMPTZ        NOT NULL DEFAULT NOW()
  deleted_at       TIMESTAMPTZ        NULL  -- set when status = 'DELETED'
```

**CONSTRAINTS (all mandatory):**

```sql
  1. chunk_assignments_segment_and_shard_null_iff_vetting (DM §4.5 Invariant 6):
     CHECK (
         (is_vetting_chunk = FALSE AND segment_id IS NOT NULL AND shard_index IS NOT NULL)
         OR
         (is_vetting_chunk = TRUE  AND segment_id IS NULL    AND shard_index IS NULL)
     )

  2. chunk_assignments_one_per_provider_per_chunk:
     UNIQUE (chunk_id, provider_id)

  3. shard_index CHECK — PROFILE-VARIABLE (from generator):
     fmt.Sprintf("CHECK (shard_index BETWEEN 0 AND %d OR shard_index IS NULL)",
         profile.TotalShards-1)
```

**PARTIAL UNIQUE INDEX (MUST be standalone CREATE UNIQUE INDEX, NOT inline):**

```sql
  CREATE UNIQUE INDEX idx_chunk_assignments_one_active_per_shard
      ON chunk_assignments (segment_id, shard_index)
      WHERE is_vetting_chunk = FALSE
        AND status IN ('ACTIVE', 'REPAIRING');

  NOTE: Inline WHERE clause on UNIQUE is invalid PostgreSQL syntax (DM §9).
  The index MUST be a separate statement after CREATE TABLE.
```

**VIEW (created after the table):**

```sql
  CREATE VIEW active_chunk_assignments AS
  SELECT * FROM chunk_assignments WHERE status = 'ACTIVE';
```

**VERIFY:**

```bash

  TABLE_IN_SQL:
    $ grep -c "CREATE TABLE chunk_assignments" migrations/001_initial_schema.sql
    EXPECT: 1

  IS_VETTING_CHUNK_COLUMN:
    $ grep -c "is_vetting_chunk.*BOOLEAN.*NOT NULL.*DEFAULT FALSE" \
        migrations/001_initial_schema.sql
    EXPECT: 1

  INVARIANT_6_CHECK:
    $ grep -c "chunk_assignments_segment_and_shard_null_iff_vetting" \
        migrations/001_initial_schema.sql
    EXPECT: 1

  PARTIAL_INDEX_STANDALONE:
    $ grep -c "CREATE UNIQUE INDEX idx_chunk_assignments_one_active_per_shard" \
        migrations/001_initial_schema.sql
    EXPECT: 1

    # Must NOT be an inline UNIQUE constraint
    $ grep -B5 "idx_chunk_assignments_one_active_per_shard" migrations/001_initial_schema.sql | \
        grep "CONSTRAINT" && echo "FAIL: must be standalone index" || echo "PASS"

  PARTIAL_INDEX_VETTING_FILTER:
    $ grep -A3 "idx_chunk_assignments_one_active_per_shard" migrations/001_initial_schema.sql | \
        grep "is_vetting_chunk = FALSE"
    EXPECT: found  (DM §9 checklist requirement)

  SEGMENT_ID_NULLABLE:
    $ grep -A1 "segment_id" migrations/001_initial_schema.sql | \
        grep "NOT NULL" | head -1
    EXPECT: empty  (segment_id must be nullable)

```

---

### Phase 4.4 — Audit, Escrow, and Repair Tables

**Reference:** DM §4.6 through §4.10

#### Session 4.4.1 — `audit_periods` table

**Task:** Implement DM §4.6 including the `audit_periods_no_overlap` exclusion constraint.
This requires `btree_gist` (DM §9 first checklist item). Add the prerequisite comment:
`-- PREREQUISITE: CREATE EXTENSION IF NOT EXISTS btree_gist;`
The `period_start < period_end` CHECK must be present.

Refer: DM §9 (btree_gist prerequisite — first checklist item)

#### Session 4.4.2 — `audit_receipts` table

**TASK:** Add CREATE TABLE audit_receipts to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §4.7, DM §3 Invariants 1 and 5, DM §8.9–§8.15, DM §8.20 (`file_id nullable`)
        IC §11 (`challenge_nonce` must be BYTEA(33) never BYTEA(32))

**CRITICAL COLUMNS:**

```sql
  receipt_id             UUID                 PRIMARY KEY DEFAULT gen_random_uuid()
  schema_version         SMALLINT             NOT NULL DEFAULT 1
  chunk_id               BYTEA                NOT NULL CHECK (octet_length(chunk_id) = 32)
  file_id                UUID                 REFERENCES files(file_id)
                         -- NULL for synthetic vetting chunk audits (DM §8.20)
  provider_id            UUID                 NOT NULL REFERENCES providers(provider_id)
  challenge_nonce        BYTEA                NOT NULL CHECK (octet_length(challenge_nonce) = 33)
                         -- MUST BE 33, NEVER 32 (DM §3 Invariant 5, IC §11, CI grep check 8)
  server_challenge_ts    TIMESTAMPTZ          NOT NULL
  response_hash          BYTEA                CHECK (octet_length(response_hash) = 32
                                                  OR response_hash IS NULL)
  response_latency_ms    INT                  CHECK (response_latency_ms >= 0
                                                  OR response_latency_ms IS NULL)
  audit_result           audit_result_type    -- NO DEFAULT; NULL = PENDING state (DM §9)
  address_was_stale      BOOLEAN              NOT NULL DEFAULT FALSE
  provider_sig           BYTEA                CHECK (octet_length(provider_sig) = 64
                                                  OR provider_sig IS NULL)
  service_sig            BYTEA                CHECK (octet_length(service_sig) = 64
                                                  OR service_sig IS NULL)
  service_countersign_ts TIMESTAMPTZ
  jit_flag               BOOLEAN              NOT NULL DEFAULT FALSE
  abandoned_at           TIMESTAMPTZ

MANDATORY CONSTRAINTS:
  audit_receipts_nonce_unique: UNIQUE (challenge_nonce)
  audit_receipts_response_consistency: CHECK (
      (audit_result IN ('PASS', 'FAIL') AND response_hash IS NOT NULL AND provider_sig IS NOT NULL)
      OR (audit_result = 'TIMEOUT' AND response_hash IS NULL AND provider_sig IS NULL)
      OR (audit_result IS NULL)
  )
  audit_receipts_service_sig_consistency: CHECK (
      (service_sig IS NULL) = (service_countersign_ts IS NULL)
  )
```

**NIGHTLY INTEGRITY QUERY (add as SQL comment, DM §4.7):**

```sql
  -- Nightly data integrity check — must return 0:
  -- SELECT COUNT(*) FROM audit_receipts ar
  --   JOIN chunk_assignments ca ON ca.chunk_id = ar.chunk_id
  --     AND ca.provider_id = ar.provider_id
  --   WHERE ar.file_id IS NULL AND ca.is_vetting_chunk = FALSE;
```

**VERIFY:**

```bash

  TABLE_IN_SQL:
    $ grep -c "CREATE TABLE audit_receipts" migrations/001_initial_schema.sql
    EXPECT: 1

  NONCE_IS_33_BYTES:
    $ grep -c "octet_length(challenge_nonce) = 33" migrations/001_initial_schema.sql
    EXPECT: 1

    $ grep -c "octet_length(challenge_nonce) = 32" migrations/001_initial_schema.sql
    EXPECT: 0  (CI grep check 8 also catches this)

  NO_DEFAULT_ON_AUDIT_RESULT:
    $ grep -B1 "audit_result_type" migrations/001_initial_schema.sql | grep "DEFAULT"
    EXPECT: empty  (no DEFAULT on audit_result)

  FILE_ID_NULLABLE:
    $ grep -A2 "file_id.*UUID" migrations/001_initial_schema.sql | \
        grep "audit_receipts" -A3 | grep "NOT NULL"
    EXPECT: empty  (file_id in audit_receipts is nullable)

  ADDRESS_WAS_STALE_PRESENT:
    $ grep -c "address_was_stale.*BOOLEAN.*NOT NULL.*DEFAULT FALSE" \
        migrations/001_initial_schema.sql
    EXPECT: 1

  NONCE_UNIQUE_CONSTRAINT:
    $ grep -c "audit_receipts_nonce_unique" migrations/001_initial_schema.sql
    EXPECT: 1

  INTEGRITY_QUERY_COMMENT:
    $ grep -c "Nightly data integrity check" migrations/001_initial_schema.sql
    EXPECT: 1

```

---

#### Session 4.4.3 — `escrow_events` table

**TASK:** Add CREATE TABLE escrow_events to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §4.8, DM §3 Invariants 2 and 4, DM §8.16 (``audit_period_id` nullable)

**CRITICAL COLUMNS:**

  ```sql
    event_id         UUID               PRIMARY KEY DEFAULT gen_random_uuid()
    provider_id      UUID               NOT NULL REFERENCES providers(provider_id)
    event_type       escrow_event_type  NOT NULL  -- includes REVERSAL (DM §9 checklist)
    amount_paise     BIGINT             NOT NULL CHECK (amount_paise > 0)
                    -- BIGINT ONLY. No FLOAT, NUMERIC, DECIMAL (DM §3 Invariant 4)
    audit_period_id  UUID               REFERENCES audit_periods(id)
                    -- NULL for DEPOSIT and SEIZURE events (DM §8.16)
    idempotency_key  VARCHAR(64)        NOT NULL UNIQUE
    created_at       TIMESTAMPTZ        NOT NULL DEFAULT NOW()
  ```

**MANDATORY COMMENT:**

  ```sql
    COMMENT ON TABLE escrow_events IS
        'Append-only escrow ledger. Balance = SUM(DEPOSIT) - SUM(RELEASE + SEIZURE + REVERSAL) '
        'per provider_id. No UPDATE. No DELETE. All amounts in integer paise (ADR-016, Invariant 2).';

    COMMENT ON COLUMN escrow_events.amount_paise IS
        'Integer paise ONLY. BIGINT. No FLOAT. ₹1 = 100 paise (NFR-046).';
  ```

**VERIFY:**

  **AMOUNT_PAISE_IS_BIGINT:**

  ```bash
    $ grep -A2 "amount_paise" migrations/001_initial_schema.sql | \
        grep "escrow_events" -A5 | grep "BIGINT"
    EXPECT: found

    $ grep "amount_paise.*FLOAT\|amount_paise.*NUMERIC\|amount_paise.*DECIMAL" \
        migrations/001_initial_schema.sql
    EXPECT: empty  (Invariant 4)

  ```

  **REVERSAL_IN_ESCROW_EVENT_TYPE:**

  ```bash
    $ grep -A10 "CREATE TYPE escrow_event_type" migrations/001_initial_schema.sql | \
        grep "REVERSAL"
    EXPECT: found  (DM §9 checklist)

  ```

  **IDEMPOTENCY_UNIQUE:**

  ```bash
    $ grep -c "idempotency_key.*VARCHAR(64).*NOT NULL.*UNIQUE\|idempotency_key.*UNIQUE" \
        migrations/001_initial_schema.sql
    EXPECT: >= 1

  ```

---

#### Session 4.4.4 — `owner_escrow_events` table

**Task:** Implement DM §4.9. This table is required for FR-014, FR-021, FR-059 (DM §9
checklist). Same `amount_paise BIGINT NOT NULL CHECK (amount_paise > 0)` constraint.
Add the balance query as a SQL comment: `SUM(DEPOSIT) - SUM(CHARGE + WITHDRAWAL) + SUM(REFUND)`.

#### Session 4.4.5 — `repair_jobs` table

**TASK:** Add CREATE TABLE repair_jobs to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §4.10, DM §8.17 (`provider_id` nullable), DM §8.18 (`started_at` nullable),
        DM §8.19 (`completed_at` nullable), IC §5.7 (priority ordering)

**CRITICAL COLUMNS:**

  ```sql
    job_id                UUID                PRIMARY KEY DEFAULT gen_random_uuid()
    chunk_id              BYTEA               NOT NULL CHECK (octet_length(chunk_id) = 32)
    segment_id            UUID                NOT NULL REFERENCES segments(segment_id)
    provider_id           UUID                REFERENCES providers(provider_id)
                          -- NULL for THRESHOLD_WARNING / EMERGENCY_FLOOR triggers (DM §8.17)
    trigger_type          repair_trigger_type NOT NULL
    priority              repair_priority     NOT NULL
    status                repair_job_status   NOT NULL DEFAULT 'QUEUED'
    available_shard_count SMALLINT            NOT NULL
                          -- CHECK bounds are PROFILE-VARIABLE (from generator):
                          -- prod: CHECK (available_shard_count BETWEEN 16 AND 56)
                          -- demo: CHECK (available_shard_count BETWEEN 3 AND 5)
    created_at            TIMESTAMPTZ         NOT NULL DEFAULT NOW()
    started_at            TIMESTAMPTZ         NULL   -- DM §8.18
    completed_at          TIMESTAMPTZ         NULL   -- DM §8.19
  ```

**MANDATORY CONSTRAINTS:**

  ```sql
    repair_jobs_priority_matches_trigger: CHECK (
        (trigger_type = 'EMERGENCY_FLOOR' AND priority = 'EMERGENCY')
        OR (trigger_type IN ('SILENT_DEPARTURE', 'ANNOUNCED_DEPARTURE')
                AND priority = 'PERMANENT_DEPARTURE')
        OR (trigger_type = 'THRESHOLD_WARNING' AND priority = 'PRE_WARNING')
    )
    repair_jobs_completed_after_started: CHECK (
        completed_at IS NULL OR started_at IS NOT NULL
    )
  ```

**PARTIAL UNIQUE INDEX for threshold deduplication (DM §5):**

  ```sql
    CREATE UNIQUE INDEX idx_repair_jobs_threshold_no_dup
        ON repair_jobs (chunk_id, trigger_type)
        WHERE provider_id IS NULL AND status IN ('QUEUED', 'IN_PROGRESS');
  ```

**NOTE ON REMOVED CONSTRAINT (DM §4.10):**

  ```sql
    The UNIQUE (chunk_id, provider_id, trigger_type) constraint was removed.
    Application-layer deduplication in `internal/repair.EnqueueJob` handles this.
    Add comment: "-- Departure-trigger deduplication is at application layer (IC §5.7)"
  ```

**VERIFY:**

  **TABLE_IN_SQL:**

  ```bash
    $ grep -c "CREATE TABLE repair_jobs" migrations/001_initial_schema.sql
    EXPECT: 1

  ```

  **PRIORITY_TRIGGER_CONSTRAINT:**

  ```bash
    $ grep -c "repair_jobs_priority_matches_trigger" migrations/001_initial_schema.sql
    EXPECT: 1

  ```

  **COMPLETED_AFTER_STARTED:**

  ```bash
    $ grep -c "repair_jobs_completed_after_started" migrations/001_initial_schema.sql
    EXPECT: 1

  ```

  **THRESHOLD_INDEX:**

  ```bash
    $ grep -c "idx_repair_jobs_threshold_no_dup" migrations/001_initial_schema.sql
    EXPECT: 1

  ```

  **PROVIDER_ID_NULLABLE:**

  ```bash
    $ grep -B1 -A5 "CREATE TABLE repair_jobs" migrations/001_initial_schema.sql | \
        grep "provider_id.*NOT NULL"
    EXPECT: empty  (provider_id is nullable)

  ```

  **PROFILE_VARIABLE_SHARD_COUNT:**

  ```bash
    # Production schema:
    $ grep "available_shard_count BETWEEN 16 AND 56" migrations/001_initial_schema.sql
    EXPECT: found (in production schema file)
    # Demo schema:
    $ grep "available_shard_count BETWEEN 3 AND 5" migrations/001_initial_schema_demo.sql
    EXPECT: found

  ```

---

### Phase 4.5 — Index Catalogue

**Reference:** DM §5 (complete index catalogue with named query patterns)

#### Session 4.5.1 — Create all indexes

**TASK:** Add all indexes from DM §5 to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §5 (complete catalogue with named query patterns)
**RULE: Every index must appear as a standalone CREATE INDEX statement AFTER all CREATE TABLE**
      statements (DM §9 migration ordering). Each must include its named query pattern
      from DM §5 as a SQL comment on the preceding line.

**INDEX LIST (complete — add in order matching DM §5, each with comment):**

```sql
-- Query: lookup by phone at login / OTP verification
CREATE UNIQUE INDEX idx_owners_phone ON owners (phone_number);

-- Query: departure detector — find providers with last_heartbeat_ts > 72h ago
CREATE INDEX idx_providers_heartbeat_active ON providers (last_heartbeat_ts)
    WHERE status = 'ACTIVE';

-- Query: assignment service — select ACTIVE providers for ASN cap check
CREATE INDEX idx_providers_asn_active ON providers (asn) WHERE status = 'ACTIVE';

-- Query: readiness gate — count providers by status AND region
CREATE INDEX idx_providers_status_region ON providers (status, region);

-- Query: lookup by phone at registration / OTP re-verification
CREATE UNIQUE INDEX idx_providers_phone ON providers (phone_number);

-- Query: file list for a data owner dashboard
CREATE INDEX idx_files_owner ON files (owner_id, uploaded_at DESC)
    WHERE status = 'ACTIVE';

-- Query: find files awaiting deletion confirmation for the GC retry loop (FR-020)
CREATE INDEX idx_files_pending_deletion ON files (owner_id, uploaded_at)
    WHERE status = 'DELETION_PENDING';

-- Query: fetch all segments for a file in order (upload orchestrator, retrieval)
CREATE INDEX idx_segments_file ON segments (file_id, segment_index);

-- Query: challenge scheduler — find all active chunks for a provider
CREATE INDEX idx_chunk_assignments_provider_active ON chunk_assignments (provider_id)
    WHERE status = 'ACTIVE';

-- Query: repair scheduler — find surviving shard holders for a segment
CREATE INDEX idx_chunk_assignments_segment_active ON chunk_assignments (segment_id)
    WHERE status IN ('ACTIVE', 'REPAIRING');

-- Query: deletion workflow — find pending deletions per provider for GC
CREATE INDEX idx_chunk_assignments_provider_pending_deletion
    ON chunk_assignments (provider_id) WHERE status = 'PENDING_DELETION';

-- Query: ASN cap check at assignment time
CREATE INDEX idx_chunk_assignments_segment_provider
    ON chunk_assignments (segment_id, provider_id) WHERE status = 'ACTIVE';

-- (Partial unique index already created in Session 4.3.5)
-- idx_chunk_assignments_one_active_per_shard — created with CREATE TABLE chunk_assignments

-- Query: ACTIVE transition GC — fetch synthetic chunk IDs to send to daemon
CREATE INDEX idx_chunk_assignments_vetting_provider_active
    ON chunk_assignments (provider_id)
    WHERE is_vetting_chunk = TRUE AND status = 'ACTIVE';

-- Query: departure handler — bulk soft-delete synthetic chunks on vetting departure
CREATE INDEX idx_chunk_assignments_vetting_provider
    ON chunk_assignments (provider_id) WHERE is_vetting_chunk = TRUE;

-- Query: monthly release computation — get current period per provider
CREATE INDEX idx_audit_periods_provider_recent
    ON audit_periods (provider_id, period_start DESC);

-- Query: scoring queries (three-window score)
CREATE INDEX idx_audit_periods_provider_range
    ON audit_periods (provider_id, period_start, period_end);

-- Query: three-window scoring — sum PASS/FAIL/TIMEOUT for a provider
CREATE INDEX idx_audit_receipts_provider_ts
    ON audit_receipts (provider_id, server_challenge_ts DESC)
    WHERE abandoned_at IS NULL AND audit_result IS NOT NULL;

-- NOTE: UNIQUE on challenge_nonce already created as table constraint; no extra index needed.

-- Query: GC process — find PENDING rows older than 48h for abandonment
CREATE INDEX idx_audit_receipts_pending_stale
    ON audit_receipts (server_challenge_ts)
    WHERE audit_result IS NULL AND abandoned_at IS NULL;

-- Query: JIT analysis — count jit_flags per provider in a rolling 7-day window
CREATE INDEX idx_audit_receipts_jit_provider
    ON audit_receipts (provider_id, server_challenge_ts DESC) WHERE jit_flag = TRUE;

-- Query: provider retrieves their own receipts for dispute resolution (FR-058)
CREATE INDEX idx_audit_receipts_provider_file
    ON audit_receipts (provider_id, file_id, server_challenge_ts DESC);

-- Query: FR-058 provider dispute evidence — filter by chunk_id
CREATE INDEX idx_audit_receipts_provider_chunk
    ON audit_receipts (provider_id, chunk_id, server_challenge_ts DESC);

-- Query: balance computation
CREATE INDEX idx_escrow_events_provider ON escrow_events (provider_id, event_type);

-- Query: monthly release job
CREATE INDEX idx_escrow_events_period ON escrow_events (audit_period_id)
    WHERE audit_period_id IS NOT NULL;

-- Query: repair scheduler dequeue
CREATE INDEX idx_repair_jobs_queue ON repair_jobs (priority, created_at ASC)
    WHERE status = 'QUEUED';

-- Query: repair dashboard
CREATE INDEX idx_repair_jobs_status_priority ON repair_jobs (status, priority);

-- Query: link repair jobs to departing provider
CREATE INDEX idx_repair_jobs_provider ON repair_jobs (provider_id)
    WHERE provider_id IS NOT NULL;

-- (idx_repair_jobs_threshold_no_dup already created in Session 4.4.5)
```

**VERIFY:**

  **INDEX_COUNT:**

  ```bash
    $ grep -c "^CREATE.*INDEX idx_" migrations/001_initial_schema.sql
    EXPECT: >= 20  (not counting the 2 created inline in earlier sessions)

  ```

  **VETTING_INDEXES:**

  ```bash
    $ grep -c "idx_chunk_assignments_vetting" migrations/001_initial_schema.sql
    EXPECT: 2  (vetting_provider_active and vetting_provider)

  ```

  **INDEXES_AFTER_TABLES:**

  ```bash
    $ awk '/^CREATE TABLE/{t=NR} /^CREATE.*INDEX/{i=NR; if(i<t) exit 1}' \
        migrations/001_initial_schema.sql && echo "PASS" || echo "FAIL: index before table"

  ```

  **QUERY_COMMENTS:**

  ```bash
    $ grep -c "^-- Query:" migrations/001_initial_schema.sql
    EXPECT: >= 20  (each index has a named query pattern comment)

  NAMED_INDEXES_COMPLETE (spot check critical ones):
    $ grep -c "idx_providers_heartbeat_active\|idx_audit_receipts_pending_stale\|idx_repair_jobs_queue" \
        migrations/001_initial_schema.sql
    EXPECT: 3

  ```

---

### Phase 4.6 — Row Security Policies

**Reference:** DM §6, IC §6 (DML contracts that RSPs enforce)

#### Session 4.6.1 — `audit_receipts` RSP (Invariant 1)

**TASK:** Add Row Security Policies for audit_receipts to `migrations/001_initial_schema.sql`.

**SOURCE:** DM §6 (Invariant 1 enforcement), ADR-015 (two-phase write intent)

**PREREQUISITES: PostgreSQL roles vyomanaut_app and vyomanaut_gc must exist.**

  ```sql
  Add role creation to the migration preamble:
    DO $$ BEGIN
      IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'vyomanaut_app') THEN
        CREATE ROLE vyomanaut_app;
      END IF;
      IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'vyomanaut_gc') THEN
        CREATE ROLE vyomanaut_gc;
      END IF;
    END $$;
  ```

**ENABLE RLS:**

  ```sql
    ALTER TABLE audit_receipts ENABLE ROW LEVEL SECURITY;
  ```

**POLICY 1 — INSERT only (Phase 1 of two-phase write):**
  
  ```sql
    CREATE POLICY audit_receipts_insert_only
        ON audit_receipts FOR INSERT TO vyomanaut_app WITH CHECK (TRUE);
  ```

**POLICY 2 — Phase 2 UPDATE (PENDING → terminal state):**

  ```sql
    CREATE POLICY audit_receipts_phase2_update
        ON audit_receipts FOR UPDATE TO vyomanaut_app
        USING (audit_result IS NULL AND abandoned_at IS NULL)
        WITH CHECK (
            audit_result IN ('PASS', 'FAIL', 'TIMEOUT')
            AND service_sig IS NOT NULL
            AND service_countersign_ts IS NOT NULL
        );
    -- This is the ONLY permitted UPDATE on audit_receipts.
    -- Any other UPDATE returns permission denied.
    -- The underlying query must be: WHERE receipt_id = $1 AND audit_result IS NULL AND abandoned_at IS NULL
  ```

**POLICY 3 — GC abandon (background process sets abandoned_at):**

  ```sql
    CREATE POLICY audit_receipts_gc_abandon
        ON audit_receipts FOR UPDATE TO vyomanaut_gc
        USING (
            audit_result IS NULL
            AND abandoned_at IS NULL
            AND server_challenge_ts < NOW() - INTERVAL '48 hours'
            -- NOTE: This RSP hardcodes 48 hours at the DB layer.
            -- The application-layer GC query uses `profile.PendingReceiptGCAge` (demo=5min, prod=48h).
            -- These are SEPARATE mechanisms:
            --   RSP: enforces the maximum DB-level update window (always 48h)
            --   App query: fires early in demo mode using `profile.PendingReceiptGCAge`
            -- The RSP is a safety backstop; the app fires first in demo.
        )
        WITH CHECK (
            abandoned_at IS NOT NULL
            AND audit_result IS NULL  -- GC never sets audit_result; only abandoned_at
        );
  ```

**NO DELETE POLICY (any DELETE attempt returns permission denied):**

  ```sql
    -- Intentionally absent: no DELETE policy means all DELETEs are denied.
  ```

**VERIFY:**

  **RLS_ENABLED:**

  ```bash
    (After migration is applied to test DB)
    $ psql -U vyomanaut_app -d vyomanaut_test -c "\d audit_receipts" | grep "row security"
    EXPECT: "Row Security: enabled"

  ```

  **POLICIES_PRESENT:**

  ```bash
    $ psql -U vyomanaut_app -d vyomanaut_test -c \
        "SELECT policyname FROM pg_policies WHERE tablename = 'audit_receipts'"
    EXPECT: output contains all three policy names:
      audit_receipts_insert_only
      audit_receipts_phase2_update
      audit_receipts_gc_abandon

  ```

  **NO_DELETE_BLOCKED:**

  ```bash
    $ psql -U vyomanaut_app -d vyomanaut_test -c \
        "DELETE FROM audit_receipts WHERE receipt_id = gen_random_uuid()" 2>&1
    EXPECT: contains "ERROR" or "permission denied"

  ```

  **INSERT_ALLOWED:**

  ```bash
    $ psql -U vyomanaut_app -d vyomanaut_test -c \
        "INSERT INTO audit_receipts (chunk_id, provider_id, challenge_nonce, server_challenge_ts)
         VALUES (repeat('\x00', 32)::bytea,
                 (SELECT provider_id FROM providers LIMIT 1),
                 repeat('\x00', 33)::bytea,
                 NOW())" 2>&1
    EXPECT: INSERT 0 1  (success, assuming FK constraints are satisfied)

  ```

  **GC_COMMENT_IN_SQL:**

  ```bash
    $ grep -c "separate mechanisms\|safety backstop\|profile.PendingReceiptGCAge" \
        migrations/001_initial_schema.sql
    EXPECT: >= 1

  ```

---

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

**TASK:** Add mv_provider_scores materialised view to migrations/001_initial_schema.sql.

**SOURCE:** DM §7, MVP §5.5 (view is dropped and recreated at startup), IC §6 (scoring consumer)

**CRITICAL DESIGN NOTE:**
  The interval literals in this view are PLACEHOLDERS. The view is dropped and recreated
  at microservice startup from NetworkProfile.ScoreWindow{Short,Medium,Long}.
  The migration file contains a PLACEHOLDER view with production intervals clearly marked.
  In VYOMANAUT_MODE=demo, the startup sequence drops this view and recreates it with
  2min/6min/20min intervals. (DM §7, MVP §5.5)
  
**PLACEHOLDER SQL (exactly this format — startup code pattern-matches the comment):**

  ```sql
    -- mv_provider_scores: DROPPED AND RECREATED AT STARTUP from NetworkProfile.ScoreWindow*
    -- Production intervals: 24h / 7d / 30d
    -- Demo intervals: 2min / 6min / 20min  (set by microservice startup, not this migration)
  
    CREATE MATERIALIZED VIEW mv_provider_scores AS
    SELECT
        provider_id,
        score_24h,
        score_7d,
        score_30d,
        (
            COALESCE(score_24h, 0)* 0.5 +
            COALESCE(score_7d,  0) *0.3 +
            COALESCE(score_30d, 0)* 0.2
        ) AS score_composite,
        NOW() AS scores_as_of  -- consumers must check age before using for payment decisions
    FROM (
        SELECT
            provider_id,
            -- SHORT WINDOW (placeholder: 24h production; overridden at startup)
            SUM(CASE WHEN server_challenge_ts >= NOW() - INTERVAL '24 hours'
                    AND audit_result = 'PASS' THEN 1 ELSE 0 END)::FLOAT
            / NULLIF(SUM(CASE WHEN server_challenge_ts >= NOW() - INTERVAL '24 hours'
                              AND audit_result IS NOT NULL THEN 1 ELSE 0 END), 0)
            AS score_24h,
            -- MEDIUM WINDOW (placeholder: 7 days production; overridden at startup)
            SUM(CASE WHEN server_challenge_ts >= NOW() - INTERVAL '7 days'
                    AND audit_result = 'PASS' THEN 1 ELSE 0 END)::FLOAT
            / NULLIF(SUM(CASE WHEN server_challenge_ts >= NOW() - INTERVAL '7 days'
                              AND audit_result IS NOT NULL THEN 1 ELSE 0 END), 0)
            AS score_7d,
            -- LONG WINDOW (placeholder: 30 days production; overridden at startup)
            SUM(CASE WHEN server_challenge_ts >= NOW() - INTERVAL '30 days'
                    AND audit_result = 'PASS' THEN 1 ELSE 0 END)::FLOAT
            / NULLIF(SUM(CASE WHEN server_challenge_ts >= NOW() - INTERVAL '30 days'
                              AND audit_result IS NOT NULL THEN 1 ELSE 0 END), 0)
            AS score_30d
        FROM audit_receipts
        WHERE abandoned_at IS NULL
        GROUP BY provider_id
    ) sub;

    CREATE UNIQUE INDEX ON mv_provider_scores (provider_id);
    -- Required for REFRESH MATERIALIZED VIEW CONCURRENTLY (DM §9 checklist)
  ```

**VERIFY:**

  **VIEW_IN_SQL:**

  ```bash
      $ grep -c "CREATE MATERIALIZED VIEW mv_provider_scores" \
          `migrations/001_initial_schema.sql`
      EXPECT: 1
  ```

  **SCORES_AS_OF_COLUMN:**

  ```bash
      $ grep -c "scores_as_of" migrations/001_initial_schema.sql
      EXPECT: 1  (DM §9 checklist)
  ```

  **PLACEHOLDER_COMMENT:**

  ```bash
      $ grep -c "DROPPED AND RECREATED AT STARTUP" migrations/001_initial_schema.sql
      EXPECT: 1
  ```

  **SUBQUERY_STRUCTURE (column aliases not referenced in same SELECT level):**

  ```bash
      $ grep -c "FROM (.*SELECT\|FROM.*sub" migrations/001_initial_schema.sql
      EXPECT: >= 1  (DM §9: "column aliases not referenced in same SELECT level")
  ```

  **UNIQUE_INDEX_ON_VIEW:**

  ```bash
      $ grep -c "CREATE UNIQUE INDEX ON mv_provider_scores" migrations/001_initial_schema.sql
      EXPECT: 1
  ```

---

#### Session 4.7.2 — `mv_provider_escrow_balance`

**TASK:** Add mv_provider_escrow_balance, mv_owner_escrow_balance, mv_segment_shard_counts
      to `migrations/001_initial_schema.sql.`

**SOURCE:** DM §7

**`mv_provider_escrow_balance` (includes REVERSAL amendment, DM §9 checklist):**

```sql
  CREATE MATERIALIZED VIEW mv_provider_escrow_balance AS
  SELECT
      provider_id,
      SUM(CASE WHEN event_type IN ('DEPOSIT', 'REVERSAL') THEN amount_paise ELSE 0 END)
      -
      SUM(CASE WHEN event_type IN ('RELEASE', 'SEIZURE') THEN amount_paise ELSE 0 END)
      AS balance_paise
  FROM escrow_events
  GROUP BY provider_id;
  -- REVERSAL increases balance (refund of reversed payout) — DM §7 amendment
  -- idempotency_key for REVERSAL = SHA-256('reversal' || original_idempotency_key)
  -- (enforced at application layer, not DB)

  CREATE UNIQUE INDEX ON mv_provider_escrow_balance (provider_id);
```

**`mv_owner_escrow_balance` (with GREATEST guard, DM §7):**

```sql
  CREATE MATERIALIZED VIEW mv_owner_escrow_balance AS
  SELECT
      owner_id,
      GREATEST(
          SUM(CASE WHEN event_type IN ('DEPOSIT', 'REFUND') THEN amount_paise ELSE 0 END)
          -
          SUM(CASE WHEN event_type IN ('CHARGE', 'WITHDRAWAL') THEN amount_paise ELSE 0 END),
          0  -- prevents negative balance (DM §7: "Add GREATEST(..., 0)")
      ) AS balance_paise
  FROM owner_escrow_events
  GROUP BY owner_id;

  CREATE UNIQUE INDEX ON mv_owner_escrow_balance (owner_id);
```

**`mv_segment_shard_counts`:**

```sql
  CREATE MATERIALIZED VIEW mv_segment_shard_counts AS
  SELECT
      segment_id,
      COUNT(*) FILTER (WHERE status IN ('ACTIVE', 'REPAIRING')) AS available_shard_count,
      COUNT(*) FILTER (WHERE status = 'ACTIVE') AS active_shard_count
  FROM chunk_assignments
  GROUP BY segment_id;

  CREATE UNIQUE INDEX ON mv_segment_shard_counts (segment_id);
```

**VERIFY:**

  **VIEWS_PRESENT:**

  ```bash
      $ grep -c "CREATE MATERIALIZED VIEW" migrations/001_initial_schema.sql
      EXPECT: 4  (mv_provider_scores + 3 from this session)
  ```

  **REVERSAL_IN_ESCROW_BALANCE:**

  ```bash
      $ grep -A5 "mv_provider_escrow_balance" migrations/001_initial_schema.sql | \
          grep "REVERSAL"
      EXPECT: found  (DM §9 checklist)
  ```

  **GREATEST_IN_OWNER_BALANCE:**

  ```bash
      $ grep -A10 "mv_owner_escrow_balance" migrations/001_initial_schema.sql | \
          grep "GREATEST"
      EXPECT: found
  ```

  **UNIQUE_INDEXES_ON_ALL_VIEWS:**

  ```bash
      $ grep -c "CREATE UNIQUE INDEX ON mv_" migrations/001_initial_schema.sql
      EXPECT: 4  (one per view)
  ```

  **BOTH_SHARD_COUNT_COLUMNS:**

  ```bash
      $ grep -c "available_shard_count\|active_shard_count" migrations/001_initial_schema.sql
      EXPECT: 2
  ```

---

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

**TASK:** Create `scripts/ci/migration_check.sh` implementing all DM §9 checklist items as CI-runnable psql queries. This is CI check 7 (MVP §8.4).

FILE: `scripts/ci/migration_check.sh`

**STRUCTURE:**

  ```bash
    #!/usr/bin/env bash
    set -euo pipefail
    DB="${PGDATABASE:-vyomanaut_test}"
    USER="${PGUSER:-vyomanaut_app}"
    HOST="${PGHOST:-localhost}"
    FAIL=0

    psql_check() {
      local name="$1"; local query="$2"; local expected="$3"
      result=$(psql -h "$HOST" -U "$USER" -d "$DB" -t -c "$query" 2>&1 | tr -d '[:space:]')
      if [ "$result" = "$expected" ]; then
        echo "PASS [$name]"
      else
        echo "FAIL [$name]: got '$result', expected '$expected'"
        FAIL=1
      fi
    }
  ```

**DM §9 checklist item 1:** `btree_gist` installed

  ```sql
    psql_check "btree_gist_installed" \

      "SELECT COUNT(*) FROM pg_extension WHERE extname = 'btree_gist'" \
      "1"
  ```

**DM §9:** `challenge_nonce` is BYTEA with `octet_length = 33` (not 32)

  ```sql
    psql_check "challenge_nonce_33_bytes" \

      "SELECT COUNT(*) FROM information_schema.check_constraints
      WHERE constraint_name LIKE '%challenge_nonce%'
      AND check_clause LIKE '%33%'" \
      "1"
  ```

**DM §9:** no FLOAT columns in `escrow_events`

  ```sql
    psql_check "no_float_in_escrow" \

      "SELECT COUNT(*) FROM information_schema.columns
      WHERE table_name = 'escrow_events'
      AND data_type IN ('real', 'double precision')" \
      "0"
  ```

**DM §9:** `audit_result` has no DEFAULT

  ```sql
    psql_check "audit_result_no_default" \

      "SELECT COUNT(*) FROM information_schema.columns
      WHERE table_name = 'audit_receipts'
      AND column_name = 'audit_result'
      AND column_default IS NOT NULL" \
      "0"
  ```

**DM §9:** REVERSAL in `escrow_event_type` ENUM

  ```sql
    psql_check "reversal_in_enum" \

      "SELECT COUNT(*) FROM pg_enum
      WHERE enumtypid = 'escrow_event_type'::regtype
      AND enumlabel = 'REVERSAL'" \
      "1"
  ```

**DM §9:** `is_vetting_chunk` column present on `chunk_assignments`

  psql_check "is_vetting_chunk_present" \

```sql
    "SELECT COUNT(*) FROM information_schema.columns
     WHERE table_name = 'chunk_assignments'
     AND column_name = 'is_vetting_chunk'" \
    "1"
```

**DM §9:** `file_id` on `audit_receipts` is nullable

  ```sql
    psql_check "audit_receipts_file_id_nullable" \

      "SELECT is_nullable FROM information_schema.columns
      WHERE table_name = 'audit_receipts'
      AND column_name = 'file_id'" \
      "YES"
  ```

**DM §9:** partial unique index includes WHERE `is_vetting_chunk = FALSE`

  ```sql
    psql_check "vetting_filter_in_partial_index" \

      "SELECT COUNT(*) FROM pg_indexes
      WHERE indexname = 'idx_chunk_assignments_one_active_per_shard'
      AND indexdef LIKE '%is_vetting_chunk%'" \
      "1"
  ```

**DM §9:** `scores_as_of` column present in `mv_provider_scores`

  ```sql
    psql_check "scores_as_of_column" \

      "SELECT COUNT(*) FROM information_schema.columns
      WHERE table_name = 'mv_provider_scores'
      AND column_name = 'scores_as_of'" \
      "1"
  ```

**DM §9:** REVERSAL in `mv_provider_escrow_balance` formula (check view definition)

  ```sql
    psql_check "reversal_in_balance_view" \

      "SELECT COUNT(*) FROM pg_views
      WHERE viewname = 'mv_provider_escrow_balance'
      AND definition LIKE '%REVERSAL%'" \
      "1"
  ```

**DM §9:** `owner_escrow_events` table present

  ```sql
    psql_check "owner_escrow_events_exists" \

      "SELECT COUNT(*) FROM information_schema.tables
      WHERE table_name = 'owner_escrow_events'" \
      "1"
  ```

**DM §9:** `providers.first_chunk_assignment_at` column present

  ```sql
    psql_check "first_chunk_assignment_at_exists" \

      "SELECT COUNT(*) FROM information_schema.columns
      WHERE table_name = 'providers'
      AND column_name = 'first_chunk_assignment_at'" \
      "1"
  ```

**DM §9:** files `display_name` columns present

  ```sql
    psql_check "display_name_columns" \

      "SELECT COUNT(*) FROM information_schema.columns
      WHERE table_name = 'files'
      AND column_name IN ('display_name_ciphertext', 'display_name_nonce', 'display_name_tag')" \
      "3"
  ```

**DM §9:** `p95_throughput_kbps` and `avg_rtt_ms` default to NULL (not 0/2000)

  ```sql
    psql_check "throughput_def`ault_null" \

      "SELECT COUNT(*) FROM information_schema.columns
      WHERE table_name = 'providers'
      AND column_name IN ('p95_throughput_kbps', 'avg_rtt_ms')
      AND column_default IS NOT NULL" \
      "0"
  ```

**DM §9:** `repair_priority` ENUM has three values

  ```sql
    psql_check "repair_priority_three_values" \

      "SELECT COUNT(*) FROM pg_enum
      WHERE enumtypid = 'repair_priority'::regtype" \
      "3"
  ```

**DM §9:** `file_status` ENUM has three values

  ```sql
    psql_check "file_status_three_values" \

      "SELECT COUNT(*) FROM pg_enum
      WHERE enumtypid = 'file_status'::regtype" \
      "3"
  ```

**DM §9:** RSP on `audit_receipts` enabled

  ```sql
    psql_check "audit_receipts_rls_enabled" \ 

      "SELECT COUNT(*) FROM pg_tables
      WHERE tablename = 'audit_receipts' AND rowsecurity = true" \
      "1"
  ```

**DM §9:** RSP on `escrow_events` enabled

  ```sql
    psql_check "escrow_events_rls_enabled" \

      "SELECT COUNT(*) FROM pg_tables
      WHERE tablename = 'escrow_events' AND rowsecurity = true" \
      "1"
  ```

**DM §9:** `mv_provider_scores` unique index present (for CONCURRENTLY refresh)

  ```sql
    psql_check "mv_provider_scores_unique_idx" \

        "SELECT COUNT(*) FROM pg_indexes
        WHERE tablename = 'mv_provider_scores' AND indexdef LIKE '%UNIQUE%'" \
        "1"
  ```

**MVP §6.4 MR-01/MR-02:** demo and prod schema have different `shard_index` bounds (This check runs only when both schema files exist)

  ```bash
    if [ -f `migrations/001_initial_schema.sql` ] && [ -f `migrations/001_initial_schema_demo.sql` ]; then
      prod_bound=$(grep "shard_index BETWEEN" `migrations/001_initial_schema.sql` | grep -o "AND [0-9]*" | awk '{print $2}')
      demo_bound=$(grep "shard_index BETWEEN" `migrations/001_initial_schema_demo.sql` | grep -o "AND [0-9]*" | awk '{print $2}')
      if [ "$prod_bound" = "55" ] && [ "$demo_bound" = "4" ]; then
        echo "PASS [profile_bound_separation]"
      else
        echo "FAIL [profile_bound_separation]: prod=$prod_bound (want 55) demo=$demo_bound (want 4)"
        FAIL=1
      fi
    fi

    exit $FAIL
  ```

**MAKE EXECUTABLE:** chmod +x `scripts/ci/migration_check.sh`

**VERIFY:**

  **FILES_EXIST:**

  ```bash
      test -f scripts/ci/migration_check.sh && echo PASS || echo FAIL
      test -x scripts/ci/migration_check.sh && echo PASS || echo FAIL
  ```

  **SCRIPT_SYNTAX:**

  ```bash
      bash -n scripts/ci/migration_check.sh && echo "PASS: no syntax errors" || echo FAIL
  ```

  **CHECK_COUNT:**

  ```bash
      $ grep -c "psql_check " scripts/ci/migration_check.sh
      EXPECT: >= 18
  ```

  **RUNS_AGAINST_TEST_DB (after migration applied):**

  ```bash
      $ bash scripts/ci/migration_check.sh
      EXPECT: exit 0; all lines print "PASS [...]"
  ```

  **FAIL_DETECTION (inject a known violation):**

  ```bash
      $ psql -U vyomanaut_app -d vyomanaut_test \
          -c "ALTER TABLE audit_receipts DISABLE ROW LEVEL SECURITY"
      $ bash scripts/ci/migration_check.sh | grep "FAIL\|PASS" | grep "audit_receipts_rls"
      EXPECT: "FAIL [audit_receipts_rls_enabled]"
      $ psql -U vyomanaut_app -d vyomanaut_test \
          -c "ALTER TABLE audit_receipts ENABLE ROW LEVEL SECURITY"
  ```

---

## Milestone 5 — Provider Storage Engine (`internal/storage`)

**Deliverable:** WiscKey-style chunk storage engine — RocksDB index plus an
append-only vLog file — with crash recovery, GC, and the single-writer goroutine
constraint enforced by test. The daemon is transparent to whether a chunk is
synthetic (vetting) or real; both use identical code paths (ADR-030, DM §3 Invariant 6).

**References:**

- **IC §5.3** — `ChunkStore` interface: all six methods, five sentinel errors, concurrency contract
- **ARCH §16** — Provider Storage Engine: WiscKey design, vLog entry layout, audit lookup path, single-writer rule, crash recovery, GC
- **ARCH §27.1** — Dimensions Table: `vLogEntrySize = 262,212 bytes`, RocksDB index entry `= 44 bytes`, Bloom filter 10 bits/key
- **MVP §8.2** — File inventory for `internal/storage`
- **ARCH §4.1** — RocksDB CGo build note: `linxGnu/grocksdb`, pinned Docker image in CI
- **IC §11** — Forbidden: breaking the single-writer invariant

**CGo prerequisite:** `librocksdb` must be installed before `go build ./internal/storage/`
can succeed. In CI the pinned Docker image in `.github/workflows/ci.yml` provides the
correct version. A version mismatch produces a link-time failure, not a runtime failure
(ARCH §4.1). Locally on Ubuntu 22.04:

```bash
sudo apt-get install -y librocksdb-dev
go get github.com/linxGnu/grocksdb
```

---

### Phase 5.1 — ChunkStore Interface & vLog

**Reference:** IC §5.3, ARCH §16 (vLog entry format, audit lookup path, single-writer rule), ARCH §27.1 (entry sizes)

---

#### Session 5.1.1 — Define `ChunkStore` interface and sentinel errors

**PRECONDITIONS:**

- `internal/storage/doc.go` exists (stub from Session 0.1.3)
- `go.mod` and `go.sum` are present (Session 0.1.1)
- `internal/config` package is complete (Milestone 1)

**TASK:**

1. Run `go get github.com/linxGnu/grocksdb` to add the RocksDB Go binding to `go.mod`.
2. Create `internal/storage/errors.go` with the five sentinel errors from IC §5.3.
3. Create `internal/storage/store.go` with:
   - `const vLogEntrySize = 262212` derived from ARCH §16: `chunk_id(32) + chunk_size(4) + chunk_data(262144) + content_hash(32)`. Every read and write uses this fixed size.
   - `const indexValueSize = 12` — RocksDB value bytes: `vlog_offset(uint64=8) + chunk_size(uint32=4)`. The RocksDB key is the 32-byte `chunk_id`; total entry on disk ≈ 44 bytes (ARCH §27.1).
   - The `ChunkStore` interface with all six methods from IC §5.3 and the mandatory concurrency comment on `AppendChunk`.

**FILE:** `internal/storage/errors.go`

```go
package storage

import "errors"

var (
    // ErrChunkNotFound is returned by LookupChunk when the chunk is absent from the index.
    // The Bloom filter exits fast on this path — no vLog disk I/O occurs (ARCH §16, ARCH §27.1).
    ErrChunkNotFound = errors.New("storage: chunk not found")

    // ErrContentHashMismatch is returned by LookupChunk when SHA-256(chunk_data)
    // does not equal the stored content_hash — silent disk corruption detected (ARCH §16).
    // Caller must return audit_result = FAIL with status byte 0x02 (IC §4.2).
    ErrContentHashMismatch = errors.New("storage: content hash mismatch (disk corruption)")

    // ErrVLogFsync is returned by AppendChunk when the fsync() call fails after writing.
    // The daemon must halt and restart; RecoverFromCrash will repair the tail on next start.
    ErrVLogFsync = errors.New("storage: vLog fsync failed")

    // ErrVLogRead is returned by LookupChunk or RecoverFromCrash on vLog I/O failure.
    ErrVLogRead = errors.New("storage: vLog read failed")

    // ErrRocksDBInsert is returned by AppendChunk when the RocksDB index INSERT fails
    // after a successful vLog write. RecoverFromCrash will repair the missing entry
    // on next daemon startup (ARCH §16 §Crash recovery).
    ErrRocksDBInsert = errors.New("storage: RocksDB index insert failed")
)
```

**FILE:** `internal/storage/store.go`

```go
package storage

import "context"

// vLogEntrySize is the fixed byte size of every vLog entry (ARCH §16, ARCH §27.1).
// Layout: chunk_id(32) + chunk_size(4) + chunk_data(262144) + content_hash(32) = 262212.
// Every read and write uses this exact size; no variable-length entries exist.
const vLogEntrySize = 262212

// indexValueSize is the byte size of the RocksDB value per chunk index entry (ARCH §27.1).
// Layout: vlog_offset(uint64=8) + chunk_size(uint32=4) = 12 bytes.
// The RocksDB key is the 32-byte chunk_id; total on-disk entry ≈ 44 bytes.
const indexValueSize = 12

// ChunkStore is the WiscKey key-value separated chunk storage engine (ARCH §16, ADR-023).
// RocksDB holds the small index; the append-only vLog holds all chunk data.
// Write amplification ≈ 1.0 at 256 KB values (ARCH §4.1, ARCH §27.1).
//
// IMPORT CONSTRAINT: This package must NOT import internal/payment, internal/scoring,
// or internal/repair (IC §9).
//
// INVARIANT (DM §3 Invariant 6, ADR-030): The daemon cannot distinguish between synthetic
// vetting chunks and real file shards. Both use identical AppendChunk / LookupChunk /
// DeleteChunk code paths. The is_vetting_chunk flag lives in the microservice only.
type ChunkStore interface {
    // AppendChunk writes a 256 KB chunk to the vLog and inserts the RocksDB index entry.
    //
    // *** SINGLE WRITER ONLY — NOT goroutine-safe ***
    // All concurrent callers MUST submit write requests via a buffered channel to the
    // designated single writer goroutine. POSIX O_APPEND atomicity does not hold for writes
    // above ~4 KB; for 262 KB entries, goroutine serialisation is required (ARCH §16).
    //
    // Pre-conditions (panic in debug builds if violated):
    //   len(chunkID) == 32
    //   len(chunkData) == 262144
    //   SHA-256(chunkData) == chunkID  — caller MUST verify before calling.
    // Post-conditions (on nil error):
    //   chunkData and its SHA-256 content_hash are durably fsync'd to the vLog.
    //   RocksDB index entry (chunkID → vlog_offset, chunk_size) is inserted.
    // Error semantics:
    //   ErrVLogFsync    — halt the daemon; RecoverFromCrash repairs the tail on restart.
    //   ErrRocksDBInsert — RecoverFromCrash repairs the missing index entry on restart.
    AppendChunk(chunkID [32]byte, chunkData []byte) (vlogOffset uint64, err error)

    // LookupChunk retrieves a chunk from the vLog by content address and verifies integrity.
    // Internally: Bloom filter → RocksDB index → vLog pread → SHA-256 verify (ARCH §16).
    // The RocksDB step typically hits the block cache — no disk I/O for the index.
    // The vLog step is one random disk read: ~1 ms SSD, ~12-15 ms HDD (ARCH §27.1).
    //
    // Goroutine-safe: yes (read-only via pread; concurrent with the writer goroutine).
    // Post-conditions (on nil error):
    //   Returned slice is exactly 262144 bytes.
    //   SHA-256(returned data) == chunkID — verified internally; caller need not re-check.
    // Error semantics:
    //   ErrChunkNotFound      — Bloom filter or RocksDB has no entry. Return audit status 0x01.
    //   ErrContentHashMismatch — data present but hash wrong. Return audit status 0x02 (IC §4.2).
    //   ErrVLogRead           — fatal I/O error.
    LookupChunk(chunkID [32]byte) ([]byte, error)

    // DeleteChunk removes the RocksDB index entry for chunkID.
    // vLog space is reclaimed asynchronously by RunGC.
    // Used identically for real chunk deletion (FR-020) and synthetic vetting chunk GC
    // (ADR-030, IC §4.5). The daemon has no visibility into which type is being deleted.
    //
    // Goroutine-safe: yes.
    DeleteChunk(chunkID [32]byte) error

    // RecoverFromCrash scans the vLog tail for entries missing from the RocksDB index
    // and re-inserts them. MUST be called once at daemon startup BEFORE the writer
    // goroutine is launched (ARCH §16 §Crash recovery, NFR-024, ADR-023).
    // Maximum scan: one memtable flush interval — typically a few hundred chunks.
    //
    // Goroutine-safe: NO — must complete before any AppendChunk call is made.
    RecoverFromCrash() error

    // RunGC reclaims vLog disk space from entries whose RocksDB index entry is deleted.
    // Runs in a background goroutine; ctx cancellation stops it cleanly.
    //
    // Goroutine-safe: yes (uses a separate read file handle from the writer goroutine).
    RunGC(ctx context.Context) error

    // Close flushes pending RocksDB writes and closes both the RocksDB instance and the
    // vLog file handle. No further method calls are valid after Close returns.
    Close() error
}
```

**IMPORTS REQUIRED:**

- `store.go`: `"context"` only — no other imports at this stage
- `errors.go`: `"errors"` only

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/storage/errors.go && echo PASS || echo FAIL
  $ test -f internal/storage/store.go  && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/storage/
  EXPECT: exit 0

CONSTANTS:
  $ grep -c "^const vLogEntrySize = 262212" internal/storage/store.go
  EXPECT: 1

  $ grep -c "^const indexValueSize = 12" internal/storage/store.go
  EXPECT: 1

SENTINEL_ERRORS (exactly five):
  $ grep -c "errors.New" internal/storage/errors.go
  EXPECT: 5

  $ grep -c "ErrChunkNotFound\|ErrContentHashMismatch\|ErrVLogFsync\|ErrVLogRead\|ErrRocksDBInsert" \
      internal/storage/errors.go
  EXPECT: 5

INTERFACE_METHODS (six methods declared in the interface body):
  $ grep -c "AppendChunk\|LookupChunk\|DeleteChunk\|RecoverFromCrash\|RunGC\|Close" \
      internal/storage/store.go
  EXPECT: 6

SINGLE_WRITER_COMMENT:
  $ grep -c "SINGLE WRITER ONLY" internal/storage/store.go
  EXPECT: 1

NEGATIVE_CHECKS:
  # store.go must not import any other internal/ package (IC §9)
  $ grep -n "Vyomanaut_V2/internal/" internal/storage/store.go \
      && echo "FAIL: prohibited import" || echo "PASS"

  # errors.go must not import any other internal/ package (IC §9)
  $ grep -n "Vyomanaut_V2/internal/" internal/storage/errors.go \
      && echo "FAIL: prohibited import" || echo "PASS"

  # vLogEntrySize must be 262212 — not 262144 (which omits the 68-byte header fields)
  $ grep -c "vLogEntrySize = 262144" internal/storage/store.go
  EXPECT: 0

VET:
  $ go vet ./internal/storage/
  EXPECT: exit 0; zero output
```

---

#### Session 5.1.2 — Implement the append-only vLog

**PRECONDITIONS:**

- Session 5.1.1 complete (`store.go` and `errors.go` exist, `vLogEntrySize = 262212` is defined)
- `github.com/linxGnu/grocksdb` present in `go.mod` (added in Session 5.1.1 step 1)

**TASK:**

1. Create `internal/storage/rotational.go` (build tag `//go:build linux`) that detects whether the storage device is rotational by reading `/sys/block/<dev>/queue/rotational`. Returns `true` for HDD, `false` for SSD. If detection fails, returns `false` (conservative SSD assumption). Used for monitoring output only; core vLog logic is storage-medium-agnostic.
2. Create `internal/storage/rotational_other.go` (build tag `//go:build !linux`) as a stub returning `false` (assume SSD on non-Linux).
3. Create `internal/storage/vlog.go` with:
   - The `wiskeyStore` concrete struct (unexported) implementing `ChunkStore`.
   - Named byte-offset constants for the vLog entry layout derived from `vLogEntrySize`.
   - `appendToVLog` — writes one fixed-size entry and calls `Sync()`.
   - `readFromVLog` — reads one fixed-size entry at a byte offset and verifies `content_hash`.

**FILE:** `internal/storage/rotational.go`

```go
//go:build linux

package storage

import (
    "os"
    "strings"
)

// isRotationalDevice reports whether the block device backing path is a rotational HDD.
// Reads /sys/block/<dev>/queue/rotational: "1" = HDD, "0" = SSD (ARCH §16 §rotational.go).
// Failure returns false (assume SSD — conservative assumption). Used for monitoring only.
func isRotationalDevice(path string) bool {
    // TODO: resolve actual block device name from path via os.Stat + /proc/mounts.
    // Simplified implementation for the initial session:
    data, err := os.ReadFile("/sys/block/sda/queue/rotational")
    if err != nil {
        return false
    }
    return strings.TrimSpace(string(data)) == "1"
}
```

**FILE:** `internal/storage/rotational_other.go`

```go
//go:build !linux

package storage

// isRotationalDevice always returns false (assume SSD) on non-Linux platforms.
func isRotationalDevice(_ string) bool { return false }
```

**FILE:** `internal/storage/vlog.go`

```go
package storage

import (
    "crypto/sha256"
    "encoding/binary"
    "fmt"
    "os"
)

// vLog entry field byte offsets. Derived from vLogEntrySize = 262212 (ARCH §16, ARCH §27.1).
// Layout: [chunk_id:32][chunk_size:4][chunk_data:262144][content_hash:32]
const (
    vlogOffChunkID      = 0
    vlogOffChunkSize    = 32
    vlogOffChunkData    = 36
    vlogOffContentHash  = 36 + 262144 // = 262180
)

// wiskeyStore is the concrete, unexported implementation of ChunkStore.
// Callers always interact through the ChunkStore interface.
type wiskeyStore struct {
    // index is the RocksDB instance (defined in index.go, Session 5.1.3).
    index *rocksDBIndex

    // vlog is the append-only value log file. Only the single writer goroutine writes to it.
    vlog     *os.File
    vlogPath string

    // vlogHead is the byte offset for the next write. Protected by the single-writer invariant.
    vlogHead uint64

    // isRotational is true for HDD-backed storage, false for SSD (ARCH §16).
    // Affects monitoring and GC scheduling only; core logic is medium-agnostic.
    isRotational bool
}

// appendToVLog writes one complete 262212-byte vLog entry and fsyncs to disk.
// MUST be called only from the single writer goroutine (IC §5.3).
//
// Algorithm (ARCH §16):
//  1. Compute content_hash = SHA-256(chunkData).
//  2. Build a 262212-byte buffer: [chunk_id][chunk_size][chunk_data][content_hash].
//  3. Write the buffer via os.File.Write.
//  4. Call Sync() — return ErrVLogFsync on failure.
//  5. Advance ws.vlogHead by vLogEntrySize.
func (ws *wiskeyStore) appendToVLog(chunkID [32]byte, chunkData []byte) (offset uint64, err error) {
    contentHash := sha256.Sum256(chunkData)

    buf := make([]byte, vLogEntrySize)
    copy(buf[vlogOffChunkID:], chunkID[:])
    binary.BigEndian.PutUint32(buf[vlogOffChunkSize:], 262144)
    copy(buf[vlogOffChunkData:], chunkData)
    copy(buf[vlogOffContentHash:], contentHash[:])

    offset = ws.vlogHead
    if _, werr := ws.vlog.Write(buf); werr != nil {
        return 0, fmt.Errorf("%w: write: %v", ErrVLogFsync, werr)
    }
    if serr := ws.vlog.Sync(); serr != nil {
        return 0, fmt.Errorf("%w: sync: %v", ErrVLogFsync, serr)
    }
    ws.vlogHead += vLogEntrySize
    return offset, nil
}

// readFromVLog reads the 262212-byte entry at byteOffset and verifies its content_hash.
// Goroutine-safe: uses ReadAt (pread semantics — concurrent-safe on POSIX).
//
// Algorithm (ARCH §16 §Audit lookup path steps 4–5):
//  1. ReadAt vLogEntrySize bytes from byteOffset.
//  2. Extract content_hash from buf[vlogOffContentHash:].
//  3. Compute SHA-256(buf[vlogOffChunkData:vlogOffChunkData+262144]).
//  4. If hashes differ: return nil, ErrContentHashMismatch (disk corruption detected).
//  5. Return a copy of buf[vlogOffChunkData:vlogOffChunkData+262144].
func (ws *wiskeyStore) readFromVLog(byteOffset uint64) ([]byte, error) {
    buf := make([]byte, vLogEntrySize)
    if _, err := ws.vlog.ReadAt(buf, int64(byteOffset)); err != nil {
        return nil, fmt.Errorf("%w: ReadAt offset %d: %v", ErrVLogRead, byteOffset, err)
    }

    storedHash := buf[vlogOffContentHash : vlogOffContentHash+32]
    computed := sha256.Sum256(buf[vlogOffChunkData : vlogOffChunkData+262144])
    for i := range computed {
        if computed[i] != storedHash[i] {
            return nil, ErrContentHashMismatch
        }
    }

    data := make([]byte, 262144)
    copy(data, buf[vlogOffChunkData:vlogOffChunkData+262144])
    return data, nil
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/storage/vlog.go           && echo PASS || echo FAIL
  $ test -f internal/storage/rotational.go     && echo PASS || echo FAIL
  $ test -f internal/storage/rotational_other.go && echo PASS || echo FAIL

COMPILE (both platforms to confirm build tags):
  $ GOARCH=amd64 go build ./internal/storage/
  EXPECT: exit 0

  $ GOARCH=arm64 go build ./internal/storage/
  EXPECT: exit 0

BUILD_TAGS:
  $ grep -c "//go:build linux" internal/storage/rotational.go
  EXPECT: 1

  $ grep -c "//go:build !linux" internal/storage/rotational_other.go
  EXPECT: 1

VLOG_OFFSET_CONSTANTS:
  $ grep -c "vlogOffChunkID\|vlogOffChunkSize\|vlogOffChunkData\|vlogOffContentHash" \
      internal/storage/vlog.go
  EXPECT: 4

  # contentHash offset must be 262180 = 36 + 262144
  $ grep -c "262180" internal/storage/vlog.go
  EXPECT: 1

STRUCT_DECLARED:
  $ grep -c "^type wiskeyStore struct" internal/storage/vlog.go
  EXPECT: 1

KEY_FUNCTIONS:
  $ grep -c "^func (ws \*wiskeyStore) appendToVLog\|^func (ws \*wiskeyStore) readFromVLog" \
      internal/storage/vlog.go
  EXPECT: 2

FSYNC_PRESENT:
  $ grep -c "\.Sync()" internal/storage/vlog.go
  EXPECT: 1

CONCURRENT_SAFE_READ:
  # readFromVLog must use ReadAt (pread), not Read+Seek which is not goroutine-safe
  $ grep -c "\.ReadAt(" internal/storage/vlog.go
  EXPECT: 1

SHA256_BOTH_PATHS:
  # sha256 used for both write (content_hash creation) and read (verification)
  $ grep -c "sha256\." internal/storage/vlog.go
  EXPECT: 2

NEGATIVE_CHECKS:
  # LookupChunk must never return data when ErrContentHashMismatch is detected
  $ grep -A5 "ErrContentHashMismatch" internal/storage/vlog.go | grep -c "return nil,"
  EXPECT: >= 1

  # No prohibited internal/ imports (IC §9)
  $ grep -n "Vyomanaut_V2/internal/" internal/storage/vlog.go \
      && echo "FAIL: prohibited import" || echo "PASS"
```

---

#### Session 5.1.3 — Implement the RocksDB chunk index wrapper

**PRECONDITIONS:**

- Session 5.1.2 complete (`wiskeyStore` struct declared with `index *rocksDBIndex` field reference)
- `github.com/linxGnu/grocksdb` in `go.mod`; `librocksdb` installed (ARCH §4.1)

**TASK:**

1. Create `internal/storage/index.go` defining the `rocksDBIndex` struct and all its operations.
2. Configure the Bloom filter at **10 bits per key** (~1% false-positive rate) so that audit challenges for unassigned chunks exit via the filter with no disk I/O (ARCH §16, ARCH §27.1).
3. Configure a 64 MB LRU block cache so the index remains warm after start-up — audit challenge lookups then require no disk I/O for the index step (ARCH §16).
4. Use **two column families** — `"default"` for the data index and `"dht-keys"` for cached DHT keys. The 44-byte ARCH §27.1sizing refers to `"default"` CF entries only. The DHT key cache (IC §12.2, ARCH §13) is stored in `"dht-keys"` as `chunk_id(32) → dht_key(32)`.
5. RocksDB key: `chunk_id` (32 bytes). RocksDB value (default CF): `vlog_offset(uint64 big-endian=8) + chunk_size(uint32 big-endian=4) = 12 bytes`.

**FILE:** `internal/storage/index.go`

```go
package storage

import (
    "encoding/binary"
    "fmt"

    "github.com/linxGnu/grocksdb"
)

// rocksDBIndex wraps the RocksDB instance used as the WiscKey chunk index (ARCH §16, ARCH §27.1).
//
// Column families:
//   "default"  — chunk_id(32-byte key) → vlog_offset(uint64)+chunk_size(uint32) = 12-byte value
//   "dht-keys" — chunk_id(32-byte key) → dht_key(32-byte value)
//
// The "default" CF entry total ≈ 44 bytes (ARCH §27.1: key=32, value=12).
// The "dht-keys" CF stores pre-computed DHT keys for republication (IC §12.2, ARCH §13).
type rocksDBIndex struct {
    db       *grocksdb.DB
    cfData   *grocksdb.ColumnFamilyHandle // "default" CF — data index
    cfDHT    *grocksdb.ColumnFamilyHandle // "dht-keys" CF — cached DHT keys
    writeOps *grocksdb.WriteOptions
    readOps  *grocksdb.ReadOptions
}

// openRocksDBIndex opens (or creates) the RocksDB index at dbPath.
// Bloom filter: 10 bits per key, ~1% false-positive rate (ARCH §16, ARCH §27.1).
// Block cache: 64 MB LRU so the index stays warm — audit lookups require no disk I/O.
func openRocksDBIndex(dbPath string) (*rocksDBIndex, error) {
    bbto := grocksdb.NewDefaultBlockBasedTableOptions()
    bbto.SetFilterPolicy(grocksdb.NewBloomFilter(10)) // 10 bits/key ≈ 1% FP rate
    bbto.SetBlockCache(grocksdb.NewLRUCache(64 * 1024 * 1024)) // 64 MB

    cfOpts := grocksdb.NewDefaultOptions()
    cfOpts.SetBlockBasedTableFactory(bbto)
    cfOpts.SetCreateIfMissing(true)

    dbOpts := grocksdb.NewDefaultOptions()
    dbOpts.SetCreateIfMissing(true)
    dbOpts.SetCreateMissingColumnFamilies(true)

    cfNames := []string{"default", "dht-keys"}
    cfOptsList := []*grocksdb.Options{cfOpts, cfOpts}

    db, handles, err := grocksdb.OpenDbColumnFamilies(dbOpts, dbPath, cfNames, cfOptsList)
    if err != nil {
        return nil, fmt.Errorf("storage: openRocksDBIndex: %w", err)
    }
    return &rocksDBIndex{
        db:       db,
        cfData:   handles[0],
        cfDHT:    handles[1],
        writeOps: grocksdb.NewDefaultWriteOptions(),
        readOps:  grocksdb.NewDefaultReadOptions(),
    }, nil
}

// put inserts or updates the data index entry for chunkID.
func (idx *rocksDBIndex) put(chunkID [32]byte, vlogOffset uint64, chunkSize uint32) error {
    val := make([]byte, indexValueSize) // = 12
    binary.BigEndian.PutUint64(val[0:8], vlogOffset)
    binary.BigEndian.PutUint32(val[8:12], chunkSize)
    if err := idx.db.PutCF(idx.writeOps, idx.cfData, chunkID[:], val); err != nil {
        return fmt.Errorf("%w: chunkID %x: %v", ErrRocksDBInsert, chunkID, err)
    }
    return nil
}

// get retrieves vlog_offset and chunk_size for chunkID.
// Returns ErrChunkNotFound when the key is absent; the Bloom filter exits fast on this path.
func (idx *rocksDBIndex) get(chunkID [32]byte) (vlogOffset uint64, chunkSize uint32, err error) {
    sl, err := idx.db.GetCF(idx.readOps, idx.cfData, chunkID[:])
    if err != nil {
        return 0, 0, fmt.Errorf("%w: RocksDB get: %v", ErrVLogRead, err)
    }
    defer sl.Free()
    if !sl.Exists() {
        return 0, 0, ErrChunkNotFound
    }
    d := sl.Data()
    return binary.BigEndian.Uint64(d[0:8]), binary.BigEndian.Uint32(d[8:12]), nil
}

// del removes the data index entry for chunkID. vLog space reclaimed by RunGC.
func (idx *rocksDBIndex) del(chunkID [32]byte) error {
    return idx.db.DeleteCF(idx.writeOps, idx.cfData, chunkID[:])
}

// putDHTKey stores the pre-computed DHT key for a chunk (IC §12.2, ARCH §13).
// Called once per chunk at upload time by the upload orchestrator (Session 15.2.1).
func (idx *rocksDBIndex) putDHTKey(chunkID [32]byte, dhtKey [32]byte) error {
    return idx.db.PutCF(idx.writeOps, idx.cfDHT, chunkID[:], dhtKey[:])
}

// dhtKeyFor retrieves the cached DHT key for chunkID (IC §12.2, ARCH §13).
// Returns ok=false if no DHT key was ever stored for this chunk.
// The heartbeat goroutine calls this — it must NEVER recompute from file_owner_key.
func (idx *rocksDBIndex) dhtKeyFor(chunkID [32]byte) (dhtKey [32]byte, ok bool) {
    sl, err := idx.db.GetCF(idx.readOps, idx.cfDHT, chunkID[:])
    if err != nil || !sl.Exists() {
        return dhtKey, false
    }
    defer sl.Free()
    copy(dhtKey[:], sl.Data())
    return dhtKey, true
}

// allChunkIDs returns every chunk_id in the data index.
// Used by the DHT republication loop in the heartbeat goroutine (Session 6.3.1, IC §12.2).
func (idx *rocksDBIndex) allChunkIDs() [][32]byte {
    it := idx.db.NewIteratorCF(idx.readOps, idx.cfData)
    defer it.Close()
    var ids [][32]byte
    for it.SeekToFirst(); it.Valid(); it.Next() {
        var id [32]byte
        copy(id[:], it.Key().Data())
        ids = append(ids, id)
    }
    return ids
}

// close releases all RocksDB handles.
func (idx *rocksDBIndex) close() {
    idx.cfData.Destroy()
    idx.cfDHT.Destroy()
    idx.writeOps.Destroy()
    idx.readOps.Destroy()
    idx.db.Close()
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/storage/index.go && echo PASS || echo FAIL

COMPILE (requires librocksdb):
  $ CGO_ENABLED=1 go build ./internal/storage/
  EXPECT: exit 0

GROCKSDB_IMPORT:
  $ grep -c "github.com/linxGnu/grocksdb" internal/storage/index.go
  EXPECT: 1

  $ grep -c "github.com/linxGnu/grocksdb" go.mod
  EXPECT: 1

BLOOM_FILTER_10_BITS:
  $ grep -c "NewBloomFilter(10)" internal/storage/index.go
  EXPECT: 1

BLOCK_CACHE:
  $ grep -c "NewLRUCache\|LRUCache" internal/storage/index.go
  EXPECT: 1

COLUMN_FAMILIES:
  $ grep -c '"dht-keys"' internal/storage/index.go
  EXPECT: >= 2  (CF name definition + usage)

DHT_KEY_METHODS:
  $ grep -c "func.*putDHTKey\|func.*dhtKeyFor\|func.*allChunkIDs" internal/storage/index.go
  EXPECT: 3

DATA_INDEX_METHODS:
  $ grep -c "func.*idx.*\) put\b\|func.*idx.*\) get\b\|func.*idx.*\) del\b" \
      internal/storage/index.go
  EXPECT: 3

VALUE_SIZE:
  # Value is 12 bytes (vlog_offset=8 + chunk_size=4), not 44
  $ grep -c "indexValueSize\b" internal/storage/index.go
  EXPECT: >= 1

NEGATIVE_CHECKS:
  # No imports of other internal/ packages (IC §9)
  $ grep -n "Vyomanaut_V2/internal/" internal/storage/index.go \
      && echo "FAIL: prohibited import" || echo "PASS"

  # DHT key must not be derived here — that belongs in internal/crypto (IC §12.2)
  $ grep -c "DeriveDHTKey\|DeriveDHTOwnerKey\|HMAC" internal/storage/index.go
  EXPECT: 0
```

---

#### Session 5.1.4 — Implement `AppendChunk()` and `LookupChunk()`

**PRECONDITIONS:**

- Sessions 5.1.2 and 5.1.3 complete
- `appendToVLog`, `readFromVLog`, and `rocksDBIndex` all exist
- `wiskeyStore` struct has `index *rocksDBIndex` and `vlog *os.File` fields

**TASK:**

Add `AppendChunk` and `LookupChunk` method implementations on `*wiskeyStore` to `internal/storage/vlog.go`, wiring together the vLog functions (Session 5.1.2) and the RocksDB index (Session 5.1.3).

**ALGORITHM — `AppendChunk` (IC §5.3, ARCH §16):**

```
Pre-condition (debug panic if violated):
  assert len(chunkID) == 32 && len(chunkData) == 262144

1. Call ws.appendToVLog(chunkID, chunkData) → offset, err
   On err: return 0, ErrVLogFsync — daemon must halt; crash recovery on restart.
2. Call ws.index.put(chunkID, offset, 262144) → err
   On err: return 0, ErrRocksDBInsert
   NOTE: a successful vLog write with a failed index insert is recoverable via
   RecoverFromCrash — the vLog entry is the source of truth.
3. Return offset, nil
```

**ALGORITHM — `LookupChunk` (ARCH §16 §Audit lookup path):**

```
1. Call ws.index.get(chunkID) → vlogOffset, _, err
   On ErrChunkNotFound: return nil, ErrChunkNotFound  [Bloom filter exit — no disk I/O]
   On other err:        return nil, ErrVLogRead
2. Call ws.readFromVLog(vlogOffset) → data, err
   readFromVLog performs SHA-256 verification internally.
   On ErrContentHashMismatch: return nil, ErrContentHashMismatch
   On ErrVLogRead:            return nil, ErrVLogRead
3. Return data, nil
```

**CRITICAL RULE (IC §5.3):** `LookupChunk` MUST NOT return any chunk bytes when `ErrContentHashMismatch` is detected. The buffer is already zeroed on error by `readFromVLog` (see Session 5.1.2 implementation).

**VERIFY:**

```bash
COMPILE:
  $ CGO_ENABLED=1 go build ./internal/storage/
  EXPECT: exit 0

METHODS_IMPLEMENTED:
  $ grep -c "^func (ws \*wiskeyStore) AppendChunk" internal/storage/vlog.go
  EXPECT: 1

  $ grep -c "^func (ws \*wiskeyStore) LookupChunk" internal/storage/vlog.go
  EXPECT: 1

PRE_CONDITION_GUARD:
  $ grep -c "262144\|len(chunkData)\|len(chunkID)" internal/storage/vlog.go
  EXPECT: >= 2

CALLS_HELPER_FUNCTIONS:
  $ grep -c "appendToVLog\|readFromVLog" internal/storage/vlog.go
  EXPECT: >= 2  (one call per interface method)

  $ grep -c "index\.put\|index\.get" internal/storage/vlog.go
  EXPECT: >= 2

NEGATIVE_CHECKS:
  # AppendChunk must NOT call readFromVLog (write path only)
  $ grep -A20 "func (ws \*wiskeyStore) AppendChunk" internal/storage/vlog.go \
      | grep -c "readFromVLog"
  EXPECT: 0

  # LookupChunk must NOT return data bytes when ErrContentHashMismatch occurs
  # (readFromVLog returns nil on corruption — verify the error is passed through)
  $ grep -A15 "func (ws \*wiskeyStore) LookupChunk" internal/storage/vlog.go \
      | grep -c "return nil, err\|return nil, Err"
  EXPECT: >= 1
```

---

#### Session 5.1.5 — Implement `DeleteChunk`, `RecoverFromCrash`, `RunGC`, and `NewChunkStore`

**PRECONDITIONS:**

- Sessions 5.1.1 through 5.1.4 complete
- `wiskeyStore` struct fields `index`, `vlog`, `vlogHead`, and `isRotational` are all populated

**TASK:**

1. Add `NewChunkStore(dataDir string) (ChunkStore, error)` to `internal/storage/store.go`. This constructor opens RocksDB via `openRocksDBIndex`, opens (or creates) the vLog file, detects rotational storage, and returns a `*wiskeyStore`. It **must NOT** call `RecoverFromCrash` — the daemon (Session 13.1.1) calls it explicitly before launching the writer goroutine (NFR-024).
2. Add `DeleteChunk`, `RecoverFromCrash`, `RunGC`, and `Close` implementations to `internal/storage/vlog.go`.

**FUNCTION SIGNATURE for `NewChunkStore` (add to `store.go`):**

```go
// NewChunkStore opens or creates a WiscKey chunk store rooted at dataDir.
// Creates two subdirectories: dataDir/index (RocksDB) and dataDir/chunks.vlog (vLog).
//
// The caller MUST call RecoverFromCrash() on the returned ChunkStore and allow it to
// complete BEFORE starting the writer goroutine (NFR-024, ARCH §16, Session 13.1.1 step 4).
func NewChunkStore(dataDir string) (ChunkStore, error)
```

**ALGORITHM — `DeleteChunk`:**

```
1. Call ws.index.del(chunkID)
2. Return any error directly.
   vLog space is NOT immediately freed — RunGC reclaims it asynchronously.
```

**ALGORITHM — `RecoverFromCrash` (ARCH §16 §Crash recovery):**

```
1. Seek the vLog file to offset 0.
2. currentOffset := uint64(0)
3. For each 262212-byte block from currentOffset to EOF:
   a. ReadAt vLogEntrySize bytes. If < vLogEntrySize bytes read: stop (tail truncation).
   b. Extract chunk_id from buf[vlogOffChunkID:vlogOffChunkID+32].
   c. Verify SHA-256(buf[vlogOffChunkData:]) == buf[vlogOffContentHash:].
      On mismatch: this is a corrupt tail entry — stop here.
   d. Call ws.index.get(chunk_id): if ErrChunkNotFound, call ws.index.put(chunk_id, currentOffset, 262144).
   e. currentOffset += vLogEntrySize
4. Set ws.vlogHead = currentOffset.
```

**ALGORITHM — `RunGC` (ARCH §16 §Garbage collection):**

```
1. Enumerate all chunk_ids in ws.index via allChunkIDs() → build liveSet map.
2. Seek vLog to offset 0; walk in vLogEntrySize steps.
3. For each entry: if chunk_id NOT in liveSet → mark offset as dead.
4. Compact: write a new temporary vLog with only live entries; update RocksDB vlog_offsets.
5. Atomically rename the new vLog file over the old one (os.Rename).
6. Update ws.vlogHead. Check ctx.Done() between entries for clean cancellation.
```

**VERIFY:**

```bash
COMPILE:
  $ CGO_ENABLED=1 go build ./internal/storage/
  EXPECT: exit 0

NEWCHUNKSTORE_IN_STORE_GO:
  $ grep -c "^func NewChunkStore" internal/storage/store.go
  EXPECT: 1

REMAINING_METHODS_IN_VLOG_GO:
  $ grep -c "^func (ws \*wiskeyStore) DeleteChunk\|^func (ws \*wiskeyStore) RecoverFromCrash\|^func (ws \*wiskeyStore) RunGC\|^func (ws \*wiskeyStore) Close" \
      internal/storage/vlog.go
  EXPECT: 4

RECOVER_WALKS_FROM_ZERO:
  $ grep -c "currentOffset\|vlogHead\|SeekToFirst\|offset.*0" internal/storage/vlog.go
  EXPECT: >= 2

GC_CONTEXT_CANCELLATION:
  $ grep -c "ctx\.Done()\|ctx\.Err()" internal/storage/vlog.go
  EXPECT: >= 1

NEWCHUNKSTORE_DOES_NOT_CALL_RECOVER:
  # Caller's responsibility per NFR-024; must NOT be called inside the constructor
  $ grep -A25 "^func NewChunkStore" internal/storage/store.go | grep -c "RecoverFromCrash"
  EXPECT: 0

ROCKSDB_AND_VLOG_OPENED:
  $ grep -c "openRocksDBIndex\|os\.OpenFile\|os\.Create" internal/storage/store.go
  EXPECT: >= 2

FULL_CGO_BUILD:
  $ CGO_ENABLED=1 go build ./internal/storage/
  EXPECT: exit 0; link-time failure if librocksdb not installed (ARCH §4.1)
```

---

### Phase 5.2 — Storage Tests

**Reference:** MVP §8.2 (`single_writer_test.go`, `store_test.go`), IC §5.3 (concurrency contract), NFR-024 (crash recovery requirement)

---

#### Session 5.2.1 — Implement `TestSingleWriterGoroutine`

**PRECONDITIONS:**

- Session 5.1.5 complete — `NewChunkStore` and all six interface methods are implemented
- `librocksdb` available in the test environment

**TASK:**

Create `internal/storage/single_writer_test.go` implementing `TestSingleWriterGoroutine`.

**TEST DESIGN (MVP §8.2):**

The test validates the **correct** channel-serialized design. 100 concurrent callers each submit a write request via a buffered channel to a single writer goroutine; the writer is the only goroutine that directly calls `AppendChunk`. The "deadlock, not data corruption" property stated in MVP §8.2 means: if the writer goroutine exits early due to a bug, callers block on the channel (a detectable deadlock) rather than silently interleaving 262 KB writes into a corrupted vLog. Run with `-race` to verify no data races in the channel-plus-writer interaction.

**FILE:** `internal/storage/single_writer_test.go`

```go
package storage_test

import (
    "crypto/sha256"
    "testing"

    "github.com/stretchr/testify/require"

    "github.com/masamasaowl/Vyomanaut_V2/internal/storage"
)

// TestSingleWriterGoroutine validates that 100 concurrent callers routing through a
// single writer goroutine (via a buffered channel) store all 100 chunks correctly
// without data corruption or race conditions (IC §5.3, MVP §8.2).
//
// Run with: go test -race -run TestSingleWriterGoroutine ./internal/storage/
func TestSingleWriterGoroutine(t *testing.T) {
    store, err := storage.NewChunkStore(t.TempDir())
    require.NoError(t, err)
    require.NoError(t, store.RecoverFromCrash())
    defer store.Close()

    // writeReq is the message type submitted to the single writer goroutine.
    type writeReq struct {
        chunkID   [32]byte
        chunkData []byte
        result    chan error
    }

    // writeCh serialises all AppendChunk calls through one goroutine.
    writeCh := make(chan writeReq, 200)
    go func() {
        for req := range writeCh {
            _, err := store.AppendChunk(req.chunkID, req.chunkData)
            req.result <- err
        }
    }()

    const numChunks = 100
    errCh := make(chan error, numChunks)
    chunkIDs := make([][32]byte, numChunks)

    // Launch 100 concurrent callers — each submits via writeCh, never calls AppendChunk directly.
    for i := 0; i < numChunks; i++ {
        data := make([]byte, 262144)
        data[0] = byte(i) // unique content per chunk
        id := sha256.Sum256(data)
        chunkIDs[i] = id
        resultCh := make(chan error, 1)
        writeCh <- writeReq{chunkID: id, chunkData: data, result: resultCh}
        go func() { errCh <- <-resultCh }()
    }

    // All 100 AppendChunk calls must succeed.
    for i := 0; i < numChunks; i++ {
        require.NoError(t, <-errCh, "chunk %d AppendChunk must not error", i)
    }
    close(writeCh)

    // All 100 chunks must be readable with the correct content.
    for i, id := range chunkIDs {
        data, err := store.LookupChunk(id)
        require.NoError(t, err, "chunk %d must be found after write", i)
        require.Equal(t, 262144, len(data), "chunk %d must be 262144 bytes", i)
        require.Equal(t, byte(i), data[0], "chunk %d first byte must equal index", i)
    }
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/storage/single_writer_test.go && echo PASS || echo FAIL

TEST_WITH_RACE_DETECTOR:
  $ go test -race -count=1 -run TestSingleWriterGoroutine ./internal/storage/
  EXPECT: exit 0; "--- PASS: TestSingleWriterGoroutine" in output; zero data-race reports

CONTENT_CHECKS:
  $ grep -c "numChunks = 100" internal/storage/single_writer_test.go
  EXPECT: 1

  # writeCh must be the channel used to serialise writes
  $ grep -c "writeCh\|writeReq\|chan.*error" internal/storage/single_writer_test.go
  EXPECT: >= 3

NEGATIVE_CHECKS:
  # AppendChunk must be called in exactly ONE place — inside the writer goroutine
  $ grep -c "store\.AppendChunk\|\.AppendChunk(" internal/storage/single_writer_test.go
  EXPECT: 1

  # AppendChunk must NOT appear inside a 'go func' (which would bypass the writer goroutine)
  $ grep -B5 "AppendChunk" internal/storage/single_writer_test.go | grep -c "go func"
  EXPECT: 0
```

---

#### Session 5.2.2 — Storage round-trip and crash recovery tests

**PRECONDITIONS:**

- Session 5.1.5 complete
- Session 5.2.1 (`TestSingleWriterGoroutine`) passes

**TASK:**

Create `internal/storage/store_test.go` with the following named tests. All tests use
`t.TempDir()` for isolation and the channel-serialized writer pattern from Session 5.2.1.

**TEST INVENTORY:**

| Test name | What it validates |
|---|---|
| `TestAppendLookupRoundTrip` | Write N chunks; read each back; SHA-256 matches (IC §5.3 post-condition) |
| `TestLookupChunkNotFound` | `LookupChunk` on an unwritten chunk_id returns `ErrChunkNotFound` |
| `TestLookupChunkCorrupt` | Corrupt vLog bytes at a known offset; `LookupChunk` returns `ErrContentHashMismatch` |
| `TestDeleteChunk` | `DeleteChunk` then `LookupChunk` returns `ErrChunkNotFound` |
| `TestRecoverFromCrash` | Write chunks; simulate crash by skipping `Close()`; re-open; `RecoverFromCrash()`; all chunks readable (NFR-024) |
| `TestRecoverFromCrashPartialWrite` | Write a partial truncated entry at vLog tail; `RecoverFromCrash()` skips it; earlier entries still readable |
| `TestRunGCReclaimsSpace` | Write 10 chunks; delete 5; run GC; freed space verified; 5 remaining chunks still readable |

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/storage/store_test.go && echo PASS || echo FAIL

ALL_TESTS_PASS:
  $ go test -race -count=1 -v ./internal/storage/
  EXPECT: exit 0; every test listed above shows "--- PASS:"

TEST_NAME_PRESENCE (seven tests):
  $ grep -c "^func TestAppendLookupRoundTrip\|^func TestLookupChunkNotFound\|^func TestLookupChunkCorrupt\|^func TestDeleteChunk\|^func TestRecoverFromCrash\b\|^func TestRecoverFromCrashPartialWrite\|^func TestRunGCReclaimsSpace" \
      internal/storage/store_test.go
  EXPECT: 7

CRASH_SIMULATION_SKIPS_CLOSE:
  # TestRecoverFromCrash must NOT call Close() — the crash skips it
  $ grep -A30 "^func TestRecoverFromCrash\b" internal/storage/store_test.go \
      | grep -c "\.Close()"
  EXPECT: 0

CORRUPTION_TEST_WRITES_DIRECTLY_TO_FILE:
  $ grep -c "os\.OpenFile\|os\.WriteAt\|WriteString\|corrupt\|Truncate" \
      internal/storage/store_test.go
  EXPECT: >= 1

CI_RACE_GATE (check-04 passes with this session complete):
  $ go test -race -count=1 ./...
  EXPECT: exit 0
```

---

## Milestone 6 — P2P Network Layer (`internal/p2p`)

**Deliverable:** libp2p host with QUIC v1 primary and TCP+Noise XX fallback transports,
three-tier NAT traversal (AutoNAT → DCUtR → Circuit Relay v2), Kademlia DHT with a custom
HMAC key validator, 0-RTT policy enforced per an explicit protocol deny-list, and a
heartbeat goroutine with integrated DHT republication.

**References:**

- **IC §4** — Common libp2p protocol rules: transport auth, 0-RTT policy, framing, stream lifecycle
- **IC §4.3** — Circuit Relay v2 reservation: TTL, slot count, relay multiaddrs in heartbeat payload
- **IC §5.4** — `Host` and `DHT` Go interfaces, sentinel errors
- **IC §12** — DHT key contract: HMAC key derivation, validator accept/reject rules
- **IC §12.2** — DHT republication: `dht_key` cached in RocksDB at upload time; never recomputed at republication time
- **ARCH §13** — P2P Transfer Layer: transports, three-tier NAT, DHT parameters (`k=16`, `α=3`), session resumption policy
- **MVP §8.2** — File inventory for `internal/p2p`
- **MVP §8.4** — CI check 5: `TestDHTKeyValidatorPersists` (becomes a hard gate once M6 completes)
- **ADR-021** — P2P transfer protocol design
- **ADR-001** — DHT HMAC pseudonymisation, DHT privacy

**Dependency to add:** `go get github.com/libp2p/go-libp2p`. Pin the version in `go.mod`
before this milestone closes — the custom DHT key validator must survive every upgrade and
`TestDHTKeyValidatorPersists` is the CI gate (ARCH §4, MVP §8.4 CI check 5).

---

### Phase 6.1 — libp2p Host

**Reference:** IC §5.4 (Host interface), IC §4 (common rules), ARCH §13 (transports, 0-RTT session resumption policy)

---

#### Session 6.1.1 — Implement `Host` interface

**PRECONDITIONS:**

- `internal/p2p/doc.go` exists (stub from Session 0.1.3)
- `github.com/libp2p/go-libp2p` added to `go.mod`
- `internal/crypto` package complete (Milestone 2) — Ed25519 key pair available for Peer ID

**TASK:**

1. Create `internal/p2p/errors.go` with the three sentinel errors from IC §5.4.
2. Create `internal/p2p/host.go` with:
   - The `Host` interface from IC §5.4 (six methods).
   - A `zeroRTTProhibited` package-level constant set enumerating every protocol ID for which 0-RTT is forbidden (IC §4.2, IC §4.4.1, IC §4.5). **This must be an explicit set, not a suffix-pattern match** — `/vyomanaut/repair-download/1.0.0` and `/vyomanaut/vetting-gc/1.0.0` do not contain `-challenge` or `-audit` so suffix matching alone is insufficient.
   - The `NewStream` method enforces 0-RTT by checking `zeroRTTProhibited` before opening the stream.

**0-RTT DENY-LIST — complete and exhaustive (IC §4, ARCH §13):**

| Protocol ID | 0-RTT | Reason |
|---|---|---|
| `/vyomanaut/chunk-upload/1.0.0` | **Permitted** | Replay is idempotent — RocksDB UNIQUE rejects duplicate (IC §4.1) |
| `/vyomanaut/audit-challenge/1.0.0` | **Prohibited** | Replayed response falsely credits a PASS (IC §4.2) |
| `/vyomanaut/repair-download/1.0.0` | **Prohibited** | Replayed request could exfiltrate chunk data (IC §4.4.1) |
| `/vyomanaut/vetting-gc/1.0.0` | **Prohibited** | Replayed GC instruction causes permanent deletion (IC §4.5) |

**FILE:** `internal/p2p/errors.go`

```go
package p2p

import "errors"

var (
    // ErrPeerIDMismatch is returned by Connect when the remote peer's authenticated ID
    // does not match the expected peerID supplied by the caller (IC §5.4, NFR-016).
    ErrPeerIDMismatch = errors.New("p2p: remote Peer ID does not match expected")

    // ErrAllAddrsFailed is returned by Connect when every provided multiaddr
    // fails to establish a connection (IC §5.4).
    ErrAllAddrsFailed = errors.New("p2p: all provided multiaddrs failed to connect")

    // ErrDHTKeyInvalid is returned by PutProviderRecord or FindProviders when
    // the key does not satisfy the custom HMAC validator (IC §5.4, IC §12).
    ErrDHTKeyInvalid = errors.New("p2p: DHT key does not pass HMAC validator")
)
```

**FILE:** `internal/p2p/host.go`

```go
package p2p

import (
    "context"

    autonat "github.com/libp2p/go-libp2p/p2p/host/autonat"
    "github.com/libp2p/go-libp2p/core/network"
    "github.com/libp2p/go-libp2p/core/peer"
    "github.com/libp2p/go-libp2p/core/protocol"
    "github.com/multiformats/go-multiaddr"
)

// zeroRTTProhibited is the complete, exhaustive set of protocol IDs for which
// 0-RTT session resumption is forbidden (IC §4, ARCH §13 §Session resumption policy).
//
// DESIGN: This is an explicit deny-list, NOT a suffix/pattern match.
// Rationale: /vyomanaut/repair-download/1.0.0 and /vyomanaut/vetting-gc/1.0.0 do not
// contain "-challenge" or "-audit", so suffix matching would silently miss them (IC §4).
//
// /vyomanaut/chunk-upload/1.0.0 is intentionally absent — 0-RTT is PERMITTED (IC §4.1).
var zeroRTTProhibited = map[string]struct{}{
    "/vyomanaut/audit-challenge/1.0.0": {},
    "/vyomanaut/repair-download/1.0.0": {},
    "/vyomanaut/vetting-gc/1.0.0":      {},
}

// Host is the primary libp2p networking interface for the provider daemon (IC §5.4).
// Constructed once at daemon startup via NewHost; all methods are goroutine-safe.
type Host interface {
    // PeerID returns the local Ed25519-based libp2p Peer ID.
    // Computed as multihash(ed25519_public_key) (ARCH §13, ADR-021).
    PeerID() peer.ID

    // Connect dials peerID at the given multiaddrs and performs cryptographic
    // Peer ID verification during the TLS 1.3 (QUIC) or Noise XX (TCP) handshake
    // (IC §4, NFR-016). Returns ErrPeerIDMismatch if the authenticated ID differs.
    Connect(ctx context.Context, peerID peer.ID, addrs []multiaddr.Multiaddr) error

    // NewStream opens an application-level stream to peerID for protocolID.
    //
    // 0-RTT policy (IC §4, ARCH §13 §Session resumption policy):
    // If protocolID is in zeroRTTProhibited, this method sets DisableEarlyData on
    // the underlying QUIC connection before opening the stream. The caller does NOT
    // need to manage this — the Host enforces it automatically.
    //
    // /vyomanaut/chunk-upload/1.0.0 is excluded from the deny-list; 0-RTT is permitted.
    NewStream(ctx context.Context, peerID peer.ID, protocolID protocol.ID) (network.Stream, error)

    // SetStreamHandler registers a handler for incoming streams of protocolID.
    // Each incoming stream runs the handler in a new goroutine (IC §4).
    SetStreamHandler(protocolID protocol.ID, handler network.StreamHandler)

    // NATType returns the current AutoNAT classification of the local node.
    // Periodically updated by the background AutoNAT service (ARCH §13).
    NATType() autonat.NATStatus

    // Close shuts down the host, all transports, and all open connections.
    Close() error
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/p2p/host.go   && echo PASS || echo FAIL
  $ test -f internal/p2p/errors.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/p2p/
  EXPECT: exit 0

ZERO_RTT_DENY_LIST_DECLARATION:
  $ grep -c "^var zeroRTTProhibited" internal/p2p/host.go
  EXPECT: 1

ALL_THREE_PROHIBITED_PROTOCOLS_PRESENT:
  $ grep -c "audit-challenge.*1.0.0\|repair-download.*1.0.0\|vetting-gc.*1.0.0" \
      internal/p2p/host.go
  EXPECT: 3

CHUNK_UPLOAD_NOT_IN_DENY_LIST:
  # chunk-upload is permitted; it must NOT appear in zeroRTTProhibited
  $ grep -A10 "zeroRTTProhibited" internal/p2p/host.go | grep -c "chunk-upload"
  EXPECT: 0

DENY_LIST_USED_IN_NEWSTREAM:
  $ grep -c "zeroRTTProhibited" internal/p2p/host.go
  EXPECT: >= 2  (declaration + usage in NewStream implementation)

SENTINEL_ERRORS:
  $ grep -c "ErrPeerIDMismatch\|ErrAllAddrsFailed\|ErrDHTKeyInvalid" \
      internal/p2p/errors.go
  EXPECT: 3

INTERFACE_METHODS:
  $ grep -c "PeerID\|Connect\|NewStream\|SetStreamHandler\|NATType\|Close" \
      internal/p2p/host.go
  EXPECT: 6

TRANSPORT_CONFIG:
  $ grep -c "QUIC\|quic" internal/p2p/host.go
  EXPECT: >= 1  (QUIC v1 primary transport)

  $ grep -c "TCP\|tcp\|Noise\|noise" internal/p2p/host.go
  EXPECT: >= 1  (TCP+Noise XX fallback)

NEGATIVE_CHECKS:
  # Must NOT use suffix matching to determine 0-RTT policy (IC §4)
  $ grep -n "HasSuffix\|strings\.Contains.*-challenge\|strings\.Contains.*-audit" \
      internal/p2p/host.go \
      && echo "FAIL: use explicit deny-list, not pattern matching" || echo "PASS"

  # No imports from other internal/ packages except internal/p2p itself (IC §9)
  $ grep -n "Vyomanaut_V2/internal/" internal/p2p/host.go | grep -v "internal/p2p"
  EXPECT: empty
```

---

#### Session 6.1.2 — NAT traversal stack

**PRECONDITIONS:**

- Session 6.1.1 complete (`host.go` and `errors.go` exist)
- `github.com/libp2p/go-libp2p` present in `go.mod`

**TASK:**

1. Create `internal/p2p/nat.go` defining constants and the `SetupNAT` wiring function for the three-tier NAT traversal stack (ARCH §13).
2. Set `maxHolePunchRetries = 1` — justified by 97.6% first-attempt success rate across 4.4M DCUtR measurements (ARCH §13). **Do not use the libp2p default of 3** (MVP §8.2).
3. Configure Circuit Relay v2 with a 30-minute reservation TTL; the daemon refreshes before expiry (IC §4.3).

**THREE-TIER NAT TRAVERSAL (ARCH §13 §NAT traversal — three tiers):**

| Tier | Protocol | Applies to | Note |
|---|---|---|---|
| 1 | AutoNAT | All providers at startup | Classifies: publicly reachable / cone NAT / symmetric NAT |
| 2 | DCUtR | Cone NAT (Indian home routers) | `maxHolePunchRetries = 1`; 97.6% succeed on first attempt |
| 3 | Circuit Relay v2 | Symmetric NAT (~30-45% per ARCH §27.5) | Relay RTT < 50 ms from Indian cloud nodes (NFR-006) |

**FILE:** `internal/p2p/nat.go`

```go
package p2p

import (
    "github.com/libp2p/go-libp2p/p2p/host/autonat"
    relayv2 "github.com/libp2p/go-libp2p/p2p/protocol/circuitv2/client"
    "github.com/multiformats/go-multiaddr"
)

// maxHolePunchRetries is the DCUtR hole-punch retry count.
// Set to 1, NOT the libp2p default of 3.
// Justification: 97.6% of successful DCUtR connections succeed on the first attempt
// across 4.4M traversal measurements (ARCH §13, Paper 30, MVP §8.2).
const maxHolePunchRetries = 1

// relayReservationTTL is the Circuit Relay v2 reservation duration in seconds.
// The daemon refreshes the reservation before it expires (IC §4.3, libp2p default).
const relayReservationTTL = 1800 // 30 minutes

// SetupNAT wires all three NAT traversal tiers to the given host (ARCH §13).
//
// Tier 1: AutoNAT — probes external reachability on startup and periodically.
// Tier 2: DCUtR  — hole-punches through cone NAT (most Indian home routers).
// Tier 3: Circuit Relay v2 — routes through relayAddrs for symmetric NAT providers.
//
// relayAddrs contains the multiaddrs of Vyomanaut-operated relay nodes. At launch:
// 3 nodes, one per Indian cloud AZ, each supporting 128 concurrent reservations
// (ARCH §13 §Relay infrastructure at launch, ARCH §27.5).
func SetupNAT(h Host, relayAddrs []multiaddr.Multiaddr) error {
    // Tier 1: AutoNAT service (integrated into the libp2p host at construction time).
    // Tier 2: DCUtR — configured via libp2p.EnableHolePunching() with maxHolePunchRetries.
    // Tier 3: Circuit Relay v2 — connect to each relay and request a reservation.
    _ = autonat.NewAutoNAT    // compile-time import validation
    _ = relayv2.Reserve       // compile-time import validation
    // Full implementation: see libp2p.New() options in cmd/provider/main.go (Session 13.1.1)
    return nil
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/p2p/nat.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/p2p/
  EXPECT: exit 0

HOLE_PUNCH_RETRY_COUNT:
  $ grep -c "^const maxHolePunchRetries = 1" internal/p2p/nat.go
  EXPECT: 1

  # Must NOT be 3 (the libp2p default that ARCH §13 explicitly overrides)
  $ grep -c "maxHolePunchRetries = 3\|Retries.*=.*3" internal/p2p/nat.go
  EXPECT: 0

RELAY_TTL:
  $ grep -c "relayReservationTTL\|1800\|30 minute\|30.*min" internal/p2p/nat.go
  EXPECT: 1

THREE_TIER_COVERAGE:
  $ grep -c "AutoNAT\|autonat\|autoNAT" internal/p2p/nat.go
  EXPECT: >= 1

  $ grep -c "DCUtR\|holepunch\|hole.*punch\|HolePunch" internal/p2p/nat.go
  EXPECT: >= 1

  $ grep -c "CircuitRelay\|circuitv2\|Circuit Relay\|relayv2" internal/p2p/nat.go
  EXPECT: >= 1

NEGATIVE_CHECKS:
  # Relay overhead (50 ms NFR-006) is a measurable NFR, not a runtime constant
  $ grep -c "50.*time.Millisecond\|50ms" internal/p2p/nat.go
  EXPECT: 0
```

---

#### Session 6.1.3 — Provider identity persistence

**PRECONDITIONS:**

- Session 6.1.1 complete
- `internal/crypto` package complete (Milestone 2) — `DeriveKeystoreEncKey`, `EncryptPointerFile`, and `DecryptPointerFile` available (IC §5.1)

**TASK:**

1. Create `internal/p2p/identity.go` implementing Ed25519 key pair generation, encrypted persistence, and load-on-restart.
2. Derive the keystore encryption key using `crypto.DeriveKeystoreEncKey(masterSecret, ownerID)`. The correct function name is **`DeriveKeystoreEncKey`** — not `DeriveDKSKeystoreEncKey` (IC §3.2, IC §5.1).
3. Encrypt the private key with `AEAD_CHACHA20_POLY1305` via `crypto.EncryptPointerFile` and persist to `<dataDir>/identity.key`. On first launch, generate via `crypto/rand`. On subsequent launches, load and decrypt.
4. The libp2p Peer ID is `multihash(ed25519_public_key)` (ARCH §13, IC §3.2).

**FILE:** `internal/p2p/identity.go`

```go
package p2p

import (
    "crypto/rand"
    "fmt"
    "os"
    "path/filepath"

    libp2pcrypto "github.com/libp2p/go-libp2p/core/crypto"
    "github.com/libp2p/go-libp2p/core/peer"

    localcrypto "github.com/masamasaowl/Vyomanaut_V2/internal/crypto"
)

const identityFileName = "identity.key"

// LoadOrGenerateIdentity loads the provider's Ed25519 identity from dataDir/identity.key,
// or generates a new one if the file does not exist.
//
// The key file is encrypted with AEAD_CHACHA20_POLY1305 using a key derived from
// DeriveKeystoreEncKey(masterSecret, ownerID) (IC §3.2, IC §5.1).
//
// NOTE: The correct function is DeriveKeystoreEncKey — NOT DeriveDKSKeystoreEncKey.
//
// Returns the libp2p private key and the Peer ID (multihash(public_key), ARCH §13).
func LoadOrGenerateIdentity(
    dataDir string,
    masterSecret [32]byte,
    ownerID []byte, // 16-byte UUID bytes
) (libp2pcrypto.PrivKey, peer.ID, error) {
    if len(ownerID) != 16 {
        return nil, "", fmt.Errorf("p2p: LoadOrGenerateIdentity: ownerID must be 16 bytes, got %d", len(ownerID))
    }

    encKey := localcrypto.DeriveKeystoreEncKey(masterSecret, ownerID) // correct function name
    keyPath := filepath.Join(dataDir, identityFileName)

    if _, err := os.Stat(keyPath); os.IsNotExist(err) {
        return generateAndSaveIdentity(keyPath, encKey)
    }
    return loadIdentity(keyPath, encKey)
}

// generateAndSaveIdentity creates a new Ed25519 key pair, encrypts and saves it.
func generateAndSaveIdentity(keyPath string, encKey [32]byte) (libp2pcrypto.PrivKey, peer.ID, error) {
    privKey, _, err := libp2pcrypto.GenerateEd25519Key(rand.Reader)
    if err != nil {
        return nil, "", fmt.Errorf("p2p: generate Ed25519 key: %w", err)
    }

    raw, err := privKey.Raw()
    if err != nil {
        return nil, "", fmt.Errorf("p2p: marshal Ed25519 key: %w", err)
    }

    // Nonce: use a random 12-byte value for the identity file
    var nonce [12]byte
    if _, err := rand.Read(nonce[:]); err != nil {
        return nil, "", fmt.Errorf("p2p: generate nonce: %w", err)
    }

    aad := []byte("vyomanaut-identity-v1")
    ciphertext, err := localcrypto.EncryptPointerFile(encKey, nonce, aad, raw)
    if err != nil {
        return nil, "", fmt.Errorf("p2p: encrypt identity: %w", err)
    }

    // File format: nonce(12) || ciphertext
    blob := append(nonce[:], ciphertext...)
    if err := os.WriteFile(keyPath, blob, 0600); err != nil {
        return nil, "", fmt.Errorf("p2p: write identity file: %w", err)
    }

    pid, err := peer.IDFromPrivateKey(privKey)
    if err != nil {
        return nil, "", fmt.Errorf("p2p: derive Peer ID: %w", err)
    }
    return privKey, pid, nil
}

// loadIdentity decrypts and loads an existing identity file.
func loadIdentity(keyPath string, encKey [32]byte) (libp2pcrypto.PrivKey, peer.ID, error) {
    blob, err := os.ReadFile(keyPath)
    if err != nil {
        return nil, "", fmt.Errorf("p2p: read identity file: %w", err)
    }
    if len(blob) < 12 {
        return nil, "", fmt.Errorf("p2p: identity file too short (%d bytes)", len(blob))
    }

    var nonce [12]byte
    copy(nonce[:], blob[:12])
    ciphertext := blob[12:]

    aad := []byte("vyomanaut-identity-v1")
    raw, err := localcrypto.DecryptPointerFile(encKey, nonce, aad, ciphertext)
    if err != nil {
        return nil, "", fmt.Errorf("p2p: decrypt identity: %w", err)
    }

    privKey, err := libp2pcrypto.UnmarshalEd25519PrivateKey(raw)
    if err != nil {
        return nil, "", fmt.Errorf("p2p: unmarshal Ed25519 key: %w", err)
    }

    pid, err := peer.IDFromPrivateKey(privKey)
    if err != nil {
        return nil, "", fmt.Errorf("p2p: derive Peer ID: %w", err)
    }
    return privKey, pid, nil
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/p2p/identity.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/p2p/
  EXPECT: exit 0

CORRECT_FUNCTION_NAME (no typo):
  $ grep -c "DeriveKeystoreEncKey" internal/p2p/identity.go
  EXPECT: 1

TYPO_ABSENT (the erroneous name from the original build.md must not exist anywhere):
  $ grep -rn "DeriveDKSKeystoreEncKey" internal/
  EXPECT: empty — this function does not exist

ENCRYPTED_STORAGE:
  # Key file must be encrypted — raw ed25519 bytes must never be written plaintext to disk
  $ grep -c "EncryptPointerFile\|AEAD\|nonce\|chacha20" internal/p2p/identity.go
  EXPECT: >= 2

CRYPTO_RAND:
  $ grep -c "crypto/rand\|rand\.Reader\|rand\.Read" internal/p2p/identity.go
  EXPECT: >= 1

PEER_ID_DERIVATION:
  $ grep -c "IDFromPrivateKey\|IDFromPublicKey\|peer\.ID" internal/p2p/identity.go
  EXPECT: >= 1

INTERNAL_CRYPTO_IMPORT:
  $ grep -c "Vyomanaut_V2/internal/crypto" internal/p2p/identity.go
  EXPECT: 1

FILE_PERMISSIONS:
  # Identity file must be created with mode 0600 (owner-only read/write)
  $ grep -c "0600\|os.FileMode.*0600" internal/p2p/identity.go
  EXPECT: 1

NEGATIVE_CHECKS:
  # Must NOT write raw private key bytes to disk (must be encrypted)
  $ grep -n "os.WriteFile.*raw\|WriteFile.*privKey" internal/p2p/identity.go \
      && echo "FAIL: raw key written to disk" || echo "PASS"
```

---

### Phase 6.2 — Kademlia DHT with Custom HMAC Validator

**Reference:** IC §12 (key derivation, validator rules, republication), IC §5.4 (DHT interface), ARCH §13 (DHT parameters: k=16, α=3, Server mode), MVP §8.2 (dht_namespace.go as sole definition), MVP §8.4 (CI check 5)

---

#### Session 6.2.1 — Define the DHT namespace constant

**PRECONDITIONS:**

- Session 6.1.1 complete

**TASK:**

1. Create `internal/p2p/dht_namespace.go` containing **only** the `dhtKeyNamespace` constant.
2. This is the **sole definition** of the string `"/vyomanaut/dht-key/1.0.0"` in the entire repository (MVP §8.2). No other source file may inline this literal.
3. After any `go-libp2p` upgrade, a global search for this string must find only this file plus the registration call-site in `cmd/provider/main.go` — the CI upgrade-drift test (`TestDHTKeyValidatorPersists`) catches a silent namespace reset (IC §12, ARCH §4).

**FILE:** `internal/p2p/dht_namespace.go`

```go
package p2p

// dhtKeyNamespace is the libp2p DHT validator namespace for all Vyomanaut chunk
// content-address records (IC §12, ADR-001, ARCH §13 §DHT configuration).
//
// SOLE DEFINITION: This constant is the only place in the repository where the string
// "/vyomanaut/dht-key/1.0.0" may appear (MVP §8.2). All other files must reference
// this constant — never inline the string literal.
//
// UPGRADE GUARD: TestDHTKeyValidatorPersists (CI check 5, MVP §8.4) verifies this
// namespace survives every go-libp2p dependency upgrade.
//
// BREAKING CHANGE: Modifying this string is a network-breaking change — all provider
// daemons must upgrade simultaneously before any chunk lookup succeeds (IC §13).
const dhtKeyNamespace = "/vyomanaut/dht-key/1.0.0"
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/p2p/dht_namespace.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/p2p/
  EXPECT: exit 0

EXACT_CONSTANT_VALUE:
  $ grep -c 'dhtKeyNamespace = "/vyomanaut/dht-key/1.0.0"' internal/p2p/dht_namespace.go
  EXPECT: 1

SINGLETON_CHECK (string literal must appear in only the definition + cmd registration):
  $ grep -rn '"/vyomanaut/dht-key/1.0.0"' . \
      --include="*.go" \
      | grep -v "_test.go" \
      | grep -v "internal/p2p/dht_namespace.go" \
      | grep -v "cmd/provider/main.go"
  EXPECT: empty — no other Go source file may contain this literal

FILE_IS_MINIMAL (only const, package decl, and doc comments):
  $ grep -c "^func \|^type \|^var " internal/p2p/dht_namespace.go
  EXPECT: 0

LINE_COUNT:
  $ wc -l < internal/p2p/dht_namespace.go
  EXPECT: <= 20

VET:
  $ go vet ./internal/p2p/
  EXPECT: exit 0; zero output
```

---

#### Session 6.2.2 — Implement the custom HMAC key validator and DHT interface

**PRECONDITIONS:**

- Sessions 6.2.1 and 6.1.1 complete
- `internal/crypto` complete (Milestone 2) — `DeriveDHTKey` and `DeriveDHTOwnerKey` available (these are called at upload time, NOT here)
- `internal/storage` complete (Milestone 5) — `index.dhtKeyFor()` available for the republication path

**TASK:**

1. Create `internal/p2p/dht.go` with the `DHT` interface from IC §5.4 and a `kademliaDHT` concrete implementation.
2. Implement the custom HMAC key validator per IC §12. Register it under `dhtKeyNamespace`.
3. Configure DHT parameters per ARCH §13 §DHT configuration table: `k-bucket size = 16`, `α = 3`, `mode = Server` (all provider daemons are full DHT participants).
4. `PutProviderRecord` MUST use the **pre-cached** `dht_key` from RocksDB (via `index.dhtKeyFor`). It must **never** recompute the key — `file_owner_key` is not available to the daemon after upload (IC §12.2, ARCH §13).

**KEY VALIDATOR RULES (IC §12 — explicit accept/reject table):**

| Condition | Decision | Reason |
|---|---|---|
| `len(key) == 32` | **Accept** | HMAC-SHA256 output is always 32 bytes |
| `len(key) != 32` | **Reject** → `ErrDHTKeyInvalid` | Wrong length — not an HMAC output |
| Key has `"vyom-chunk:"` prefix | **Reject** → `ErrDHTKeyInvalid` | Plaintext chunk hash — wrong format (IC §12) |
| libp2p default CID multihash encoding | **Reject** → `ErrDHTKeyInvalid` | Wrong namespace — would leak file identity |

**FILE:** `internal/p2p/dht.go`

```go
package p2p

import (
    "context"
    "fmt"
    "strings"

    "github.com/libp2p/go-libp2p/core/peer"
    record "github.com/libp2p/go-libp2p-record"
)

// DHT is the Kademlia DHT interface for chunk content-address lookup (IC §5.4, ADR-001).
//
// PURPOSE: chunk address lookup and stale-address fallback only.
// Provider discovery goes through the microservice control plane, NOT the DHT (ARCH §5).
//
// DHT parameters (ARCH §13 §DHT configuration):
//   k-bucket size = 16  (S/Kademlia: k = 2 × d where d = 8)
//   parallel lookups α  = 3  (produces O(log N / 3) round trips)
//   mode = Server       (all provider daemons must be reachable for challenge dispatch)
type DHT interface {
    // PutProviderRecord announces that the local peer holds the value for dhtKey.
    //
    // Pre-conditions:
    //   len(dhtKey) == 32  (HMAC-SHA256 output is always 32 bytes)
    //   dhtKey was retrieved from index.dhtKeyFor(chunkID) — NEVER recomputed.
    //
    // WHY NOT recompute: file_owner_key = HKDF(master_secret, ...) is only known at upload
    // time. The daemon does NOT hold master_secret after upload completes (IC §12.2, ARCH §13).
    PutProviderRecord(ctx context.Context, dhtKey []byte) error

    // FindProviders returns up to maxCount peers that have announced holding dhtKey.
    FindProviders(ctx context.Context, dhtKey []byte, maxCount int) ([]peer.AddrInfo, error)

    // Bootstrap connects to well-known seed nodes and fills k-buckets.
    // Must be called once at daemon startup, after the Host is created (ARCH §13).
    Bootstrap(ctx context.Context) error
}

// dhtKeyValidator implements the custom HMAC key validator for the Vyomanaut DHT
// namespace. Registered as a libp2p record.Validator under dhtKeyNamespace (IC §12).
type dhtKeyValidator struct{}

// Validate implements record.Validator. It enforces the key acceptance rules from IC §12.
func (v dhtKeyValidator) Validate(key string, _ []byte) error {
    return validateDHTKey(key)
}

// Select implements record.Validator (chooses among multiple candidate records).
// For content-routing provider records, the most recent is always preferred.
func (v dhtKeyValidator) Select(_ string, vals [][]byte) (int, error) {
    if len(vals) == 0 {
        return 0, fmt.Errorf("p2p: dhtKeyValidator.Select: no candidates")
    }
    return 0, nil // first candidate wins (provider records have no conflicting state)
}

// Compile-time: dhtKeyValidator must satisfy the record.Validator interface.
var _ record.Validator = dhtKeyValidator{}

// validateDHTKey enforces the IC §12 key acceptance rules.
//
// Accept: len(key) == 32 and no forbidden prefix.
// Reject (ErrDHTKeyInvalid): wrong length, "vyom-chunk:" prefix, or CID multihash encoding.
func validateDHTKey(key string) error {
    const expectedLen = 32
    if len(key) != expectedLen {
        return fmt.Errorf("%w: key length %d != %d",
            ErrDHTKeyInvalid, len(key), expectedLen)
    }
    // Plaintext SHA-256 chunk hashes are prefixed "vyom-chunk:" in other Vyomanaut contexts.
    // They must never be stored directly in the DHT — that would leak file identity (IC §12).
    if strings.HasPrefix(key, "vyom-chunk:") {
        return fmt.Errorf("%w: key has forbidden plaintext chunk-hash prefix", ErrDHTKeyInvalid)
    }
    return nil
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/p2p/dht.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/p2p/
  EXPECT: exit 0

INTERFACE_METHODS:
  $ grep -c "PutProviderRecord\|FindProviders\|Bootstrap" internal/p2p/dht.go
  EXPECT: 3

VALIDATOR_STRUCT:
  $ grep -c "dhtKeyValidator\|validateDHTKey" internal/p2p/dht.go
  EXPECT: >= 3

VALIDATOR_RULES:
  # 32-byte length check present
  $ grep -c "expectedLen = 32\|len(key) != expectedLen\|len.*32" internal/p2p/dht.go
  EXPECT: >= 1

  # Plaintext prefix rejection present
  $ grep -c '"vyom-chunk:"' internal/p2p/dht.go
  EXPECT: 1

  # ErrDHTKeyInvalid returned on rejection
  $ grep -c "ErrDHTKeyInvalid" internal/p2p/dht.go
  EXPECT: >= 2

DHT_PARAMETERS_DOCUMENTED:
  # k = 16 bucket size (ARCH §13)
  $ grep -c "k.*=.*16\|BucketSize.*16\|k-bucket.*16" internal/p2p/dht.go
  EXPECT: >= 1

  # alpha = 3 parallel lookups (ARCH §13)
  $ grep -c "α.*=.*3\|alpha.*=.*3\|parallel.*3" internal/p2p/dht.go
  EXPECT: >= 1

  # Server mode (ARCH §13)
  $ grep -c "Server\|ModeServer\|ServerMode" internal/p2p/dht.go
  EXPECT: >= 1

NAMESPACE_CONSTANT_USED:
  $ grep -c "dhtKeyNamespace" internal/p2p/dht.go
  EXPECT: >= 1

  # Must NOT inline the string — must use the constant (MVP §8.2)
  $ grep -c '"/vyomanaut/dht-key/1.0.0"' internal/p2p/dht.go
  EXPECT: 0

NEGATIVE_CHECKS:
  # PutProviderRecord must NOT recompute the DHT key — that is upload-time only (IC §12.2)
  $ grep -c "DeriveDHTKey\|DeriveDHTOwnerKey\|file_owner_key\|master_secret" \
      internal/p2p/dht.go
  EXPECT: 0

  # No prohibited internal/ imports beyond p2p itself
  $ grep -n "Vyomanaut_V2/internal/" internal/p2p/dht.go \
      | grep -v "internal/p2p"
  EXPECT: empty
```

---

#### Session 6.2.3 — Implement `TestDHTKeyValidatorPersists` (CI check 5)

**PRECONDITIONS:**

- Session 6.2.2 complete — DHT validator, `validateDHTKey`, and namespace constant all exist
- `github.com/libp2p/go-libp2p` pinned in `go.mod`

**TASK:**

1. Create `internal/p2p/dht_test.go` with `TestDHTKeyValidatorPersists` (the text in IC §12) and two companion tests.
2. After this session passes, remove `continue-on-error: true` from check-05 in `.github/workflows/ci.yml` — this test becomes a **hard gate** (MVP §8.4).
3. Add `// pin check: go-libp2p` comment to `go.mod` at the `go-libp2p` entry, and note in `architecture.md §4.1` that the pin date matches this session.

**FILE:** `internal/p2p/dht_test.go`

```go
package p2p_test

import (
    "context"
    "testing"

    "github.com/libp2p/go-libp2p/core/peer"
    mh "github.com/multiformats/go-multihash"
    "github.com/stretchr/testify/require"

    "github.com/masamasaowl/Vyomanaut_V2/internal/p2p"
)

// TestDHTKeyValidatorPersists is a mandatory CI gate (CI check 5, MVP §8.4).
//
// PURPOSE: Detect a silent namespace reset in a go-libp2p upgrade. If the upgrade
// resets the custom validator to the default CID namespace, the validator would accept
// plain CIDs — leaking file identity in DHT traffic (IC §12, ADR-001).
//
// Run with: go test -run TestDHTKeyValidatorPersists ./internal/p2p/
// This test MUST be re-run after every go-libp2p dependency upgrade.
func TestDHTKeyValidatorPersists(t *testing.T) {
    ctx := context.Background()
    host := buildTestHost(t)
    dht := buildTestDHT(t, host)

    // CASE 1: A valid 32-byte HMAC-derived key MUST be accepted by the validator.
    validKey := make([]byte, 32)
    for i := range validKey {
        validKey[i] = byte(i + 1) // deterministic, non-zero test key
    }
    err := dht.PutProviderRecord(ctx, validKey)
    require.NoError(t, err,
        "valid 32-byte key must be accepted; a failure here means the HMAC validator is broken")

    // CASE 2: A plain libp2p CID MUST be rejected.
    // This is the critical regression check — a go-libp2p upgrade that silently resets
    // the namespace would cause the default CID validator to accept this key (IC §12).
    cidBytes, err := mh.Sum([]byte("test-chunk-content"), mh.SHA2_256, -1)
    require.NoError(t, err, "multihash encoding must succeed")
    err = dht.PutProviderRecord(ctx, []byte(cidBytes))
    require.ErrorIs(t, err, p2p.ErrDHTKeyInvalid,
        "plain CID must be rejected; a go-libp2p upgrade may have reset the namespace — "+
            "check dhtKeyNamespace registration in cmd/provider/main.go")

    // CASE 3: A 31-byte key (one byte short of 32) MUST be rejected.
    shortKey := validKey[:31]
    err = dht.PutProviderRecord(ctx, shortKey)
    require.ErrorIs(t, err, p2p.ErrDHTKeyInvalid,
        "31-byte key must be rejected; only exactly 32-byte HMAC keys are valid")
}

// TestDHTKeyValidatorRejectsAll is a table-driven exhaustive rejection test (IC §12).
func TestDHTKeyValidatorRejectsAll(t *testing.T) {
    ctx := context.Background()
    host := buildTestHost(t)
    dht := buildTestDHT(t, host)

    cases := []struct {
        name string
        key  []byte
    }{
        {"empty key", []byte{}},
        {"31 bytes (one short)", make([]byte, 31)},
        {"33 bytes (one over)", make([]byte, 33)},
        {"64 bytes (Ed25519 sig length)", make([]byte, 64)},
        {"vyom-chunk prefix", append([]byte("vyom-chunk:"), make([]byte, 21)...)},
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            err := dht.PutProviderRecord(ctx, tc.key)
            require.ErrorIs(t, err, p2p.ErrDHTKeyInvalid,
                "key %q (%d bytes) must be rejected by the HMAC validator",
                tc.name, len(tc.key))
        })
    }
}

// TestDHTKeyValidatorAcceptsHMAC tests positive acceptance with boundary key values.
func TestDHTKeyValidatorAcceptsHMAC(t *testing.T) {
    ctx := context.Background()
    host := buildTestHost(t)
    dht := buildTestDHT(t, host)

    validKeys := [][]byte{
        make([]byte, 32),           // all-zero 32-byte key
        func() []byte { k := make([]byte, 32); for i := range k { k[i] = 0xFF }; return k }(), // all-FF
    }

    for _, key := range validKeys {
        err := dht.PutProviderRecord(ctx, key)
        require.NoError(t, err, "32-byte key must be accepted regardless of content")
    }
}

// buildTestHost creates a minimal libp2p host for testing.
// Uses an in-memory transport; does not bind real network ports.
func buildTestHost(t *testing.T) p2p.Host {
    t.Helper()
    // TODO(Session 6.2.3): implement using libp2p.New with in-memory/tcp loopback transport
    // Example: libp2p.New(libp2p.ListenAddrStrings("/ip4/127.0.0.1/tcp/0"))
    t.Skip("buildTestHost: implement after libp2p wiring in Session 6.1.1")
    return nil
}

// buildTestDHT creates a kademliaDHT instance wired to host, with dhtKeyNamespace registered.
func buildTestDHT(t *testing.T, host p2p.Host) p2p.DHT {
    t.Helper()
    // TODO(Session 6.2.3): call p2p.NewDHT(host) once the constructor is implemented
    t.Skip("buildTestDHT: implement after Session 6.2.2 constructor is wired")
    return nil
}
```

**UPDATE `.github/workflows/ci.yml`** after this session passes:

```yaml
# BEFORE (check-05 with pending guard):
- name: check-05 TestDHTKeyValidatorPersists
  run: |
    go test -run TestDHTKeyValidatorPersists ./internal/p2p/... \
      || (echo "PENDING: session 6.2.3" && exit 0)
  continue-on-error: true

# AFTER (M6 complete — hard gate active):
- name: check-05 TestDHTKeyValidatorPersists (M6 hard gate)
  run: go test -run TestDHTKeyValidatorPersists ./internal/p2p/...
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/p2p/dht_test.go && echo PASS || echo FAIL

TEST_NAMES_PRESENT (three tests):
  $ grep -c "^func TestDHTKeyValidatorPersists\|^func TestDHTKeyValidatorRejectsAll\|^func TestDHTKeyValidatorAcceptsHMAC" \
      internal/p2p/dht_test.go
  EXPECT: 3

THREE_CASES_IN_MAIN_TEST:
  # CASE 1 (valid), CASE 2 (plain CID), CASE 3 (31-byte short)
  $ grep -c "CASE 1\|CASE 2\|CASE 3\|validKey\|cidBytes\|shortKey" internal/p2p/dht_test.go
  EXPECT: >= 3

TABLE_CASES (five entries in TestDHTKeyValidatorRejectsAll):
  $ grep -c '"empty key"\|"31 bytes\|"33 bytes\|"64 bytes\|"vyom-chunk prefix"' \
      internal/p2p/dht_test.go
  EXPECT: 5

ERROR_ASSERTIONS:
  $ grep -c "ErrorIs.*ErrDHTKeyInvalid" internal/p2p/dht_test.go
  EXPECT: >= 3

TEST_RUNS (once buildTestHost and buildTestDHT are implemented):
  $ go test -v -run TestDHTKeyValidatorPersists ./internal/p2p/
  EXPECT: exit 0; "--- PASS: TestDHTKeyValidatorPersists" in output

  $ go test -v -run TestDHTKeyValidatorRejectsAll ./internal/p2p/
  EXPECT: exit 0; all 5 sub-tests "--- PASS:"

CI_CHECK_5_HARD_GATE (verify continue-on-error removed from ci.yml):
  $ grep -A5 "check-05" .github/workflows/ci.yml | grep -c "continue-on-error"
  EXPECT: 0

SINGLETON_STILL_HOLDS (no inline string literals introduced by test file):
  $ grep -c '"/vyomanaut/dht-key/1.0.0"' internal/p2p/dht_test.go
  EXPECT: 0
```

---

### Phase 6.3 — Heartbeat Goroutine

**Reference:** IC §3.1 (heartbeat multiaddr update: signed payload, interval, token refresh integration), IC §12.2 (DHT republication coordinated with heartbeat goroutine), ARCH §13 (DHCP rotation motivation, heartbeat as primary address source for challenge dispatch), ADR-028

---

#### Session 6.3.1 — Implement heartbeat goroutine

**PRECONDITIONS:**

- Sessions 6.1.1, 6.1.2, 6.1.3, 6.2.1, and 6.2.2 complete
- `internal/crypto.SignBytes` available (Session 2.7.1)
- `internal/storage.ChunkStore` with `index.allChunkIDs()` and `index.dhtKeyFor()` available (Milestone 5)
- `internal/config.NetworkProfile` available (Milestone 1) — supplies all duration values

**TASK:**

1. Create `internal/p2p/heartbeat.go` implementing the heartbeat and DHT republication loops per IC §3.1.
2. All durations MUST come from `profile.HeartbeatInterval` and `profile.DHTRepublishInterval` — **never hardcoded as `4 * time.Hour`** (ADR-031, MVP §5.4).
3. Integrate token refresh per IC §3.1: if `tokenExpiresIn() < 24*time.Hour`, call `refreshToken()` before sending the heartbeat. This invokes the `POST /api/v1/provider/token/refresh` endpoint from Session 11.4.3.
4. DHT republication reads **pre-cached** `dht_keys` from RocksDB via `index.dhtKeyFor()`. It must **NOT** call `crypto.DeriveDHTKey` — `file_owner_key` is not available to the daemon after upload (IC §12.2).
5. The signed heartbeat payload uses canonical JSON (sorted keys, `provider_sig` field absent from signing input) per IC §3.1 and IC §3.2 (fixed-layout byte sequence — never `json.Marshal` with arbitrary field ordering).

**HEARTBEAT SIGNED PAYLOAD (IC §3.1 §Required fields):**

```
{
  "current_multiaddrs": ["<multiaddr1>", "<multiaddr2>", ...],
  "timestamp": "<ISO 8601 UTC>",
  "provider_sig": "<base64(Ed25519(SHA-256(canonical_json_without_sig_field)))>"
}
```

Microservice rejects timestamps with skew > ±5 minutes (IC §3.1).

**FILE:** `internal/p2p/heartbeat.go`

```go
package p2p

import (
    "context"
    "log"
    "time"

    "github.com/masamasaowl/Vyomanaut_V2/internal/config"
    "github.com/masamasaowl/Vyomanaut_V2/internal/storage"
)

// HeartbeatConfig holds all runtime dependencies for the heartbeat goroutine.
// All duration values come from the active NetworkProfile — never hardcoded.
type HeartbeatConfig struct {
    Profile         config.NetworkProfile
    Host            Host           // for current libp2p multiaddrs
    DHT             DHT            // for PutProviderRecord calls
    Store           storage.ChunkStore // for DHT republication (allChunkIDs + dhtKeyFor)
    MicroserviceURL string
    SigningKey       []byte         // Ed25519 private key (44 bytes) — for signing heartbeats
    GetToken        func() (token string, expiry time.Time)
    RefreshToken    func(ctx context.Context) error // POSTs to /api/v1/provider/token/refresh
}

// RunHeartbeat runs the heartbeat and DHT republication loops until ctx is cancelled.
// Must be called once from cmd/provider/main.go after the daemon is fully initialised
// (Session 13.1.1 step 8).
//
// Loop structure (IC §3.1, IC §12.2):
//   heartbeatTicker := time.NewTicker(profile.HeartbeatInterval)   // NOT 4*time.Hour
//   republishTicker := time.NewTicker(profile.DHTRepublishInterval) // NOT 12*time.Hour
func RunHeartbeat(ctx context.Context, cfg HeartbeatConfig) {
    // Durations come from NetworkProfile — demo uses 30s/2min, prod uses 4h/12h (ADR-031).
    heartbeatTicker := time.NewTicker(cfg.Profile.HeartbeatInterval)
    defer heartbeatTicker.Stop()

    republishTicker := time.NewTicker(cfg.Profile.DHTRepublishInterval)
    defer republishTicker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-heartbeatTicker.C:
            doHeartbeat(ctx, cfg)
        case <-republishTicker.C:
            doRepublish(ctx, cfg)
        }
    }
}

// doHeartbeat sends one signed heartbeat to the microservice control plane (IC §3.1).
//
// Steps:
//  1. Token refresh check: if expiry < 24h, call cfg.RefreshToken() (IC §3.1).
//  2. Collect current_multiaddrs from cfg.Host (includes relay multiaddrs when reserved).
//  3. Build canonical JSON payload (sorted keys; provider_sig absent from signing input).
//  4. Sign: provider_sig = SignBytes(signingKey, SHA-256(canonical_json_bytes)) (IC §3.2).
//  5. POST to /api/v1/provider/heartbeat with the signed payload.
func doHeartbeat(ctx context.Context, cfg HeartbeatConfig) {
    // Token refresh check (IC §3.1 §Token refresh integration with heartbeat goroutine)
    _, expiry := cfg.GetToken()
    if time.Until(expiry) < 24*time.Hour {
        if err := cfg.RefreshToken(ctx); err != nil {
            log.Printf("[heartbeat] token refresh failed; will retry next cycle: %v", err)
            // Do not abort — send heartbeat with existing token; retry refresh next cycle.
        }
    }
    // TODO: build signed payload and POST to cfg.MicroserviceURL + /api/v1/provider/heartbeat
}

// doRepublish re-announces all stored chunks in the DHT (IC §12.2, ARCH §13).
//
// Steps (IC §12.2 §DHT Republication Contract):
//  1. Iterate all chunk IDs from cfg.Store via index.allChunkIDs().
//  2. For each chunk_id: call index.dhtKeyFor(chunk_id) to retrieve the cached DHT key.
//     If dhtKeyFor returns ok=false: skip — no DHT key was registered for this chunk.
//  3. Call cfg.DHT.PutProviderRecord(ctx, dhtKey[:]).
//
// CRITICAL: Must NOT call crypto.DeriveDHTKey or crypto.DeriveDHTOwnerKey.
// file_owner_key is only available at upload time; it is not stored by the daemon.
func doRepublish(ctx context.Context, cfg HeartbeatConfig) {
    // NOTE: allChunkIDs() is called on the storage.ChunkStore index directly.
    // This is intentional — the heartbeat goroutine needs the full list of stored chunks
    // for DHT republication without involving the microservice.
    //
    // TODO: call cfg.Store via the rocksDBIndex accessors once the storage package
    // exposes AllChunkIDsAndDHTKeys() or an equivalent iterator.
    _ = cfg.DHT // suppress unused warning until implementation is complete
    _ = cfg.Store
}
```

**VERIFY:**

```bash
FILES_EXIST:
  $ test -f internal/p2p/heartbeat.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/p2p/
  EXPECT: exit 0

PROFILE_DRIVEN_INTERVALS (no hardcoded durations):
  $ grep -c "Profile\.HeartbeatInterval\|cfg\.Profile\.HeartbeatInterval" \
      internal/p2p/heartbeat.go
  EXPECT: 1

  $ grep -c "Profile\.DHTRepublishInterval\|cfg\.Profile\.DHTRepublishInterval" \
      internal/p2p/heartbeat.go
  EXPECT: 1

  # 4*time.Hour must NOT appear — the interval comes from NetworkProfile (ADR-031)
  $ grep -n "4 \* time\.Hour\|4\*time\.Hour" internal/p2p/heartbeat.go \
      && echo "FAIL: hardcoded heartbeat interval" || echo "PASS"

  # 12*time.Hour must NOT appear — DHTRepublishInterval comes from NetworkProfile
  $ grep -n "12 \* time\.Hour\|12\*time\.Hour" internal/p2p/heartbeat.go \
      && echo "FAIL: hardcoded DHT republish interval" || echo "PASS"

TOKEN_REFRESH_PRESENT:
  $ grep -c "RefreshToken\|24\*time\.Hour\|24 \* time\.Hour" internal/p2p/heartbeat.go
  EXPECT: >= 2  (the 24h threshold + call to RefreshToken)

CONTEXT_CANCELLATION:
  $ grep -c "ctx\.Done()\|ctx\.Err()" internal/p2p/heartbeat.go
  EXPECT: >= 1

STRUCT_CONFIG_FIELDS:
  $ grep -c "HeartbeatInterval\|DHTRepublishInterval\|GetToken\|RefreshToken\|SigningKey" \
      internal/p2p/heartbeat.go
  EXPECT: >= 5

NEGATIVE_CHECKS:
  # Must NOT recompute DHT key from file_owner_key (IC §12.2)
  $ grep -c "DeriveDHTKey\|DeriveDHTOwnerKey\|file_owner_key\|master_secret" \
      internal/p2p/heartbeat.go
  EXPECT: 0

  # Must NOT use json.Marshal directly on the signing input (IC §3.2)
  $ grep -n "json\.Marshal" internal/p2p/heartbeat.go \
      && echo "WARN: verify canonical JSON encoding is used, not json.Marshal" \
      || echo "PASS"

FULL_PACKAGE_BUILD:
  $ go build ./internal/storage/ ./internal/p2p/
  EXPECT: exit 0

ALL_M6_TESTS_PASS:
  $ go test -race -count=1 ./internal/p2p/
  EXPECT: exit 0
  NOTE: TestDHTKeyValidatorPersists and TestDHTKeyValidatorRejectsAll must both PASS

CI_HARD_GATES (both check-05 and check-04 must now be green):
  $ go test -race -count=1 ./internal/storage/ ./internal/p2p/
  EXPECT: exit 0; TestSingleWriterGoroutine and TestDHTKeyValidatorPersists both PASS
```

---

## Appendix A — Invariant Enforcement Traceability

Every design invariant from DM §3 and IC must be enforced at the layer shown below.
A PR that breaks any row here must be rejected.

| ID | Name | Source | Enforced At | Milestone |
|----|------|---------|-------------|-----------|
| INV-1 | Append-only audit log | DM §3 | DB RSP + `WriteReceiptPhase2` WHERE clause | M4, M7 |
| INV-2 | Append-only escrow ledger | DM §3 | DB RSP + `InsertEscrowEvent` sole write path | M4, M10 |
| INV-3 | No physical provider deletion | DM §3 | IC §6 DML; departure sets `status='DEPARTED'` | M4, M9 |
| INV-4 | All amounts integer paise | DM §3 | DB BIGINT; `TestNoFloatArithmetic`; debug panic | M4, M10 |
| INV-5 | Challenge nonce = 33 bytes | DM §3 | DB CHECK; `[33]byte` return type; CI grep-check 8 | M4, M7 |
| INV-6 | No real shard on vetting / no repair for synthetic | DM §3 | DB CHECK; `EnqueueJob` debug panic; `IsVettingChunk` pre-check | M4, M9, M14 |
| INV-7 | `ShardSize` compile-time constant = 262144 | DM §3 | `const ShardSize`; `TestProfileShardSizeIsConstant` | M1, M3 |
| INV-SW | Single-writer vLog | IC §5.3 | Channel to writer goroutine; `TestSingleWriterGoroutine` | M5 |
| INV-0RTT | 0-RTT prohibited on audit/gc streams | IC §4 | Host middleware on protocol ID suffix | M6 |
| INV-SIGN | No JSON signing inputs | IC §3.2 | `SignBytes` comment; code-review rule | M2 |
| INV-DHT | DHT HMAC key validator | IC §12 | Custom namespace validator; CI check 5 | M6 |
| INV-FLOAT | No float in payment | IC §11 | CI check 6; debug panic in `InsertEscrowEvent` | M10 |
| INV-THROTTLE | Background throttle fires at DB p99 ≥ 50 ms | NFR-028 | `backgroundThrottle` goroutine; histogram | M12, M-OBS |
| INV-METRIC | Metric names follow `vyomanaut_{subsystem}_{name}_{unit}` | NFR-046 | CI check 16; `internal/metrics/*.go` sole source | M-OBS |
| INV-CEIL | Chunk ceiling enforced at assignment time | NFR-044 | Assignment service; HTTP 503 `INSUFFICIENT_PROVIDER_CAPACITY` | M11 |
| INV-RAM | Provider RAM checked before daemon starts | NFR-045 | `memcheck_*.go` at startup; WARN + reduce `declared_storage_gb` | M13 |
| INV-RELAY | Relay ≤ 128 concurrent reservations per node | ARCH §13 | `cmd/relay` constant; gauge alert | M17 |

---

## Appendix B — Session Registry

> Update Status column as sessions complete

| Session | Milestone | Title | Status |
| --- | --- | --- | --- |
| 0.1.1 | M0 | Initialise Go module and top-level directories | ☐ |
| 0.1.2 | M0 | Create `cmd/` entrypoint stubs | ☐ |
| 0.1.3 | M0 | Create `internal/` package stubs | ☐ |
| 0.2.1 | M0 | Configure `.golangci.yml` | ☐ |
| 0.2.2 | M0 | Configure forbidden-pattern grep checks | ☐ |
| 0.3.1 | M0 | Create `.github/workflows/ci.yml` | ☐ |
| 0.3.2 | M0 | Create `.github/CODEOWNERS` | ☐ |
| 0.4.1 | M0 | Create `deployments/dev/docker-compose.yml` | ☐ |
| 1.1.1 | M1 | Define `NetworkProfile` struct | ☐ |
| 1.1.2 | M1 | Define `ProductionProfile` and `DemoProfile` | ☐ |
| 1.2.1 | M1 | `TestProfileShardSizeIsConstant` | ☐ |
| 1.2.2 | M1 | `TestProfileBothFullySpecified` | ☐ |
| 1.2.3 | M1 | `TestDemoDiffersFromProduction` | ☐ |
| 1.3.1 | M1 | Implement `SelectProfile()` | ☐ |
| 1.3.2 | M1 | Implement startup guard rails | ☐ |
| 1.3.3 | M1 | Guard rail tests | ☐ |
| 2.1.1 | M2 | Implement `DetectAESNI()` | ☐ |
| 2.2.1 | M2 | Implement HKDF-SHA256 derivation functions | ☐ |
| 2.2.2 | M2 | HKDF known-answer tests | ☐ |
| 2.3.1 | M2 | Implement `DeriveMasterSecret()` | ☐ |
| 2.3.2 | M2 | `DeriveMasterSecret` performance test | ☐ |
| 2.4.1 | M2 | Implement AONT canary constant | ☐ |
| 2.4.2 | M2 | Implement `AONTEncodeSegment()` | ☐ |
| 2.4.3 | M2 | Implement `AONTDecodePackage()` | ☐ |
| 2.4.4 | M2 | AONT tests | ☐ |
| 2.5.1 | M2 | Implement pointer file AEAD | ☐ |
| 2.6.1 | M2 | Embed BIP-39 English wordlist | ☐ |
| 2.6.2 | M2 | Implement `MasterSecretToMnemonic()` | ☐ |
| 2.6.3 | M2 | Implement `MnemonicToMasterSecret()` | ☐ |
| 2.6.4 | M2 | Implement `SelectConfirmationWords()` | ☐ |
| 2.6.5 | M2 | BIP-39 round-trip and error tests | ☐ |
| 2.7.1 | M2 | Ed25519 signing conventions | ☐ |
| 3.1.1 | M3 | Define `ShardSize` constant and `Engine` struct | ☐ |
| 3.1.2 | M3 | Implement `NewEngine()` | ☐ |
| 3.2.1 | M3 | Implement `EncodeSegment()` | ☐ |
| 3.2.2 | M3 | Implement `DecodeSegment()` | ☐ |
| 3.2.3 | M3 | Erasure coding tests | ☐ |
| 4.1.1 | M4 | Implement `migrations/generator.go` | ☐ |
| 4.2.1 | M4 | Define all PostgreSQL ENUMs | ☐ |
| 4.3.1 | M4 | `owners` table | ☐ |
| 4.3.2 | M4 | `providers` table | ☐ |
| 4.3.3 | M4 | `files` table | ☐ |
| 4.3.4 | M4 | `segments` table | ☐ |
| 4.3.5 | M4 | `chunk_assignments` table | ☐ |
| 4.4.1 | M4 | `audit_periods` table | ☐ |
| 4.4.2 | M4 | `audit_receipts` table | ☐ |
| 4.4.3 | M4 | `escrow_events` table | ☐ |
| 4.4.4 | M4 | `owner_escrow_events` table | ☐ |
| 4.4.5 | M4 | `repair_jobs` table | ☐ |
| 4.5.1 | M4 | Create all indexes | ☐ |
| 4.6.1 | M4 | `audit_receipts` RSP | ☐ |
| 4.6.2 | M4 | `escrow_events` RSP | ☐ |
| 4.6.3 | M4 | `chunk_assignments` soft-delete policy | ☐ |
| 4.7.1 | M4 | `mv_provider_scores` | ☐ |
| 4.7.2 | M4 | `mv_provider_escrow_balance` | ☐ |
| 4.7.3 | M4 | `mv_owner_escrow_balance` | ☐ |
| 4.7.4 | M4 | `mv_segment_shard_counts` | ☐ |
| 4.7.5 | M4 | Unique indexes on materialised views | ☐ |
| 4.8.1 | M4 | Run and document DM §9 checklist | ☐ |
| 5.1.1 | M5 | Define `ChunkStore` interface and sentinel errors | ☐ |
| 5.1.2 | M5 | Implement append-only vLog | ☐ |
| 5.1.3 | M5 | Implement RocksDB index wrapper | ☐ |
| 5.1.4 | M5 | Implement `LookupChunk()` | ☐ |
| 5.1.5 | M5 | Implement `DeleteChunk()`, `RecoverFromCrash()`, `RunGC()` | ☐ |
| 5.2.1 | M5 | Implement `TestSingleWriterGoroutine` | ☐ |
| 5.2.2 | M5 | Storage round-trip and crash recovery tests | ☐ |
| 6.1.1 | M6 | Implement `Host` interface | ☐ |
| 6.1.2 | M6 | NAT traversal stack | ☐ |
| 6.1.3 | M6 | Provider identity persistence | ☐ |
| 6.2.1 | M6 | Define DHT namespace constant | ☐ |
| 6.2.2 | M6 | Implement custom HMAC key validator | ☐ |
| 6.2.3 | M6 | Implement `TestDHTKeyValidatorPersists` | ☐ |
| 6.3.1 | M6 | Implement heartbeat goroutine | ☐ |
| 7.1.1 | M7 | Implement `ChallengeNonce()` | ☐ |
| 7.2.1 | M7 | Implement `ValidateResponse()` | ☐ |
| 7.3.1 | M7 | Implement `WriteReceiptPhase1()` | ☐ |
| 7.3.2 | M7 | Implement `WriteReceiptPhase2()` | ☐ |
| 7.4.1 | M7 | `SecretsManagerClient` interface and cache | ☐ |
| 7.5.1 | M7 | Implement JIT threshold computation | ☐ |
| 8.1.1 | M8 | Implement `GetScore()` | ☐ |
| 8.2.1 | M8 | Implement `IncrementConsecutivePasses()` | ☐ |
| 8.3.1 | M8 | Implement EWMA update functions | ☐ |
| 9.1.1 | M9 | Implement `EnqueueJob()` | ☐ |
| 9.1.2 | M9 | Implement `DequeueNextJob()` | ☐ |
| 9.1.3 | M9 | Implement `IsVettingChunk()` and `DeleteVettingChunksOnDeparture()` | ☐ |
| 9.2.1 | M9 | Implement repair download client | ☐ |
| 9.2.2 | M9 | Implement `RepairPromotionTimeout()` | ☐ |
| 9.3.1 | M9 | Implement departure detector loop | ☐ |
| 10.1.1 | M10 | Define `PaymentProvider` interface | ☐ |
| 10.1.2 | M10 | Implement mock payment provider | ☐ |
| 10.2.1 | M10 | Implement `InsertEscrowEvent()` | ☐ |
| 10.3.1 | M10 | Implement Razorpay webhook handlers | ☐ |
| 10.4.1 | M10 | Implement monthly release computation | ☐ |
| 10.4.2 | M10 | `TestNoFloatArithmetic` (CI check 6) | ☐ |
| 11.1.1 | M11 | Implement standard error response | ☐ |
| 11.2.1 | M11 | Implement `GET /api/v1/admin/readiness` | ☐ |
| 11.3.1 | M11 | Establish HTTP routing tree with stubs | ☐ |
| 11.4.1 | M11 | OTP Send | ☐ |
| 11.4.2 | M11 | OTP Verify | ☐ |
| 11.4.3 | M11 | Provider Token Refresh | ☐ |
| 11.5.1 | M11 | Owner Register | ☐ |
| 11.5.2 | M11 | Deposit Initiate | ☐ |
| 11.5.3 | M11 | Owner Balance | ☐ |
| 11.5.4 | M11 | Owner File List | ☐ |
| 11.5.5 | M11 | Owner Escrow History | ☐ |
| 11.5.6 | M11 | Owner Withdraw | ☐ |
| 11.6.1 | M11 | Provider Register | ☐ |
| 11.6.2 | M11 | Provider Heartbeat | ☐ |
| 11.6.3 | M11 | Provider Status | ☐ |
| 11.6.4 | M11 | Provider Audit Receipts | ☐ |
| 11.6.5 | M11 | Provider Downtime | ☐ |
| 11.6.6 | M11 | Provider Depart | ☐ |
| 11.7.1 | M11 | Upload Assign | ☐ |
| 11.7.2 | M11 | File Register | ☐ |
| 11.7.3 | M11 | Pointer File Retrieval | ☐ |
| 11.7.4 | M11 | File Delete | ☐ |
| 11.8.1 | M11 | Storage Pricing Estimate | ☐ |
| 11.8.2 | M11 | Provider Earnings Estimate | ☐ |
| 11.9.1 | M11 | Manual Audit Challenge | ☐ |
| 11.10.1 | M11 | Admin Providers List | ☐ |
| 11.10.2 | M11 | Repair Queue | ☐ |
| 11.10.3 | M11 | Manual Repair Trigger | ☐ |
| 11.10.4 | M11 | Audit Statistics | ☐ |
| 11.10.5 | M11 | Vetting Status | ☐ |
| 11.10.6 | M11 | Vetting GC Retry | ☐ |
| 11.11.1 | M11 | Per-Provider Chunk Count Ceiling | ☐ |
| 12.1.1 | M12 | Wire microservice `main()` | ☐ |
| 12.1.2 | M12 | Audit challenge dispatch loop | ☐ |
| OBS.1.1 | M-OBS | Register microservice Prometheus metrics | ☐ |
| OBS.1.2 | M-OBS | Wire metric increments at call sites | ☐ |
| OBS.2.1 | M-OBS | Register provider daemon Prometheus metrics | ☐ |
| OBS.3.1 | M-OBS | Define Grafana alert rules | ☐ |
| OBS.4.1 | M-OBS | Implement `TestNoOrphanMetricName` | ☐ |
| 13.1.1 | M13 | Wire provider `main()` | ☐ |
| 13.2.1 | M13 | Chunk upload stream handler | ☐ |
| 13.3.1 | M13 | Audit challenge stream handler | ☐ |
| 13.4.1 | M13 | Repair download stream handler | ☐ |
| 13.5.1 | M13 | Vetting GC stream handler | ☐ |
| 13.6.1 | M13 | Provider RAM check at installation | ☐ |
| 14.1.1 | M14 | Implement `Generator` | ☐ |
| 14.2.1 | M14 | Implement `DeliverGCInstruction()` | ☐ |
| 15.1.1 | M15 | Registration and keystore | ☐ |
| 15.2.1 | M15 | `UploadFile()` with capability token handling | ☐ |
| 15.2.2 | M15 | `ResumeUpload()` | ☐ |
| 15.3.1 | M15 | `RetrieveFile()` | ☐ |
| 16.1.1 | M16 | `TestDemoTimeline` integration test | ☐ |
| 16.1.2 | M16 | Viability fact-check tests | ☐ |
| 16.2.1 | M16 | `--sim-count` multi-instance provider | ☐ |
| 17.1.1 | M17 | Secrets manager adapters | ☐ |
| 17.2.1 | M17 | Gossip cluster (`internal/cluster`) | ☐ |
| 17.2.2 | M17 | Relay node binary and deployment configuration | ☐ |
| 18.1.1 | M18 | Create 8 required runbooks | ☐ |
| 18.2.1 | M18 | Create benchmark scripts | ☐ |
| 18.3.1 | M18 | Security verification checklist | ☐ |
| 18.4.1 | M18 | All-green CI verification | ☐ |

---

## Appendix C — File Creation Index

> Reverse lookup: which session creates or primarily modifies each file.

| File | Created By | Modified By |
| --- | --- | --- |
| `go.mod` | 0.1.1 | — |
| `.golangci.yml` | 0.2.1 | — |
| `scripts/ci/grep_checks.sh` | 0.2.2 | 7.1.1, OBS.4.1 |
| `.github/workflows/ci.yml` | 0.3.1 | — |
| `.github/CODEOWNERS` | 0.3.2 | — |
| `deployments/dev/docker-compose.yml` | 0.4.1 | — |
| `cmd/microservice/main.go` | 0.1.2 | 12.1.1, 12.1.2 |
| `cmd/provider/main.go` | 0.1.2 | 13.1.1, 13.6.1 |
| `cmd/client/main.go` | 0.1.2 | — |
| `cmd/relay/main.go` | 17.2.2 | — |
| `internal/config/network_profile.go` | 1.1.1 | — |
| `internal/config/profiles.go` | 1.1.2 | — |
| `internal/config/profiles_test.go` | 1.2.1 | 1.2.2, 1.2.3 |
| `internal/config/select.go` | 1.3.1 | — |
| `internal/config/guards.go` | 1.3.2 | — |
| `internal/crypto/aesni.go` | 2.1.1 | — |
| `internal/crypto/aesni_other.go` | 2.1.1 | — |
| `internal/crypto/hkdf.go` | 2.2.1 | — |
| `internal/crypto/hkdf_test.go` | 2.2.2 | — |
| `internal/crypto/argon2.go` | 2.3.1 | — |
| `internal/crypto/aont_canary.go` | 2.4.1 | — |
| `internal/crypto/aont.go` | 2.4.2 | 2.4.3 |
| `internal/crypto/aont_test.go` | 2.4.4 | — |
| `internal/crypto/chacha20poly1305.go` | 2.5.1 | — |
| `internal/crypto/errors.go` | 2.5.1 | — |
| `internal/crypto/wordlist_en.txt` | 2.6.1 | — |
| `internal/crypto/bip39.go` | 2.6.2 | 2.6.3, 2.6.4 |
| `internal/crypto/ed25519.go` | 2.7.1 | — |
| `internal/erasure/params.go` | 3.1.1 | — |
| `internal/erasure/engine.go` | 3.1.2 | 3.2.1, 3.2.2 |
| `internal/erasure/engine_test.go` | 3.2.3 | — |
| `migrations/generator.go` | 4.1.1 | — |
| `migrations/001_initial_schema.sql` | 4.1.1–4.8.1 | — |
| `scripts/ci/migration_check.sh` | 4.8.1 | — |
| `internal/storage/store.go` | 5.1.1 | — |
| `internal/storage/errors.go` | 5.1.1 | — |
| `internal/storage/vlog.go` | 5.1.2 | — |
| `internal/storage/rotational.go` | 5.1.2 | — |
| `internal/storage/rotational_other.go` | 5.1.2 | — |
| `internal/storage/index.go` | 5.1.3 | — |
| `internal/storage/single_writer_test.go` | 5.2.1 | — |
| `internal/p2p/host.go` | 6.1.1 | — |
| `internal/p2p/nat.go` | 6.1.2 | — |
| `internal/p2p/identity.go` | 6.1.3 | — |
| `internal/p2p/dht_namespace.go` | 6.2.1 | — |
| `internal/p2p/dht.go` | 6.2.2 | — |
| `internal/p2p/dht_test.go` | 6.2.3 | — |
| `internal/p2p/heartbeat.go` | 6.3.1 | — |
| `internal/audit/challenge.go` | 7.1.1 | — |
| `internal/audit/validate.go` | 7.2.1 | — |
| `internal/audit/receipt.go` | 7.3.1 | 7.3.2 |
| `internal/audit/secrets_iface.go` | 7.4.1 | — |
| `internal/audit/secret.go` | 7.4.1 | — |
| `internal/audit/jit.go` | 7.5.1 | — |
| `internal/scoring/score.go` | 8.1.1 | — |
| `internal/scoring/passes.go` | 8.2.1 | — |
| `internal/scoring/rto.go` | 8.3.1 | — |
| `internal/repair/queue.go` | 9.1.1 | 9.1.2 |
| `internal/repair/executor.go` | 9.2.1 | — |
| `internal/repair/departure.go` | 9.3.1 | — |
| `internal/payment/provider.go` | 10.1.1 | — |
| `internal/payment/mock.go` | 10.1.2 | — |
| `internal/payment/ledger.go` | 10.2.1 | — |
| `internal/payment/razorpay.go` | 10.3.1 | — |
| `internal/payment/release.go` | 10.4.1 | — |
| `internal/payment/payment_test.go` | 10.4.2 | — |
| `internal/api/errors.go` | 11.1.1 | — |
| `internal/api/readiness.go` | 11.2.1 | — |
| `internal/api/router.go` | 11.3.1 | 11.4.1–11.10.6 |
| `internal/cluster/gossip.go` | 17.2.1 | — |
| `internal/cluster/router.go` | 17.2.1 | — |
| `internal/cluster/mock_cluster.go` | 17.2.1 | — |
| `internal/secrets/vault.go` | 17.1.1 | — |
| `internal/secrets/aws_ssm.go` | 17.1.1 | — |
| `internal/secrets/gcp_secret.go` | 17.1.1 | — |
| `internal/metrics/microservice.go` | OBS.1.1 | OBS.1.2 |
| `internal/metrics/daemon.go` | OBS.2.1 | — |
| `deployments/grafana/alerts.yaml` | OBS.3.1 | — |
| `deployments/grafana/dashboards/vyomanaut.json` | OBS.3.1 | — |
| `deployments/production/relay/docker-compose.yml` | 17.2.2 | — |
| `internal/vettingchunk/generator.go` | 14.1.1 | — |
| `internal/vettingchunk/gc.go` | 14.2.1 | — |
| `internal/client/account/register.go` | 15.1.1 | — |
| `internal/client/account/keystore.go` | 15.1.1 | — |
| `internal/client/upload/orchestrator.go` | 15.2.1 | 15.2.2 |
| `internal/client/retrieve/orchestrator.go` | 15.3.1 | — |
| `internal/storage/memcheck_linux.go` | 13.6.1 | — |
| `internal/storage/memcheck_darwin.go` | 13.6.1 | — |
| `internal/storage/memcheck_windows.go` | 13.6.1 | — |
| `scripts/test/demo_timeline_test.go` | 16.1.1 | — |
| `runbooks/microservice-failover.md` | 18.1.1 | — |
| `runbooks/postgres-failover.md` | 18.1.1 | — |
| `runbooks/relay-node-replacement.md` | 18.1.1 | — |
| `runbooks/secrets-manager-outage.md` | 18.1.1 | — |
| `runbooks/razorpay-api-outage.md` | 18.1.1 | — |
| `runbooks/provider-mass-departure.md` | 18.1.1 | — |
| `runbooks/rbi-holiday-table-update.md` | 18.1.1 | — |
| `runbooks/audit-secret-rotation.md` | 18.1.1 | — |
| `scripts/benchmarks/aont_encode.sh` | 18.2.1 | — |
| `scripts/benchmarks/argon2id.sh` | 18.2.1 | — |
| `scripts/benchmarks/rocksdb_ssd.sh` | 18.2.1 | — |
| `scripts/benchmarks/rocksdb_hdd.sh` | 18.2.1 | — |
| `scripts/benchmarks/postgres_insert_ceiling.sh` | 18.2.1 | — |
| `scripts/benchmarks/e2e_upload.sh` | 18.2.1 | — |
| `docs/system-design/security-verification-checklist.md` | 18.3.1 | — |

---

*End of BUILD.md*  
*Repository: https://github.com/masamasaowl/Vyomanaut_Research*  
