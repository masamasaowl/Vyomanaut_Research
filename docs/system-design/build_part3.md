# Vyomanaut V2 — Build Procedure Part 3

**Status:** Authoritative build specification  
**Version:** 1.0  
**Date:** July 2026  
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
- [Milestone 12 — Coordination Microservice (`cmd/microservice`)](#milestone-12--coordination-microservice-cmdmicroservice)
  - [Phase 12.1 — Microservice Startup](#phase-121--microservice-startup)
- [Milestone M-OBS — Observability & Metrics](#milestone-m-obs--observability--metrics)
  - [Phase OBS.1 — Microservice Metrics](#phase-obs1--microservice-metrics)
  - [Phase OBS.2 — Provider Daemon Metrics](#phase-obs2--provider-daemon-metrics)
  - [Phase OBS.3 — Operational Alerts](#phase-obs3--operational-alerts)
  - [Phase OBS.4 — Prometheus Metric Naming CI Gate](#phase-obs4--prometheus-metric-naming-ci-gate)
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

**Deliverable:** every NFR-025/NFR-026 metric exported under the NFR-046 naming contract; Grafana alert rules + dashboard defined; the background-throttle **windowed** signal wired (ADR-033); `TestNoOrphanMetricName` passes as CI check 16.
**Reference:** architecture.md §23, requirements.md §5.6 (NFR-025, 026, 027, 028) and §5.5 (NFR-046). *[Corrected per A5: OBS wires increments into M7/M9/M10 and the M12 throttle under the authoritative numbering.]*

---

### Phase OBS.1 — Microservice metrics

#### Session OBS.1.1 — Register all microservice Prometheus metrics

**Reference:** NFR-025 (requirements.md §5.6), architecture.md §23, NFR-046 (as restated in A6), ADR-033.

**PRECONDITIONS**

- `internal/metrics/` does not yet exist (net-new package; the observability leaf per A2/Option A).
- `prometheus/client_golang` is already a module dependency (M0).

**TASK**

1. Create `internal/metrics/microservice.go`. Register every NFR-025 metric using `vyomanaut_{subsystem}_{name}_{unit}`, `{subsystem}` from the allow-list `{audit, scoring, repair, payment, cluster, db}` (A6). Names are a frozen contract:

   | Metric name | Type | Labels | Emitting site (informational) |
   |---|---|---|---|
   | `vyomanaut_audit_challenges_issued_total` | Counter | — | audit |
   | `vyomanaut_audit_results_total` | Counter | `result`=PASS\|FAIL\|TIMEOUT | audit |
   | `vyomanaut_scoring_provider_score` | Histogram | — | scoring |
   | `vyomanaut_repair_queue_depth` | Gauge | — | repair |
   | `vyomanaut_repair_jobs_completed_total` | Counter | — | repair |
   | `vyomanaut_payment_escrow_events_total` | Counter | `type`=DEPOSIT\|RELEASE\|SEIZURE\|REVERSAL | payment |
   | `vyomanaut_cluster_replica_count` | Gauge | `state`=healthy\|degraded | cmd/microservice |
   | `vyomanaut_db_read_latency_seconds` | Histogram | — *(A11: no percentile label)* | audit/cross-cutting |

   > These names apply A6's allow-list consistently (`scoring`, `payment`, `cluster`) so `{subsystem}` is always a valid package/component. If you prefer to preserve the older `vyomanaut_provider_score` / `vyomanaut_escrow_events_total` / `vyomanaut_microservice_replica_count` spellings, keep them **only if** A6's allow-list explicitly grants `provider`, `escrow`, `microservice` — decide this once, here, because it freezes the contract.

2. `vyomanaut_db_read_latency_seconds` bucket boundaries: `{0.01, 0.025, 0.04, 0.05, 0.1}` (ADR-033 adds `0.04`; `0.05` is the throttle threshold).
3. **ADR-033 windowed signal:** add a 60-second sliding-window p99 estimator; expose `ForegroundReadLatency.Observe(d)` (updates histogram **and** window) and `ForegroundReadP99Window() time.Duration`. The throttle reads the window, never the cumulative histogram.
4. Expose all metrics at `/metrics` on a configurable admin port (default `9090`).

**FILES**

- `internal/metrics/microservice.go`
- `internal/metrics/window.go` *(ADR-033 estimator; keep pure-stdlib so the leaf imports nothing internal)*

**VERIFY**

```bash
FILES_EXIST:
  $ test -f internal/metrics/microservice.go && echo PASS || echo FAIL
  $ test -f internal/metrics/window.go       && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/metrics/
  EXPECT: exit 0

LEAF_HAS_NO_INTERNAL_IMPORTS:
  $ grep -c "masamasaowl/Vyomanaut_V2/internal/" internal/metrics/*.go
  EXPECT: 0                      # A2: internal/metrics is a zero-internal-dep leaf

ALL_EIGHT_NFR025_METRICS_REGISTERED:
  $ grep -oE "vyomanaut_[a-z_]+" internal/metrics/microservice.go | sort -u | wc -l
  EXPECT: >= 8

METRIC_NAMES_PRESENT:
  $ for m in audit_challenges_issued_total audit_results_total scoring_provider_score \
             repair_queue_depth repair_jobs_completed_total payment_escrow_events_total \
             cluster_replica_count db_read_latency_seconds; do \
      grep -q "vyomanaut_$m" internal/metrics/microservice.go && echo "PASS $m" || echo "FAIL $m"; done
  EXPECT: eight PASS lines

THROTTLE_BUCKET_BOUNDARIES:
  $ grep -c "0.05" internal/metrics/microservice.go
  EXPECT: >= 1                   # 50 ms throttle threshold present as a bucket
  $ grep -c "0.04" internal/metrics/microservice.go
  EXPECT: >= 1                   # ADR-033 early-warning bucket

NO_INVALID_HISTOGRAM_LABEL:
  $ grep -c "percentile" internal/metrics/microservice.go
  EXPECT: 0                      # A11: histograms carry no p99 label

WINDOWED_SIGNAL_EXPORTED:
  $ grep -c "func ForegroundReadP99Window" internal/metrics/window.go
  EXPECT: 1
  $ grep -c "func .*Observe" internal/metrics/window.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestMetricsRegistry ./internal/metrics/
  EXPECT: exit 0; tests include:
    TestAllNFR025NamesRegistered
    TestDBReadHistogramHasThrottleBuckets
    TestForegroundP99WindowReflectsRecentBurstNotLifetime   # ADR-033 core guarantee

VET:
  $ go vet ./internal/metrics/
  EXPECT: exit 0; zero output
```

---

#### Session OBS.1.2 — Wire metric increments at call sites

**Reference:** NFR-025, NFR-046, A2 (import-DAG resolution), ADR-033.

**PRECONDITIONS**

- OBS.1.1 complete. IC §9 / build-skill §4 amended per **A2/Option A** (`internal/metrics` is an importable leaf). If A2/Option B is chosen instead, wire all increments in `cmd/microservice` and adjust the VERIFY paths accordingly.

**TASK**

1. `internal/audit` receipt-write site (**M7** Session 7.3.2): `metrics.AuditResultsTotal.With(prometheus.Labels{"result": result.String()}).Inc()`.
2. `internal/repair` enqueue/dequeue (**M9** Phase 9.1): `metrics.RepairQueueDepth.Set(float64(depth))`.
3. `internal/payment.InsertEscrowEvent` (**M10** Phase 10.2.1): `metrics.EscrowEventsTotal.With(...).Inc()`.
4. Foreground DB reads on the audit hot path call `metrics.ForegroundReadLatency.Observe(elapsed)` (ADR-033).
   All names must be the exact frozen strings from OBS.1.1 — any rename is a breaking change (NFR-046).

**FILES** *(edits only)*

- `internal/audit/*.go` (receipt write), `internal/repair/*.go` (enqueue/dequeue), `internal/payment/*.go` (`InsertEscrowEvent`)

**VERIFY**

```bash
IMPORT_LEAF_ALLOWED:      # A2/Option A must be in force, else these imports fail the DAG gate
  $ grep -rl "internal/metrics" internal/audit internal/repair internal/payment | wc -l
  EXPECT: >= 3

AUDIT_RESULT_INCREMENT:
  $ grep -c "AuditResultsTotal" internal/audit/*.go
  EXPECT: >= 1

REPAIR_DEPTH_SET:
  $ grep -c "RepairQueueDepth.Set" internal/repair/*.go
  EXPECT: >= 1

ESCROW_INCREMENT_IN_INSERT:
  $ grep -c "EscrowEventsTotal" internal/payment/*.go
  EXPECT: >= 1

THROTTLE_OBSERVE_ON_HOT_PATH:
  $ grep -c "ForegroundReadLatency.Observe" internal/audit/*.go
  EXPECT: >= 1

NO_RENAMED_METRICS:
  $ grep -roE "vyomanaut_[a-z_]+" internal/audit internal/repair internal/payment | sort -u \
      | while read n; do grep -q "$n" internal/metrics/microservice.go || echo "ORPHAN $n"; done
  EXPECT: no ORPHAN lines

BUILD_AFFECTED:
  $ go build ./internal/audit/ ./internal/repair/ ./internal/payment/
  EXPECT: exit 0
```

---

### Phase OBS.2 — Provider daemon metrics

#### Session OBS.2.1 — Register all provider-daemon metrics

**Reference:** NFR-026, architecture.md §23, NFR-046 (allow-list restated in A6).

**TASK**

1. Create `internal/metrics/daemon.go`. Register (subsystems from the allow-list; names frozen):

   | Metric name | Type |
   |---|---|
   | `vyomanaut_daemon_chunks_stored_total` | Counter |
   | `vyomanaut_daemon_audit_responses_sent_total` | Counter |
   | `vyomanaut_daemon_audit_response_latency_milliseconds` | Histogram |
   | `vyomanaut_daemon_vlog_append_latency_milliseconds` | Histogram |
   | `vyomanaut_daemon_content_hash_failures_total` | Counter |
   | `vyomanaut_daemon_heartbeat_sent_total` | Counter |
   | `vyomanaut_daemon_ram_constrained` | Gauge (`constrained`) |

   *(Using the single `daemon` subsystem for all daemon-side metrics keeps NFR-046's allow-list rule satisfiable without renaming across packages — A6.)*
2. Expose at a **local-only** HTTP server on `localhost:9091` (not publicly reachable). The FR-029 CLI status view reads this endpoint (no separate in-memory state).

**FILES**

- `internal/metrics/daemon.go`

**VERIFY**

```bash
FILES_EXIST:
  $ test -f internal/metrics/daemon.go && echo PASS || echo FAIL

COMPILE:
  $ go build ./internal/metrics/
  EXPECT: exit 0

SEVEN_DAEMON_METRICS:
  $ grep -oE "vyomanaut_daemon_[a-z_]+" internal/metrics/daemon.go | sort -u | wc -l
  EXPECT: 7

RAM_CONSTRAINED_GAUGE_PRESENT:     # consumed by Session 13.6.1 (A1)
  $ grep -c "vyomanaut_daemon_ram_constrained" internal/metrics/daemon.go
  EXPECT: >= 1

LOCAL_ONLY_BIND:
  $ grep -c "localhost:9091\|127.0.0.1:9091" internal/metrics/daemon.go
  EXPECT: >= 1                    # must NOT bind 0.0.0.0

VET:
  $ go vet ./internal/metrics/
  EXPECT: exit 0; zero output
```

---

### Phase OBS.3 — Operational alerts

#### Session OBS.3.1 — Define Grafana alert rules + dashboard

**Reference:** NFR-027 (requirements.md §5.6 — architecture ref corrected to **§23** per A8), architecture.md §23.

**TASK**

1. Create `deployments/grafana/alerts.yaml` with the four mandatory NFR-027 rules (names must match the frozen metric strings from OBS.1.1):

   | Alert | Condition | Severity |
   |---|---|---|
   | `RepairQueueDepthHigh` | `vyomanaut_repair_queue_depth > 1000` | warning |
   | `AuditTimeoutRateHigh` | `rate(vyomanaut_audit_results_total{result="TIMEOUT"}[1h]) / rate(vyomanaut_audit_results_total[1h]) > 0.05` | critical |
   | `ContentHashFailureDetected` | `increase(vyomanaut_daemon_content_hash_failures_total[7d]) > 0` | critical |
   | `MicroserviceReplicasDegraded` | `vyomanaut_cluster_replica_count{state="healthy"} < 3` | critical |

   *(A10: `architecture.md §23`'s fifth "Release multiplier 0.00" alert is intentionally omitted here — it has no NFR-025 backing metric. Resolve via A10 option (a) add-metric-and-alert, or (b) mark it V3-deferred in §23, before this session is considered complete.)*
2. Add `deployments/grafana/dashboards/vyomanaut.json` referencing metric names by exact string.
3. Add CI check `TestGrafanaMetricNamesMatchRegistry` to `scripts/ci/grep_checks.sh`: every metric string in the dashboard JSON must exist in `internal/metrics/*.go`.

**FILES**

- `deployments/grafana/alerts.yaml`, `deployments/grafana/dashboards/vyomanaut.json`, `scripts/ci/grep_checks.sh` (edit)

**VERIFY**

```bash
FILES_EXIST:
  $ test -f deployments/grafana/alerts.yaml                     && echo PASS || echo FAIL
  $ test -f deployments/grafana/dashboards/vyomanaut.json       && echo PASS || echo FAIL

FOUR_MANDATORY_ALERTS:
  $ grep -cE "RepairQueueDepthHigh|AuditTimeoutRateHigh|ContentHashFailureDetected|MicroserviceReplicasDegraded" deployments/grafana/alerts.yaml
  EXPECT: 4

ALERTS_REFERENCE_LIVE_METRICS_ONLY:
  $ grep -oE "vyomanaut_[a-z_]+" deployments/grafana/alerts.yaml | sort -u \
      | while read n; do grep -q "$n" internal/metrics/*.go || echo "ORPHAN $n"; done
  EXPECT: no ORPHAN lines

TIMEOUT_THRESHOLD_IS_5PCT:
  $ grep -c "0.05" deployments/grafana/alerts.yaml
  EXPECT: >= 1

DASHBOARD_NAMES_IN_REGISTRY:
  $ grep -oE "vyomanaut_[a-z_]+" deployments/grafana/dashboards/vyomanaut.json | sort -u \
      | while read n; do grep -q "$n" internal/metrics/*.go || echo "ORPHAN $n"; done
  EXPECT: no ORPHAN lines

CI_CHECK_ADDED:
  $ grep -c "TestGrafanaMetricNamesMatchRegistry" scripts/ci/grep_checks.sh
  EXPECT: 1
```

---

### Phase OBS.4 — Prometheus metric-naming CI gate

#### Session OBS.4.1 — Implement `TestNoOrphanMetricName` (CI check 16)

**Reference:** NFR-046 (requirements.md §5.5, restated in A6).

**TASK**
Add a grep-fail check to `scripts/ci/grep_checks.sh`: extract every metric-name string from `deployments/grafana/dashboards/vyomanaut.json` **and** `deployments/grafana/alerts.yaml`; verify each appears in ≥1 `.go` file under `internal/metrics/`, and (A6) that each `{subsystem}` is in the allow-list. Any name in dashboards/alerts but absent from the registry (or vice-versa), or any out-of-allow-list subsystem, fails with:
`"orphan metric name: {name} — update metrics/*.go and dashboards simultaneously"`.
Register as **CI check 16** (add to Phase 0.3 Session 0.3.1 step list).

**FILES**

- `scripts/ci/grep_checks.sh` (edit)

**VERIFY**

```bash
CHECK_PRESENT:
  $ grep -c "TestNoOrphanMetricName" scripts/ci/grep_checks.sh
  EXPECT: 1

ORPHAN_MESSAGE_EXACT:
  $ grep -c "orphan metric name:" scripts/ci/grep_checks.sh
  EXPECT: >= 1

BIDIRECTIONAL_SOURCES:
  $ grep -c "vyomanaut.json" scripts/ci/grep_checks.sh
  EXPECT: >= 1
  $ grep -c "alerts.yaml" scripts/ci/grep_checks.sh
  EXPECT: >= 1

ALLOWLIST_ENFORCED:       # A6: subsystem must be in {audit,scoring,repair,payment,cluster,storage,daemon,db}
  $ grep -cE "audit|scoring|repair|payment|cluster|storage|daemon|db" scripts/ci/grep_checks.sh
  EXPECT: >= 1

GATE_RUNS_GREEN_ON_CURRENT_TREE:
  $ bash scripts/ci/grep_checks.sh
  EXPECT: exit 0
```

---

## Milestone 13 — Provider Daemon Core (`cmd/provider`)

**Deliverable:** provider daemon with stream handlers for all four libp2p protocols, startup identity, heartbeat, and the corrected pre-install RAM check.

**Reference:** IC §4.1, §4.2, §4.4.1, §4.5, §3.1, §4 (transport auth, 0-RTT, framing), §12/§12.2 (DHT), MVP §5.3/§8.3.

**One Flag:** `cmd/provider/main.go` is currently the M0 stub (`fmt.Printf("[STARTUP]...mode=STUB")`) —
Milestone 13's session that replaces it has not been executed against this repo yet.
`ADR-038`/`ADR-047` both assume the tray process and the daemon logic are **the same
process**, started in-process by a Task Scheduler logon trigger, with no service/companion
split. That only works cleanly if Milestone 13 Session 13.1.1's eventual `main()` is
written as a thin wrapper around an exported, context-cancelable entry point (e.g.
`provider.StartDaemon(ctx context.Context, cfg Config) (*provider.Handle, error)` in an
`internal/`-reachable location) rather than a `main()` that owns the whole process
lifecycle inline. This costs nothing extra today — M13 hasn't been built yet — and avoids a
forced refactor later, once a Wails app needs to call the exact same startup sequence
in-process instead of `exec`-ing a separate binary.

---

### Phase 13.1 — Provider startup

#### Session 13.1.1 — Wire provider `main()`

**PRECONDITIONS**

- `internal/p2p/host.go` (M6) provides `Host`, `NewHost`, and `zeroRTTProhibited`.
- `internal/storage.ChunkStore` (M5) provides `RecoverFromCrash`, `AppendChunk`, `LookupChunk`, `DeleteChunk`.
- `cmd/provider/main.go` is currently the M0 stub.

**TASK** — replace the stub with, in order:

1. Parse flags per MVP §8.3 (`--microservice-url`, `--data-dir`, `--declared-storage-gb`, `--relay-addrs`, `--sim-count`, `--sim-base-port`, `--sim-data-dir`, `--sim-asn-count`).
2. `config.SelectProfile()` then `config.ValidateStartupGuards(profile)`.
3. **RAM check (Session 13.6.1) runs here, before `ChunkStore` init** — see 13.6.1 (A1).
4. Load/generate Ed25519 identity (`internal/p2p/identity.go`).
5. `ChunkStore.RecoverFromCrash()` **before** starting the writer goroutine.
6. Start the single writer goroutine (only caller of `AppendChunk`).
7. `NewHost` with QUIC+TCP; **responder-side 0-RTT rejection for every protocol in `zeroRTTProhibited`** (A7).
8. Register the four stream handlers (Phases 13.2–13.5).
9. Heartbeat goroutine + DHT republication (IC §3.1, §12.2); register the DHT custom validator from `dht_namespace.go` (IC §12).

**FILES** — `cmd/provider/main.go` (rewrite)

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/provider/
  EXPECT: exit 0

NO_BUSINESS_LOGIC_IN_CMD:      # IC §11 — cmd/ is wiring only
  $ grep -cE "func (verify|compute|encode|decode|reconstruct)[A-Z]" cmd/provider/main.go
  EXPECT: 0

STARTUP_ORDER_RAM_BEFORE_STORE:
  $ awk '/RAM|memcheck|RequiredRAM/{r=NR} /RecoverFromCrash/{c=NR} END{print (r>0 && r<c)?"PASS":"FAIL"}' cmd/provider/main.go
  EXPECT: PASS

RECOVER_BEFORE_WRITER:
  $ awk '/RecoverFromCrash/{r=NR} /go .*[Ww]riter|writerLoop|startWriter/{w=NR} END{print (r>0 && r<w)?"PASS":"FAIL"}' cmd/provider/main.go
  EXPECT: PASS

ALL_FOUR_HANDLERS_REGISTERED:
  $ grep -cE "chunk-upload/1.0.0|audit-challenge/1.0.0|repair-download/1.0.0|vetting-gc/1.0.0" cmd/provider/main.go
  EXPECT: >= 4

RESPONDER_SIDE_ZERO_RTT:       # A7 — deny-list consulted on the responder, not a suffix
  $ grep -c "zeroRTTProhibited\|DisableEarlyData" cmd/provider/main.go
  EXPECT: >= 1

DHT_VALIDATOR_FROM_CONSTANT:
  $ grep -c "dht_namespace\|DHTNamespace\|CustomValidator" cmd/provider/main.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestProviderStartup ./cmd/provider/
  EXPECT: exit 0; tests include:
    TestStartupRunsRAMCheckBeforeChunkStore
    TestStartupRecoversBeforeWriterGoroutine
    TestStartupRegistersAllFourProtocolHandlers

VET:
  $ go vet ./cmd/provider/
  EXPECT: exit 0; zero output
```

---

### Phase 13.2 — Chunk upload stream handler

#### Session 13.2.1 — `/vyomanaut/chunk-upload/1.0.0`

**Reference:** IC §4.1 (full Frame 1 verification, capability token, all status codes), OAS `ShardAssignment.capability_token`. 0-RTT is **permitted** here (IC §4.1).

**PRECONDITIONS** — writer goroutine and `ChunkStore` live; capability-token verify key available from config.

**TASK**

1. Read 4-byte length prefix; reject `0x01` if `length > 262252` *(= 32+4+72+262144; matches on-disk `capabilityTokenSize`/`chunkIDFieldSize` constants)*.
2. Parse `chunk_id`(32) ‖ `shard_index`(4) ‖ `capability_token`(72) ‖ `chunk_data`(262144).
3. Capability-token verify (IC §4.1 steps 1–5): 72-byte token = `expiry_unix_ms`(8, be int64) ‖ Ed25519 sig(64) → else `0x03`; `expiry_unix_ms > NOW − 30_000` → else `0x07 CAPABILITY_EXPIRED`; verify sig → else `0x03`; `chunk_id` bound in signing input → same `0x03`.
4. `SHA-256(chunk_data) == chunk_id` → else `0x02` **before any disk write**.
5. Write via the writer-goroutine channel (single-writer rule — never call `AppendChunk` directly).
6. On `0x00`: `provider_sig = Ed25519(SHA-256(chunk_id ‖ shard_index ‖ provider_id_bytes ‖ timestamp_unix_ms))`.
7. Frame 2 = `status` ‖ (on success) `provider_sig`. Handle `0x06 ALREADY_STORED` idempotently.
8. Pre-condition: `providers.status ∈ {ACTIVE, VETTING}`; if `DEPARTED`, reset stream immediately.

**FILES** — `cmd/provider/handler_upload.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/provider/
  EXPECT: exit 0

FRAME_SIZE_CAP:
  $ grep -c "262252" cmd/provider/handler_upload.go
  EXPECT: >= 1

ALL_EIGHT_UPLOAD_STATUS_CODES:
  $ grep -cE "0x00|0x01|0x02|0x03|0x04|0x05|0x06|0x07" cmd/provider/handler_upload.go
  EXPECT: >= 8

HASH_CHECK_BEFORE_WRITE:      # 0x02 must precede any channel send to the writer
  $ awk '/SHA-256|sha256.Sum256/{h=NR} /writer|AppendChunk|writeCh/{w=NR} END{print (h>0 && h<w)?"PASS":"FAIL"}' cmd/provider/handler_upload.go
  EXPECT: PASS

SINGLE_WRITER_RULE:           # handler must not call AppendChunk directly
  $ grep -c "\.AppendChunk(" cmd/provider/handler_upload.go
  EXPECT: 0

CAPABILITY_EXPIRY_WINDOW:
  $ grep -c "30_000\|30000" cmd/provider/handler_upload.go
  EXPECT: >= 1

DEPARTED_RESETS_STREAM:
  $ grep -c "DEPARTED" cmd/provider/handler_upload.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestChunkUploadHandler ./cmd/provider/
  EXPECT: exit 0; tests include:
    TestUploadRejectsExpiredCapabilityToken            # 0x07
    TestUploadRejectsContentHashMismatchBeforeWrite    # 0x02, no disk write
    TestUploadIdempotentAlreadyStored                  # 0x06
    TestUploadRejectsDepartedProvider
```

---

### Phase 13.3 — Audit challenge stream handler

#### Session 13.3.1 — `/vyomanaut/audit-challenge/1.0.0`

**Reference:** IC §4.2. 0-RTT **prohibited** — enforced by **membership in `zeroRTTProhibited`** (A7), *not* a `-challenge` suffix, and rejected on the **responder** side.

**TASK**

1. Frame 1: `chunk_id`(32) ‖ `challenge_nonce`(33) ‖ `server_challenge_ts_ms`(8) — reject `0x03` if nonce ≠ 33 bytes (Invariant 5).
2. Check nonce version byte against valid `server_secret_vN` versions.
3. Bloom filter → `LookupChunk()`.
4. Not found → `0x01` with `provider_sig` (§4.2: `0x01`/`0x02` are 1+64 bytes).
5. `SHA-256(chunk_data) == content_hash` → else `0x02`.
6. `response_hash = SHA-256(chunk_data ‖ challenge_nonce)`.
7. `provider_sig = Ed25519(SHA-256(response_hash ‖ challenge_nonce ‖ server_challenge_ts_ms ‖ provider_id))`.
8. Frame 2: `0x00` ‖ `response_hash` ‖ `provider_sig`.
9. Concurrency ≥ 32 simultaneous streams; `LookupChunk` is goroutine-safe.

**FILES** — `cmd/provider/handler_audit.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/provider/
  EXPECT: exit 0

NONCE_LENGTH_33_ENFORCED:
  $ grep -c "33" cmd/provider/handler_audit.go
  EXPECT: >= 1
  $ grep -c "BYTEA(32)\|\\[32\\]byte.*nonce\|nonce.*\\[32\\]byte" cmd/provider/handler_audit.go
  EXPECT: 0                        # Invariant 5 / IC §11 forbidden

ZERO_RTT_BY_DENYLIST_NOT_SUFFIX:   # A7
  $ grep -c "zeroRTTProhibited" cmd/provider/handler_audit.go
  EXPECT: >= 1
  $ grep -c "HasSuffix.*challenge\|strings.HasSuffix" cmd/provider/handler_audit.go
  EXPECT: 0

ERROR_RESPONSES_ARE_1_PLUS_64:
  $ grep -c "1 + 64\|1+64\|65" cmd/provider/handler_audit.go
  EXPECT: >= 1

SIGNING_INPUT_FIXED_LAYOUT:        # no json.Marshal in the signing path (IC §11)
  $ grep -c "json.Marshal" cmd/provider/handler_audit.go
  EXPECT: 0

UNIT_TESTS:
  $ go test -v -run TestAuditChallengeHandler ./cmd/provider/
  EXPECT: exit 0; tests include:
    TestAuditRejectsNon33ByteNonce
    TestAuditNotFoundReturnsSignedResponse             # 0x01, 1+64 bytes
    TestAuditContentHashMismatchReturns0x02
    TestAudit32ConcurrentStreams
```

---

### Phase 13.4 — Repair download stream handler

#### Session 13.4.1 — `/vyomanaut/repair-download/1.0.0`

**Reference:** IC §4.4.1. 0-RTT prohibited (deny-list membership, responder side).

**TASK** *(base — buildable today)*

1. Verify requesting Peer ID ∈ registered-microservice replica set (locally cached; refreshed via DHT + heartbeat acks). Unregistered → `0x02 NOT_AUTHORISED` immediately.
2. Verify `repair_auth_sig = Ed25519(SHA-256(chunk_id ‖ request_ts_ms ‖ microservice_peer_id))`.
3. `LookupChunk()` → `0x01` not found, `0x03` corruption.
4. Frame 2: `0x00` ‖ `chunk_data`. Timeout 10 000 ms.

**⛔ SECURITY ADDENDUM — apply after ADR-036 accepted (A3):**

- Between steps 2 and 3, add: reject when `|now − request_ts_ms| > profile.AuthRequestFreshnessWindow` → `0x02 NOT_AUTHORISED`. (The `request_ts_ms` is already signed; the handler simply never checked freshness.)

**FILES** — `cmd/provider/handler_repair.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/provider/
  EXPECT: exit 0

PEER_AUTHZ_FIRST:                 # unregistered peer rejected before any lookup
  $ awk '/NOT_AUTHORISED|0x02/{a=NR} /LookupChunk/{l=NR} END{print (a>0 && a<l)?"PASS":"FAIL"}' cmd/provider/handler_repair.go
  EXPECT: PASS

REPAIR_AUTH_SIG_VERIFIED:
  $ grep -c "repair_auth_sig\|repairAuthSig\|ed25519.Verify" cmd/provider/handler_repair.go
  EXPECT: >= 1

STATUS_CODES_PRESENT:
  $ grep -cE "0x00|0x01|0x02|0x03" cmd/provider/handler_repair.go
  EXPECT: >= 4

TIMEOUT_10S:
  $ grep -c "10_000\|10000\|10 \* time.Second" cmd/provider/handler_repair.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestRepairDownloadHandler ./cmd/provider/
  EXPECT: exit 0; tests include:
    TestRepairRejectsUnregisteredPeerBeforeLookup      # 0x02
    TestRepairRejectsInvalidAuthSig
    TestRepairReturnsChunkDataOnSuccess

# ── VERIFY (enable after ADR-036 accepted) ──
FRESHNESS_WINDOW_ENFORCED:
  $ grep -c "AuthRequestFreshnessWindow" cmd/provider/handler_repair.go
  EXPECT: >= 1
UNIT_TESTS_ADR036:
  $ go test -v -run TestRepairRejectsStaleRequest ./cmd/provider/
  EXPECT: exit 0
```

---

### Phase 13.5 — Vetting GC stream handler

#### Session 13.5.1 — `/vyomanaut/vetting-gc/1.0.0`

**Reference:** IC §4.5, IC §5.3 (`DeleteChunk`). 0-RTT prohibited (deny-list membership, responder side).

> **🟠 SECURITY (A3):** As specified in the current IC §4.5, this handler deletes chunks with **no caller authorization**. Since chunk IDs are DHT-discoverable and the daemon cannot distinguish synthetic from real chunks (DM §3 Invariant 6), the base handler is exploitable for real-data destruction. The base session below is faithful to the current contract; **ADR-036 (proposed) closes the hole** and its addendum should be applied as soon as ADR-036 is accepted — ideally before this handler ships to any network carrying real data.

**TASK** *(base — matches current IC §4.5)*

1. Frame 1: `chunk_count`(4) ‖ `chunk_ids`(`chunk_count×32`). Reject if `chunk_count > 10 000`.
2. For each chunk ID: `DeleteChunk()`.
3. Build `failure_bitmap`: bit N set iff `DeleteChunk(chunk_ids[N])` failed.
4. Frame 2: `0x00` all-ok · `0x01` ‖ bitmap partial · `0x02 INTERNAL_ERROR`.
5. ≤ 10 000 IDs/frame; multiple sequential frames per stream.
6. Timeout 30 000 ms/frame. Handler remains in the binary indefinitely (IC §13).

**⛔ SECURITY ADDENDUM — apply after ADR-036 accepted (A3):**

- Parse the two new Frame-1 fields `request_ts_ms`(8) ‖ `gc_auth_sig`(64).
- **Before any `DeleteChunk`:** (a) requesting Peer ID ∈ registered-microservice set → else `0x03 NOT_AUTHORISED`; (b) `|now − request_ts_ms| ≤ profile.AuthRequestFreshnessWindow` → else `0x04 STALE_REQUEST`; (c) verify `gc_auth_sig = Ed25519(SHA-256(chunk_ids ‖ request_ts_ms ‖ microservice_peer_id))` → else `0x03`.
- Renumber `INTERNAL_ERROR` → `0x05`.

**FILES** — `cmd/provider/handler_vetting_gc.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/provider/
  EXPECT: exit 0

MAX_10K_IDS_PER_FRAME:
  $ grep -c "10_000\|10000" cmd/provider/handler_vetting_gc.go
  EXPECT: >= 1

FAILURE_BITMAP_SIZE:              # ceil(chunk_count/8)
  $ grep -c "/ 8\|>> 3\|(chunk_count+7)" cmd/provider/handler_vetting_gc.go
  EXPECT: >= 1

TIMEOUT_30S:
  $ grep -c "30_000\|30000\|30 \* time.Second" cmd/provider/handler_vetting_gc.go
  EXPECT: >= 1

DELETE_CALLED:
  $ grep -c "DeleteChunk" cmd/provider/handler_vetting_gc.go
  EXPECT: >= 1

UNIT_TESTS:
  $ go test -v -run TestVettingGCHandler ./cmd/provider/
  EXPECT: exit 0; tests include:
    TestVettingGCPartialFailureBitmap
    TestVettingGCRejectsOver10kIDs
    TestVettingGCMultipleSequentialFrames

# ── VERIFY (enable after ADR-036 accepted) — the security-critical checks ──
AUTHZ_BEFORE_ANY_DELETE:
  $ awk '/NOT_AUTHORISED|0x03/{a=NR} /DeleteChunk/{d=NR} END{print (a>0 && a<d)?"PASS":"FAIL"}' cmd/provider/handler_vetting_gc.go
  EXPECT: PASS
GC_AUTH_SIG_VERIFIED:
  $ grep -c "gc_auth_sig\|gcAuthSig\|ed25519.Verify" cmd/provider/handler_vetting_gc.go
  EXPECT: >= 1
FRESHNESS_WINDOW_ENFORCED:
  $ grep -c "AuthRequestFreshnessWindow" cmd/provider/handler_vetting_gc.go
  EXPECT: >= 1
UNIT_TESTS_ADR036:
  $ go test -v -run 'TestVettingGCRejectsUnauthorizedPeer|TestVettingGCRejectsForgedSig|TestVettingGCRejectsStaleRequest' ./cmd/provider/
  EXPECT: exit 0
```

---

### Phase 13.6 — Provider RAM check at installation

#### Session 13.6.1 — Pre-installation RAM check *(A1 fix folded in)*

**Reference:** NFR-045 (requirements.md §5.5), architecture.md §27.5.

**PRECONDITIONS** — runs in `main()` **before** `ChunkStore` init (Session 13.1.1 step 3). `vyomanaut_daemon_ram_constrained` exists (OBS.2.1).

**TASK**

1. Compute required RAM with the **corrected constant** (A1):

   ```
   ChunksPerGB        = (1 << 30) / ShardSize          // 4096
   DHTRecordSizeBytes = 200
   required_mb        = ceil(declared_storage_gb × ChunksPerGB × DHTRecordSizeBytes / (1 << 20))
   ```

   Sanity: 50 GB → 40 MB, 200 GB → 160 MB, 500 GB → ~400 MB (matches §27.5 / NFR-045).
2. Read free RAM via platform syscall — `internal/storage/memcheck_linux.go` (`/proc/meminfo`), `_darwin.go` (`sysctl`), `_windows.go` (`GlobalMemoryStatusEx`), plus a `_other.go` build-tagged stub.
3. If free < required: WARN (do not halt) —
   `"[WARN] Declared storage requires ~{required_mb} MB free RAM for DHT cache; only {available_mb} MB detected. Chunk assignment will be limited until RAM is freed."` — reduce `declared_storage_gb` to the safe ceiling and set `vyomanaut_daemon_ram_constrained{constrained="true"}`.
4. Installer runs this before completing the wizard and surfaces the shortfall.

**FILES** — `cmd/provider/main.go` (edit), `internal/storage/memcheck_linux.go`, `_darwin.go`, `_windows.go`, `_other.go`

**VERIFY**

```bash
COMPILE_ALL_PLATFORMS:
  $ GOOS=linux   go build ./internal/storage/ && echo linux_PASS
  $ GOOS=darwin  go build ./internal/storage/ && echo darwin_PASS
  $ GOOS=windows go build ./internal/storage/ && echo windows_PASS
  EXPECT: three *_PASS lines

CORRECTED_CONSTANT_NOT_400:      # A1 — the 10× bug must be gone
  $ grep -c "ChunksPerGB\|(1 << 30) / \|4096" internal/storage/*.go cmd/provider/main.go
  EXPECT: >= 1
  $ grep -nE "declared_storage_gb \* 400|declaredStorageGB \* 400|\* 400 \* 200" cmd/provider/main.go internal/storage/*.go
  EXPECT: no matches            # the erroneous *400 formula must not appear

REQUIRED_MB_TABLE_CORRECT:
  $ go test -v -run TestDHTCacheRAMFormula ./internal/storage/
  EXPECT: exit 0; asserts 50GB->40, 200GB->160, 500GB->~400

DOES_NOT_HALT_ON_SHORTFALL:
  $ grep -c "os.Exit\|log.Fatal" cmd/provider/main.go | head -1
  # (RAM shortfall path must WARN, not exit — verify manually the WARN branch has no Fatal)
  EXPECT: reviewer-confirmed: no Fatal on the RAM-shortfall branch

RAM_CONSTRAINED_METRIC_SET:
  $ grep -c "vyomanaut_daemon_ram_constrained\|RamConstrained" cmd/provider/main.go
  EXPECT: >= 1

OTHER_STUB_EXISTS:               # build-tag hygiene (skill §3)
  $ test -f internal/storage/memcheck_other.go && echo PASS || echo FAIL

UNIT_TESTS:
  $ go test -v -run TestRAMCheck ./cmd/provider/
  EXPECT: exit 0; tests include:
    TestRAMCheckWarnsAndReducesDeclaredStorageOnShortfall
    TestRAMCheckDoesNotHalt
```

---

## Milestone 14 — Vetting & Synthetic Chunks (`internal/vettingchunk`)

**Deliverable:** synthetic-chunk lifecycle — generation, upload, GC delivery on ACTIVE transition, departure cleanup. Enforces Invariant 6 end-to-end.

**Reference:** IC §5.10 (`Generator`, `GCDelivery`, sentinels), IC §4.5 (client side), DM §3 Invariant 6, DM §4.5. Import scope (IC §9): `internal/vettingchunk → config, crypto, storage, p2p` (+ `metrics` if A2/Option A is adopted).

---

### Phase 14.1 — Synthetic chunk generator

#### Session 14.1.1 — Implement `Generator`

**PRECONDITIONS** — `internal/vettingchunk/doc.go` stub exists; upload client (M15) or a shared upload helper available.

**TASK** — implement `GenerateChunk()` per IC §5.10:

1. 256 KB via `crypto/rand` — not retained after confirmation.
2. `chunkID = SHA-256(data)`.
3. Upload via `/vyomanaut/chunk-upload/1.0.0`.
4. On success: INSERT `chunk_assignments` with `is_vetting_chunk = TRUE`, `segment_id = NULL`, `shard_index = NULL` (DM §4.5, Invariant 6).
   Cap: `Cap(declaredStorageGB) = floor(declaredStorageGB × 400)` (the vetting cap — **distinct** from the RAM formula constant in 13.6.1; do not conflate, per A1). Caller checks `CurrentCount < Cap` before calling.

**FILES** — `internal/vettingchunk/generator.go`, `internal/vettingchunk/errors.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/vettingchunk/
  EXPECT: exit 0

IMPORT_CONSTRAINTS:              # IC §9 — must NOT import scoring/repair/payment
  $ grep -cE "internal/(scoring|repair|payment)" internal/vettingchunk/*.go
  EXPECT: 0

CRYPTO_RAND_USED:
  $ grep -c "crypto/rand" internal/vettingchunk/generator.go
  EXPECT: >= 1

INVARIANT6_INSERT_SHAPE:        # vetting rows: segment_id NULL, shard_index NULL, is_vetting TRUE
  $ grep -c "is_vetting_chunk" internal/vettingchunk/generator.go
  EXPECT: >= 1
  $ grep -c "NULL\|nil" internal/vettingchunk/generator.go
  EXPECT: >= 1

CAP_FORMULA:
  $ grep -c "400" internal/vettingchunk/generator.go
  EXPECT: >= 1                   # vetting cap = floor(GB*400); correct here (cf. A1)

UNIT_TESTS:
  $ go test -v -run TestGenerator ./internal/vettingchunk/
  EXPECT: exit 0; tests include:
    TestGenerateChunkProduces256KBRandom
    TestGeneratedRowIsVettingWithNullSegmentAndShard   # Invariant 6
    TestCapIsFloorGBTimes400

VET:
  $ go vet ./internal/vettingchunk/
  EXPECT: exit 0; zero output
```

---

### Phase 14.2 — GC delivery

#### Session 14.2.1 — Implement `DeliverGCInstruction()`

**Reference:** IC §5.10 (`GCDelivery`, `ErrProviderOffline`), IC §4.5 (client side).

**PRECONDITIONS** — triggered immediately after `providers.status → ACTIVE`. p2p `Host` injected by the M12 entrypoint (vettingchunk never imports `cmd/`).

**TASK** *(base — matches current IC §4.5)*

1. Query synthetic IDs: `WHERE is_vetting_chunk = TRUE AND provider_id = $1 AND status = 'ACTIVE'`.
2. Batch ≤ 10 000 IDs/frame.
3. Open `/vyomanaut/vetting-gc/1.0.0`.
4. Per frame: send `VettingGCRequest`, await `VettingGCResponse` ≤ 30 000 ms.
5. `0x00` → mark batch `DELETED`.
6. `0x01` → retry failed entries next connection.
7. Provider offline → all rows `PENDING_DELETION`, return `ErrProviderOffline`.
   Backoff: `profile.GCRetryBackoff` (prod `[5m,15m,60m]`, demo `[10s,30s,2m]` — never hardcoded).

**⛔ SECURITY ADDENDUM — apply after ADR-036 accepted (A3):** when building `VettingGCRequest`, append `request_ts_ms`(8) and `gc_auth_sig`(64) = `Ed25519(microservice_signing_key, SHA-256(chunk_ids ‖ request_ts_ms ‖ microservice_peer_id))`. Handle new response codes `0x03 NOT_AUTHORISED` / `0x04 STALE_REQUEST` (log + alert; do not mark `DELETED`).

**FILES** — `internal/vettingchunk/gc.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/vettingchunk/
  EXPECT: exit 0

QUERY_FILTERS_VETTING_ACTIVE:
  $ grep -c "is_vetting_chunk = TRUE" internal/vettingchunk/gc.go
  EXPECT: >= 1

BATCH_10K:
  $ grep -c "10_000\|10000" internal/vettingchunk/gc.go
  EXPECT: >= 1

TIMEOUT_30S:
  $ grep -c "30_000\|30000\|30 \* time.Second" internal/vettingchunk/gc.go
  EXPECT: >= 1

BACKOFF_FROM_PROFILE_NOT_HARDCODED:
  $ grep -c "profile.GCRetryBackoff" internal/vettingchunk/gc.go
  EXPECT: >= 1
  $ grep -cE "5 \* time.Minute|15 \* time.Minute|60 \* time.Minute" internal/vettingchunk/gc.go
  EXPECT: 0                       # values live in NetworkProfile, never inline

OFFLINE_SETS_PENDING_DELETION:
  $ grep -c "PENDING_DELETION\|ErrProviderOffline" internal/vettingchunk/gc.go
  EXPECT: >= 2

SENTINEL_NOT_INLINE:            # errors.Is pattern; no inline errors.New at call site
  $ grep -c "errors.New" internal/vettingchunk/gc.go
  EXPECT: 0

UNIT_TESTS:
  $ go test -v -run TestDeliverGC ./internal/vettingchunk/
  EXPECT: exit 0; tests include:
    TestDeliverGCMarksDeletedOn0x00
    TestDeliverGCPartialFailureRetriesFailedOnly
    TestDeliverGCOfflineSetsPendingDeletionReturnsErrProviderOffline
    TestDeliverGCBackoffFromProfile

# ── VERIFY (enable after ADR-036 accepted) ──
SIGNS_GC_REQUEST:
  $ grep -c "gc_auth_sig\|gcAuthSig\|ed25519.Sign" internal/vettingchunk/gc.go
  EXPECT: >= 1
HANDLES_NEW_STATUS_CODES:
  $ grep -cE "0x03|0x04|NOT_AUTHORISED|STALE_REQUEST" internal/vettingchunk/gc.go
  EXPECT: >= 1
```

---

## Milestone 15 — Client SDK (`internal/client`)

**Deliverable:** Account management, upload orchestrator, retrieval orchestrator, **and
file management** (Phase 15.4 — new). The client SDK does NOT import `cmd/` (IC §9; see
Design Correction A-2 for the corrected positive-space import list).

**Reference:** IC §5.9 (`UploadOrchestrator` interface, ERRATA note, all sentinel errors),
IC §4.1 (upload stream — client initiator side), IC §11 (forbidden: `json.Marshal` in any
Ed25519 signing path — see A-6; no business logic in `cmd/`), `mvp.md` §8.2 (authoritative
per-file inventory — every `FILES:` block below is copied from it verbatim, see A-7), `mvp.md`
§8.3 (`cmd/client` subcommand table, corrected per A-5)

---

### Phase 15.1 — Account Management

**Reference:** IC §5.1 (crypto primitives used here — `DeriveMasterSecret`,
`DeriveKeystoreEncKey`, `MasterSecretToMnemonic`/`MnemonicToMasterSecret`,
`SelectConfirmationWords`), `mvp.md` §8.2 (`internal/client/account/` file inventory)

#### Session 15.1.1 — Implement registration, mnemonic gate, keystore, and recovery

**PRECONDITIONS** — `internal/client/account/doc.go` stub exists (*"Registration, BIP-39
mnemonic, Session key derivation, Keystore"*); `internal/crypto` (M2) and `internal/config`
(M1) complete.

**TASK:**

1. `register.go` — `Register()`: generate an Ed25519 key pair; derive the master secret via
   `crypto.DeriveMasterSecret(passphrase, ownerID, profile.Argon2Time, profile.Argon2Memory,
   profile.Argon2Threads)` — **parameters always come from the active `NetworkProfile`, never
   hardcoded** (IC §5.1's own caller-responsibility note, ADR-031).
2. `mnemonic.go` — generate the 24-word BIP-39 phrase via `crypto.MasterSecretToMnemonic`;
   display it; run `crypto.SelectConfirmationWords()` and gate on the two returned indices
   **unless** `profile.SkipMnemonicConfirm == true` (demo mode) — the gate function is still
   called in demo mode (IC §5.1's own note: *"this function may still be called; the caller
   simply does not block on user input"*), only the blocking prompt is skipped.
3. `master_secret.go` — the UI gate: holds the derived master secret in memory only,
   never persisted, and is the single call site every other client package (`upload`,
   `retrieve`, `manage`) receives it from as a parameter — no package below this one reads
   `master_secret.go`'s internals directly.
4. `keystore.go` — encrypt the Ed25519 key and the pointer-file AEAD nonce counter (IC §2.5's
   monotone counter) under `crypto.DeriveKeystoreEncKey(masterSecret, ownerID)`.
5. `recover.go` — two recovery paths: passphrase (re-run step 1's derivation) and mnemonic
   (`crypto.MnemonicToMasterSecret(words)` — on `crypto.ErrInvalidMnemonic`, surface *"Invalid
   recovery phrase — please check your words and try again"* without indicating which word
   failed, per IC §5.1's explicit timing-oracle warning).
**FILES** — `internal/client/account/register.go`, `master_secret.go`, `mnemonic.go`,
`keystore.go`, `recover.go`, `account_test.go` (`mvp.md` §8.2 — see A-7)

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/client/account/
  EXPECT: exit 0
 
IMPORT_CONSTRAINTS:                # IC §9 corrected (A-2): config+crypto only, no cmd/, no p2p/erasure needed here
  $ grep -cE "Vyomanaut_V2/cmd|Vyomanaut_V2/internal/(p2p|erasure|storage|audit|scoring|repair|payment)" internal/client/account/*.go
  EXPECT: 0
 
ARGON2_PARAMS_FROM_PROFILE_NOT_HARDCODED:
  $ grep -c "profile.Argon2Time\|profile.Argon2Memory\|profile.Argon2Threads" internal/client/account/register.go
  EXPECT: >= 3
  $ grep -cE "argon2Time\s*=\s*[0-9]|argon2Memory\s*=\s*[0-9]{4,}" internal/client/account/register.go
  EXPECT: 0                        # no inline literal cost parameters
 
CONFIRMATION_GATE_SKIPPED_ONLY_IN_DEMO:
  $ grep -c "profile.SkipMnemonicConfirm" internal/client/account/mnemonic.go
  EXPECT: >= 1
  $ grep -c "SelectConfirmationWords(" internal/client/account/mnemonic.go
  EXPECT: >= 1                     # called in both modes, per IC §5.1
 
KEYSTORE_ENCRYPTS_ED25519_AND_NONCE_COUNTER:
  $ grep -c "DeriveKeystoreEncKey" internal/client/account/keystore.go
  EXPECT: >= 1
  $ grep -cE "nonceCounter|noncCounter|pointerNonce" internal/client/account/keystore.go
  EXPECT: >= 1
 
RECOVERY_NEVER_LEAKS_WHICH_WORD_FAILED:   # NFR-class timing-oracle guard, IC §5.1
  $ grep -nE "word\[|words\[[0-9]" internal/client/account/recover.go | grep -i "err\|invalid"
  EXPECT: no matches
  $ grep -c "ErrInvalidMnemonic" internal/client/account/recover.go
  EXPECT: >= 1
 
MASTER_SECRET_NEVER_WRITTEN_TO_DISK:
  $ grep -cE "os.WriteFile|ioutil.WriteFile" internal/client/account/master_secret.go
  EXPECT: 0
 
UNIT_TESTS:
  $ go test -v -run TestAccount ./internal/client/account/
  EXPECT: exit 0; tests include:
    TestRegisterUsesProfileArgon2Params           # fails if params are hardcoded
    TestRegisterSkipsBlockingPromptInDemoOnly
    TestRegisterStillCallsSelectConfirmationWordsInDemo
    TestKeystoreRoundTripsEd25519KeyAndNonceCounter
    TestRecoverFromPassphraseReproducesMasterSecret
    TestRecoverFromMnemonicReproducesMasterSecret
    TestRecoverInvalidMnemonicDoesNotIdentifyFailingWord
 
VET:
  $ go vet ./internal/client/account/
  EXPECT: exit 0; zero output
```

---

### Phase 15.2 — Upload Orchestrator

**Reference:** IC §5.9 (`UploadOrchestrator` ERRATA note — capability tokens; `ErrInsufficientEscrow`,
`ErrNetworkNotReady`, `ErrUploadIncomplete`), IC §4.1 (chunk upload stream, client side — capability
token verification steps, frame sizes, status codes), `mvp.md` §8.2 (file split), OAS
`ShardAssignment.capability_token`, `SegmentAssignment`, `UploadAssignResponse`,
`FileRegisterRequest` (owner_sig fixed per A-6), `PointerFilePlaintextSegment`

#### Session 15.2.1 — Implement `UploadFile()` with capability-token handling

**TASK:**

1. `orchestrator.go` — segment the file, padding to `profile.DataShards × profile.ShardSize`
   minimum (never hardcode `16` or `262144` — read both from the active profile: demo uses
   `DataShards=3`).
2. AONT-encode each segment via `crypto.AONTEncodeSegment()`; RS-encode via
   `erasure.Engine.EncodeSegment()`.
3. `assign.go` — `POST /api/v1/upload/assign`; on HTTP 503 return `ErrNetworkNotReady`
   (readiness gate closed); on HTTP 409 return `ErrInsufficientEscrow` (IC §5.9
   pre-condition — 30-day escrow check enforced server-side).
4. **ERRATA (IC §5.9), in `transfer.go`:** include each `capability_token` verbatim (hex,
   144 chars per OAS `ShardAssignment.capability_token` pattern) in the corresponding
   `UploadRequest` frame — never re-derive or reconstruct it client-side. On `0x07
   CAPABILITY_EXPIRED`, re-call `assign.go`'s assignment request with the same `file_id`
   (idempotent — same providers, fresh tokens).
5. `transfer.go` — upload all shards in parallel (bounded goroutine pool); collect
   `provider_sig` receipts.
6. `pointer.go` — construct the `PointerFilePlaintextSegment` structure (OAS — informational
   schema, never transmitted; client-side reference only), AEAD-encrypt it, and
   `POST /api/v1/file/register`. Populate `display_name_ciphertext`/`_nonce`/`_tag` from the
   locally-encrypted filename (FR-019) — key = `HKDF-SHA256(master_secret, salt=owner_id,
   info="vyomanaut-filename-v1" ‖ file_id)`.
7. **`owner_sig` construction — fixed-layout, per A-6, NOT canonical JSON:**
   `owner_sig_input = SHA-256("vyomanaut-file-register-v1" ‖ file_id ‖
   SHA-256(pointer_ciphertext) ‖ pointer_nonce ‖ pointer_tag ‖ original_size_bytes(8,BE) ‖
   display_name_present(1) ‖ SHA-256(display_name_ciphertext‖display_name_nonce‖display_name_tag) ‖
   schema_version(4,BE))`; `owner_sig = Ed25519_sign(...)`. This is the corrected replacement
   for the OAS's current "canonical JSON, keys sorted" description — the OAS field
   description is corrected in the same PR (A-6).
8. `session.go` — persist session state for resume (FR-060): `file_id`, `chunk_ids`,
   `ack_status[TotalShards]`, cleaned up only after step 6 succeeds.
**FILES** — `internal/client/upload/orchestrator.go`, `session.go`, `assign.go`,
`transfer.go`, `pointer.go`, `upload_test.go` (`mvp.md` §8.2 — see A-7)

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/client/upload/
  EXPECT: exit 0
 
IMPORT_CONSTRAINTS:                # IC §9 corrected (A-2)
  $ grep -c "Vyomanaut_V2/cmd" internal/client/upload/*.go
  EXPECT: 0
 
SEGMENTATION_USES_PROFILE_NOT_LITERALS:
  $ grep -c "profile.DataShards\|profile.ShardSize" internal/client/upload/orchestrator.go
  EXPECT: >= 2
  $ grep -cE "\* 16\b|262144\b" internal/client/upload/orchestrator.go
  EXPECT: 0
 
CAPABILITY_TOKEN_INCLUDED_VERBATIM:
  $ grep -c "capability_token\|CapabilityToken" internal/client/upload/transfer.go
  EXPECT: >= 1
  $ grep -cE "signing_input|reconstructToken|deriveToken" internal/client/upload/transfer.go
  EXPECT: 0                        # client must never attempt to re-derive a token
 
CAPABILITY_EXPIRED_TRIGGERS_IDEMPOTENT_REASSIGN:
  $ grep -c "0x07\|CAPABILITY_EXPIRED" internal/client/upload/transfer.go
  EXPECT: >= 1
  $ grep -c "file_id" internal/client/upload/assign.go
  EXPECT: >= 1
 
ESCROW_AND_READINESS_ERRORS_MAPPED:
  $ grep -c "409\|ErrInsufficientEscrow" internal/client/upload/assign.go
  EXPECT: >= 1
  $ grep -c "503\|ErrNetworkNotReady" internal/client/upload/assign.go
  EXPECT: >= 1
 
DISPLAY_NAME_POPULATED_WITH_CORRECT_HKDF_INFO:
  $ grep -c "vyomanaut-filename-v1" internal/client/upload/pointer.go
  EXPECT: >= 1
  $ grep -c "display_name_ciphertext" internal/client/upload/pointer.go
  EXPECT: >= 1
 
OWNER_SIG_IS_FIXED_LAYOUT_NOT_JSON:              # A-6 — critical fix
  $ grep -c "json.Marshal" internal/client/upload/pointer.go
  EXPECT: 0
  $ grep -c "vyomanaut-file-register-v1" internal/client/upload/pointer.go
  EXPECT: >= 1
  $ grep -c "sha256.Sum256" internal/client/upload/pointer.go
  EXPECT: >= 2                     # ciphertext hash + display-name-block hash, at minimum
 
SESSION_STATE_FIELDS_PRESENT_FR060:
  $ grep -cE "file_id|chunk_ids|ack_status" internal/client/upload/session.go
  EXPECT: >= 3
  $ grep -c "TotalShards" internal/client/upload/session.go
  EXPECT: >= 1
 
SESSION_STATE_CLEANED_UP_ONLY_AFTER_REGISTER:
  $ awk '/file\/register|FileRegisterRequest/{r=NR} /os.Remove|session.*[Cc]leanup|DeleteSessionState/{c=NR} END{print (r>0 && c>0 && r<c)?"PASS":"FAIL"}' internal/client/upload/orchestrator.go internal/client/upload/pointer.go internal/client/upload/session.go
  EXPECT: PASS
 
UNIT_TESTS:
  $ go test -v -run TestUpload ./internal/client/upload/
  EXPECT: exit 0; tests include:
    TestUploadFileSegmentsPadUsingProfileNotLiterals
    TestUploadIncludesCapabilityTokenVerbatimPerShard
    TestUploadReassignsIdempotentlyOnCapabilityExpired
    TestUploadMapsHTTP409ToErrInsufficientEscrow
    TestUploadMapsHTTP503ToErrNetworkNotReady
    TestUploadPopulatesDisplayNameCiphertextFromMasterSecret
    TestOwnerSigUsesFixedByteLayoutNotJSON              # A-6
    TestUploadPersistsSessionStateForResume
    TestUploadCleansUpSessionStateOnlyAfterRegisterSucceeds
 
VET:
  $ go vet ./internal/client/upload/
  EXPECT: exit 0; zero output
```

#### Session 15.2.2 — Implement `ResumeUpload()`

**TASK:** Load persisted session state (`session.go`); identify unacknowledged shards from
`ack_status[TotalShards]`; re-upload only those via `transfer.go` (FR-060 — never
retransmit acknowledged shards, IC §5.9); on completion, follow the same pointer-registration
and cleanup path as Session 15.2.1 step 6–8.

**FILES** — `internal/client/upload/session.go` (edit — resume entry point), `orchestrator.go`
(edit — `ResumeUpload()` per IC §5.9 signature)

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/client/upload/
  EXPECT: exit 0
 
RESUME_SKIPS_ACKNOWLEDGED_SHARDS:
  $ grep -c "ack_status\[" internal/client/upload/orchestrator.go internal/client/upload/session.go
  EXPECT: >= 1
  $ grep -cE "range.*ack_status|for.*acked" internal/client/upload/orchestrator.go
  EXPECT: >= 1
 
RESUME_REUSES_UPLOAD_TRANSFER_PATH:               # no duplicated transfer logic
  $ grep -c "transfer\." internal/client/upload/orchestrator.go
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -v -run TestResumeUpload ./internal/client/upload/
  EXPECT: exit 0; tests include:
    TestResumeUploadRetransmitsOnlyUnacknowledgedShards
    TestResumeUploadReusesExistingSessionID
    TestResumeUploadCompletesRegistrationOnSuccess
 
VET:
  $ go vet ./internal/client/upload/
  EXPECT: exit 0; zero output
```

---

### Phase 15.3 — Retrieval Orchestrator

**Reference:** IC §5.9 (`RetrieveFile`, `ErrPointerTagMismatch`, `ErrTooFewShards`,
`ErrCanaryMismatch`), IC §5.1 (`DecryptPointerFile`, `AONTDecodePackage` — both with
explicit "caller MUST NOT return any plaintext" contracts), `mvp.md` §8.2 (file split),
OAS `PointerFilePlaintextSegment`, FR-016 (fastest-16 parallel-fetch-with-cancel), FR-018
(canary enforcement)

#### Session 15.3.1 — Implement `RetrieveFile()`

**TASK:**

1. `pointer.go` — fetch pointer ciphertext (`GET /api/v1/file/{file_id}/pointer`); derive
   the pointer key via `crypto.DerivePointerEncKey()`; call `crypto.DecryptPointerFile()` —
   on `crypto.ErrTagMismatch`, return `client.ErrPointerTagMismatch` **with no plaintext
   returned under any circumstance** (IC §5.1, IC §5.9).
2. `download.go` — for each segment: dial all `TotalShards` providers in parallel via
   libp2p; cancel remaining dials once `profile.DataShards` valid responses are received
   (FR-016's "fastest-responding, cancel-on-k" pattern — **never hardcode `16`**; demo mode
   cancels at 3). Verify each shard's content address (`SHA-256(shard_data) == chunk_id`)
   before it is handed to RS decode.
3. `decode.go` — RS-decode via `erasure.Engine.DecodeSegment()`; `crypto.AONTDecodePackage()`
   — on `crypto.ErrCanaryMismatch`: **zero the buffer and return `client.ErrCanaryMismatch`**
   (IC §5.1, IC §5.9 — "caller MUST NOT return any plaintext"); strip padding to
   `original_size_bytes`.
4. `orchestrator.go` — concatenate segments in `segment_index` order; if fewer than
   `profile.DataShards` providers are reachable for any segment after retry, return
   `client.ErrTooFewShards`.
**FILES** — `internal/client/retrieve/orchestrator.go`, `pointer.go`, `download.go`,
`decode.go`, `retrieve_test.go` (`mvp.md` §8.2 — see A-7)

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/client/retrieve/
  EXPECT: exit 0
 
IMPORT_CONSTRAINTS:
  $ grep -c "Vyomanaut_V2/cmd" internal/client/retrieve/*.go
  EXPECT: 0
 
TAG_MISMATCH_RETURNS_NO_PLAINTEXT:                # order-sensitive — check happens before any use
  $ awk '/DecryptPointerFile/{d=NR} /ErrPointerTagMismatch/{e=NR} END{print (d>0 && e>0)?"PASS":"FAIL"}' internal/client/retrieve/pointer.go
  EXPECT: PASS
  $ grep -A5 "ErrTagMismatch" internal/client/retrieve/pointer.go | grep -c "return.*plaintext\|return.*data,"
  EXPECT: 0                         # the error branch must not return decrypted bytes
 
CANCEL_AFTER_PROFILE_DATASHARDS_NOT_HARDCODED_16:
  $ grep -c "profile.DataShards" internal/client/retrieve/download.go
  EXPECT: >= 1
  $ grep -cE "== 16\b|>= 16\b" internal/client/retrieve/download.go
  EXPECT: 0
 
CONTENT_ADDRESS_VERIFIED_BEFORE_RS_DECODE:
  $ awk '/sha256.Sum256|SHA-256/{h=NR} /DecodeSegment/{d=NR} END{print (h>0 && d>0 && h<d)?"PASS":"FAIL"}' internal/client/retrieve/download.go internal/client/retrieve/decode.go
  EXPECT: PASS
 
CANARY_MISMATCH_ZEROES_BUFFER_BEFORE_RETURN:
  $ grep -c "ErrCanaryMismatch" internal/client/retrieve/decode.go
  EXPECT: >= 1
  $ grep -B3 "ErrCanaryMismatch" internal/client/retrieve/decode.go | grep -cE "zero|clear|for.*range.*= 0"
  EXPECT: >= 1
 
TOO_FEW_SHARDS_SENTINEL_PRESENT:
  $ grep -c "ErrTooFewShards" internal/client/retrieve/orchestrator.go
  EXPECT: >= 1
 
SEGMENTS_CONCATENATED_IN_INDEX_ORDER:
  $ grep -cE "segment_index|SegmentIndex" internal/client/retrieve/orchestrator.go
  EXPECT: >= 1
  $ grep -c "sort\." internal/client/retrieve/orchestrator.go
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -v -run TestRetrieve ./internal/client/retrieve/
  EXPECT: exit 0; tests include:
    TestRetrieveTagMismatchReturnsNoPlaintext
    TestRetrieveCancelsAfterProfileDataShardsValidResponses
    TestRetrieveVerifiesContentAddressBeforeDecode
    TestRetrieveCanaryMismatchZeroesBufferAndReturnsSentinel
    TestRetrieveTooFewShardsReturnsSentinel
    TestRetrieveConcatenatesSegmentsInOrder
    TestRetrieveStripsPaddingToOriginalSizeBytes
 
VET:
  $ go vet ./internal/client/retrieve/
  EXPECT: exit 0; zero output
```

---

### Phase 15.4 — File & Account Management *(new phase — closes A-4/A-5)*

**Reference:** `mvp.md` §8.2 (`internal/client/manage/**files.go, delete.go, escrow.go` —
already specified, never built until now), `mvp.md` §8.3 (`ls`, `rm`, `balance`, `deposit`
subcommands, `withdraw` added per A-5), OAS `GET /api/v1/owner/{owner_id}/files`,
`DELETE /api/v1/file/{file_id}`, `GET /api/v1/owner/{owner_id}/balance`,
`GET /api/v1/owner/{owner_id}/escrow`, `POST /api/v1/owner/deposit`,
`POST /api/v1/owner/withdraw`, FR-019/FR-020/FR-021/FR-059, IC §14 (user-facing copy —
pending merge per ADR-034; referenced here as "pending" not "authoritative today"),
ADR-035 (`intent_url` — pending merge; see PRECONDITIONS below)

#### Session 15.4.1 — Implement file list and delete (`ls`, `rm`)

**TASK:**

1. `files.go` — `GET /api/v1/owner/{owner_id}/files`; decrypt `display_name_ciphertext`
   locally when present (same HKDF derivation as Session 15.2.1 step 6 — `info=
   "vyomanaut-filename-v1"‖file_id`), else display `file_id`. Render the `availability`
   enum (`OK`/`DEGRADED`/`CRITICAL`) through the **canonical label mapping from IC §14.2**
   — never the raw enum string directly to the user. **PRECONDITION:** IC §14.2 is not yet
   merged into `interface-contracts.md` (per prior session — filed locally as
   `interface-contracts-section-14.md`, pending push). Until it merges, this session
   implements the mapping call site as a named, swappable function
   (`copy.AvailabilityLabel(status string) string`) with the three enum values as
   placeholder pass-through strings, so merging §14.2 later is a one-file diff, not a
   re-implementation.
2. `delete.go` — `DELETE /api/v1/file/{file_id}`; surface `assignments_marked`,
   `providers_notified`, `providers_pending` from the response (FR-020's own
   pending-provider retry language — this is not an error state, it's expected steady-state
   text). Treat HTTP 409 `FILE_ALREADY_DELETED` as an idempotent success on the client side
   (the file is, from the data owner's point of view, in the state they asked for).
**FILES** — `internal/client/manage/files.go`, `delete.go`, `manage_test.go` (`mvp.md` §8.2
inventory has no explicit test file for this package — added here for consistency with
every other package's convention, which always pairs implementation files with a `_test.go`)

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/client/manage/
  EXPECT: exit 0
 
IMPORT_CONSTRAINTS:
  $ grep -c "Vyomanaut_V2/cmd" internal/client/manage/*.go
  EXPECT: 0
 
DISPLAY_NAME_DECRYPTED_WITH_SAME_HKDF_INFO_AS_UPLOAD:
  $ grep -c "vyomanaut-filename-v1" internal/client/manage/files.go
  EXPECT: >= 1
 
AVAILABILITY_NEVER_RENDERED_AS_RAW_ENUM:
  $ grep -c "copy.AvailabilityLabel\|AvailabilityLabel(" internal/client/manage/files.go
  EXPECT: >= 1
  $ grep -nE 'fmt\.Print.*"OK"|fmt\.Print.*"DEGRADED"|fmt\.Print.*"CRITICAL"' internal/client/manage/files.go
  EXPECT: no matches                 # raw enum string must never reach a Print/output call directly
 
DELETE_SURFACES_PROVIDER_NOTIFICATION_COUNTS:
  $ grep -cE "assignments_marked|providers_notified|providers_pending" internal/client/manage/delete.go
  EXPECT: >= 3
 
DELETE_409_TREATED_AS_IDEMPOTENT_SUCCESS:
  $ grep -c "FILE_ALREADY_DELETED" internal/client/manage/delete.go
  EXPECT: >= 1
  $ grep -A3 "FILE_ALREADY_DELETED" internal/client/manage/delete.go | grep -cE "return nil|success"
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -v -run "TestFileList|TestFileDelete" ./internal/client/manage/
  EXPECT: exit 0; tests include:
    TestFileListDecryptsDisplayNameWhenPresent
    TestFileListFallsBackToFileIDWhenNamePlaintextAbsent
    TestFileListRendersAvailabilityThroughLabelMapping
    TestFileDeleteSurfacesProviderNotificationCounts
    TestFileDeleteTreats409AsIdempotentSuccess
 
VET:
  $ go vet ./internal/client/manage/
  EXPECT: exit 0; zero output
```

#### Session 15.4.2 — Implement escrow balance, deposit, and withdrawal (`balance`, `deposit`, `withdraw`)

**Reference:** FR-021 (balance view — current balance, 30-day reserved, available,
transaction history), FR-059 (withdrawal — idempotency key `SHA-256(owner_id +
withdrawal_request_id)`, blocked while any upload is in-flight), ADR-035 (`intent_url`
construction — **pending merge**, see PRECONDITIONS)

**PRECONDITIONS** — `docs/api/openapi.yaml`'s live `DepositInitiateResponse` currently
returns only `vpa`/`qr_code_url`/`expires_at`; it does **not** yet include `intent_url`
(confirmed against the live OAS — ADR-035 is Proposed, not merged). This session builds
`deposit`'s rendering path against **both** possible response shapes: if `intent_url` is
present, render it as the primary, copyable output (ADR-035 §3's specified client
behavior); if absent, fall back to printing `vpa` and the `qr_code_url` link, so this
session does not block on ADR-035 landing first, and requires no rework once it does.

**TASK:**

1. `escrow.go` (`balance`) — `GET /api/v1/owner/{owner_id}/balance` → render `balance_paise`,
   `reserved_next_30d_paise`, `available_paise` (OwnerBalance; all integer paise, never
   floating point client-side either — consistent with IC §11's payment-path rule, applied
   here even though this is a read-only client display, not `internal/payment` itself).
2. `escrow.go` (`deposit`) — `POST /api/v1/owner/deposit`; render per the PRECONDITIONS
   fallback logic above.
3. `escrow.go` (`withdraw`, new per A-5) — `POST /api/v1/owner/withdraw`; idempotency key
   `SHA-256(owner_id + withdrawal_request_id)`, generated client-side once per withdrawal
   attempt and reused verbatim on retry (never regenerated on retry — a fresh key on retry
   would defeat the idempotency guarantee FR-059 exists for). Surface the HTTP 409 case
   (blocked while an upload is in-flight) as an explicit, actionable message, not a generic
   error.
**FILES** — `internal/client/manage/escrow.go`, `manage_test.go` (edit — extends Session
15.4.1's test file)

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/client/manage/
  EXPECT: exit 0
 
BALANCE_RENDERS_ALL_THREE_FIGURES_INTEGER_PAISE:
  $ grep -cE "balance_paise|reserved_next_30d_paise|available_paise" internal/client/manage/escrow.go
  EXPECT: >= 3
  $ grep -cE "float32|float64" internal/client/manage/escrow.go
  EXPECT: 0
 
DEPOSIT_HANDLES_BOTH_RESPONSE_SHAPES:
  $ grep -c "intent_url" internal/client/manage/escrow.go
  EXPECT: >= 1
  $ grep -c "qr_code_url" internal/client/manage/escrow.go
  EXPECT: >= 1
 
WITHDRAW_IDEMPOTENCY_KEY_STABLE_ACROSS_RETRY:
  $ grep -c "withdrawal_request_id" internal/client/manage/escrow.go
  EXPECT: >= 1
  $ grep -cE "sha256.Sum256\(.*owner_id.*withdrawal_request_id\)|SHA-256.*owner_id.*withdrawal" internal/client/manage/escrow.go
  EXPECT: >= 1
  $ grep -nE "uuid.New\(\)|rand\." internal/client/manage/escrow.go | grep -i retry
  EXPECT: no matches                # key must not be regenerated inside a retry loop
 
WITHDRAW_409_BLOCKED_WHILE_UPLOAD_INFLIGHT_SURFACED:
  $ grep -c "409" internal/client/manage/escrow.go
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -v -run TestEscrow ./internal/client/manage/
  EXPECT: exit 0; tests include:
    TestBalanceRendersAllThreeFiguresAsIntegerPaise
    TestDepositRendersIntentURLWhenPresent
    TestDepositFallsBackToVPAAndQRWhenIntentURLAbsent
    TestWithdrawReusesSameIdempotencyKeyOnRetry
    TestWithdrawSurfaces409AsActionableMessage
 
VET:
  $ go vet ./internal/client/manage/
  EXPECT: exit 0; zero output
```

---

## Milestone 16 — Demo Mode Validation

**Deliverable:** A fully runnable demo that completes the 30-minute lifecycle from `mvp.md`
§3.6, **on the platform it is committed to shipping on first** (Windows —
`ux-decisions.md` §7). `go test` validates all demo-specific behaviours and mode-invariant
properties; a native `windows-latest` CI run validates that the demo path is not
Linux/macOS-only.

**Reference:** `mvp.md` §3.6 (demo timeline), §7 (viability fact-checks), §6 (switching
requirements), §8.4 (CI checks 14–15); ADR-046 (Windows storage engine — Phase 16.0, new)

---

### Phase 16.0 — Cross-Platform Storage Prerequisite *(new phase — closes A-8)*

**Reference:** ADR-046 (Provider Storage Engine on Windows: BadgerDB, Not RocksDB/CGo)

> **Why this phase exists here and not in Milestone 5.** `internal/storage` is an M5
> deliverable, already merged, with no `//go:build` tags on its RocksDB import — confirmed
> against the live repo. It compiles today only because nothing has yet tried to
> cross-compile it for Windows. Milestone 16 is the first point in the build plan where
> that stops being true: Session 16.2.1's `--sim-count` harness is explicitly meant to run
> "at a hackathon or investor meeting" (`mvp.md` §3.1), and the product's committed
> platform order ships Windows first. This phase is sequenced as **16.0**, before 16.1/16.2,
> specifically so it does not renumber the two already-cross-referenced existing phases —
> it is a prerequisite insert, not a milestone restructure.

#### Session 16.0.1 — Retrofit `internal/storage` with build-tag-selected engines

**PRECONDITIONS** — `internal/storage/index.go` currently imports `github.com/linxGnu/grocksdb`
unconditionally (no build tag on any file in the package). ADR-046 is approved for
implementation per Karma's direction on this document.

**TASK:**

1. Rename `internal/storage/index.go` → `internal/storage/engine_rocksdb.go`; add
   `//go:build linux || darwin` (ADR-046 §1). No logic change — this is a pure rename +
   tag, the existing RocksDB implementation and its CI-proven Linux/macOS build are
   unchanged.
2. Create `internal/storage/engine_badger.go`, `//go:build windows`, implementing the same
   internal surface `engine_rocksdb.go` exposes (`AppendChunk`, `LookupChunk`, `DeleteChunk`,
   `RunGC`, `RecoverFromCrash`, `Close`) against `badger.DB`, configured per ADR-046 §2:
   `WithValueThreshold(1024)`, `WithBloomFalsePositive(0.01)`, `WithBlockCacheSize(64<<20)`,
   `WithCompression(options.None)`. `content_hash` verification (`SHA-256(chunk_data)`
   before every audit response) is unchanged — it is application logic on top of `Get`/`Set`,
   identical on both engines (ADR-046 §2).
3. Edit `store.go`'s `NewChunkStore` so it calls a build-tag-satisfied constructor (the same
   function name on both engine files, e.g. `openIndex(dbPath)`) rather than
   `openRocksDBIndex` directly — the rest of `store.go`/`vlog.go` is otherwise unchanged
   (ADR-046 §3: Badger's own crash recovery/GC machinery replaces the hand-rolled
   single-writer-goroutine/tail-scan/GC code on the Windows path only; the Linux/macOS path
   keeps all of it, unchanged).
4. Add `github.com/dgraph-io/badger/v4` to `go.mod`.
5. Add a native `windows-latest` CI job (mirroring the existing native `macos-15` job)
   running `internal/storage`'s test suite — ADR-046's own "open constraint": cross-compiling
   proves the code compiles for Windows, not that it behaves correctly there.
**FILES** — `internal/storage/engine_rocksdb.go` (renamed from `index.go`, tagged),
`internal/storage/engine_badger.go` (new, tagged), `internal/storage/store.go` (edit —
engine-selection call site only), `go.mod` (edit), `.github/workflows/ci.yml` (edit —
new `windows-latest` job)

**VERIFY**

```bash
COMPILE_LINUX_UNCHANGED:
  $ go build ./internal/storage/...
  EXPECT: exit 0
 
CROSS_COMPILE_WINDOWS:                     # ADR-046 §4 — the immediate, no-new-toolchain check
  $ GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./internal/storage/...
  EXPECT: exit 0
 
BUILD_TAGS_PRESENT_ON_BOTH_ENGINES:
  $ head -1 internal/storage/engine_rocksdb.go | grep -c "go:build linux || darwin"
  EXPECT: 1
  $ head -1 internal/storage/engine_badger.go | grep -c "go:build windows"
  EXPECT: 1
 
NO_UNCONDITIONAL_GROCKSDB_IMPORT:          # every grocksdb-importing file must carry a build tag
  $ for f in internal/storage/*.go; do grep -q "go:build" "$f" || grep -l "grocksdb" "$f"; done
  EXPECT: no output
 
BADGER_CONFIG_MATCHES_ADR046:
  $ grep -cE "WithValueThreshold\(1024\)|WithBloomFalsePositive\(0.01\)|WithBlockCacheSize\(64 ?<< ?20\)|WithCompression\(options.None\)" internal/storage/engine_badger.go
  EXPECT: >= 4
 
CHUNKSTORE_INTERFACE_SURFACE_IDENTICAL_ON_BOTH_ENGINES:
  $ grep -cE "func.*AppendChunk|func.*LookupChunk|func.*DeleteChunk|func.*RunGC|func.*RecoverFromCrash|func.*Close" internal/storage/engine_badger.go
  EXPECT: >= 6
  $ grep -cE "func.*AppendChunk|func.*LookupChunk|func.*DeleteChunk|func.*RunGC|func.*RecoverFromCrash|func.*Close" internal/storage/engine_rocksdb.go
  EXPECT: >= 6
 
STORE_GO_DISPATCHES_BY_BUILD_TAG_NOT_RUNTIME_OS_CHECK:
  $ grep -cE "runtime.GOOS" internal/storage/store.go
  EXPECT: 0                                 # selection must be compile-time (build tags), not a runtime branch
 
CONTENT_HASH_VERIFICATION_IDENTICAL_ON_BOTH_PATHS:
  $ grep -c "sha256.Sum256\|SHA-256" internal/storage/engine_badger.go
  EXPECT: >= 1
 
WINDOWS_CI_JOB_ADDED:
  $ grep -c "windows-latest" .github/workflows/ci.yml
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test ./internal/storage/...
  EXPECT: exit 0                            # existing M5 suite, unaffected on Linux/macOS
  $ go test -v -run TestEngineBadger ./internal/storage/
  EXPECT: exit 0 (on a windows-latest runner); tests include:
    TestBadgerAppendLookupDeleteRoundTrip
    TestBadgerContentHashVerifiedBeforeAuditResponse
    TestBadgerAndRocksDBSatisfySameChunkStoreInterface
 
VET:
  $ go vet ./internal/storage/...
  EXPECT: exit 0; zero output
```

---

### Phase 16.1 — End-to-End Demo Test

**Reference:** `mvp.md` §3.6

#### Session 16.1.1 — Implement `TestDemoTimeline` integration test

**TASK:** Create `scripts/test/demo_timeline_test.go`. Using `--sim-count=5
--sim-asn-count=5` and `VYOMANAUT_MODE=demo`, run the demo timeline from `mvp.md` §3.6:

- Assert readiness gate passes within 60s of startup.
- Assert file upload succeeds for a file ≤ 1.25 MB.
- Assert first audit PASS is recorded within 3 minutes.
- Assert VETTING→ACTIVE transition completes within 12 minutes.
- Assert synthetic chunk GC is delivered after ACTIVE transition.
- Kill one simulated daemon; assert departure detection within `profile.DepartureThreshold`
  (10 min in demo).
- Assert repair job created and completed.
- Assert `ReadinessResponse.mode == "demo"` when `VYOMANAUT_MODE=demo` (OAS
  `ReadinessResponse.properties.mode` and the `DemoReady` example both confirm this field —
  verified present in the live OAS). Demo thresholds, confirmed against the same example:
  `active_vetted_providers.required_value=5`, `distinct_asns.required_value=5`,
  `distinct_metro_regions.required_value=1`, `microservice_quorum.required_value=1`,
  `relay_nodes_deployed.required_value=0`.
This test is tagged `//go:build integration` and runs separately from unit tests.

**FILES** — `scripts/test/demo_timeline_test.go`

**VERIFY**

```bash
COMPILE:
  $ go build -tags integration ./scripts/test/
  EXPECT: exit 0
 
BUILD_TAG_PRESENT:
  $ head -1 scripts/test/demo_timeline_test.go | grep -c "go:build integration"
  EXPECT: 1
 
ALL_SEVEN_TIMELINE_ASSERTIONS_PRESENT:
  $ grep -cE "readiness|upload.*1.25|audit.*PASS|VETTING.*ACTIVE|GC.*deliver|DepartureThreshold|repair.*complet" scripts/test/demo_timeline_test.go
  EXPECT: >= 7
 
MODE_FIELD_ASSERTED:
  $ grep -c '"demo"' scripts/test/demo_timeline_test.go
  EXPECT: >= 1
 
DEMO_THRESHOLDS_MATCH_OAS_DEMOREADY_EXAMPLE:
  $ grep -cE "required_value.*5|required_value.*1|required_value.*0" scripts/test/demo_timeline_test.go
  EXPECT: >= 3
 
UNIT_TESTS:
  $ go test -tags integration -v -run TestDemoTimeline ./scripts/test/
  EXPECT: exit 0 (full run takes ~30-35 min per mvp.md §3.6 — this is an integration test, not a fast unit test)
 
VET:
  $ go vet -tags integration ./scripts/test/
  EXPECT: exit 0; zero output
```

#### Session 16.1.2 — Validate all viability fact-checks from `mvp.md` §7

**TASK:** Add individual tests for each fact-check:

- §7.1 (ASN cap): assert `MinDistinctASNs=5` in `DemoProfile` (already covered in M1 tests
  — this session asserts it again from the demo-timeline harness, not a duplicate: it
  confirms the *running* profile matches, not just the struct literal).
- §7.2 (RS math): assert repair succeeds when 2 of 5 providers are offline.
- §7.3 (vetting timing): assert ACTIVE transition happens at ~10 min in demo.
- §7.7 (mock idempotency): assert duplicate webhook delivery produces exactly one
  `escrow_events` row.
**FILES** — `scripts/test/demo_timeline_test.go` (edit — additional test functions in the
same file; no new file needed, per the file's existing scope)

**VERIFY**

```bash
COMPILE:
  $ go build -tags integration ./scripts/test/
  EXPECT: exit 0
 
UNIT_TESTS:
  $ go test -tags integration -v -run "TestViability" ./scripts/test/
  EXPECT: exit 0; tests include:
    TestViabilityASNCapMatchesRunningDemoProfile
    TestViabilityRepairSucceedsWithTwoOfFiveOffline
    TestViabilityActiveTransitionAtTenMinutes
    TestViabilityDuplicateWebhookProducesExactlyOneEscrowRow
 
DUPLICATE_WEBHOOK_ROW_COUNT_ASSERTED:
  $ grep -c "COUNT(\*)\|len(.*escrow_events\|rowCount" scripts/test/demo_timeline_test.go
  EXPECT: >= 1
 
VET:
  $ go vet -tags integration ./scripts/test/
  EXPECT: exit 0; zero output
```

---

### Phase 16.2 — Simulation Mode

**Reference:** `mvp.md` §8.3 (`--sim-count` flag), IC §10 (simulation mode paths)

#### Session 16.2.1 — Implement `--sim-count` multi-instance provider *(rewritten — closes A-8)*

**PRECONDITIONS** — Phase 16.0 complete (`internal/storage.NewChunkStore` compiles on both
`linux || darwin` and `windows`).

**TASK:** In `cmd/provider/main.go`, when `--sim-count=N`, spawn N goroutines each running
an independent provider daemon instance with:

- Separate Ed25519 identity.
- **Separate `storage.ChunkStore` instance** at `/tmp/vyomanaut-sim/{zero-padded-index}/db/`
  — the same `storage.NewChunkStore` call used in normal (non-simulation) mode; the engine
  selected underneath it is RocksDB on Linux/macOS or BadgerDB on Windows (ADR-046, Phase
  16.0), transparently, via the build tag already resolved at compile time. **This session's
  code must never name the engine directly** — that coupling is exactly what made the
  original spec Linux/macOS-only.
- Value-log path unchanged in shape: `/tmp/vyomanaut-sim/{zero-padded-index}/vlog/chunks.vlog`
  on the RocksDB engine; the equivalent Badger data directory on Windows, both produced by
  the same `dbPath` argument to `storage.NewChunkStore` — no simulation-specific path
  branching by OS.
- Auto-assigned synthetic ASN `SIM-AS{1..N}` from `--sim-asn-count`.
Paths must use the exact format from IC §10 naming conventions: `{instance_id}` is
zero-padded to 4 digits (e.g. `0000`, `0001`).

**FILES** — `cmd/provider/main.go` (edit — `--sim-count` branch)

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/provider/
  EXPECT: exit 0
 
CROSS_COMPILE_WINDOWS_SIM_PATH:                     # the actual fix this session exists for
  $ GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./cmd/provider/
  EXPECT: exit 0
 
NO_ENGINE_NAMED_IN_SIM_CODE_PATH:
  $ grep -niE "rocksdb|grocksdb|badger" cmd/provider/main.go
  EXPECT: no matches                                 # engine choice must stay inside internal/storage
 
USES_SHARED_NEWCHUNKSTORE_CALL:
  $ grep -c "storage.NewChunkStore" cmd/provider/main.go
  EXPECT: >= 1
 
INSTANCE_ID_ZERO_PADDED_4_DIGITS:
  $ grep -cE "%04d|Sprintf.*04d" cmd/provider/main.go
  EXPECT: >= 1
 
SIM_PATH_MATCHES_IC10_CONVENTION:
  $ grep -c "/tmp/vyomanaut-sim/" cmd/provider/main.go
  EXPECT: >= 1
 
ASN_AUTO_ASSIGNMENT_PRESENT:
  $ grep -c "SIM-AS" cmd/provider/main.go
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -v -run TestSimCount ./cmd/provider/
  EXPECT: exit 0 (Linux/macOS CI); tests include:
    TestSimCountSpawnsNIndependentInstances
    TestSimCountAssignsDistinctEd25519Identities
    TestSimCountZeroPadsInstanceIDToFourDigits
    TestSimCountAssignsSyntheticASNsFromFlag
    TestSimCountNeverReferencesStorageEngineDirectly    # static-analysis-style test asserting the grep above
  $ go test -v -run TestSimCount ./cmd/provider/                      # re-run on windows-latest runner
  EXPECT: exit 0 — same test names, same pass criteria, no Windows-specific skips
 
VET:
  $ go vet ./cmd/provider/
  EXPECT: exit 0; zero output
```

---

## MILESTONE 17 — Demo Completion *(rewritten; replaces Production Hardening)*

**Deliverable:** a demo a human can run. The CLI exists, `scripts/test/` is complete, and the old
M17's production content is formally relocated to the LTS track.

**Status:** `internal/client/{account,upload,retrieve,manage}` are built and tested (M15).
`cmd/client/main.go` is an eight-line stub. No session in any build document has ever revisited it.
This milestone is the wiring.

**Dependency:** M16 → **M17** → M18.
**Reference:** `MVP §8.3` (subcommand table), IC §5.9 (`UploadOrchestrator`), IC §11 (`cmd/` is
wiring only), IC §14 (copy codes), ADR-035 (`intent_url` server-owned), ADR-064.
**Sessions:** 5.

---

### Phase 17.1 — The Data Owner CLI *(closes N-01)*

#### Session 17.1.1 — CLI skeleton, `register`, `recover`

**PRECONDITIONS** — M16 complete. `internal/client/account` builds; its tests pass.

**TASK**

1. Replace `cmd/client/main.go`'s stub with a subcommand dispatcher over the eight `MVP §8.3` names:
   `register`, `recover`, `upload`, `retrieve`, `ls`, `rm`, `balance`, `deposit`. Unknown
   subcommand → usage text, exit 2.
2. Global flags: `--mode` (falls back to `VYOMANAUT_MODE`; flag wins, MVP §5.3),
   `--microservice-url` (required), `--data-dir` (default `$HOME/.vyomanaut`), `--json`.
3. `register` → `account.Register`. Argon2id at `profile.Argon2Time/Memory/Threads` — never
   hardcoded. Mnemonic displayed **once** to stdout behind an explicit confirmation prompt.
4. `recover` → `account.Recover`, with `--passphrase` and `--mnemonic` paths.
5. **The mnemonic never enters `--json` output, any log line, or any file.**
6. One `renderError` helper maps every error to an IC §14 copy code.
**FILES** — `cmd/client/main.go` (rewrite), `cmd/client/dispatch.go`, `cmd/client/account_cmds.go`,
`cmd/client/render.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/client/
  EXPECT: exit 0
 
STUB_IS_GONE:                               # N-01
  $ grep -c "mode=STUB" cmd/client/main.go
  EXPECT: 0
  $ grep -c "TODO: wire subsystems" cmd/client/main.go
  EXPECT: 0
 
ALL_EIGHT_SUBCOMMANDS_DISPATCHED:           # MVP §8.3
  $ grep -cE '"(register|recover|upload|retrieve|ls|rm|balance|deposit)"' cmd/client/dispatch.go
  EXPECT: >= 8
 
UNKNOWN_SUBCOMMAND_EXITS_2:
  $ go run ./cmd/client nonsense >/dev/null 2>&1; echo "exit=$?"
  EXPECT: exit=2
 
MODE_FLAG_OVERRIDES_ENV:                    # MVP §5.3
  $ grep -c "VYOMANAUT_MODE" cmd/client/main.go
  EXPECT: >= 1
 
CMD_IS_WIRING_ONLY:                         # IC §11
  $ grep -cE "func (Derive|Encode|Decode|Sign|Compute|Validate)[A-Z]" cmd/client/*.go
  EXPECT: 0
 
MNEMONIC_NEVER_LOGGED_OR_SERIALISED:
  $ grep -cE "log\.|slog\.|os.WriteFile" cmd/client/account_cmds.go
  EXPECT: 0
  $ grep -cE 'json:"[a-z_]*mnemonic' cmd/client/*.go
  EXPECT: 0
 
ARGON2_PARAMS_FROM_PROFILE:
  $ grep -cE "profile\.Argon2(Time|Memory|Threads)" cmd/client/account_cmds.go
  EXPECT: >= 1
  $ grep -cE "argon2\.IDKey\([^)]*, ?3, ?65536|64 \* 1024" cmd/client/*.go
  EXPECT: 0
 
UNIT_TESTS:
  $ go test -v ./cmd/client/
  EXPECT: exit 0; tests include:
    TestDispatchRecognisesAllEightSubcommands
    TestUnknownSubcommandExitsTwo
    TestModeFlagOverridesEnvVar
    TestMnemonicAbsentFromJSONOutput
 
VET:
  $ go vet ./cmd/client/
  EXPECT: exit 0; zero output
```

---

#### Session 17.1.2 — `upload` and `retrieve`

**PRECONDITIONS** — 17.1.1 complete. `internal/client/upload` and `internal/client/retrieve` pass.

**TASK**

1. `upload <path>` → `upload.UploadFile` (IC §5.9). Prints `file_id` to stdout. Percentage progress
   to **stderr**, keeping stdout parseable.
2. `retrieve <file_id> [-o out]` → `retrieve.RetrieveFile`. Default filename from the pointer file.
3. `--resume <session_id>` → `upload.ResumeUpload`.
4. Error paths render IC §14 codes only: `ErrInsufficientEscrow` (with shortfall and a pointer to
   `deposit`), `ErrNetworkNotReady` (readiness gate unsatisfied — a normal state, not an alarm),
   `ErrTooFewShards`, `ErrCanaryMismatch`.
5. **Apply D-10.** `internal/client/retrieve/decode.go:43` returns the package-local
   `ErrCanaryMismatch` **without wrapping**, so `errors.Is(err, crypto.ErrCanaryMismatch)` is false
   above the client boundary. Change to `fmt.Errorf("retrieve: %w", crypto.ErrCanaryMismatch)`.
   The buffer-zeroing behaviour is already correct — **do not touch it**.
6. **Apply D-07.** `internal/payment`'s doc comment cites `NFR-046` for the no-float rule. The
   no-float rule is **NFR-038**; NFR-046 is metric naming.
**FILES** — `cmd/client/transfer_cmds.go`, `internal/client/retrieve/decode.go` (edit),
`internal/payment/ledger.go` (edit — comment)

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/client/ ./internal/client/... ./internal/payment/
  EXPECT: exit 0
 
CLI_CALLS_THE_SDK_NOT_REIMPLEMENTS_IT:
  $ grep -cE "upload\.(UploadFile|ResumeUpload)" cmd/client/transfer_cmds.go
  EXPECT: >= 2
  $ grep -c "retrieve.RetrieveFile" cmd/client/transfer_cmds.go
  EXPECT: >= 1
 
PROGRESS_ON_STDERR_STDOUT_PARSEABLE:
  $ grep -c "os.Stderr" cmd/client/transfer_cmds.go
  EXPECT: >= 1
 
FOUR_ERROR_PATHS_RENDERED:
  $ grep -cE "ErrInsufficientEscrow|ErrNetworkNotReady|ErrTooFewShards|ErrCanaryMismatch" cmd/client/transfer_cmds.go
  EXPECT: >= 4
 
NO_RAW_GO_ERRORS_TO_STDOUT:                 # IC §14
  $ grep -c "err.Error()" cmd/client/transfer_cmds.go
  EXPECT: 0
 
D10_CANARY_ERROR_NOW_WRAPS:
  $ grep -c 'fmt.Errorf("retrieve: %w", crypto.ErrCanaryMismatch)' internal/client/retrieve/decode.go
  EXPECT: 1
 
D10_BUFFER_ZEROING_UNCHANGED:               # regression guard
  $ grep -cE "for i := range|zero\(|clear\(" internal/client/retrieve/decode.go
  EXPECT: >= 1
 
D07_FLOAT_RULE_CITES_NFR038:
  $ grep -c "NFR-046" internal/payment/ledger.go
  EXPECT: 0
  $ grep -c "NFR-038" internal/payment/ledger.go
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -v ./cmd/client/ ./internal/client/retrieve/
  EXPECT: exit 0; tests include:
    TestUploadPrintsFileIDOnStdoutOnly
    TestRetrieveDefaultsToPointerFilename
    TestInsufficientEscrowRendersIC14CodeAndPointsAtDeposit
    TestCanaryMismatchIsErrorsIsCryptoSentinel
 
VET_AND_RACE:
  $ go vet ./cmd/client/ && go test -race ./internal/client/...
  EXPECT: exit 0
```

---

#### Session 17.1.3 — `ls`, `rm`, `balance`, `deposit`

**PRECONDITIONS** — 17.1.2 complete. `internal/client/manage` passes.

**TASK**

1. `ls` → `manage.ListFiles`. Table output; availability labels from **IC §14.2**, never invented.
2. `rm <file_id>` → `manage.DeleteFile`, confirmation prompt unless `--yes`.
3. `balance` → `manage.Balance`, through the single paise formatter.
4. `deposit --amount-paise=N` → `manage.Deposit`. **Demo-critical, not production-only.** Upload is
   gated on escrow balance at `internal/api/upload.go:238`, and `MockProvider.InitiateEscrow`
   credits the ledger **synchronously** in demo mode — its own comment records *"no async webhook in
   demo mode."* So `deposit` is the prerequisite of the first upload. Print the mock VPA and QR URL;
   render `intent_url` **exactly as returned** — ADR-035 makes it server-owned and the client must
   never construct a `upi://pay?…` string.
5. **One paise formatter, `int64`, no floating point anywhere** (IC §11, NFR-038).
**FILES** — `cmd/client/manage_cmds.go`, `cmd/client/money.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/client/
  EXPECT: exit 0
 
FOUR_SUBCOMMANDS_CALL_THE_SDK:
  $ grep -cE "manage\.(ListFiles|DeleteFile|Balance|Deposit)" cmd/client/manage_cmds.go
  EXPECT: >= 4
 
NO_FLOAT_ON_THE_MONEY_PATH:                 # IC §11, NFR-038
  $ grep -cE "float64|float32|ParseFloat|%\.2f" cmd/client/money.go cmd/client/manage_cmds.go
  EXPECT: 0
 
SINGLE_PAISE_FORMATTER:
  $ grep -c "func formatPaise(paise int64) string" cmd/client/money.go
  EXPECT: 1
  $ grep -rc "/ 100" cmd/client/ | grep -v "money.go" | grep -v ":0" | wc -l
  EXPECT: 0
 
INTENT_URL_RENDERED_NOT_CONSTRUCTED:        # ADR-035
  $ grep -cE "intent_url|IntentURL" cmd/client/manage_cmds.go
  EXPECT: >= 1
  $ grep -crE "upi://pay\?|\bpa=|\bpn=|\bam=" cmd/client/ | grep -v ":0" | wc -l
  EXPECT: 0
 
AVAILABILITY_LABELS_FROM_IC14_2:
  $ grep -oE '"[A-Z][A-Z_]{3,}"' cmd/client/manage_cmds.go | tr -d '"' | sort -u \
      | while read c; do grep -q "$c" docs/system-design/interface-contracts.md \
          || echo "NOT_IN_IC14 $c"; done
  EXPECT: no NOT_IN_IC14 lines
 
RM_CONFIRMS_UNLESS_YES:
  $ grep -c -- "--yes\|skipConfirm" cmd/client/manage_cmds.go
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -v ./cmd/client/
  EXPECT: exit 0; tests include:
    TestFormatPaiseRendersIntegerRupees
    TestFormatPaiseRejectsFloatInputAtTypeLevel
    TestDepositRendersServerSuppliedIntentURLVerbatim
    TestListUsesIC14AvailabilityLabels
    TestRemoveRequiresConfirmationWithoutYesFlag
 
VET:
  $ go vet ./cmd/client/
  EXPECT: exit 0; zero output
```

---

### Phase 17.2 — `scripts/test/` completeness *(closes N-04)*

#### Session 17.2.1 — CLI-driven end-to-end harness

**PRECONDITIONS** — Phase 17.1 complete.

**Why this session exists.** `scripts/test/` is empty in the live tree. M16 Session 16.1.1 fills it
with `demo_timeline_test.go`, which drives the **SDK**. That proves the code path. It does not prove
the artifact you will demonstrate, because the CLI did not exist when 16.1.1 was written. This
session adds the CLI-driven harness and — defensively — verifies 16.1.1's deliverable is actually
present, since a milestone under construction is not a milestone finished.

**TASK**

1. Verify `scripts/test/demo_timeline_test.go` exists and passes. **If M16.1.1 was not completed,
   this session builds it to the M16.1.1 spec before proceeding** — `scripts/test/` must be complete
   when M17 closes.
2. Add `scripts/test/demo_cli_test.go`, tagged `//go:build integration`, driving the **compiled
   `cmd/client` binary** through the full sequence: `register` → `deposit` → `upload` → `ls` →
   `retrieve` → byte-compare → `balance` → `rm`.
3. Every assertion parses `--json` output — no screen-scraping of human-readable text.
4. Assert the retrieved bytes are **identical** to the uploaded bytes. This is the single most
   important assertion on the demo track.
5. Assert `ReadinessResponse.mode == "demo"` and that no `prod` profile path is reachable.
**FILES** — `scripts/test/demo_cli_test.go`, `scripts/test/demo_timeline_test.go` (verify/create),
`scripts/test/helpers_test.go`

**VERIFY**

```bash
SCRIPTS_TEST_IS_NOT_EMPTY:                  # N-04
  $ ls scripts/test/*.go | wc -l
  EXPECT: >= 2
 
COMPILE:
  $ go build -tags integration ./scripts/test/
  EXPECT: exit 0
 
BUILD_TAGS_PRESENT:
  $ head -1 scripts/test/demo_cli_test.go | grep -c "go:build integration"
  EXPECT: 1
 
M16_TIMELINE_TEST_PRESENT_AND_PASSING:
  $ test -f scripts/test/demo_timeline_test.go && echo PASS
  EXPECT: PASS
  $ go test -tags integration -run TestDemoTimeline ./scripts/test/
  EXPECT: exit 0
 
CLI_BINARY_DRIVEN_NOT_SDK:                  # the artifact under test must be the artifact demoed
  $ grep -cE "exec.Command|clientBin" scripts/test/demo_cli_test.go
  EXPECT: >= 1
  $ grep -cE "internal/client/(upload|retrieve|manage)" scripts/test/demo_cli_test.go
  EXPECT: 0
 
ALL_SEVEN_SUBCOMMANDS_EXERCISED:
  $ grep -cE '"(register|deposit|upload|ls|retrieve|balance|rm)"' scripts/test/demo_cli_test.go
  EXPECT: >= 7
 
BYTE_IDENTITY_ASSERTED:
  $ grep -cE "bytes.Equal|sha256.Sum256.*==|cmp\(" scripts/test/demo_cli_test.go
  EXPECT: >= 1
 
JSON_OUTPUT_PARSED_NOT_SCRAPED:
  $ grep -c -- "--json" scripts/test/demo_cli_test.go
  EXPECT: >= 5
  $ grep -c "json.Unmarshal" scripts/test/demo_cli_test.go
  EXPECT: >= 1
 
DEMO_MODE_ONLY:
  $ grep -c '"demo"' scripts/test/demo_cli_test.go
  EXPECT: >= 1
  $ grep -cE "prod|VYOMANAUT_MODE=prod" scripts/test/demo_cli_test.go
  EXPECT: 0
 
UNIT_TESTS:
  $ go test -tags integration -v -run TestDemoCLI ./scripts/test/
  EXPECT: exit 0; tests include:
    TestDemoCLIFullLifecycle
    TestDemoCLIRetrievedBytesIdenticalToUploaded
    TestDemoCLIUploadFailsBeforeDeposit
    TestDemoCLIReadinessReportsDemoMode
 
VET:
  $ go vet -tags integration ./scripts/test/
  EXPECT: exit 0; zero output
```

---

### Phase 17.3 — Relocation record *(closes N-03)*

#### Session 17.3.1 — Relocate production hardening to the LTS track

**PRECONDITIONS** — ADR-062 accepted.

**TASK**

1. In `build_part3.md`, replace the old Milestone 17 body with a **relocation notice** naming what
   moved and where its content now lives: secrets-manager adapters (Vault / AWS SSM / GCP Secret
   Manager, IC §8), the three-replica gossip cluster (ARCH §18), ADR-048's gossip-authentication
   addendum, `cmd/relay`, and `internal/cluster`'s real routing.
2. The relocated content is written verbatim into `build_part4.md` under **LTS — Production
   Hardening**, with **no milestone number assigned** (per your instruction 8; numbering follows the
   demo run outcomes).
3. Update `build.md`'s dependency graph: the demo edge is `M16 → M17 → M18`.
4. Update `internal/cluster/`'s and `internal/secrets/`'s package doc comments — both currently
   point at "Milestone 17 Session 17.2.1" and "Milestone 17 Phase 17.1", which now refer to the CLI
   milestone. Point them at the LTS Production Hardening milestone by name, not by number.
5. Same for `deployments/dev/docker-compose.yml`'s three `TRANSITION RULES` comments, one of which
   says *"M17 — replace relay placeholder with cmd/relay binary."*
**FILES** — `docs/system-design/build_part3.md` (edit), `docs/system-design/build_part4.md` (new),
`docs/system-design/build.md` (edit graph), `internal/cluster/membership.go` (comment),
`internal/cluster/router.go` (comment), `internal/secrets/doc.go` (comment),
`deployments/dev/docker-compose.yml` (comment)

**VERIFY**

```bash
OLD_M17_CONTENT_RELOCATED_NOT_DELETED:
  $ grep -cE "Vault|AWS SSM|GCP Secret Manager|GossipCluster|cmd/relay" docs/system-design/build_part4.md
  EXPECT: >= 5
 
M17_IS_NOW_DEMO_COMPLETION:
  $ grep -cE "^## Milestone 17 — Demo Completion" docs/system-design/build_part3.md
  EXPECT: 1
 
DEPENDENCY_GRAPH_UPDATED:
  $ grep -cE "M16 ?(──▶|->|→) ?M17 ?(──▶|->|→) ?M18" docs/system-design/build.md
  EXPECT: >= 1
 
NO_STALE_M17_POINTERS_IN_CODE:              # these now mean the CLI milestone
  $ grep -rcE "Milestone 17 (Phase 17\.1|Session 17\.2\.1)" internal/ deployments/ | grep -v ":0" | wc -l
  EXPECT: 0
 
NO_HARDCODED_LTS_MILESTONE_NUMBERS:         # instruction 8
  $ grep -cE "Milestone 2[0-9]|M2[0-9]\b" docs/system-design/build_part4.md
  EXPECT: 0
 
BUILD_STILL_GREEN:
  $ go build ./... && go vet ./...
  EXPECT: exit 0
```

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

## M18/M19 Forward-Compatibility: Desktop GUI Shell Readiness

**Why this section exists.** `build.md`'s Milestone 18 ("Launch Readiness") is the last
milestone currently in the plan. `ux-decisions.md` §7/§9 and ten already-drafted ADRs
(038–047) commit this product to shipping two Wails-based Windows desktop apps — and none
of that is reflected anywhere in `build.md`/`build_part2.md`/`build_part3.md` today. This
section does three things: confirms what M15–17, as rewritten above, already got right for
that future; flags the one thing outside my reviewed scope (M13) worth fixing while it's
still cheap; and drafts a placeable Milestone 19 outline so `build.md` has somewhere
concrete to grow into, rather than stopping at M18 with the GUI direction undocumented in
the build plan itself.

### What's already GUI-ready, and why

`ADR-039` (UI–Backend Communication) decides that both Wails apps call Go functions
**directly, in-process** — *"the existing `cmd/client` and `cmd/provider` logic is what
the app runs."* That only works if `internal/client/*` is already something a new
entrypoint can import and drive without going through `cmd/`. It already is:
`interface-contracts.md` §9's own permitted-dependency graph (corrected per A-2) is
`cmd/* → internal/client/* → (...)` with **no constraint on which `cmd/*` does the
importing** — a future `cmd/desktop-owner` (or wherever the Wails Go backend lives) is just
another member of that same permitted left-hand side, no different in kind from
`cmd/client` today. Nothing in Part C's Milestone 15 rewrite needed to change for this;
IC §5.9's `UploadOrchestrator` interface (`UploadFile`/`ResumeUpload`/`RetrieveFile`) and
Phase 15.4's file/escrow functions are exactly the bindable surface ADR-039 needs, and they
were designed (by IC, before this review) with no `cmd/`-specific assumptions baked in.

`ADR-035`'s `intent_url` field (Session 15.4.2, Part C) was written with **both** consumers
in mind — *"the Wails Data Owner app... renders it as a tappable link... `cmd/client
deposit`... prints `intent_url` as its primary, copyable output"* — so Session 15.4.2 does
not need a GUI-specific variant later; the CLI and the future GUI read the same field off
the same response.

### Candidate Milestone 19 — Desktop GUI Shell (Data Owner & Provider Apps)

**Not a finished, `VERIFY`-ready set of sessions** — a phase-level outline, grounded in the
ten ADRs that already exist, sized so Karma can confirm numbering/placement (immediately
after M18, or interleaved with it) before it is expanded to session-level detail in a
follow-up pass, the same way this document expanded M15–17.

| Phase | Grounded in | Scope |
| --- | --- | --- |
| 19.1 — Windows Storage Validation | ADR-046 open constraint | Native `windows-latest` test run of `internal/storage`'s Badger path under real load; re-run the HDD-equivalent compaction benchmark against Badger's own tuning surface (ADR-046 explicitly does not assume RocksDB's tuned values transfer) |
| 19.2 — Provider App Shell | ADR-038, ADR-040, ADR-042, ADR-047 | Wails-wrap `cmd/provider` (post F.2's `StartDaemon` refactor); goroutine-pattern system tray; Task Scheduler logon-trigger installer step, no elevation anywhere in the install or runtime path |
| 19.3 — Data Owner App Shell | ADR-038, ADR-039, ADR-035 | Wails-wrap `internal/client/*` directly (Part C's Phase 15.1–15.4 orchestrators, called in-process, no local port); render `intent_url` per ADR-035 §3 |
| 19.4 — Shared Component Library & Design System | `ux-decisions.md` §9.5 | One component set (buttons, cards, status badges, toast/error system) shared by both apps; status colors wired to the same IC §14 copy table both CLIs already render from (A-4's Phase 15.4, once §14.2 merges) |
| 19.5 — Windows Packaging, Signing & CI | ADR-041 | NSIS via `wails build -nsis`; Azure Trusted Signing from the first external build; GitHub Actions producing a signed installer on every release |
| 19.6 — Provider Setup & Transparency Surfaces | ADR-043, ADR-044, ADR-045, ADR-016 addendum | No manual port-forwarding/DDNS in setup; environmental-claim copy scoped per ADR-044; the ADR-016 addendum's `gross_amount_paise`/`release_multiplier_bps` rendered as the "why was my payout smaller" explanation ADR-045 and the addendum both anticipate |

**Recommendation, for your decision, not mine to make:** `build.md`'s dependency graph
currently ends `M17 → M18`. Milestone 18 ("Launch Readiness") is written entirely in terms
of the *network* — runbooks, benchmarks, the security checklist, the 15-check CI gate —
and none of its four phases reference a GUI at all. Two placements are both defensible:
(a) insert Milestone 19 as a genuine sequel, after M18's all-green CI gate, treating GUI
shipment as a fast-follow on top of a network that's already launch-ready on its own terms;
or (b) interleave GUI phases into M18 itself, if "launch" is meant to mean "the GUI is what
most users actually install," not "the backend network passes its own checklist." Nothing
in M15–17 as rewritten above forecloses either choice — that's the point of F.1 — so this
is deliberately left as a placement decision rather than a fix.

---
