# Vyomanaut V2 — Repair Flow Sequence Diagram

**Document ID:** `VYOM-SEQ-003`
**Version:** 1.0
**Status:** Authoritative
**Date:** April 2026
**Author:** Vyomanaut Engineering
**Repository:** [masamasaowl/Vyomanaut_Research](https://github.com/masamasaowl/Vyomanaut_Research)
**Companion documents:**
- [architecture.md §15 Repair System](../architecture.md#15-repair-system)
- [architecture.md §22 Runtime Flows — Flow 3](../architecture.md#22-runtime-flows)
- [requirements.md §6.9 Repair System](../requirements.md#69-repair-system)
- [ADR-004](../../decisions/ADR-004-repair-protocol.md) · [ADR-007](../../decisions/ADR-007-provider-exit-states.md) · [ADR-014](../../decisions/ADR-014-adversarial-defences.md) · [ADR-024](../../decisions/ADR-024-economic-mechanism.md) · [ADR-029](../../decisions/ADR-029-bootstrap-minimum-viable-network.md)

---

## Overview

This diagram covers the self-healing loop: how the system detects that a provider has
gone permanently silent, triggers repair for its lost fragments, downloads surviving
shards, re-encodes the missing ones, and distributes them to new providers — all without
human intervention. The primary correctness properties are: (1) lazy repair defers
reconstruction until `available_shard_count ≤ 24` (s + r0), never firing on every
transient absence; (2) `PERMANENT_DEPARTURE` jobs always drain before `PRE_WARNING`
jobs, preventing permanently degraded stripes from accumulating; (3) the 20% ASN cap is
re-enforced at replacement selection, maintaining the co-requisite for the LossRate < 10⁻¹⁵
durability guarantee. These properties derive from [ADR-004](../../decisions/ADR-004-repair-protocol.md),
[ADR-014](../../decisions/ADR-014-adversarial-defences.md), and [ADR-003](../../decisions/ADR-003-erasure-coding.md).

---

## Participants

| Participant label | Role in this flow | Described in |
|---|---|---|
| `Microservice` | Departure detector; repair scheduler; replacement assignment service | [architecture.md §18](../architecture.md#18-coordination-microservice) |
| `PostgreSQL` | Stores `providers`, `chunk_assignments`, `repair_jobs`, `escrow_events` | [architecture.md §6](../architecture.md#6-component-overview) |
| `ProviderDaemon[×16]` | Surviving shard holders; contacted during repair download phase | [architecture.md §16](../architecture.md#16-provider-storage-engine) |
| `ProviderDaemon[replacement]` | Newly selected provider that receives the re-encoded shard | [architecture.md §16](../architecture.md#16-provider-storage-engine) |

---

## Happy Path 1 — Silent Departure Detected → Repair Triggered

The departure detector runs continuously. When a provider's `last_heartbeat_ts` exceeds
the 72-hour threshold, it is declared silently departed. All its chunk assignments are
removed from the routing table (to stop further challenge issuance), escrow is frozen and
seized into the repair reserve fund, and repair jobs are enqueued for every affected chunk.
The repair scheduler then executes the actual shard reconstruction.

```mermaid
sequenceDiagram
    %% Repair Flow — Silent Departure → Repair
    %% ADR-004 (lazy repair, priority ordering), ADR-007 (exit states),
    %% ADR-014 (20% ASN cap), ADR-024 (escrow seizure)

    participant MS  as Microservice
    participant PG  as PostgreSQL
    participant PSH as ProviderDaemon[×16]
    participant PRP as ProviderDaemon[replacement]

    Note over MS: Departure detector polls every 60 s:<br/>SELECT provider_id FROM providers<br/>WHERE status = 'ACTIVE'<br/>AND last_heartbeat_ts < NOW() - INTERVAL '72 hours'<br/>(ADR-006, ADR-007)

    MS->>PG: UPDATE providers SET status='DEPARTED',<br/>departed_at=NOW()<br/>WHERE provider_id = ?
    Note over PG: Row is NEVER physically deleted (Invariant 3).<br/>Physical deletion is prohibited — payment history,<br/>audit history, and chunk assignments all reference<br/>this row (ADR-007, ADR-013).
    PG-->>MS: OK

    MS->>PG: DELETE FROM chunk_assignments<br/>WHERE provider_id = ? AND status = 'ACTIVE'
    Note over PG: Hard-remove assignments to stop challenge<br/>issuance to the departed provider.<br/>The departed provider's vLog data on<br/>disk is incidental — not tracked, not paid.<br/>(ADR-007 returning provider section)
    PG-->>MS: N rows deleted (one per assigned chunk)

    Note over MS: Freeze escrow and seize holdings (ADR-024 §5):<br/>SET providers.frozen = true<br/>INSERT INTO escrow_events<br/>{ event_type='SEIZURE',<br/>  amount_paise = rolling_30d_balance,<br/>  idempotency_key = SHA-256(provider_id || 'departure') }<br/>All amounts are integer paise — no float (NFR-046).

    MS->>PG: INSERT escrow_events (SEIZURE)
    PG-->>MS: OK

    Note over MS: Enqueue one repair job per affected chunk.<br/>trigger_type = 'SILENT_DEPARTURE'<br/>priority = 'PERMANENT_DEPARTURE'<br/>(drains before PRE_WARNING jobs, ADR-004)

    MS->>PG: INSERT INTO repair_jobs (×N chunks)<br/>{ trigger_type='SILENT_DEPARTURE',<br/>  priority='PERMANENT_DEPARTURE',<br/>  status='QUEUED',<br/>  available_shard_count = current_count }
    PG-->>MS: OK

    Note over MS: ── REPAIR SCHEDULER DEQUEUES ──<br/>Dequeue order: PERMANENT_DEPARTURE FIFO,<br/>then PRE_WARNING FIFO.<br/>A PRE_WARNING job already queued waits<br/>until all PERMANENT_DEPARTURE jobs ahead of<br/>it are processed (Paper 39, ADR-004).

    MS->>PG: UPDATE repair_jobs SET status='IN_PROGRESS',<br/>started_at=NOW()<br/>WHERE job_id = ? AND priority='PERMANENT_DEPARTURE'
    PG-->>MS: OK

    %% ── Download surviving shards ──────────────────────────────────────────
    Note over MS: Contact k=16 surviving fragment holders<br/>via chunk_assignments table:<br/>SELECT provider_id, chunk_id<br/>FROM chunk_assignments<br/>WHERE segment_id = ? AND status='ACTIVE'<br/>LIMIT 16

    MS->>PSH: libp2p QUIC dial + CHUNK_REQUEST(chunk_id[0..15])
    Note over PSH: Each surviving provider reads from vLog,<br/>verifies content_hash, streams 256 KB shard.<br/>(ADR-023 audit lookup path — same code path<br/>as audit response, no special case needed.)
    PSH-->>MS: 16 × 256 KB fragment streams

    %% ── Re-encode missing shards ───────────────────────────────────────────
    Note over MS: RS decode 16 fragments → AONT package.<br/>Re-encode all N missing shards from package.<br/>(Same RS(16,56) library used in encoding pipeline,<br/>ADR-003. No AONT key needed — working at<br/>the RS codeword level, not plaintext level.)

    %% ── Select replacement providers ───────────────────────────────────────
    Note over MS: Power of Two Choices selection for<br/>each missing shard (ADR-005).<br/>20% ASN cap RE-ENFORCED at replacement<br/>selection — same constraint as original<br/>assignment (ADR-014, NFR-002).<br/><br/>Check: ceil(0.20 × 56) = 11 shards max<br/>per ASN. Reject any candidate that would<br/>push their ASN over 11 shards for this segment.

    MS->>PG: INSERT INTO chunk_assignments<br/>{ chunk_id, segment_id,<br/>  shard_index, provider_id=replacement,<br/>  status='ACTIVE' }<br/>(one row per re-encoded shard)
    PG-->>MS: OK

    %% ── Upload to replacement providers ────────────────────────────────────
    MS->>PRP: libp2p QUIC dial + CHUNK_UPLOAD(chunk_id, shard_data 256 KB)
    Note over PRP: Provider writes to vLog:<br/>  entry = chunk_id || chunk_size || chunk_data || content_hash<br/>  content_hash = SHA-256(chunk_data)<br/>  fsync() before RocksDB insert (ADR-023).<br/>  Single writer goroutine serialises append.
    PRP-->>MS: Signed upload receipt (Ed25519)

    %% ── Mark repair complete ───────────────────────────────────────────────
    MS->>PG: UPDATE repair_jobs SET status='COMPLETED',<br/>completed_at=NOW()<br/>WHERE job_id = ?
    PG-->>MS: OK

    Note over MS: Fragment count for this segment<br/>restored to 56 (full redundancy).<br/>Steady-state BWavg ≈ 39 Kbps/peer at<br/>MTTF=300d (Giroire Formula 1, Paper 10).<br/>Full repair of one provider at N=1,000<br/>completes in ~8h — within 12h safety window<br/>(ADR-004, capacity.md §3.2).
```

### Cross-reference: diagram steps to ADRs and requirements

| Step # | Description | ADR / Requirement |
|---|---|---|
| 1 | Departure detector threshold: `last_heartbeat_ts < NOW() - INTERVAL '72 hours'` | [ADR-006](../../decisions/ADR-006-polling-interval.md), [ADR-007](../../decisions/ADR-007-provider-exit-states.md) |
| 2 | `providers.status = 'DEPARTED'`; physical deletion prohibited — Invariant 3 | [ADR-007](../../decisions/ADR-007-provider-exit-states.md), [ADR-013](../../decisions/ADR-013-consistency-model.md) |
| 3 | `chunk_assignments` hard-deleted to stop challenge issuance to departed provider | [ADR-007](../../decisions/ADR-007-provider-exit-states.md) |
| 4 | `escrow_events` SEIZURE — integer paise, idempotency key, no float | [ADR-024](../../decisions/ADR-024-economic-mechanism.md), [ADR-016](../../decisions/ADR-016-payment-db-schema.md) |
| 5 | `repair_jobs` enqueued with `priority = 'PERMANENT_DEPARTURE'` | [ADR-004](../../decisions/ADR-004-repair-protocol.md), [FR-035](../requirements.md#67-provider--exit-and-departure) |
| 6 | Priority queue drains `PERMANENT_DEPARTURE` before `PRE_WARNING` | [ADR-004](../../decisions/ADR-004-repair-protocol.md), Paper 39 |
| 7 | 16 surviving fragment holders contacted via `chunk_assignments` table; DHT not required | [ADR-001](../../decisions/ADR-001-coordination-architecture.md), [ADR-023](../../decisions/ADR-023-provider-storage-engine.md) |
| 8 | RS decode 16 → AONT package; re-encode missing shards; no AONT key needed | [ADR-003](../../decisions/ADR-003-erasure-coding.md), [ADR-022](../../decisions/ADR-022-encryption-erasure-order.md) |
| 9 | 20% ASN cap re-enforced at replacement selection — co-requisite for LossRate < 10⁻¹⁵ | [ADR-014](../../decisions/ADR-014-adversarial-defences.md), [FR-045](../requirements.md#69-repair-system) |
| 10 | Provider vLog write: `fsync()` before RocksDB insert; single writer goroutine | [ADR-023](../../decisions/ADR-023-provider-storage-engine.md), [NFR-023](../requirements.md#75-reliability-and-correctness) |

### What this diagram does not show

- How the provider's DHT records are removed — driven by the availability service on the next 12-hour republication cycle ([ADR-001](../../decisions/ADR-001-coordination-architecture.md)).
- The escrow seizure's interaction with Razorpay Route reversals — covered in [04-payment-release.md](./04-payment-release.md).
- How the repair scheduler handles correlated burst failures (multiple simultaneous departures from the same ASN) — the 20% ASN cap bounds this to ≤11 simultaneous shard losses; the repair queue simply accumulates multiple PERMANENT_DEPARTURE jobs and processes them FIFO.
- The Giroire Qpeek formula details — documented in [capacity.md §3.2](../capacity.md#32-burst-repair-bandwidth-qpeek); note that at N < 500 providers, repair window exceeds 12 hours and manual oversight of `repair_queue_depth` is required.

---

## Failure Path 1 — Emergency Floor (Fragment Count Reaches s=16)

When multiple providers depart in a short window, the available fragment count can drop
to s=16 — the reconstruction floor — before a repair job has completed. This triggers
an immediate emergency response that bypasses all normal queue ordering.

```mermaid
sequenceDiagram
    %% Repair Flow — Emergency Floor: available_shard_count reaches s=16
    %% ADR-004 (emergency floor), FR-044

    participant MS  as Microservice
    participant PG  as PostgreSQL

    Note over MS: Materialised view mv_segment_shard_counts<br/>refreshed after each chunk_assignment change.<br/>Repair monitor detects:<br/>  available_shard_count = 16 for segment S<br/>  (reached the reconstruction floor s=16).<br/>  ADR-004: "emergency floor — trigger immediately,<br/>  bypassing normal threshold and queue ordering."

    MS->>PG: INSERT INTO repair_jobs<br/>{ trigger_type='EMERGENCY_FLOOR',<br/>  priority='PERMANENT_DEPARTURE',<br/>  available_shard_count=16,<br/>  status='QUEUED' }
    Note over PG: EMERGENCY_FLOOR jobs are enqueued<br/>as PERMANENT_DEPARTURE priority —<br/>they drain at the front of the queue,<br/>ahead of any PRE_WARNING jobs and ahead<br/>of any regular PERMANENT_DEPARTURE<br/>jobs with a later created_at. (FR-044)
    PG-->>MS: OK

    Note over MS: Alert fires immediately:<br/>  repair_queue_depth metric observed;<br/>  operator notified via PagerDuty.<br/>  (NFR-027, architecture.md §24)<br/><br/>At N < 500 providers, repair window<br/>may exceed 12 hours — operator must<br/>manually confirm no second failure<br/>occurs during the repair window.<br/>(capacity.md §3.2)

    Note over MS: Repair scheduler immediately dequeues<br/>the EMERGENCY_FLOOR job (bypassing<br/>any PRE_WARNING jobs that arrived<br/>earlier). Execution follows the same<br/>download → re-encode → upload path<br/>as the Happy Path above.
```

### Cross-reference

| Step # | Description | ADR / Requirement |
|---|---|---|
| 1 | Fragment count monitored via `mv_segment_shard_counts` materialised view | [data-model.md §7](../data-model.md#7-materialised-views) |
| 2 | `EMERGENCY_FLOOR` jobs enqueued as `PERMANENT_DEPARTURE` priority — front of queue | [ADR-004](../../decisions/ADR-004-repair-protocol.md), [FR-044](../requirements.md#69-repair-system) |
| 3 | Alert fires on `repair_queue_depth > 1,000` OR on any `EMERGENCY_FLOOR` creation | [NFR-027](../requirements.md#76-observability-and-operability) |

---

## Failure Path 2 — Pre-Warning: Fragment Count Drops to s+r0=24

The lazy repair threshold fires before a provider is declared formally departed — the
redundancy has eroded due to multiple prior silent departures, each individually within
the lazy window. This generates a `PRE_WARNING` job that waits behind any active
`PERMANENT_DEPARTURE` jobs.

```mermaid
sequenceDiagram
    %% Repair Flow — Lazy threshold: available_shard_count drops to s+r0=24
    %% ADR-004 (lazy repair threshold), ADR-003 (r0=8)

    participant MS  as Microservice
    participant PG  as PostgreSQL

    Note over MS: mv_segment_shard_counts refresh detects:<br/>  available_shard_count = 24<br/>  = s + r0 = 16 + 8  (lazy repair threshold)<br/>  No single departure caused this drop —<br/>  cumulative drift from several<br/>  temporary absences (ADR-004).

    MS->>PG: INSERT INTO repair_jobs<br/>{ trigger_type='THRESHOLD_WARNING',<br/>  priority='PRE_WARNING',<br/>  available_shard_count=24,<br/>  provider_id=NULL }
    Note over PG: provider_id is NULL because no single<br/>departure triggered this job.<br/>(data-model.md §8.15 nullable justification)
    PG-->>MS: OK

    Note over MS: PRE_WARNING job sits behind all<br/>PERMANENT_DEPARTURE jobs in the queue.<br/>Scheduler will only dequeue it once<br/>all higher-priority jobs are COMPLETED.<br/>(ADR-004, Paper 39)<br/><br/>If a new PERMANENT_DEPARTURE job<br/>arrives for a different chunk while<br/>this PRE_WARNING is waiting:<br/>  → The new PERMANENT_DEPARTURE job<br/>    is processed first.<br/>  → This PRE_WARNING continues waiting.<br/>This is correct: silently departed<br/>providers' chunks are more urgently<br/>degraded than a pre-warning threshold.

    Note over MS: If available_shard_count drops further<br/>to 16 (EMERGENCY_FLOOR) before this<br/>PRE_WARNING is processed:<br/>  → An EMERGENCY_FLOOR job is enqueued<br/>    (PERMANENT_DEPARTURE priority).<br/>  → It supersedes this PRE_WARNING job<br/>    for the same segment.<br/>  → This PRE_WARNING job is marked<br/>    COMPLETED after the EMERGENCY job<br/>    restores full shard count.
```

---

## Failure Path 3 — ASN Cap Unsatisfiable During Replacement Selection

At small network sizes (near the 56-provider floor), the assignment service may not
be able to find a replacement provider for a shard without violating the 20% ASN cap.
The repair job is retried rather than proceeding with a cap violation.

```mermaid
sequenceDiagram
    %% Repair Flow — ASN cap unsatisfiable during replacement
    %% ADR-014, ADR-029, mvp.md risk register R-01

    participant MS  as Microservice
    participant PG  as PostgreSQL

    Note over MS: Repair job dequeued. RS decode complete.<br/>Replacement selection for shard_index=7:<br/>  Available ACTIVE providers: 56<br/>  Providers already holding shards<br/>  from ASN 'AS24560' for this segment: 11<br/>  (= 20% cap of 56 shards)<br/>  All remaining providers without existing<br/>  assignments belong to AS24560.<br/>  → CAP VIOLATION: cannot place shard.

    MS->>PG: UPDATE repair_jobs SET status='FAILED',<br/>  completed_at=NOW(),<br/>  failure_reason='ASN_CAP_UNSATISFIABLE'<br/>WHERE job_id = ?
    PG-->>MS: OK

    Note over MS: Log warning: "Repair job for chunk_id X<br/>  failed: no eligible provider outside<br/>  ASN cap. Will retry when new providers<br/>  join from a different ASN."<br/><br/>The repair job is NOT retried immediately<br/>with a cap violation — doing so would<br/>silently invalidate the LossRate guarantee<br/>(ADR-003 + ADR-014 co-requisite).<br/><br/>The fragment count for this segment<br/>remains below full redundancy.<br/>A new repair attempt is enqueued<br/>automatically when a new provider<br/>from a different ASN joins and<br/>clears vetting (ACTIVE status).<br/>(mvp.md risk register R-01)

    Note over MS: If the fragment count then reaches<br/>EMERGENCY_FLOOR (s=16) before a new<br/>provider joins: the EMERGENCY_FLOOR<br/>job overrides and the assignment<br/>service uses best-effort (best available<br/>provider even with ASN overlap) with<br/>a logged warning. Data preservation<br/>takes priority over strict cap enforcement<br/>at the reconstruction floor.
```

---

## Happy Path 2 — Announced Departure: Pre-emptive Repair

When a provider explicitly calls `POST /api/v1/provider/depart`, repair is triggered
immediately — before any 72-hour window elapses. The provider receives proportional
escrow release rather than seizure.

```mermaid
sequenceDiagram
    %% Repair Flow — Announced departure: pre-emptive repair
    %% ADR-007, ADR-024 §4, FR-034

    participant PD  as ProviderDaemon
    participant MS  as Microservice
    participant PG  as PostgreSQL

    PD->>MS: POST /api/v1/provider/depart<br/>{ depart_at (optional future timestamp),<br/>  provider_sig (Ed25519) }

    Note over MS: Verify provider_sig against<br/>registered ed25519_public_key.<br/>Compute proportional release:<br/>  fraction_completed = days_elapsed_in_30d_window / 30<br/>  release_paise = escrow_window_balance × fraction_completed<br/>  (ADR-024 §4 announced departure row)<br/>All arithmetic in integer paise. (NFR-046)

    MS->>PG: UPDATE providers<br/>SET status='DEPARTED', departed_at=NOW()<br/>WHERE provider_id = ?
    PG-->>MS: OK

    MS->>PG: DELETE FROM chunk_assignments<br/>WHERE provider_id = ? AND status='ACTIVE'
    PG-->>MS: OK

    MS->>PG: INSERT INTO escrow_events<br/>{ event_type='RELEASE',<br/>  amount_paise=proportional_release,<br/>  idempotency_key=SHA-256(provider_id||'departure_release') }
    Note over PG: RELEASE event (not SEIZURE) for<br/>announced departure — provider earns<br/>their proportional fraction.<br/>Remaining escrow is retained in<br/>the repair reserve fund. (ADR-024)
    PG-->>MS: OK

    Note over MS: Enqueue repair jobs at<br/>PERMANENT_DEPARTURE priority.<br/>Announced departure uses same<br/>priority as silent departure —<br/>both involve permanent shard loss.<br/>(ADR-004)

    MS->>PG: INSERT INTO repair_jobs (×N chunks)<br/>{ trigger_type='ANNOUNCED_DEPARTURE',<br/>  priority='PERMANENT_DEPARTURE',<br/>  status='QUEUED' }
    PG-->>MS: OK

    MS-->>PD: 200 OK<br/>{ repair_jobs_queued: N,<br/>  escrow_release_paise: proportional_release }

    Note over PD: Provider daemon receives confirmation.<br/>Can safely shut down — repair is in<br/>progress. Attempting to reconnect later<br/>will return HTTP 403 (FR-036).<br/>Re-joining requires a new registration<br/>with a new phone number.
```

### Cross-reference

| Step # | Description | ADR / Requirement |
|---|---|---|
| 1 | Provider-signed departure request; microservice verifies Ed25519 signature | [ADR-021](../../decisions/ADR-021-p2p-transfer-protocol.md), [FR-034](../requirements.md#67-provider--exit-and-departure) |
| 2 | Proportional RELEASE computed as fraction of 30-day window completed | [ADR-024](../../decisions/ADR-024-economic-mechanism.md) §4 |
| 3 | `RELEASE` event (not `SEIZURE`) — announced departure is not penalised | [ADR-024](../../decisions/ADR-024-economic-mechanism.md), [FR-034](../requirements.md#67-provider--exit-and-departure) |
| 4 | Repair enqueued at `PERMANENT_DEPARTURE` priority — same as silent departure | [ADR-004](../../decisions/ADR-004-repair-protocol.md) |

---

## Invariants Demonstrated

| Invariant | Where it appears in this flow | Source |
|---|---|---|
| Physical provider row deletion is prohibited | Happy Path: `UPDATE status='DEPARTED'` — no DELETE | [ADR-007](../../decisions/ADR-007-provider-exit-states.md), Invariant 3 in [data-model.md](../data-model.md#3-design-invariants) |
| 20% ASN cap re-enforced at replacement selection | Happy Path step 9; Failure Path 3 shows what happens when the cap cannot be satisfied | [ADR-014](../../decisions/ADR-014-adversarial-defences.md), Invariant 4 in [trade-offs.md](../trade-offs.md) |
| All escrow amounts are integer paise | Both seizure and proportional release annotated "integer paise — no float" | [ADR-016](../../decisions/ADR-016-payment-db-schema.md), Invariant 4 in [data-model.md](../data-model.md#3-design-invariants) |
| `PERMANENT_DEPARTURE` jobs drain before `PRE_WARNING` | Failure Path 2 explicitly shows a PRE_WARNING job waiting behind PERMANENT_DEPARTURE | [ADR-004](../../decisions/ADR-004-repair-protocol.md) |
| Emergency floor (s=16) triggers immediate repair regardless of threshold | Failure Path 1 shows EMERGENCY_FLOOR job enqueued ahead of queue | [ADR-004](../../decisions/ADR-004-repair-protocol.md), [FR-044](../requirements.md#69-repair-system) |

---

## Related Diagrams

- **[02-audit-cycle.md](./02-audit-cycle.md)** — the sustained FAIL/TIMEOUT results from the audit cycle feed the departure detector that triggers this flow; content hash failures trigger accelerated re-audit which may subsequently trigger `THRESHOLD_WARNING` jobs.
- **[04-payment-release.md](./04-payment-release.md)** — the escrow seizure and proportional release events created in this flow are processed by the payment system; Razorpay Route reversals are shown in the payment diagram.
- **[05-provider-lifecycle.md](./05-provider-lifecycle.md)** — the `ACTIVE → DEPARTED` state transition executed in this flow is shown in its full state machine context there.