# Audit Challenge Cycle

**ADR references:** [ADR-002](../../decisions/ADR-002-proof-of-storage.md) · [ADR-006](../../decisions/ADR-006-polling-interval.md) · [ADR-008](../../decisions/ADR-008-reliability-scoring.md) · [ADR-014](../../decisions/ADR-014-adversarial-defences.md) · [ADR-015](../../decisions/ADR-015-audit-trail.md) · [ADR-017](../../decisions/ADR-017-audit-receipt-schema.md) · [ADR-023](../../decisions/ADR-023-provider-storage-engine.md) · [ADR-024](../../decisions/ADR-024-economic-mechanism.md) · [ADR-027](../../decisions/ADR-027-cluster-audit-secret.md) · [ADR-028](../../decisions/ADR-028-provider-heartbeat.md)  
**Architecture reference:** [architecture.md §14 Audit System](../architecture.md#14-audit-system) · [architecture.md §22 Runtime Flows — Flow 2](../architecture.md#22-runtime-flows)

---

## Overview

The audit system is how the microservice verifies that providers actually hold the chunks they were assigned. One challenge is issued per assigned chunk per 24-hour window. The response deadline is per-provider and timing-based — making just-in-time retrieval attacks infeasible without local disk storage ([ADR-014](../../decisions/ADR-014-adversarial-defences.md)). The receipt is written to an INSERT-only Postgres table using a crash-safe two-phase write ([ADR-015](../../decisions/ADR-015-audit-trail.md)).

---

## Phase 1 — Scheduler Fires and Challenge Generation

```
Audit Scheduler                  Microservice                  Postgres
────────────────────────────────────────────────────────────────────────────
     │                               │                               │
     │  Every 24-hour window:        │                               │
     │  select all chunk_assignments │                               │
     │  WHERE status = 'ACTIVE'      │                               │
     │  AND not yet challenged in    │                               │
     │  current window               │                               │
     │                               │                               │
     │  Challenge timing is          │                               │
     │  RANDOMISED within the window │                               │
     │  to prevent JIT pre-caching   │                               │
     │  (ADR-002, ADR-014 Defence 3) │                               │
     │                               │                               │
     │  FOR EACH chunk:              │                               │
     │                               │                               │
     ├─ Generate challenge nonce:    │                               │
     │   server_ts = NOW()           │                               │
     │   version_byte = N (mod 256)  │   N = current server_secret  │
     │                                    version (ADR-027)          │
     │   nonce = version_byte (1B)   │                               │
     │           ‖ HMAC-SHA256(      │                               │
     │               server_secret_vN,                               │
     │               chunk_id ‖ server_ts                            │
     │             )                 │                               │
     │   total: 33 bytes / 66 hex    │                               │
     │   (ADR-027; version byte      │                               │
     │    allows cross-replica       │                               │
     │    validation after failover) │                               │
     │                               │                               │
     ├─ Compute per-provider         │                               │
     │  response deadline:           │                               │
     │   deadline_ms =               │                               │
     │     ceil((chunk_size_kb /     │                               │
     │            p95_throughput_kbps)                               │
     │            × 1500)            │                               │
     │   (ADR-014 Defence 2;         │                               │
     │    p95_throughput_kbps from   │                               │
     │    vetting-period measurements│                               │
     │    — never self-declared)     │                               │
     │                               │                               │
     ├─ Compute per-provider RTO:    │                               │
     │   RTO = avg_rtt_ms            │                               │
     │         + 4 × var_rtt_ms      │                               │
     │   (TCP-style, ADR-006)        │                               │
     │   new providers: pool median  │                               │
     │   (≥5 samples required for    │                               │
     │    per-provider formula)      │                               │
     │                               │                               │
```

---

## Phase 2 — Challenge Dispatch

```
Microservice                     Provider Daemon
────────────────────────────────────────────────────────────────────────────
     │                               │
     ├─ Address resolution:          │
     │  PRIMARY: last_known_multiaddrs
     │  from providers table         │
     │  (set by 4-hourly heartbeat,  │
     │   ADR-028)                    │
     │                               │
     │  FALLBACK (if multiaddr_stale │
     │  = true AND no heartbeat      │
     │  in 8h):                      │
     │    DHT FIND_NODE lookup       │
     │    for provider's Peer ID     │
     │    (ADR-001)                  │
     │                               │
     │── libp2p dial ───────────────►│
     │   QUIC v1 (primary)           │
     │   or TCP+Noise (fallback)     │
     │   (ADR-021)                   │
     │                               │
     │   0-RTT DISABLED for          │
     │   audit challenge streams     │
     │   (replay risk, ADR-021)      │
     │                               │
     │── AUDIT_CHALLENGE stream ────►│
     │   {                           │
     │     chunk_id: "<hex32>",      │
     │     challenge_nonce: "<hex66>"│
     │   }                           │
     │                               │
     │   deadline_ms timer starts    │
     │                               │
```

---

## Phase 3 — Provider Response Computation

```
Provider Daemon (local, no network during computation)
────────────────────────────────────────────────────────────────────────────
     │
     │  receive (chunk_id, challenge_nonce)
     │
     ├─ Step 1: Bloom filter check (ADR-023)
     │   query RocksDB Bloom filter for chunk_id
     │
     │   IF absent (false negative rate ~1%):
     │     return FAIL immediately
     │     (no disk I/O — Bloom filter is in memory)
     │
     ├─ Step 2: RocksDB index lookup (ADR-023)
     │   IF Bloom filter positive:
     │     GET chunk_id from RocksDB
     │       → (vlog_offset, chunk_size)
     │     (typically from block cache: no disk I/O)
     │
     ├─ Step 3: vLog read (1 random disk I/O)
     │   read 262,212 bytes at vlog_offset
     │   target latency:
     │     SSD: ≤ 1 ms
     │     HDD: ≤ 12–15 ms
     │   (ADR-023, NFR-008)
     │
     ├─ Step 4: Content hash verification (ADR-023)
     │   computed = SHA-256(chunk_data)
     │
     │   IF computed ≠ stored content_hash:
     │     ← SILENT DISK CORRUPTION DETECTED
     │     return audit_result = FAIL
     │     with corruption error code
     │     (NFR-015: wrong hash never returned;
     │      FAIL is returned instead)
     │
     │     side effect: set accelerated_reaudit = true
     │     (Schroeder et al. Paper 32: 30× elevated
     │      failure probability after first error;
     │      all chunks on this provider get re-audited,
     │      ADR-008)
     │
     ├─ Step 5: Response hash computation (ADR-002)
     │   response_hash = SHA-256(chunk_data ‖ challenge_nonce)
     │   (uncomputable without the actual chunk data)
     │
     ├─ Step 6: Sign audit receipt (ADR-017)
     │   provider_sig = Ed25519_sign(
     │     private_key = provider_private_key,
     │     message = canonical_json({
     │       chunk_id, challenge_nonce, response_hash,
     │       server_challenge_ts, provider_id, ...
     │     })  ← all fields except provider_sig itself,
     │           sorted keys, no trailing whitespace
     │   )
     │
```

---

## Phase 4 — Receipt Submission and Two-Phase Write

```
Provider Daemon          Microservice                         Postgres
────────────────────────────────────────────────────────────────────────────
     │                        │                                    │
     │── POST /api/v1/audit/receipt ────────────────────────────  │
     │   {                    │                                    │
     │     receipt_id: "<UUIDv7>",                                 │
     │     chunk_id, file_id, │                                    │
     │     provider_id,       │                                    │
     │     challenge_nonce: "<66 hex>",
     │     server_challenge_ts,                                    │
     │     response_hash,     │                                    │
     │     response_latency_ms,                                    │
     │     provider_sig       │                                    │
     │   }                    │                                    │
     │                        │                                    │
     │                        ├─ VALIDATION:                      │
     │                        │  1. verify provider_sig against   │
     │                        │     registered ed25519_public_key │
     │                        │                                   │
     │                        │  FAILURE PATH:                    │
     │                        │  invalid signature                │
     │◄── 403 INVALID_BODY_SIGNATURE                              │
     │                        │                                   │
     │                        │  2. check challenge_nonce is      │
     │                        │     66 hex chars (33 bytes)       │
     │                        │                                   │
     │                        │  FAILURE PATH:                    │
     │                        │  64-char nonce (old format)       │
     │◄── 400 INVALID_CHALLENGE_NONCE                             │
     │                        │                                   │
     │                        │  3. extract version_byte          │
     │                        │     load server_secret_vN         │
     │                        │     (from 5-min in-memory cache   │
     │                        │      or secrets manager, ADR-027) │
     │                        │                                   │
     │                        │  4. recompute expected nonce:     │
     │                        │     expected = version_byte       │
     │                        │               ‖ HMAC-SHA256(      │
     │                        │                   server_secret_vN│
     │                        │                   chunk_id        │
     │                        │                   ‖ server_ts)    │
     │                        │     verify received nonce matches │
     │                        │                                   │
     │                        │  5. compute expected response:    │
     │                        │     look up chunk_id in           │
     │                        │     chunk_assignments             │
     │                        │     compare response_hash         │
     │                        │                                   │
     │                        ├─ PHASE 1 WRITE (crash-safe):     │
     │                        │  INSERT audit_receipts (PENDING)  │
     │                        │  {                                │
     │                        │    receipt_id, schema_version=1,  │
     │                        │    chunk_id, file_id, provider_id,│
     │                        │    challenge_nonce (33 B),        │
     │                        │    server_challenge_ts,           │
     │                        │    response_hash,                 │
     │                        │    response_latency_ms,           │
     │                        │    audit_result = NULL,    ← PENDING
     │                        │    provider_sig,                  │
     │                        │    service_sig = NULL,    ← not yet
     │                        │    jit_flag = (compute below)     │
     │                        │  }                                │
     │                        │  (durable before Phase 2,        │
     │                        │   ADR-015 two-phase write)        │
     │                        │                                   │
     │                        ├─ JIT FLAG EVALUATION (ADR-014):   │
     │                        │  anomaly_threshold =              │
     │                        │    (chunk_size_kb /               │
     │                        │     p95_throughput_kbps) × 0.3    │
     │                        │  jit_flag =                       │
     │                        │    response_latency_ms            │
     │                        │    < anomaly_threshold            │
     │                        │                                   │
     │                        │  IF 3+ jit_flags from same        │
     │                        │     provider in 7-day window:     │
     │                        │    apply 0.5× weight to that      │
     │                        │    provider's PASS results        │
     │                        │    for next 30 days               │
     │                        │                                   │
     │                        │  IF 3+ jit_flags AND identical    │
     │                        │     response_latency_ms ±5ms:     │
     │                        │    escalate to manual review      │
     │                        │    (collusion signal)             │
     │                        │                                   │
     │                        ├─ PHASE 2 WRITE:                   │
     │                        │  UPDATE audit_receipts            │
     │                        │  SET audit_result = PASS|FAIL,    │
     │                        │      service_sig = Ed25519_sign(  │
     │                        │        provider_sig ‖ service_ts),│
     │                        │      service_countersign_ts = NOW()│
     │                        │  WHERE receipt_id = ?             │
     │                        │  AND audit_result IS NULL         │
     │                        │                                   │
     │                        │  IDEMPOTENT RETRY (ADR-015):      │
     │                        │  IF duplicate challenge_nonce:    │
     │                        │    return existing countersig     │
     │                        │    (UNIQUE index on nonce)        │
     │                        │                                   │
     │◄── 200 {           ────│                                   │
     │     receipt_id,        │                                   │
     │     audit_result: PASS|FAIL,
     │     service_sig: "...",│                                   │
     │     service_countersign_ts,
     │     jit_flag           │                                   │
     │   }                    │                                   │
     │                        │                                   │
     │   daemon stores        │                                   │
     │   service_sig as       │                                   │
     │   payment evidence     │                                   │
     │                        │                                   │
```

---

## Phase 5 — Crash Recovery for PENDING Rows

If the microservice crashes between Phase 1 (INSERT PENDING) and Phase 2 (UPDATE to final result), the provider receives no countersignature and will be re-challenged on the next audit cycle.

```
Microservice (on restart)                              Postgres
────────────────────────────────────────────────────────────────────────────
     │                                                      │
     │  GC process runs (background, ADR-015):              │
     │                                                      │
     │── SELECT receipt_id FROM audit_receipts ─────────────►│
     │   WHERE audit_result IS NULL                         │
     │   AND abandoned_at IS NULL                           │
     │   AND server_challenge_ts < NOW() - INTERVAL '48 hours'
     │                                                      │
     │── UPDATE audit_receipts ──────────────────────────────►│
     │   SET abandoned_at = NOW()                           │
     │   WHERE receipt_id IN (above set)                    │
     │   (GC policy in row security; does not set result)   │
     │                                                      │
     │   ABANDONED rows are excluded from all scoring queries│
     │   (treated as neither PASS nor FAIL)                 │
     │                                                      │
     │   Provider will receive a new challenge on the next  │
     │   24-hour audit cycle (challenge_nonce uniqueness    │
     │   allows re-challenge since nonce was not repeated)  │
     │                                                      │
```

---

## Phase 6 — Reliability Score Update and Release Multiplier Recalculation

```
Microservice (async, after Phase 2 write)            Postgres
────────────────────────────────────────────────────────────────────────────
     │                                                    │
     │  Scoring is async — does not block receipt return  │
     │                                                    │
     ├─ UPDATE providers ────────────────────────────────►│
     │   SET consecutive_audit_passes +=1   (on PASS)    │
     │   SET consecutive_audit_passes = 0   (on FAIL/TIMEOUT)
     │                                                    │
     │   (consecutive_audit_passes drives VETTING→ACTIVE  │
     │    transition at 80, ADR-005)                      │
     │                                                    │
     ├─ UPDATE provider RTO stats ───────────────────────►│
     │   EWMA update of avg_rtt_ms and var_rtt_ms using  │
     │   this response's response_latency_ms              │
     │   (ADR-006)                                        │
     │                                                    │
     ├─ REFRESH MATERIALIZED VIEW mv_provider_scores ────►│
     │   (async, throttled when db_read_p99 > 50ms,      │
     │    ADR-025 background task throttling)             │
     │                                                    │
     │   mv_provider_scores computes:                     │
     │     score_24h  (weight 0.5)                        │
     │     score_7d   (weight 0.3)                        │
     │     score_30d  (weight 0.2)                        │
     │     score_composite                                │
     │   (ADR-008, three rolling windows)                 │
     │                                                    │
     ├─ DUAL-WINDOW CHECK (ADR-024):                      │
     │   dual_window_flag =                               │
     │     (score_30d − score_7d) > 0.20                 │
     │                                                    │
     │   IF dual_window_flag:                             │
     │     next month's release uses                      │
     │     min(multiplier(score_30d),                     │
     │         multiplier(score_7d))                      │
     │     (catches providers milking reputation          │
     │      before planned silent departure)              │
     │                                                    │
     ├─ RELEASE MULTIPLIER:                               │
     │   score_30d ≥ 0.95  → 1.00  (full release)       │
     │   score_30d ≥ 0.80  → 0.75  (partial)            │
     │   score_30d ≥ 0.65  → 0.50  (partial)            │
     │   score_30d < 0.65  → 0.00  (hold in full)       │
     │   withheld portions roll forward, not seized       │
     │   (ADR-024)                                        │
     │                                                    │
     ├─ ESCROW CREDIT (on PASS only):                     │
     │   payout_per_audit_pass =                          │
     │     (storage_rate_paise × chunk_size_GB)           │
     │     / audits_per_month                             │
     │                                                    │
     │   INSERT escrow_events:                            │
     │   { provider_id, event_type=DEPOSIT,               │
     │     amount_paise = payout_per_audit_pass,          │
     │     idempotency_key = SHA-256(provider_id ‖ receipt_id) }
     │   (ADR-016; idempotency prevents double-payment)   │
     │                                                    │
     │   FAIL and TIMEOUT → no DEPOSIT event              │
     │   (ADR-012: payment per audit PASSED)              │
     │                                                    │
```

---

## TIMEOUT Path

```
Audit Scheduler                  Microservice                  Postgres
────────────────────────────────────────────────────────────────────────────
     │                               │                               │
     │  RTO timer expires            │                               │
     │  (no response received within │                               │
     │   RTO = avg_rtt_ms            │                               │
     │         + 4 × var_rtt_ms)     │                               │
     │                               │                               │
     │                               ├─ PHASE 1:                    │
     │                               │  INSERT audit_receipts        │
     │                               │  { audit_result = NULL,       │
     │                               │    response_hash = NULL,      │
     │                               │    provider_sig = NULL }      │
     │                               │                               │
     │                               ├─ PHASE 2:                    │
     │                               │  UPDATE audit_receipts        │
     │                               │  SET audit_result = 'TIMEOUT',│
     │                               │      service_sig = Ed25519_sign(
     │                               │        timeout declaration),  │
     │                               │      service_countersign_ts   │
     │                               │                               │
     │                               │  (TIMEOUT is self-signed by   │
     │                               │   the microservice; no        │
     │                               │   provider_sig to countersign)│
     │                               │                               │
     │                               ├─ UPDATE mv_provider_scores ───►│
     │                               │  score decrements (TIMEOUT    │
     │                               │  counted against provider)    │
     │                               │                               │
     │                               │  NO DEPOSIT event for TIMEOUT │
     │                               │  (ADR-012)                    │
     │                               │                               │
     │   departure detector:         │                               │
     │   if last_heartbeat_ts        │                               │
     │   > NOW() − 72h:              │                               │
     │   → silent departure path     │                               │
     │   (see sequence diagram 04)   │                               │
     │                               │                               │
```

---

## Key Invariants

| Invariant | Source |
|---|---|
| `audit_receipts` is INSERT-only; PENDING→final is the only permitted UPDATE | [ADR-015](../../decisions/ADR-015-audit-trail.md), Invariant 1 |
| challenge_nonce is BYTEA(33) — 1 version byte + 32-byte HMAC. Never BYTEA(32) | [ADR-027](../../decisions/ADR-027-cluster-audit-secret.md) |
| Correct response_hash is uncomputable without the actual chunk data | [ADR-002](../../decisions/ADR-002-proof-of-storage.md) |
| Content hash mismatch returns FAIL; wrong hash is never returned | [ADR-023](../../decisions/ADR-023-provider-storage-engine.md) |
| 0-RTT disabled for audit challenge streams (replay risk) | [ADR-021](../../decisions/ADR-021-p2p-transfer-protocol.md) |
| Challenge timing is randomised within the 24h window | [ADR-002](../../decisions/ADR-002-proof-of-storage.md), [ADR-014](../../decisions/ADR-014-adversarial-defences.md) |
| Escrow DEPOSIT events only on PASS; FAIL and TIMEOUT earn nothing | [ADR-012](../../decisions/ADR-012-payment-basis.md) |
| Duplicate nonce → return existing countersig (idempotent) | [ADR-015](../../decisions/ADR-015-audit-trail.md) |