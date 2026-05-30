# Vyomanaut V2 — Build Procedure

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
- [Appendix A — Invariant Enforcement Traceability](#appendix-a--invariant-enforcement-traceability)
- [Appendix B - Session Registry](#appendix-b--session-registry)
- [Appendix C - File Creation Index](#appendix-c--file-creation-index)

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

``` go
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

```go
$ test -f scripts/ci/grep_checks.sh && echo PASS || echo FAIL

$ test -x scripts/ci/grep_checks.sh && echo PASS || echo FAIL
```

SCRIPT_RUNS:

```go
$ bash scripts/ci/grep_checks.sh
EXPECT: exit 0; all four lines print "PASS [...]"
```

SELF_TEST (inject a violation and confirm detection):

```go
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
Uses: `ubuntu-latest`, `go: '1.22'`

**Services block:** postgres:16 with `POSTGRES_DB=vyomanaut_test`,
`POSTGRES_USER=vyomanaut_app`,
`POSTGRES_PASSWORD=testpass`,
options: `--health-cmd pg_isready`

**Steps (in order, each with id matching check number):**

```go
  check-01: go build ./...
    run: go build ./...

  check-02: go vet ./...
    run: go vet ./...
  
  // Note golangci-lint version has been updated from v1.57.0 -> v2.12.2. Please build accordingly
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

```go
$ test -f .github/workflows/ci.yml && echo PASS || echo FAIL
```

CONTENT_CHECKS:

```go
$ grep -c "^ check-" .github/workflows/ci.yml
EXPECT: 16 (checks 01 through 16)
```

YAML_VALID:

```go
$ python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" \
&& echo PASS || echo FAIL
```

STEP_COUNT:

```go
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

#### Session 0.4.1 — Create `deployments/dev/docker-compose.yml`

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

```go
$ test -f deployments/dev/docker-compose.yml && echo PASS || echo FAIL

$ test -f deployments/dev/init-db.sql && echo PASS || echo FAIL
```

YAML_VALID:

```go
$ docker-compose -f deployments/dev/docker-compose.yml config > /dev/null \
&& echo PASS || echo FAIL
```

CONTENT_CHECKS:

```go
$ grep -c "btree_gist" deployments/dev/init-db.sql
EXPECT: 1

$ grep -c "TODO(M17)" deployments/dev/docker-compose.yml
EXPECT: >= 1 (relay placeholder comment)

$ grep -c "TODO(M13)" deployments/dev/docker-compose.yml
EXPECT: >= 1 (provider placeholder comment)
```

POSTGRES_STARTS:

```go
$ docker-compose -f deployments/dev/docker-compose.yml up -d postgres

$ sleep 5

$ docker-compose -f deployments/dev/docker-compose.yml exec -T postgres \
psql -U vyomanaut_app -d vyomanaut_dev -c "\dx" | grep btree_gist \
&& echo PASS || echo FAIL
```

CLEANUP:

```go
$ docker-compose -f deployments/dev/docker-compose.yml down -v
```

NEGATIVE_CHECKS:

```go
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

```sql
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

**TASK:** Implement startup guard rails in ``internal/config/guards.go.`

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
stores: `chunk_id (32)`, `chunk_size (4)`, `chunk_data (262144)`, `content_hash (32)`. The `AppendChunk` method must:
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
2. Call `IsVettingChunk()` for each assigned chunk — if true, call `DeleteVettingChunksOnDeparture()` instead of `EnqueueJob()`
3. Call `payment.Penalise()` (the departure handler wires payment — but repair package cannot import payment; This violates IC §9 which explicitly prohibits `internal/repair` from importing `internal/payment`. This can be added to Session 12.1.1 where the departure detector goroutine is wired into the microservice)
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
`EscrowSeizure`, and `EscrowReversal` 

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

**Reference:** REQ FR-049 and FR-050, IC §5.8 (`ReleaseEscrow`, `Penalise`), MVP §5.4 (release computation
cycle: calendar date in prod, ticker in demo), DM §4.2 (`frozen` column)

#### Session 10.4.1 — Implement monthly release computation

**Task:** Create `internal/payment/release.go`. The release computation:

1. For each provider with `release_computed = FALSE` for the current audit period
2. Compute the release multiplier from the 30-day reliability score using the table from FR-049: score ≥ 0.95 → multiplier 1.00 (full release); 0.80–0.94 → 0.75; 0.65–0.79 → 0.50; < 0.65 → 0.00 (hold in full). 
3. Apply the dual-window deterioration signal from FR-050: if `(score_30d − score_7d) > 0.20`, use the lower of the two score windows' multipliers. Withheld amounts must roll into the next month's escrow window, not be seized. Rolling amounts held for more than 90 days are flagged for operations team review (FR-049).
4. Call `ReleaseEscrow()` with idempotency key `SHA-256(provider_id || audit_period_id)`
5. Set `release_computed = TRUE`

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
4. Microservice quorum: when `profile.RequireQuorum == true` (production), query the gossip cluster's in-memory membership view: `clusterMembership.HealthyCount() >= 3`. When `profile.RequireQuorum == false` (demo mode), always return `satisfied: true, current_value: 1, required_value: 1`. The gossip membership object is injected at startup by M17 Phase 17.2.1; stub with a `MockClusterMembership` that always returns healthy=3 until M17 completes. The `readiness_condition.threshold` field in the response must read from `NetworkProfile.MinActiveProviders` (not hardcoded 3) per IC §3.4.
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

### Phase 11.11 — Per-Provider Chunk Count Ceiling

#### Session 11.11.1 — Enforce chunk ceiling in assignment service**

**Reference:** NFR-044 (requirements.md §5.5), architecture.md §27.3

In `POST /api/v1/upload/assign` (Session 11.7.1), before generating shard assignments, the assignment service must verify that each candidate provider's current chunk count will not push their steady-state repair bandwidth above 100 Kbps. The ceiling is derived from the provider's observed MTTF tier and the Giroire Formula 1: at MTTF=180 days approximately 70 GB; at MTTF=300 days approximately 130 GB. Providers whose current allocation would exceed their ceiling must be excluded from the candidate pool for that assignment. If the pool drops below 56 eligible providers after applying the ceiling, return HTTP 503 `INSUFFICIENT_PROVIDER_CAPACITY` with a `retry_after: 3600`. Add this check to the readiness gate evaluator as an informational field (not a hard gate condition) surfacing `providers_near_ceiling_count`. NFR-044 requires this ceiling be surfaced in the provider onboarding UI as a declared storage limit advisory — add a `storage_advisory_gb` field to `ProviderRegisterResponse` and `ProviderStatusResponse` populated from `ceil(mttf_days / 300 * 130)` GB.

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
MVP §2.3 (guard rails), OAS `components/securitySchemes, ARCH §18, REQ NFR-028

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

**Note:** We faced an import constraint violation in departure detector (Session 9.3.1). Step 3 of the departure handler lists Call `payment.Penalise()`. So please implement it in this session.  

**The authoritative JWT claim set from OAS `components/securitySchemes/BearerAuth`:**

- `sub` = entity UUID
- `role` = `"owner"` | `"provider"`
- `iss` = `"vyomanaut-microservice-v1"`
- `exp` = Unix timestamp

TTL: 24 hours for owners, 7 days for provider daemons (OAS `OtpVerifyResponse.properties.token` description).

Registration token TTL (short-lived, single-use): 1 hour, valid only for the matching register call (OAS `OtpVerifyResponse.properties.token` description, `is_new_entity = true` case).

**Gossip cluster initialisation** (architecture.md §18). When `profile.RequireQuorum == true`, initialise the gossip cluster by:

1. Reading two seed node addresses from environment variables `VYOMANAUT_SEED_NODE_1` and `VYOMANAUT_SEED_NODE_2` (pre-configured stable addresses per architecture.md §18; fail-fast at startup if absent in production mode).
2. Initialising `internal/cluster/gossip.GossipCluster` with a per-second reconciliation ticker: each tick selects one randomly chosen peer from the membership view and exchanges membership histories.
3. Waiting for at least 2 peers to ack membership before the readiness gate evaluator loop starts (prevents split-brain false-ready on cold start).

**Note:** Use `MockClusterMembership` (from `internal/cluster/mock_cluster.go`) until M17 Phase 17.2.1 is complete.

**Client-driven routing** (architecture.md §18). For latency-sensitive hot paths — audit challenge dispatch and chunk assignment decisions — the microservice must bypass the load balancer and route directly to the responsible replica. Implement `internal/cluster/router.ResponsibleReplica(opType string) *url.URL` which reads the in-memory cluster membership view and returns the direct address of the replica owning the operation shard. This reduces p99 latency by 30+ ms (architecture.md §18).

**Background task throttling** (architecture.md §18, NFR-028). Start a `backgroundThrottle` goroutine that samples `db_read_p99_latency_ms` every 60 seconds. If the metric approaches 50 ms, reduce background task concurrency (view refresh, repair queuing, Merkle log compaction) by calling `backgroundSemaphore.Reduce()`. Restore full concurrency when p99 drops below 30 ms.

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
