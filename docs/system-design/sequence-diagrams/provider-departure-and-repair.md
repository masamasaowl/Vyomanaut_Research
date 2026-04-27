# Provider Departure and Repair

**ADR references:** [ADR-003](../../decisions/ADR-003-erasure-coding.md) · [ADR-004](../../decisions/ADR-004-repair-protocol.md) · [ADR-005](../../decisions/ADR-005-peer-selection.md) · [ADR-006](../../decisions/ADR-006-polling-interval.md) · [ADR-007](../../decisions/ADR-007-provider-exit-states.md) · [ADR-013](../../decisions/ADR-013-consistency-model.md) · [ADR-014](../../decisions/ADR-014-adversarial-defences.md) · [ADR-016](../../decisions/ADR-016-payment-db-schema.md) · [ADR-022](../../decisions/ADR-022-encryption-erasure-order.md) · [ADR-024](../../decisions/ADR-024-economic-mechanism.md)  
**Architecture reference:** [architecture.md §15 Repair System](../architecture.md#15-repair-system) · [architecture.md §22 Runtime Flows — Flow 3](../architecture.md#22-runtime-flows)

---

## Overview

There are four exit states for a provider ([ADR-007](../../decisions/ADR-007-provider-exit-states.md)). Each has different repair triggers and escrow consequences. This document covers all four, with the primary focus on the most critical path: silent departure (72h threshold crossed) and the repair cycle that follows.

---

## Exit State Map

```
                    Provider departure trigger
                           │
          ┌────────────────┼────────────────────┐
          │                │                    │
          ▼                ▼                    ▼
   Temporary absence  Promised downtime   Permanent departure
   < 72h, no notice   0–72h, declared      ≥ 72h silent
   in advance         in advance           OR announced

   Action: wait        Action: wait         Action: immediate
   Score: decrement    Score: protected      repair trigger
   Escrow: no change   Escrow: no change     Escrow: seize (silent)
                       IF overrun:           OR proportional
                       reclassify →          release (announced)
                       silent departure
```

---

## Path A — Silent Departure (primary path)

### A1 — Detection (72h threshold)

```
Departure Detector (runs continuously)    Microservice         Postgres
────────────────────────────────────────────────────────────────────────────
     │                                        │                     │
     │  Periodic scan (every 5 min):          │                     │
     │                                        │                     │
     │── SELECT provider_id                   │                     │
     │   FROM providers ──────────────────────────────────────────►│
     │   WHERE status = 'ACTIVE'              │                     │
     │   AND last_heartbeat_ts                │                     │
     │       < NOW() - INTERVAL '72 hours'    │                     │
     │                                        │                     │
     │   ALSO: any provider with              │                     │
     │   last_heartbeat_ts < NOW() - 72h      │                     │
     │   in VETTING status → same treatment   │                     │
     │                                        │                     │
     │   (ADR-006: 72h threshold calibrated   │                     │
     │    to Bolosky bimodal distribution;    │                     │
     │    99.7% of weekend absences resolve   │                     │
     │    within 70h — 72h is the first safe  │                     │
     │    threshold above the weekend peak)   │                     │
     │                                        │                     │
     │   FOR EACH departed provider:          │                     │
     │                                        │                     │
```

### A2 — Escrow Seizure and Status Update

```
Microservice (payment service)                         Postgres
────────────────────────────────────────────────────────────────────────────
     │                                                      │
     │  All 6 operations below are NON-I-CONFLUENT:         │
     │  must be executed by the single authoritative        │
     │  payment microservice (ADR-013)                      │
     │                                                      │
     ├─ 1. FREEZE provider escrow:                          │
     │   UPDATE providers ───────────────────────────────►  │
     │   SET frozen = true                                  │
     │   WHERE provider_id = ?                              │
     │   (blocks new DEPOSIT events, ADR-024)               │
     │                                                      │
     ├─ 2. Compute held escrow balance:                     │
     │   SELECT SUM(DEPOSIT) - SUM(RELEASE + SEIZURE)       │
     │   FROM escrow_events                                 │
     │   WHERE provider_id = ?                              │
     │   (30-day rolling window only)                       │
     │                                                      │
     ├─ 3. INSERT escrow_events (SEIZURE): ──────────────►  │
     │   {                                                  │
     │     provider_id,                                     │
     │     event_type = 'SEIZURE',                          │
     │     amount_paise = held_balance,                     │
     │     idempotency_key =                                │
     │       SHA-256(provider_id ‖ "seizure" ‖ departed_at) │
     │   }                                                  │
     │   (ADR-016: INSERT-only escrow ledger)               │
     │   seized funds go to repair reserve fund             │
     │                                                      │
     ├─ 4. Razorpay Route reversal (if pending transfer):   │
     │   POST /v1/transfers/{id}/reversals ──► Razorpay    │
     │   (if any transfer has not yet settled)              │
     │                                                      │
     ├─ 5. UPDATE providers: ─────────────────────────────► │
     │   SET status = 'DEPARTED',                           │
     │       departed_at = NOW()                            │
     │   WHERE provider_id = ?                              │
     │   (soft delete — row never physically removed,       │
     │    ADR-007, ADR-013 Invariant 3)                     │
     │                                                      │
     ├─ 6. DELETE chunk_assignments:                        │
     │   DELETE FROM chunk_assignments ───────────────────► │
     │   WHERE provider_id = ?                              │
     │   AND status = 'ACTIVE'                              │
     │   (stops all further challenge issuance;             │
     │    this is the one permitted hard DELETE —           │
     │    these are routing records, not audit log)         │
     │                                                      │
     ├─ INSERT seizure receipt to audit log: ──────────────►│
     │   (I-confluent INSERT, ADR-013)                      │
     │                                                      │
```

### A3 — Repair Job Queuing

```
Microservice (repair scheduler)                        Postgres
────────────────────────────────────────────────────────────────────────────
     │                                                      │
     │  For each chunk previously assigned to the           │
     │  departed provider:                                  │
     │                                                      │
     ├─ Query surviving shard count: ────────────────────►  │
     │   SELECT COUNT(*) FROM chunk_assignments             │
     │   WHERE segment_id = ?                               │
     │   AND status IN ('ACTIVE', 'REPAIRING')              │
     │                                                      │
     ├─ Determine repair priority (ADR-004):                │
     │                                                      │
     │   IF count <= 16 (s = reconstruction floor):        │
     │     INSERT repair_jobs {                             │
     │       trigger_type = 'EMERGENCY_FLOOR',             │
     │       priority = 'PERMANENT_DEPARTURE',             │
     │       available_shard_count = count                 │
     │     }                                               │
     │     ← bypasses normal queue ordering                │
     │                                                      │
     │   IF 16 < count <= 24 (s + r0 lazy threshold):     │
     │     INSERT repair_jobs {                            │
     │       trigger_type = 'SILENT_DEPARTURE',            │
     │       priority = 'PERMANENT_DEPARTURE',             │
     │       available_shard_count = count                 │
     │     }                                               │
     │                                                      │
     │   IF count > 24:                                    │
     │     INSERT repair_jobs {                            │
     │       trigger_type = 'SILENT_DEPARTURE',            │
     │       priority = 'PERMANENT_DEPARTURE',             │
     │       available_shard_count = count                 │
     │     }                                               │
     │     (departure still triggers repair regardless     │
     │      of current shard count — provider is gone)     │
     │                                                      │
     │  PRIORITY ORDERING (ADR-004, Paper 39):             │
     │  PERMANENT_DEPARTURE jobs drain before PRE_WARNING  │
     │  EMERGENCY_FLOOR jobs: front of PERMANENT_DEPARTURE │
     │  Within a tier: FIFO on created_at                  │
     │                                                      │
```

### A4 — Repair Execution

```
Repair Worker       Surviving Providers (×16)      Replacement Provider
────────────────────────────────────────────────────────────────────────────
     │                    │                              │
     │  Dequeue job       │                              │
     │  from repair_jobs  │                              │
     │  WHERE status='QUEUED'
     │  priority ordered  │                              │
     │  (ADR-004)         │                              │
     │                    │                              │
     │── UPDATE repair_jobs SET status='IN_PROGRESS'     │
     │                    │                              │
     ├─ Find 16 surviving shard holders:                 │
     │   SELECT chunk_assignments                        │
     │   WHERE segment_id = ?                            │
     │   AND status = 'ACTIVE'                           │
     │   LIMIT 16                                        │
     │                    │                              │
     │── libp2p dial ────►│ (×16 parallel)               │
     │── CHUNK_REQUEST ──►│                              │
     │   { chunk_id }     │                              │
     │                    │                              │
     │   Provider reads   │                              │
     │   from vLog,       │                              │
     │   verifies         │                              │
     │   content_hash     │                              │
     │                    │                              │
     │◄── 256 KB shard ───│                              │
     │                    │                              │
     ├─ RS decode (any 16 of surviving shards):          │
     │   recover AONT package                            │
     │   (ADR-022, ADR-003)                             │
     │                    │                              │
     ├─ Re-encode missing shards:                        │
     │   RS(s=16, r=40) → remaining missing shards      │
     │   (only the missing shards; existing shards       │
     │   on surviving providers are untouched)           │
     │                    │                              │
     ├─ Select replacement provider:                     │
     │   Power of Two Choices (ADR-005)                  │
     │   weighted by reliability score                   │
     │                    │                              │
     │   ENFORCE 20% ASN CAP (ADR-014, ADR-003):        │
     │   replacement must not push any ASN over          │
     │   floor(0.20 × 56) = 11 shards for this segment  │
     │                    │                              │
     │── libp2p dial ─────────────────────────────────►│
     │── CHUNK_UPLOAD ────────────────────────────────►│
     │   { chunk_id, shard_data: 256 KB }               │
     │                    │                              │
     │                    │              stores to vLog  │
     │                    │              fsync(); INSERT │
     │                    │              RocksDB         │
     │                    │                              │
     │◄── signed upload receipt ──────────────────────────
     │                    │                              │
     │                    │                              │
     ├─ UPDATE chunk_assignments: ──────────────────────►│ (Postgres)
     │   INSERT new assignment for replacement provider  │
     │   UPDATE old assignment status → 'DELETED'        │
     │   (routing record; ADR-007)                       │
     │                    │                              │
     ├─ UPDATE repair_jobs: ────────────────────────────►│
     │   SET status = 'COMPLETED',                       │
     │       completed_at = NOW()                        │
     │                    │                              │
     │   APPEND repair completion to audit log           │
     │   (I-confluent INSERT, ADR-013)                   │
     │                    │                              │
```

---

## Path B — Announced Departure

```
Provider Daemon             Microservice                    Postgres
────────────────────────────────────────────────────────────────────────────
     │                           │                               │
     │── POST /api/v1/provider/depart ──────────────────────────►│
     │   {                       │                               │
     │     depart_at: "<ISO8601>",  ← optional future timestamp │
     │     provider_sig: "<sig>" │                               │
     │   }                       │                               │
     │                           │                               │
     │                           ├─ Compute proportional release:│
     │                           │   fraction_complete =         │
     │                           │     (NOW() - period_start)    │
     │                           │     / 30 days                 │
     │                           │   release_amount =            │
     │                           │     held_balance              │
     │                           │     × fraction_complete       │
     │                           │   (ADR-024, ADR-007)          │
     │                           │                               │
     │                           ├─ INSERT escrow_events RELEASE ►│
     │                           │   (proportional amount)       │
     │                           │   (ADR-016)                   │
     │                           │                               │
     │                           ├─ UPDATE providers:            │
     │                           │   SET status = 'DEPARTED'     │
     │                           │   (soft delete, ADR-007)      │
     │                           │                               │
     │                           ├─ Queue repair jobs            │
     │                           │   trigger_type =              │
     │                           │   'ANNOUNCED_DEPARTURE'       │
     │                           │   priority =                  │
     │                           │   'PERMANENT_DEPARTURE'       │
     │                           │   (immediate trigger,         │
     │                           │    ADR-004)                   │
     │                           │                               │
     │◄── 200 {              ────│                               │
     │     repair_jobs_queued,   │                               │
     │     escrow_release_paise  │                               │
     │   }                       │                               │
     │                           │                               │
     │   ← Provider who departs  │                               │
     │   cleanly receives        │                               │
     │   proportional earnings   │                               │
     │   rather than seizure     │                               │
     │                           │                               │
```

---

## Path C — Promised Downtime

```
Provider Daemon             Microservice                    Postgres
────────────────────────────────────────────────────────────────────────────
     │                           │                               │
     │── POST /api/v1/provider/downtime ──────────────────────►│
     │   {                       │                               │
     │     promised_return_at:   │                               │
     │       "<NOW + 1h to 72h>",│                               │
     │     provider_sig: "<sig>" │                               │
     │   }                       │                               │
     │                           │                               │
     │                           ├─ Record downtime window       │
     │                           │   in providers table          │
     │                           │   (no repair triggered)       │
     │                           │                               │
     │◄── 200 { penalty_fires_at }                              │
     │                           │                               │
     │   During downtime window: │                               │
     │   - audit FAIL results    │                               │
     │     do NOT decrement score│                               │
     │   - no escrow penalty     │                               │
     │                           │                               │
     │   IF provider does NOT    │                               │
     │   reconnect by            │                               │
     │   promised_return_at:     │                               │
     │                           │                               │
     │   departure detector      │                               │
     │   reclassifies to         │                               │
     │   SILENT_DEPARTURE        │                               │
     │   → escrow seizure        │                               │
     │   → repair triggered      │                               │
     │   (ADR-007, FR-033)       │                               │
     │                           │                               │
```

---

## Lazy Repair Trigger (independent of departure)

Repair also fires when fragment count drops to the lazy threshold, even without a single identifiable departure (e.g. multiple small silent departures over time, each within the lazy window individually).

```
Repair Trigger Monitor            Microservice                 Postgres
────────────────────────────────────────────────────────────────────────────
     │                                 │                             │
     │  Continuous monitor:            │                             │
     │  REFRESH mv_segment_shard_counts│                             │
     │  (async after each audit cycle) │                             │
     │                                 │                             │
     │── SELECT segment_id             │                             │
     │   FROM mv_segment_shard_counts ──────────────────────────────►│
     │   WHERE available_shard_count                                 │
     │         <= 24  (s + r0)          │                             │
     │   AND NOT EXISTS (               │                             │
     │     SELECT 1 FROM repair_jobs    │                             │
     │     WHERE segment_id = s.segment_id
     │     AND status IN ('QUEUED',     │                             │
     │                    'IN_PROGRESS')│                             │
     │   )                              │                             │
     │                                  │                             │
     │  FOR EACH qualifying segment:    │                             │
     │                                  │                             │
     │   IF count = 16 (floor):         │                             │
     │     trigger_type = 'EMERGENCY_FLOOR'
     │     priority = 'PERMANENT_DEPARTURE'
     │     → immediate, bypasses queue  │                             │
     │     (ADR-004)                    │                             │
     │                                  │                             │
     │   IF 16 < count <= 24:           │                             │
     │     trigger_type = 'THRESHOLD_WARNING'
     │     priority = 'PRE_WARNING'     │                             │
     │     → waits behind               │                             │
     │       PERMANENT_DEPARTURE jobs   │                             │
     │     (ADR-004, Paper 39)          │                             │
     │                                  │                             │
     │── INSERT repair_jobs ─────────────────────────────────────────►│
     │                                  │                             │
```

---

## Bandwidth Capacity Notes

From [capacity.md §3.2](../capacity.md):

| N providers | Storage/provider | Qpeek | Repair window | Within 12h budget? |
|---|---|---|---|---|
| 56 (launch floor) | 50 GB | 44 GB | ~22 hours | **No — manual oversight required** |
| 500 | 50 GB | 396 GB | ~18 hours | Borderline |
| 1,000 | 50 GB | 793 GB | ~8 hours | Yes |

At N < 500 providers (private beta), manual monitoring of `repair_queue_depth` is mandatory before accepting any new provider departure within a 24-hour window.

---

## Reconnection After Departure

```
Departed Provider Daemon     Microservice
──────────────────────────────────────────
     │                           │
     │   comes back online,      │
     │   attempts heartbeat      │
     │                           │
     │── POST /api/v1/provider/heartbeat
     │   (with old JWT)          │
     │                           │
     │   microservice checks     │
     │   providers.status        │
     │   → 'DEPARTED'            │
     │                           │
     │◄── 403 PROVIDER_DEPARTED ──│
     │                           │
     │   daemon logs 403 and     │
     │   halts operation         │
     │                           │
     │   Provider's chunk data   │
     │   on disk is incidental:  │
     │   not tracked, not audited│
     │   not paid for            │
     │                           │
     │   To re-join: full        │
     │   registration required   │
     │   with a new phone number │
     │   (ADR-007)               │
     │                           │
```

---

## Key Invariants

| Invariant | Source |
|---|---|
| Provider rows are never physically deleted — status = DEPARTED only | [ADR-007](../../decisions/ADR-007-provider-exit-states.md), [ADR-013](../../decisions/ADR-013-consistency-model.md) |
| chunk_assignments rows are hard-deleted on departure to stop challenge issuance | [ADR-007](../../decisions/ADR-007-provider-exit-states.md) |
| Seized escrow goes to repair reserve fund (funds replacement provider onboarding) | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) |
| Replacement providers must satisfy 20% ASN cap | [ADR-014](../../decisions/ADR-014-adversarial-defences.md), [ADR-003](../../decisions/ADR-003-erasure-coding.md) |
| PERMANENT_DEPARTURE jobs drain before PRE_WARNING | [ADR-004](../../decisions/ADR-004-repair-protocol.md), Paper 39 |
| EMERGENCY_FLOOR (count = 16) triggers immediately, bypasses queue ordering | [ADR-004](../../decisions/ADR-004-repair-protocol.md) |
| Lazy repair threshold: repair fires at count ≤ 24, not at every single departure | [ADR-003](../../decisions/ADR-003-erasure-coding.md), [ADR-004](../../decisions/ADR-004-repair-protocol.md) |
| 72h threshold is calibrated to Bolosky bimodal distribution (weekend peak = 70h) | [ADR-006](../../decisions/ADR-006-polling-interval.md) |