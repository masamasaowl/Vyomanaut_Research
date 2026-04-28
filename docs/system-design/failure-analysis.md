# Vyomanaut V2 — Failure Mode and Effects Analysis (FMEA)

**Status:** Authoritative — reviewed against all accepted ADRs and the V2 build plan.
Where this document conflicts with an ADR, the ADR wins. New mitigations must be introduced
via an ADR or a runbook; this document records what already exists.
**Version:** 1.0
**Date:** April 2026
**Author:** Vyomanaut Engineering
**Repository:** https://github.com/masamasaowl/Vyomanaut_Research
**Supersedes:** —
**Companion documents:**
- [`architecture.md`](./architecture.md) — component overview and runtime flows
- [`requirements.md`](./requirements.md) — functional and non-functional requirements
- [`data-model.md`](./data-model.md) — database schema and invariants
- [`interface-contracts.md`](./interface-contracts.md) — inter-component contracts
- [`capacity.md`](./capacity.md) — scale ceilings and bandwidth budgets
- [`mvp.md`](./mvp.md) — build plan and risk register
- [`ADR index`](../decisions/README.md) — all accepted architectural decisions

---

## Table of Contents

1. [Scope and Method](#1-scope-and-method)
2. [Severity and Detection Rating Scales](#2-severity-and-detection-rating-scales)
3. [FMEA Table](#3-fmea-table)
   - [3.1 Coordination Microservice Cluster](#31-coordination-microservice-cluster)
   - [3.2 Provider Daemon](#32-provider-daemon)
   - [3.3 Data Encoding Pipeline](#33-data-encoding-pipeline)
   - [3.4 Payment System](#34-payment-system)
   - [3.5 Network and NAT Layer](#35-network-and-nat-layer)
   - [3.6 Data Owner Credentials](#36-data-owner-credentials)
4. [Risk Priority Number Summary](#4-risk-priority-number-summary)
5. [Unmitigated Failure Modes](#5-unmitigated-failure-modes)
6. [Failure Mode Coverage Map](#6-failure-mode-coverage-map)

---

## 1. Scope and Method

This document is a Failure Mode and Effects Analysis (FMEA) structured to the software
adaptation of IEC 60812 / MIL-STD-1629A. It answers: for each component and each failure
mode, what is the effect on the system, what is the severity, how likely is it to occur,
how quickly is it detected, and what mitigation exists?

**What this document does and does not do.**

This document does not introduce new architectural decisions or mitigations. Every
mitigation cited here traces to an accepted ADR, an accepted NFR in
[`requirements.md`](./requirements.md), or an operational runbook already committed to the
repository. Where no mitigation exists, the failure mode is listed in
[Section 5](#5-unmitigated-failure-modes) alongside a linked open question.

This document covers the six component classes that together constitute the V2 system:
coordination microservice cluster, provider daemon, data encoding pipeline, payment system,
network and NAT layer, and data owner credential handling. It does not cover the external
Razorpay and secrets-manager services beyond the interface boundary — only the consequences
on the Vyomanaut system when those services misbehave.

**How to use this document.**

During build: before merging any PR that touches a failure mode listed here, verify that the
mitigation cited in the Source column is in place in the implementation. If a PR introduces
a new failure mode, add a row to this document as part of the same PR.

During operations: use the Detection Method column to identify which metric or alert fires
for each failure mode. Use the Source column to reach the relevant runbook or ADR.

During architecture review: use [Section 5](#5-unmitigated-failure-modes) as the residual
risk posture statement for V2. Any unmitigated item with RPN > 20 that cannot be closed by
a new ADR before launch must be escalated to engineering leadership.

---

## 2. Severity and Detection Rating Scales

**Severity (S) — 1 to 5**

| Rating | Effect |
|--------|--------|
| 5 | Permanent data loss or permanent unrecoverable financial loss with no remediation path |
| 4 | Data temporarily inaccessible, or payment incorrect, or provider escrow miscalculated; recoverable within 24 hours with operator intervention |
| 3 | Degraded audit coverage, partial scoring error, or elevated repair load; self-healing within one audit cycle (24 hours) without operator intervention |
| 2 | User-visible error with a clean failure message; no data loss, no financial impact, no lasting system state damage |
| 1 | Internal error with no user impact; logged and monitored only; resolves on next cycle |

**Occurrence (O) — 1 to 5**

| Rating | Frequency |
|--------|-----------|
| 5 | Expected to occur multiple times per month in production at V2 launch scale |
| 4 | Expected approximately once per month |
| 3 | Expected approximately once per quarter |
| 2 | Expected approximately once per year |
| 1 | Theoretically possible; not expected in production during the V2 lifetime |

**Detectability (D) — 1 to 5 (inverse scale: 1 = always detected immediately, 5 = never detected without external report)**

| Rating | Detection |
|--------|-----------|
| 1 | Detected immediately; a Prometheus alert fires and the operator is paged within minutes |
| 2 | Detected within one audit cycle (24 hours) by the audit result distribution or score windows |
| 3 | Detected within the monthly payment cycle, or by a metric that is checked weekly but not alerted |
| 4 | Detected only by explicit manual audit of stored data, receipts, or payment records |
| 5 | Not detected without an external report from a data owner or provider; no in-system signal |

**Risk Priority Number (RPN) = S × O × D.**

RPNs above 20 are flagged as high-risk and require either a linked accepted mitigation or a
linked open question in [`docs/research/open-questions.md`](../research/open-questions.md).

---

## 3. FMEA Table

### 3.1 Coordination Microservice Cluster

| ID | Failure Mode | Effect on System | S | O | D | RPN | Detection Method | Mitigation | Source |
|----|-------------|-----------------|---|---|---|-----|-----------------|------------|--------|
| MS-01 | Single replica failure (one of three replicas crashes or loses network) | Service continues: R=2, W=2 still satisfied by remaining two replicas. Audit scheduling, challenge dispatch, and payment processing uninterrupted. | 2 | 3 | 1 | 6 | `microservice_replica_count{state="healthy"} < 3` alert; PagerDuty page. | (3,2,2) quorum absorbs single replica failure without interruption. | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md) |
| MS-02 | All three replicas fail simultaneously (datacenter event, deployment bug, OOM kill cascade) | Complete halt: no audit challenges issued, no receipt recording, no repair job queuing, no payment releases. PostgreSQL continues accepting reads. Data already stored on providers is unaffected. | 4 | 1 | 2 | 8 | External uptime monitor (synthetic probe to `/api/v1/admin/readiness`) from a separate network. Internal Prometheus unreachable when all replicas are down; external probe is the only detection path. | Multi-AZ deployment across at least two AZs; keep replicas on distinct VM instances with separate power circuits. | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md), [`deployment-topology`](./architecture.md#8-deployment-topology) |
| MS-03 | Secrets manager unreachable at microservice replica startup | Replica refuses to start (fail-closed). If all three replicas restart simultaneously during a secrets manager outage, the entire cluster fails to come up. | 3 | 2 | 1 | 6 | Replica startup failure logged with `ErrSecretManagerUnavailable`; deployment orchestrator (Kubernetes, Systemd) emits a crash-loop alert. | Fail-closed at startup: unverifiable challenges are worse than no challenges. Secrets manager HA SLA must exceed microservice HA requirement. | [ADR-027](../decisions/ADR-027-cluster-audit-secret.md), [`runbooks/secrets-manager-outage.md`](../runbooks/secrets-manager-outage.md) |
| MS-04 | Secrets manager becomes unreachable during operation (after successful startup) | The 5-minute in-memory cache continues serving existing challenges. After cache expiry, `ErrSecretExpired` is returned by `ChallengeNonce`; challenge issuance halts for the duration. Receipts for challenges already dispatched can still be validated from cache. | 2 | 2 | 2 | 8 | `audit_challenges_issued_total` rate drops to zero within one 5-minute cache window; alert on challenge throughput drop > 50% in a 10-minute window. | 5-minute cache bridges short outages. Runbook covers manual secret reload. | [ADR-027](../decisions/ADR-027-cluster-audit-secret.md) |
| MS-05 | PostgreSQL primary failure | Writes fail: no new `audit_receipts`, `escrow_events`, or `chunk_assignments` can be inserted. Reads continue via read replicas. Microservice serves stale but valid data for scoring and assignment lookups. Audit receipts dispatched during primary downtime are lost if the microservice has not already committed the PENDING row. | 3 | 2 | 1 | 6 | Postgres multi-AZ managed failover alert; `db_read_p99_latency_ms` spike on failover; microservice logs write errors. | Managed PostgreSQL Multi-AZ: automatic promotion of a read replica to primary within 60–120 seconds. The two-phase receipt write ensures no receipt is silently dropped — the PENDING row must be durable before Phase 2. | [`architecture.md §8`](./architecture.md#8-deployment-topology), [ADR-015](../decisions/ADR-015-audit-trail.md) |
| MS-06 | PostgreSQL read replica lag exceeds 60 seconds | Reliability score materialised views (`mv_provider_scores`) are read from a stale replica. A provider whose score has been dropping due to recent FAILs may receive an incorrectly high release multiplier on the current monthly computation. | 2 | 3 | 3 | 18 | No direct alert for stale score data. Detectable by comparing `mv_provider_scores` timestamps against `NOW()` via a monitoring query. | Monthly release job should query the primary directly for final score values. Daily score display can use replica. Document this in the payment service implementation. | [ADR-024](../decisions/ADR-024-economic-mechanism.md), [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md) |
| MS-07 | Background task (Merkle log compaction, view refresh, repair queuing) consumes excessive Postgres read capacity | Foreground audit challenge dispatch and receipt recording latency increases. Providers' responses arrive after the per-provider RTO; TIMEOUT receipts accumulate. Reliability scores degrade for providers that are actually healthy. | 2 | 3 | 2 | 12 | `db_read_p99_latency_ms` monitored continuously; background throttle activates at 50 ms. Alert on `audit_results_total{result="TIMEOUT"}` rate exceeding 5% within a 1-hour window. | Background task slice allocation reduced when foreground p99 approaches 50 ms. | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md), [NFR-028](./requirements.md#76-observability-and-operability) |
| MS-08 | System clock skew greater than 5 minutes between a microservice replica and provider daemons | Provider heartbeat timestamp check rejects heartbeats with skew > ±5 minutes. `last_heartbeat_ts` is not updated; `multiaddr_stale` is eventually set; DHT fallback is used for challenge dispatch. If skew persists, the provider accumulates TIMEOUT receipts and score degrades. The `server_challenge_ts` embedded in nonces is set by the microservice clock; a skewed server clock embeds a wrong timestamp in the nonce, but this does not cause validation failure since validation also runs on the same replica. | 3 | 2 | 3 | 18 | Elevated TIMEOUT rate for providers across a specific geographic cluster (if ISP-level clock drift). Heartbeat rejection log entries. No direct clock-skew Prometheus metric by default — add `process_start_time_seconds` comparisons. | NTP synchronisation enforced on all VM instances; monitored by the cloud provider's infrastructure. | [`ADR-028`](../decisions/ADR-028-provider-heartbeat.md), [ADR-006](../decisions/ADR-006-polling-interval.md) |
| MS-09 | Gossip partition: two replicas lose connectivity to each other but can both reach the seed nodes | Membership views diverge temporarily (up to ~10 seconds). Both replicas continue operating against their local view. Write quorum (W=2) may be temporarily satisfied by each replica counting itself plus one other, or broken if network is asymmetric. Reads (R=2) may return inconsistent results during the partition. | 3 | 2 | 2 | 12 | `microservice_replica_count{state="healthy"}` drops to 1 on affected replicas; gossip reconciliation logs show persistent merge failures. | Seed nodes act as tie-breakers to prevent permanent partition. Gossip convergence is self-healing within seconds once the network partition resolves. | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md), [`runbooks/microservice-failover.md`](../runbooks/microservice-failover.md) |
| MS-10 | Audit receipt INSERT rate approaches or exceeds PostgreSQL single-instance ceiling (~5,000–10,000 rows/sec) | INSERT latency rises; audit receipt recording falls behind challenge issuance. Receipts queue in microservice memory. If the queue is unbounded, OOM is possible. If bounded, challenges are back-pressured and audit coverage decreases. | 3 | 1 | 2 | 6 | `db_read_p99_latency_ms` sustained rise; `audit_challenges_issued_total` rate diverges from `audit_results_total` write rate; repair queue depth grows as scoring falls behind. | Not expected at V2 launch scale. Monthly partitioning is mandatory from day one. If sampling is introduced, SHELBY Theorem 1 Condition (i) must be re-verified. | [ADR-002](../decisions/ADR-002-proof-of-storage.md), [`capacity.md §4.2`](./capacity.md#42-postgres-insert-ceiling-for-audit_receipts) |

---

### 3.2 Provider Daemon

| ID | Failure Mode | Effect on System | S | O | D | RPN | Detection Method | Mitigation | Source |
|----|-------------|-----------------|---|---|---|-----|-----------------|------------|--------|
| PD-01 | DHCP lease rotation between heartbeats (Indian residential ISPs rotate leases on 24-hour cycles) | Audit challenge dispatched to stale IP address. Connection attempt fails or is accepted by a different host. TIMEOUT receipt recorded. Provider's 24-hour score window decrements. If this occurs on every audit cycle for the 72-hour departure window, the provider could be incorrectly declared departed. | 2 | 4 | 3 | 24 | Elevated `audit_results_total{result="TIMEOUT"}` rate correlating with providers in specific ISP ASNs. `providers.multiaddr_stale` flag set after 2+ missed heartbeats. | 4-hour signed heartbeat refreshes `last_known_multiaddrs`. Microservice uses heartbeat address as primary; DHT as fallback. A provider whose IP rotates within a 4-hour window still misses at most one challenge before the next heartbeat corrects the address. | [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |
| PD-02 | vLog silent disk corruption: `SHA-256(chunk_data) ≠ stored content_hash` at audit read time | Provider returns `FAIL` with `FAIL_CORRUPTION` status code. `accelerated_reaudit` is set for all chunks on that provider. Repair is triggered if shard count drops to r0 threshold. The corrupted shard is replaced during repair. | 3 | 2 | 1 | 6 | `content_hash_failures_total` metric incremented immediately; alert fires if > 0 in a 7-day rolling window (per [NFR-027](./requirements.md#76-observability-and-operability)). | `content_hash` embedded in every vLog entry and verified on every read. Reactive re-audit accelerated by the Schroeder 30× elevated failure signal. | [ADR-023](../decisions/ADR-023-provider-storage-engine.md), [ADR-008](../decisions/ADR-008-reliability-scoring.md) |
| PD-03 | RocksDB background compaction starving audit reads (compaction I/O competing with vLog reads) | p99 audit response latency exceeds the per-provider deadline. TIMEOUT receipts accumulate. Score degrades for a provider that is actually storing data correctly. | 2 | 3 | 2 | 12 | `audit_response_latency_ms_p99` rises on provider daemon local metrics. TIMEOUT rate spike correlates with daemon-level compaction logs. | `rate_limiter` cap on RocksDB background compaction. HDD-specific `rate_limiter` set lower than SSD default (detected by rotational flag at daemon startup). Benchmark protocol Q27-1 must pass before V2 launch. | [ADR-023](../decisions/ADR-023-provider-storage-engine.md), [NFR-008](./requirements.md#73-performance) |
| PD-04 | Provider's storage device is formatted as FAT32 or another filesystem that does not support `fallocate(FALLOC_FL_PUNCH_HOLE)` | GC cannot reclaim deleted vLog entries via `fallocate`. Disk fills up with orphaned entries. Provider daemon eventually returns `STORAGE_FULL` status codes on new chunk upload requests. Assigned chunks are not stored; audit challenges for those assignments produce FAIL results. | 3 | 2 | 2 | 12 | `STORAGE_FULL` error code returned in UploadResponse; `chunks_stored_total` metric plateaus while assignments continue. Provider dashboard shows disk full. | Daemon startup must check for FAT32 and halt with a clear error: "Vyomanaut requires a filesystem supporting sparse file operations (ext4, NTFS, APFS, or equivalent). FAT32 is not supported." Document in provider hardware requirements. | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| PD-05 | Daemon killed after vLog `fsync()` completes but before RocksDB INSERT commits | vLog entry is durable; RocksDB index entry is absent. Next audit challenge for that `chunk_id` hits a Bloom filter miss and returns `FAIL_NOT_FOUND`. Microservice records a FAIL receipt. Score decrements. If the crash is not followed by a restart, the shard appears lost and repair may be triggered prematurely. | 2 | 3 | 2 | 12 | FAIL receipt with `FAIL_NOT_FOUND` status code for a chunk that was previously audited as PASS. After daemon restart, the crash recovery scan re-inserts the missing index entry; subsequent challenges produce PASS. | Crash recovery tail-scan on daemon restart re-inserts any index entry missing from RocksDB for a vLog entry that was fsync'd. Triggered automatically before the writer goroutine starts. | [ADR-023](../decisions/ADR-023-provider-storage-engine.md), [NFR-024](./requirements.md#75-reliability-and-correctness) |
| PD-06 | Provider behind symmetric NAT with all Vyomanaut relay nodes' slots exhausted | Provider is unreachable for audit challenge dispatch. TIMEOUT receipts accumulate. Score degrades. If relay exhaustion persists beyond 72 hours and no other connectivity path exists, the provider is incorrectly declared departed. | 3 | 2 | 2 | 12 | `relay_slot_utilisation` metric on each relay node; alert at > 80% utilisation. `audit_results_total{result="TIMEOUT"}` spike correlating with symmetric-NAT providers. | Relay infrastructure scaling: add a fourth relay node when provider count exceeds 570 (45% CGNAT) or 850 (30% CGNAT). Headroom formula documented in [`capacity.md §5.2`](./capacity.md#52-relay-infrastructure-scaling). | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md), [`capacity.md §5.2`](./capacity.md#52-relay-infrastructure-scaling) |
| PD-07 | Provider clock skew causes heartbeat timestamp check to fail (skew > ±5 minutes) | Heartbeat is rejected. `last_heartbeat_ts` not updated. `multiaddr_stale` is eventually set after 2+ consecutive rejections. Audit challenges fall back to DHT lookup. If DHT also has a stale record, TIMEOUT receipts accumulate. | 2 | 2 | 3 | 12 | Heartbeat rejection log entries at microservice. `heartbeat_sent_total` counter on daemon continues incrementing while `last_heartbeat_ts` in Postgres does not advance — gap detectable by operator tooling. | NTP synchronisation required on provider machines. Installer should check for NTP daemon presence on Windows/macOS/Linux and surface a warning if clock is out of sync. | [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |
| PD-08 | Single writer goroutine deadlock: a channel send or receive blocks indefinitely | All pending vLog appends block. Upload requests hang waiting for `resultChan`. The daemon stops accepting new chunk assignments. Existing stored chunks continue to serve audit challenges (read path is independent of the writer goroutine). | 3 | 1 | 2 | 6 | `vlog_append_latency_ms_p99` diverges to infinity; upload goroutines accumulate (Go runtime scheduler shows goroutine count growing unboundedly); heartbeat continues (it does not use the writer goroutine). | Channel-select with a timeout in the writer goroutine; daemon logs and restarts the writer goroutine after a configurable deadlock timeout. Unit test `TestSingleWriterGoroutine` with a 100-goroutine contention scenario in CI. | [ADR-023](../decisions/ADR-023-provider-storage-engine.md), [`interface-contracts.md §5.3`](./interface-contracts.md#53-internalstorage) |

---

### 3.3 Data Encoding Pipeline

| ID | Failure Mode | Effect on System | S | O | D | RPN | Detection Method | Mitigation | Source |
|----|-------------|-----------------|---|---|---|-----|-----------------|------------|--------|
| ENC-01 | AONT key K accidentally included in a log statement, panic stack trace, or error message | Any party who reads the log output for that session can recover K by assembling the k=16 fragments their access allows. If the operator holds the logs and holds enough fragments, they can decrypt file data — violating the zero-knowledge property. | 5 | 1 | 4 | 20 | Detected only by code audit or by a log-scanning tool that patterns-matches on 32-byte hex strings adjacent to "AONT", "key", or "K=". Not detectable at runtime. | Explicit coding rule: K is never passed to any logging function, never included in error messages, zeroed in memory immediately after `AONTDecodePackage` returns. CI grep check: `grep -r '"K"' internal/crypto/` must produce zero results in log-adjacent code paths. Documented in security verification checklist in [`mvp.md §Security Verification Checklist`](./mvp.md#security-verification-checklist). | [ADR-022](../decisions/ADR-022-encryption-erasure-order.md), [ADR-019](../decisions/ADR-019-client-side-encryption.md) |
| ENC-02 | Canary word collision: decoded plaintext is corrupt but the 16-byte canary word coincidentally matches the expected value | A corrupted segment passes the canary check. Corrupted plaintext is returned to the data owner without error. Data owner receives silently wrong data. | 5 | 1 | 5 | 25 | Completely undetectable by system monitoring. The only signal is data owner reporting wrong content after retrieval — a user-visible symptom with no in-system metric. | The canary is a fixed 16-byte value. For random single-bit corruption of the decoded segment, the probability that the corrupted bits land exactly on the canary word and exactly reproduce the canary value is 2⁻¹²⁸ — computationally negligible. This failure mode is theoretical, not a realistic operational risk. The shard-level `content_hash` verification ([ADR-023](../decisions/ADR-023-provider-storage-engine.md)) provides a second independent integrity layer at the provider side. | [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) |
| ENC-03 | Argon2id master secret derivation produces different output on different platforms due to integer width differences (32-bit vs 64-bit `int`) | Data owner derives a different `master_secret` on a 32-bit device than on a 64-bit device. Pointer file keys are also different. The data owner cannot decrypt their pointer file on the new platform. File recovery becomes impossible without access to the original platform. | 5 | 1 | 3 | 15 | Data owner reports inability to retrieve files after switching to a 32-bit device. No in-system signal — the wrong key simply produces a tag mismatch on pointer file decryption. | Argon2id implementation must use `uint64` explicitly for all internal counters and indices, not platform-native `int`. Cross-platform test matrix in CI: Windows (64-bit), macOS (Apple Silicon ARM64), Linux (x86-64), with an identical test vector verifying the same `master_secret` output on all three. | [ADR-020](../decisions/ADR-020-key-management.md) |
| ENC-04 | Pointer file nonce counter state lost (daemon keystore corrupt, manual device reset, counter not persisted durably) | The nonce counter resets to a previously-used value. Re-encryption of a pointer file with a reused `(key, nonce)` pair leaks the XOR of two plaintexts: `PT₁ XOR PT₂ = CT₁ XOR CT₂`. Additionally, the Poly1305 tag can potentially be forged if the attacker observes both ciphertexts. | 4 | 2 | 4 | 32 | Not detectable at runtime. An operator who reads the ciphertexts from Postgres cannot identify nonce reuse without knowing the plaintext. Detectable only by explicit audit comparing nonce values in `files.pointer_nonce` for the same `(owner_id, file_id)`. | Counter persisted in the encrypted local keystore with `fsync` before use. Daemon refuses to encrypt with a nonce equal to or less than the last confirmed nonce. Recovery procedure: if counter state is lost, set the counter past all known `pointer_nonce` values retrievable from the microservice's `files` table before re-encrypting any pointer file. Mandatory disclosure per [FR-005](./requirements.md#61-data-owner--registration-and-onboarding). | [ADR-019](../decisions/ADR-019-client-side-encryption.md), [ADR-020](../decisions/ADR-020-key-management.md) |
| ENC-05 | CPUID AES-NI detection result differs between the encoding and decoding device | The encoding device uses AES-256-CTR (AES-NI); the decoding device uses ChaCha20-256 (no AES-NI). The two cipher paths produce different keystreams for the same K and counter. AONT decode produces wrong plaintext; canary mismatch is detected and an error is returned. No data loss, but retrieval fails. | 3 | 1 | 1 | 3 | `ErrCanaryMismatch` returned from `AONTDecodePackage`; error surfaced to data owner as "segment integrity check failed". | The pointer file does not encode which cipher was used — both paths must produce the same output for the same K. This is a constraint on the AONT encoding implementation: both AES-256-CTR and ChaCha20-256 must produce identical word-level outputs for the same K, counter, and input. Either use a single canonical cipher on encode (ChaCha20-256 always, as the portable path) or document the cipher in the pointer file's `erasure_params`. This gap must be resolved before M1 closes. | [ADR-019](../decisions/ADR-019-client-side-encryption.md), [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) |
| ENC-06 | RS encoder produces non-systematic output (shards 0–15 are not identity-mapped from AONT package) | AONT-RS decode fails because the systematic property assumed during K recovery (`c_{s+1}` position) is violated. Recovery path: full RS decode still works, but the K recovery formula must account for non-systematic output ordering — it will produce garbage and the canary check will fail. | 4 | 1 | 1 | 4 | `ErrCanaryMismatch` on every retrieval attempt. Detected immediately on the first file retrieval test. | `klauspost/reedsolomon` supports systematic coding; `EncodeSegment` must explicitly use systematic mode (first k output shards equal input). Verified by `TestRSRoundTrip` and `TestRSAnyKShards` in the test suite. | [ADR-003](../decisions/ADR-003-erasure-coding.md), [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) |

---

### 3.4 Payment System

| ID | Failure Mode | Effect on System | S | O | D | RPN | Detection Method | Mitigation | Source |
|----|-------------|-----------------|---|---|---|-----|-----------------|------------|--------|
| PAY-01 | Razorpay Route API returns 5xx on monthly release day | Monthly payout for affected providers is delayed. The RELEASE event has already been inserted into `escrow_events` (it is inserted before the Razorpay call per the release flow). Razorpay queues a reversal automatically. The `payout.reversed` webhook triggers a corrective negative RELEASE. Retry on the next monthly cycle. | 2 | 2 | 1 | 4 | `payout.reversed` webhook received; `payment_seizure_total_paise` does not increase. Razorpay dashboard shows payout in `reversed` state. | Retry with the same idempotency key on next cycle. `payout.reversed` webhook handler inserts a corrective RELEASE event. | [ADR-012](../decisions/ADR-012-payment-basis.md), [ADR-016](../decisions/ADR-016-payment-db-schema.md), [`04-payment-release.md §Failure Path 1`](./sequence-diagrams/04-payment-release.md) |
| PAY-02 | Monthly release job triggered twice (scheduler bug, manual re-run, replica restart during job) | Second INSERT into `escrow_events` hits the `UNIQUE(idempotency_key)` constraint and returns a 409. Second Razorpay PATCH call returns the existing transfer state (idempotent). Provider receives exactly one payment. | 1 | 2 | 1 | 2 | 409 conflict logged at microservice; `escrow_events_total` counter does not double-increment. | `idempotency_key UNIQUE` constraint enforced at both Postgres level and Razorpay API level. `release_computed` flag in `audit_periods` set to `true` after first successful run; job checks flag before running. | [ADR-016](../decisions/ADR-016-payment-db-schema.md), [NFR-030](./requirements.md#77-compliance-and-payments) |
| PAY-03 | Floating-point arithmetic introduced via a third-party library imported into the payment package | Payout amounts computed with float64 accumulate rounding errors. Over many monthly cycles, providers may be over- or under-paid by up to a few paise per release. At scale this becomes a reconciliation liability. | 3 | 2 | 4 | 24 | Not detectable at runtime without an explicit paise-level reconciliation audit comparing `escrow_events` row sums against Razorpay payout history. CI static analysis is the primary prevention control. | `TestNoFloatArithmetic` static analysis test in CI: `grep -rn "float64\|float32" internal/payment/` must return zero results. This test runs on every PR that touches `internal/payment`. All monetary computations use `int64` paise. | [ADR-016](../decisions/ADR-016-payment-db-schema.md), [NFR-046](./requirements.md#77-compliance-and-payments) |
| PAY-04 | RBI bank holiday lookup table not updated during December deployment | `last_working_day_of_month()` returns an incorrect date for at least one month in the following year. `on_hold_until` is set to a date that is actually a bank holiday; Razorpay may defer the release by one additional business day. | 2 | 1 | 2 | 4 | Provider reports delayed monthly payment. No in-system metric until the holiday passes and the release does not arrive. | Annual December deployment procedure: `runbooks/rbi-holiday-table-update.md`. NFR-031 mandates this update. | [ADR-024](../decisions/ADR-024-economic-mechanism.md), [NFR-031](./requirements.md#77-compliance-and-payments) |
| PAY-05 | `payout.reversed` webhook not delivered by Razorpay (network delivery failure after all retries exhausted) | Negative RELEASE event is never inserted. The provider's escrow balance in `escrow_events` shows a RELEASE that was actually reversed by Razorpay. The balance is overstated by the reversed amount until the next manual reconciliation. | 3 | 2 | 3 | 18 | Discrepancy between Razorpay payout history and internal `escrow_events` detected during monthly reconciliation. No automated alert currently. | Monthly reconciliation runbook: cross-check all RELEASE events against Razorpay payout status. Add a reconciliation Prometheus gauge that surfaces the count of RELEASE events without a corresponding confirmed Razorpay `payout.processed` status. | [`runbooks/razorpay-api-outage.md`](../runbooks/razorpay-api-outage.md) |
| PAY-06 | Provider Linked Account cooling period check skipped (race condition or implementation bug in assignment service) | Assignment is made to a provider whose Razorpay Linked Account has not completed the 24-hour cooling period. When the first monthly release is attempted, Razorpay rejects the transfer with an error. Provider receives no payment for that month's audit passes. | 2 | 1 | 1 | 2 | Razorpay API error logged at release time; `payment_payout_total_paise` does not increase for affected provider. | `razorpay_cooling_until` check enforced in the readiness gate (one of the seven conditions in [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md)) and in the assignment service: `AND razorpay_cooling_until < NOW()` in provider selection query. | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md), [`05-provider-lifecycle.md §Sequence 2`](./sequence-diagrams/05-provider-lifecycle.md) |
| PAY-07 | Escrow seizure idempotency key collision: two providers whose `provider_id || "seizure" || departed_at` hash coincidentally collide (SHA-256 pre-image collision) | Second seizure INSERT returns a 409 unique constraint violation. The second departed provider's escrow is not seized; repair reserve fund is underfunded by that provider's held earnings. | 1 | 1 | 1 | 1 | 409 conflict logged with seizure idempotency key. Detectable in logs; no financial loss from the unique constraint violation itself (the collision is the anomaly). | SHA-256 pre-image collision probability is 2⁻²⁵⁶ — computationally impossible. Documented for completeness only. | [ADR-016](../decisions/ADR-016-payment-db-schema.md) |

---

### 3.5 Network and NAT Layer

| ID | Failure Mode | Effect on System | S | O | D | RPN | Detection Method | Mitigation | Source |
|----|-------------|-----------------|---|---|---|-----|-----------------|------------|--------|
| NET-01 | Indian ISP blocks UDP; QUIC connection attempts fail for a provider | Provider daemon falls back to TCP + Noise XX + yamux via libp2p automatic transport negotiation. Hole-punch success rate is statistically identical to QUIC (~70% for cone NAT). Relay fallback applies for symmetric NAT. No change in reliability. | 1 | 2 | 2 | 4 | `p2p_connections_active{transport="tcp"}` increases while `{transport="quic"}` decreases for affected providers. | Three-tier NAT traversal: QUIC → TCP+Noise → Circuit Relay v2. Transport selection is automatic. | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| NET-02 | Single relay node failure (1 of 3) | ~128 relay slots lost. Symmetric-NAT providers with reservations on the failed node become temporarily unreachable until they re-register with one of the remaining two relay nodes. TIMEOUT receipts accumulate until re-registration completes (typically within the next 4-hour heartbeat cycle when the daemon reports updated relay multiaddrs). | 3 | 2 | 1 | 6 | `relay_slot_utilisation` drops on failed node (or node disappears from monitoring); TIMEOUT rate spikes for ~30-minute window as affected providers re-register. Alert at relay node health check failure. | Three relay nodes provide 384 total slots — 4.3× headroom at 300 providers. Provider daemon attempts next relay in `--relay-addrs` list on reservation failure. Runbook: [`runbooks/relay-node-replacement.md`](../runbooks/relay-node-replacement.md). | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md), [`capacity.md §5.2`](./capacity.md#52-relay-infrastructure-scaling) |
| NET-03 | DHT record expires before the availability service republishes it (12-hour republish / 24-hour expiry gap) | A data owner cannot find the provider's multiaddr via DHT FIND_VALUE. The microservice uses `last_known_multiaddrs` (heartbeat-maintained) as the primary challenge dispatch address; DHT is a fallback. Impact is limited to DHT lookups initiated outside the microservice (e.g., direct data owner chunk retrieval via DHT). | 1 | 2 | 3 | 6 | DHT FIND_VALUE failure logged on client; client falls back to microservice pointer file for provider list. No audit impact. | 12-hour republish vs 24-hour expiry provides a 12-hour safety buffer. Primary challenge dispatch uses heartbeat address, not DHT. | [ADR-006](../decisions/ADR-006-polling-interval.md), [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |
| NET-04 | Correlated ISP outage affecting more than 20% of one ASN's providers simultaneously (e.g., BSNL national backbone issue) | Maximum 11 of 56 shards lost for files assigned to that ASN (ceil(0.20 × 56) = 12, but actual cap is ≤20% so ≤11 with 56 total). 45 surviving shards remain — 29 above the reconstruction threshold of s=16. Files remain accessible. Repair jobs are enqueued for the 11 missing shards. | 2 | 2 | 1 | 4 | TIMEOUT rate spike for an entire ASN detectable by `audit_results_total{result="TIMEOUT"}` segmented by provider ASN. `repair_queue_depth` increases. | 20% ASN cap per [ADR-014](../decisions/ADR-014-adversarial-defences.md) structurally limits the maximum correlated loss to 11 shards, which is categorically below the reconstruction floor of 16. Worst-case correlated loss is provably survivable. | [ADR-014](../decisions/ADR-014-adversarial-defences.md), [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| NET-05 | libp2p version upgrade silently resets the custom HMAC DHT key validator to the default CID-based namespace | DHT begins accepting plain CID lookups. Monitoring nodes observing DHT traffic can correlate lookup requests with file identity, breaking the zero-knowledge DHT privacy property. | 4 | 1 | 1 | 4 | `TestDHTKeyValidator` CI test fails: plain CID lookup is accepted when it should be rejected. This test runs on every `go-libp2p` upgrade PR. | `TestDHTKeyValidator` is mandatory in CI. Custom validator registered as a package-level constant (not inline string) so a global grep finds all registration points. Upgrade checklist includes re-running this test. | [`interface-contracts.md §9`](./interface-contracts.md#9-dht-key-contract), [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| NET-06 | DCUtR hole-punch fails on all retry attempts for a cone-NAT provider (retry count is 1 per ADR-021) | Provider falls back to Circuit Relay v2. All audit challenges and data transfers are proxied through a relay node, adding ~50 ms RTT overhead. This increases audit response latency. For providers with a tight p95_throughput_kbps, the relay latency may push some challenge responses past the deadline. | 2 | 3 | 2 | 12 | `p2p_nat_class{class="symmetric"}` count rises (providers incorrectly re-classified after DCUtR failure). Relay slot utilisation increases. Audit response latency p99 rises for relay-bound providers. | DCUtR retry count set to 1 (not 3) because 97.6% of successes occur on the first attempt (Paper 30). The 2.4% of connections that require retry will use relay. Relay overhead < 50 ms from Indian cloud-hosted nodes (NFR-006). | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md), [NFR-006](./requirements.md#72-availability) |
| NET-07 | 0-RTT session resumption data replayed on an audit challenge stream | An attacker replays a previously valid audit challenge response. The microservice receives the replayed response and — if the `challenge_nonce` unique index check passes — could record a second PASS receipt for a different challenge period, crediting the provider for an audit they did not actually perform. | 4 | 1 | 3 | 12 | The `challenge_nonce UNIQUE` constraint in `audit_receipts` rejects duplicate nonces. A replayed response for a nonce already present returns `ErrReceiptAlreadyFinal`. The UNIQUE constraint is a second line of defence. | 0-RTT is **disabled** for all connections carrying audit challenges or signed receipts (protocol ID suffix check). QUIC `DisableEarlyData: true` enforced in the libp2p host configuration for the audit protocol. | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md), [`interface-contracts.md §4.2`](./interface-contracts.md#42-audit-challenge-protocol) |

---

### 3.6 Data Owner Credentials

| ID | Failure Mode | Effect on System | S | O | D | RPN | Detection Method | Mitigation | Source |
|----|-------------|-----------------|---|---|---|-----|-----------------|------------|--------|
| DO-01 | Data owner forgets passphrase and has no BIP-39 mnemonic backup | All files stored by that owner are permanently, unrecoverably lost. No support path exists — Vyomanaut does not hold any key material. | 5 | 2 | 1 | 10 | Data owner contacts support reporting inability to access files. No in-system signal — the system cannot distinguish "owner forgot passphrase" from "new device, correct passphrase not yet entered". | By design: zero-knowledge storage requires the data owner to bear this risk. Mandatory disclosure at account creation (FR-005). Two-word mnemonic confirmation gate (FR-003) ensures the owner has seen and recorded the mnemonic before the first upload is permitted. Clear onboarding UX per [`mvp.md §8.1`](./mvp.md#81-critical-ux-moments). | [ADR-020](../decisions/ADR-020-key-management.md), [FR-003](./requirements.md#61-data-owner--registration-and-onboarding), [FR-005](./requirements.md#61-data-owner--registration-and-onboarding) |
| DO-02 | BIP-39 mnemonic backup exfiltrated (photographed by attacker, stolen from safe, cloud-synced accidentally) | Attacker reconstructs `master_secret` from the mnemonic. They can derive all pointer file decryption keys. If they can access the microservice (or capture the data owner's HTTPS traffic), they can download and decrypt all pointer files and then retrieve all stored file fragments. Full data compromise. | 5 | 1 | 5 | 25 | Not detectable at system level. The attacker uses a valid `master_secret` — indistinguishable from the legitimate owner. The data owner may only notice if a file has been retrieved without their action (if a retrieval log is implemented). | No system-level mitigation: this is a physical and personal security responsibility. UX guidance: store the mnemonic offline, never digitally, never photographed. The 24-word BIP-39 format is a widely understood offline backup standard. | [ADR-020](../decisions/ADR-020-key-management.md) |
| DO-03 | Pointer file nonce counter overflow after 2⁹⁶ pointer file write operations | Nonce counter wraps to 0. The next pointer file encryption reuses (key, nonce) pair. Poly1305 tag forgery becomes possible for a party who has observed both ciphertexts encrypted under the same (key, nonce). | 4 | 1 | 1 | 4 | Nonce counter at 2⁹⁶ − 1 is detectable by inspecting the keystore. Alert is theoretically possible but has no practical value given the occurrence probability. | A data owner would need to update their pointer file 2⁹⁶ times — requiring uploading new file versions more often than once per nanosecond for the entire age of the universe. Documented for completeness. The daemon should still check for counter rollover and halt if it occurs rather than silently reusing a nonce. | [ADR-019](../decisions/ADR-019-client-side-encryption.md) |
| DO-04 | Data owner uploads a file before Argon2id derivation completes (UI race condition between passphrase input and upload button) | Upload begins with an uninitialised or zero `master_secret`. Pointer file is encrypted with a key derived from the zero master secret. File can be retrieved only on this same session (same zero master secret in memory). On next session, the real passphrase produces a different master secret and the pointer file decryption fails. File is effectively unrecoverable after session end. | 4 | 1 | 1 | 4 | Pointer file decryption fails with `ErrTagMismatch` on second retrieval attempt. Data owner reports inability to access file. | Client UI must gate the upload button on successful Argon2id completion (non-zero master_secret in memory). Upload orchestrator checks `len(masterSecret) == 32` as a pre-condition and panics in debug builds if violated. | [ADR-020](../decisions/ADR-020-key-management.md), [`interface-contracts.md §5.9`](./interface-contracts.md#59-internalclient) |

---

## 4. Risk Priority Number Summary

The table below lists all failure modes sorted by RPN descending. Items with RPN > 20 are highlighted as requiring an accepted mitigation or an open question before V2 launch.

| ID | Component | Failure Mode (summary) | S | O | D | RPN | Status |
|----|-----------|----------------------|---|---|---|-----|--------|
| ENC-04 | Encoding Pipeline | Pointer file nonce counter lost | 4 | 2 | 4 | **32** | Mitigated: durable counter storage; recovery procedure |
| DO-02 | Data Owner Credentials | BIP-39 mnemonic exfiltrated | 5 | 1 | 5 | **25** | Unmitigated at system level — physical security |
| ENC-02 | Encoding Pipeline | Canary word collision | 5 | 1 | 5 | **25** | Theoretical; P = 2⁻¹²⁸; not a realistic risk |
| PAY-03 | Payment System | Float arithmetic via third-party library | 3 | 2 | 4 | **24** | Mitigated: `TestNoFloatArithmetic` CI check |
| PD-01 | Provider Daemon | DHCP lease rotation between heartbeats | 2 | 4 | 3 | **24** | Mitigated: 4-hour heartbeat (ADR-028) |
| ENC-01 | Encoding Pipeline | AONT key K accidentally logged | 5 | 1 | 4 | **20** | Partially mitigated: code review + CI grep |
| MS-06 | Microservice Cluster | Read replica lag > 60s | 2 | 3 | 3 | **18** | Partially mitigated: query primary for release computation |
| MS-08 | Microservice Cluster | Clock skew > 5 min | 3 | 2 | 3 | **18** | Mitigated: NTP on VMs |
| PAY-05 | Payment System | `payout.reversed` webhook not delivered | 3 | 2 | 3 | **18** | Partially mitigated: manual reconciliation runbook |
| NET-07 | Network / NAT | 0-RTT replay on audit stream | 4 | 1 | 3 | **12** | Mitigated: 0-RTT disabled + nonce UNIQUE constraint |
| PD-03 | Provider Daemon | RocksDB compaction starves audit reads | 2 | 3 | 2 | **12** | Mitigated: rate_limiter (ADR-023) |
| PD-04 | Provider Daemon | FAT32 filesystem; fallocate unsupported | 3 | 2 | 2 | **12** | Partially mitigated: startup check required |
| PD-05 | Provider Daemon | Crash between fsync and RocksDB INSERT | 2 | 3 | 2 | **12** | Mitigated: crash recovery scan (ADR-023) |
| PD-06 | Provider Daemon | Relay slots exhausted; symmetric-NAT | 3 | 2 | 2 | **12** | Mitigated: relay scaling runbook |
| PD-07 | Provider Daemon | Provider clock skew | 2 | 2 | 3 | **12** | Mitigated: NTP guidance in installer |
| MS-09 | Microservice Cluster | Gossip partition | 3 | 2 | 2 | **12** | Mitigated: seed nodes; self-healing gossip |
| MS-07 | Microservice Cluster | Background task DB saturation | 2 | 3 | 2 | **12** | Mitigated: 50 ms throttle (ADR-025) |
| NET-06 | Network / NAT | DCUtR hole-punch fails; relay fallback | 2 | 3 | 2 | **12** | Mitigated: relay headroom; <50 ms overhead |
| ENC-03 | Encoding Pipeline | Argon2id platform integer width | 5 | 1 | 3 | **15** | Partially mitigated: cross-platform CI required |
| MS-02 | Microservice Cluster | All three replicas fail simultaneously | 4 | 1 | 2 | **8** | Mitigated: multi-AZ deployment |
| MS-04 | Microservice Cluster | Secrets manager unreachable after startup | 2 | 2 | 2 | **8** | Mitigated: 5-minute cache (ADR-027) |
| PD-08 | Provider Daemon | Writer goroutine deadlock | 3 | 1 | 2 | **6** | Mitigated: channel timeout; restart logic |
| MS-01 | Microservice Cluster | Single replica failure | 2 | 3 | 1 | **6** | Mitigated: (3,2,2) quorum (ADR-025) |
| MS-03 | Microservice Cluster | Secrets manager unreachable at startup | 3 | 2 | 1 | **6** | Mitigated: fail-closed (ADR-027) |
| MS-05 | Microservice Cluster | Postgres primary failure | 3 | 2 | 1 | **6** | Mitigated: Multi-AZ managed failover |
| MS-10 | Microservice Cluster | INSERT rate ceiling exceeded | 3 | 1 | 2 | **6** | Mitigated: partitioning; not a V2 concern |
| PD-02 | Provider Daemon | vLog disk corruption | 3 | 2 | 1 | **6** | Mitigated: content_hash verification (ADR-023) |
| NET-02 | Network / NAT | Single relay node failure | 3 | 2 | 1 | **6** | Mitigated: 3 nodes; 384 slots headroom |
| NET-03 | Network / NAT | DHT record expiry | 1 | 2 | 3 | **6** | Mitigated: heartbeat address is primary |
| DO-01 | Data Owner Credentials | Passphrase forgotten, no mnemonic | 5 | 2 | 1 | **10** | By design; mandatory disclosure |
| DO-04 | Data Owner Credentials | Upload before Argon2id completes | 4 | 1 | 1 | **4** | Mitigated: UI gate; pre-condition check |
| DO-03 | Data Owner Credentials | Nonce counter overflow | 4 | 1 | 1 | **4** | Theoretical; daemon halts on rollover |
| PAY-01 | Payment System | Razorpay 5xx on release day | 2 | 2 | 1 | **4** | Mitigated: idempotent retry |
| PAY-04 | Payment System | RBI holiday table stale | 2 | 1 | 2 | **4** | Mitigated: annual December update |
| NET-01 | Network / NAT | ISP blocks UDP; QUIC unusable | 1 | 2 | 2 | **4** | Mitigated: TCP fallback |
| NET-04 | Network / NAT | Correlated ISP outage (ASN) | 2 | 2 | 1 | **4** | Mitigated: 20% ASN cap (ADR-014) |
| NET-05 | Network / NAT | libp2p upgrade resets DHT validator | 4 | 1 | 1 | **4** | Mitigated: CI TestDHTKeyValidator |
| ENC-05 | Encoding Pipeline | CPUID path divergence encode vs decode | 3 | 1 | 1 | **3** | Must be resolved before M1 closes |
| PAY-06 | Payment System | Cooling period check skipped | 2 | 1 | 1 | **2** | Mitigated: readiness gate check |
| PAY-02 | Payment System | Monthly release job double-triggered | 1 | 2 | 1 | **2** | Mitigated: idempotency_key UNIQUE |
| PAY-07 | Payment System | Seizure idempotency key SHA-256 collision | 1 | 1 | 1 | **1** | Theoretical; SHA-256 collision P = 2⁻²⁵⁶ |

---

## 5. Unmitigated Failure Modes

This section lists every failure mode that either has no mitigation or relies on a
process control (code review, annual runbook) rather than an automated enforcement
mechanism. Items with RPN > 20 that reach production without an automated mitigation
in place must be accepted explicitly by engineering leadership before the V2 launch gate
closes.

---

### UF-01 — BIP-39 Mnemonic Exfiltration (DO-02)

**RPN: 25 (S=5, O=1, D=5)**

**Failure mode:** The data owner's BIP-39 mnemonic is photographed, stolen, or
accidentally cloud-synced, giving the attacker full access to all stored files.

**Why no system-level mitigation exists:** This is a physical and personal security
problem. The system's zero-knowledge property prevents Vyomanaut from holding any backup
of the master secret. There is no in-system mechanism to detect or prevent mnemonic
exfiltration.

**What the system does:** UX guidance at onboarding directs the data owner to store the
mnemonic offline and not digitally. The two-word confirmation gate (FR-003) ensures the
owner has written it down before the first upload.

**Residual risk accepted by:** Product and engineering leadership at V2 launch. This is
the inherent risk of zero-knowledge storage and is disclosed to data owners.

**Linked open question:** OQ-FMEA-001 — Should V2 support an optional passphrase on top of
the BIP-39 mnemonic (BIP-38 style), requiring both the written mnemonic and a remembered
passphrase for recovery, to reduce the blast radius of physical mnemonic exfiltration?
Add this question to `docs/research/open-questions.md`.

---

### UF-02 — AONT Key K Logging (ENC-01)

**RPN: 20 (S=5, O=1, D=4)**

**Failure mode:** K is accidentally included in a log statement or panic stack trace.
Any party with log access can decrypt file segments.

**Mitigation exists but is process-dependent:** The CI grep check and the code review
policy are preventive controls, not automated enforcement at the type system level.
A contributor who wraps K in a named struct and logs the struct's fields could bypass
the current grep pattern.

**Gap:** Go does not have a language-level mechanism to prevent a value from being
included in a log statement. The current mitigation is best-effort.

**Recommended hardening before launch:** Wrap the AONT key in a type that implements
`fmt.Stringer` by returning `"<redacted>"` and `json.Marshaler` by returning `null`.
This ensures any accidental `log.Printf("%v", key)` produces a redacted output rather
than the raw bytes. This change does not require a new ADR — it is an implementation
hardening of [ADR-022](../decisions/ADR-022-encryption-erasure-order.md).

**Linked open question:** OQ-FMEA-002 — Implement a `SecretBytes` wrapper type in
`internal/crypto` that redacts on any `fmt`, `log`, or `encoding/json` call. Add to
`docs/research/open-questions.md`.

---

### UF-03 — Pointer File Nonce Counter Recovery (ENC-04)

**RPN: 32 (S=4, O=2, D=4)**

**Failure mode:** Pointer file nonce counter state is lost after a device reset or
keystore corruption. Re-encryption with a reused nonce breaks Poly1305 security.

**Mitigation exists but recovery is manual:** The daemon refuses to use a nonce it
has previously used. However, if the entire keystore is wiped and the counter resets to
zero, the daemon has no record of previously-used nonces and cannot enforce the check.
The recovery procedure (set counter past all `files.pointer_nonce` values in Postgres)
requires operator-level access to Postgres — a data owner cannot self-serve this
recovery.

**Gap:** The recovery procedure is documented in [ADR-020](../decisions/ADR-020-key-management.md) but not automated.
A data owner whose keystore is wiped has no self-service path to resolve this without
support intervention.

**Recommended action before launch:** Build a self-service nonce-counter recovery flow
into the data owner client: after recovering the master secret from passphrase or
mnemonic, the client calls `GET /api/v1/owner/{owner_id}/files` to retrieve all stored
pointer nonces, computes `max(pointer_nonce) + 1`, and sets the counter to that value
before any pointer file re-encryption is attempted. This makes the recovery fully
automated and does not require operator access to Postgres.

**Linked open question:** OQ-FMEA-003 — Implement automated nonce-counter recovery in
the client's new-device onboarding flow. Add to `docs/research/open-questions.md`.

---

### UF-04 — Float Arithmetic via Third-Party Library (PAY-03)

**RPN: 24 (S=3, O=2, D=4)**

**Failure mode:** A dependency of `internal/payment` introduces float64 arithmetic
indirectly (e.g., a JSON unmarshalling library that defaults to `float64` for numbers).

**Mitigation exists but has a blind spot:** The `TestNoFloatArithmetic` grep-based
static analysis checks source files in `internal/payment` for direct `float64`/`float32`
declarations. It does not catch float values flowing in via interface parameters from
external libraries.

**Gap:** If Razorpay's Go SDK returns a JSON `amount` field as `interface{}` that
unmarshals to `float64` by default, and the developer passes that value directly to
`InsertEscrowEvent` without an explicit integer conversion, the static analysis check
would not catch it.

**Recommended hardening:** All Razorpay webhook amount fields must be unmarshalled into
a custom `PaiseAmount int64` type with a dedicated JSON unmarshaller that rejects any
value that is not a JSON integer or returns an error on fractional values. Add this as
a Go type in `internal/payment/paise.go` and enforce its use via a linter rule.

**Linked open question:** OQ-FMEA-004 — Add a `PaiseAmount int64` type with strict JSON
unmarshalling and a linter rule enforcing its use for all Razorpay amount fields. Add to
`docs/research/open-questions.md`.

---

### UF-05 — Read Replica Lag in Score Computation (MS-06)

**RPN: 18 (S=2, O=3, D=3)**

**Failure mode:** Reliability scores read from a stale replica are used for release
multiplier computation, resulting in incorrect payout amounts.

**Mitigation gap:** The architecture specifies that monthly release should query the
primary, but this is not enforced in the current implementation specification. A developer
who uses the read replica connection pool for the release query (which is the default in
connection pooling libraries) will introduce stale score reads.

**Recommended action:** Explicitly document in the payment service implementation that
the release multiplier query (`GetScore`) must use a primary-directed connection. Add a
connection parameter or a helper function `GetScoreFromPrimary` that bypasses the read
replica. Add a test that verifies the correct connection type is used.

**Linked open question:** OQ-FMEA-005 — Enforce primary-directed reads for release
multiplier computation in the payment service. Add to `docs/research/open-questions.md`.

---

### UF-06 — ENC-05 Cipher Path Divergence (ENC-05)

**RPN: 3 (S=3, O=1, D=1) — Low RPN but a pre-launch blocker**

**Failure mode:** The AONT encoding uses AES-256-CTR on an AES-NI machine; decoding on
a non-AES-NI machine uses ChaCha20-256. The two produce different outputs for the same
K, making the file unrecoverable on the decode device.

**This must be resolved before M1 closes.** The pointer file must record which cipher
was used for the AONT pass, OR the encoding pipeline must always use a single canonical
cipher (ChaCha20-256 everywhere) regardless of AES-NI availability, OR the decode path
must try both ciphers on canary failure.

**Recommended resolution:** Use ChaCha20-256 as the canonical AONT cipher on all devices.
Reserve the AES-256-CTR fast path for the inner AONT word encryption only when it can be
guaranteed that decode always runs on the same device class — which cannot be guaranteed
in a distributed recovery scenario. This simplifies the specification in [ADR-019](../decisions/ADR-019-client-side-encryption.md) and
eliminates this failure mode entirely.

**Linked open question:** OQ-FMEA-006 — Resolve the AONT cipher portability gap before
M1 closes. Add to `docs/research/open-questions.md`.

---

## 6. Failure Mode Coverage Map

This section maps each failure class to the ADRs and runbooks that provide mitigations,
enabling a reviewer to quickly identify which architectural decisions are relied upon most
heavily.

### By architectural decision

| ADR | Failure modes mitigated |
|-----|------------------------|
| [ADR-001](../decisions/ADR-001-coordination-architecture.md) | NET-05 (DHT validator) |
| [ADR-002](../decisions/ADR-002-proof-of-storage.md) | MS-10 (audit ceiling) |
| [ADR-003](../decisions/ADR-003-erasure-coding.md) | NET-04 (correlated ISP outage); ENC-06 (non-systematic RS) |
| [ADR-004](../decisions/ADR-004-repair-protocol.md) | NET-04 (repair after correlated failure) |
| [ADR-006](../decisions/ADR-006-polling-interval.md) | NET-03 (DHT record expiry); MS-08 (clock skew) |
| [ADR-007](../decisions/ADR-007-provider-exit-states.md) | PD-01 (DHCP → false departure) |
| [ADR-012](../decisions/ADR-012-payment-basis.md) | PAY-01 (Razorpay 5xx retry) |
| [ADR-014](../decisions/ADR-014-adversarial-defences.md) | NET-04 (correlated outage cap) |
| [ADR-015](../decisions/ADR-015-audit-trail.md) | MS-05 (Postgres primary failure) |
| [ADR-016](../decisions/ADR-016-payment-db-schema.md) | PAY-02 (double-trigger); PAY-03 (float arithmetic); PAY-07 (idempotency collision) |
| [ADR-019](../decisions/ADR-019-client-side-encryption.md) | ENC-04 (nonce counter); ENC-05 (cipher divergence); DO-03 (counter overflow) |
| [ADR-020](../decisions/ADR-020-key-management.md) | ENC-03 (Argon2id width); ENC-04 (nonce counter recovery); DO-01 (passphrase loss); DO-04 (upload race) |
| [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) | NET-01 (UDP blocked); NET-02 (relay failure); NET-06 (DCUtR failure); NET-07 (0-RTT replay) |
| [ADR-022](../decisions/ADR-022-encryption-erasure-order.md) | ENC-01 (K logging); ENC-02 (canary collision); ENC-05 (cipher divergence); ENC-06 (non-systematic RS) |
| [ADR-023](../decisions/ADR-023-provider-storage-engine.md) | PD-02 (disk corruption); PD-03 (compaction starvation); PD-04 (fallocate); PD-05 (crash recovery); PD-08 (writer deadlock) |
| [ADR-024](../decisions/ADR-024-economic-mechanism.md) | PAY-04 (RBI holiday table); MS-06 (stale score for release) |
| [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md) | MS-01 (single replica); MS-07 (background task); MS-09 (gossip partition) |
| [ADR-027](../decisions/ADR-027-cluster-audit-secret.md) | MS-03 (secrets at startup); MS-04 (secrets during operation) |
| [ADR-028](../decisions/ADR-028-provider-heartbeat.md) | PD-01 (DHCP rotation); PD-07 (provider clock skew); NET-03 (DHT expiry) |
| [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) | PAY-06 (cooling period check) |

### By failure class

| Failure class | Count | Highest RPN | Primary ADR |
|--------------|-------|-------------|-------------|
| Data confidentiality and key management | 5 | 32 (ENC-04) | ADR-019, ADR-020, ADR-022 |
| Payment correctness | 7 | 24 (PAY-03) | ADR-016 |
| Provider connectivity and NAT | 7 | 24 (PD-01) | ADR-021, ADR-028 |
| Microservice availability | 10 | 18 (MS-06, MS-08) | ADR-025, ADR-027 |
| Provider storage engine | 7 | 12 (PD-03, PD-04, PD-05, PD-06, PD-07) | ADR-023 |
| Data owner credentials | 4 | 25 (DO-02) | ADR-020 |

### Failure modes with no ADR coverage (operational controls only)

| ID | Control type | Responsible party |
|----|-------------|------------------|
| DO-02 | UX disclosure + physical security guidance | Product / Data owner |
| ENC-01 | Code review + CI grep | Engineering |
| PAY-05 | Monthly reconciliation runbook | Operations |
| MS-06 | Implementation convention (primary connection for release query) | Engineering |

These four items represent the residual risk posture of V2. All four have either been
accepted by design (DO-02), linked to concrete implementation hardening (ENC-01, MS-06),
or have a runbook (PAY-05). None require a new ADR to address — they require implementation
discipline and operational procedure.