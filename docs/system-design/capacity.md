# Vyomanaut V2 — Capacity Estimation

**Status:** Reference document — recompute when any value in the dimensions table changes.  
**Audience:** Platform and infrastructure engineers sizing hardware, setting operational
alert thresholds, and evaluating whether a proposed scale target is safe.  
**Method:** Every ceiling is computed from values already present in the ADRs and research
log. No new assumptions are introduced. Where a value is uncertain or assumed, it is
flagged explicitly. Every quoted source is linked.

---

## Executive Summary

The current architecture supports growth to approximately [100,000 providers × 10,000 chunks before re-architecture is required](#8-hard-ceilings--what-breaks-first), at which point the Postgres audit INSERT rate (~10,000 rows/sec) becomes the binding ceiling and probabilistic audit sampling or write sharding must be introduced. The [first constraint to appear in the growth path is relay slot exhaustion](#52-relay-infrastructure-scaling), occurring at roughly 570–850 providers depending on Indian CGNAT prevalence, and can be resolved incrementally by adding one relay node (~₹1,500/month on Indian cloud) per ~280 additional providers. The [minimum viable floor is 56 active vetted providers across 5 ASNs in 3 metro regions](#7-minimum-viable-floor), with 3 relay nodes and 56 Razorpay Linked Accounts past their 24-hour cooling period — below this, the assignment service returns HTTP 503 for all uploads. At the worst-case MTTF of 180 days, [a provider must not be loaded beyond approximately 70 GB of chunk data](#31-steady-state-repair-bandwidth-bwavg) to stay within the 100 Kbps background bandwidth budget, and this ceiling must be enforced at onboarding rather than discovered operationally. The [relay headroom formula is: required slots = N × relay\_fraction × 1.5, available slots = relay\_nodes × 128](#52-relay-infrastructure-scaling); add a relay node when required slots exceed 80% of available.

---

## How to read this document

Each calculation section identifies a **binding constraint** — the single resource that
runs out first — and the **scale at which it binds**. The document closes with a two-row
summary: the minimum viable network floor, and the ceiling of the current architecture before
something must change.

**Value status labels used in the dimensions table:**

| Label | Meaning |
|---|---|
| `DESIGN` | A deliberate parameter set in an ADR. Changing it requires a new ADR. |
| `DERIVED` | Computed from other values by formula. Changes when its inputs change. |
| `MEASURED` | Taken from empirical data in a cited paper or benchmark. |
| `ASSUMED` | An estimate with no cited source. Flag for empirical validation. Treat conservatively. |

---

## Table of Contents

1. [Dimensions Table](#1-dimensions-table)
2. [Storage Capacity](#2-storage-capacity)
3. [Bandwidth Budget](#3-bandwidth-budget)
4. [Audit System Throughput](#4-audit-system-throughput)
5. [Network and DHT Scaling](#5-network-and-dht-scaling)
6. [Payment System Throughput](#6-payment-system-throughput)
7. [Minimum Viable Floor](#7-minimum-viable-floor)
8. [Hard Ceilings — What Breaks First](#8-hard-ceilings--what-breaks-first)
9. [Scale Milestones](#9-scale-milestones)
10. [Summary: Floor and Ceiling](#10-summary-floor-and-ceiling)

---

## 1. Dimensions Table

All numeric constants used in the calculations below. If you are about to quote a number
in a capacity argument and it is not in this table, either it belongs here or your argument
is not grounded.

### 1.1 Erasure coding and storage geometry

| Name | Symbol | Value | Unit | Status | Source |
|---|---|---|---|---|---|
| Reconstruction threshold | s | 16 | shards | DESIGN | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Redundancy fragments | r | 40 | shards | DESIGN | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Total shards per file segment | n | 56 | shards | DERIVED (s+r) | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Lazy repair trigger threshold | s + r0 | 24 | available shards | DESIGN | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Lazy repair buffer above floor | r0 | 8 | shards | DESIGN | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Fragment (chunk) size | lf | 256 | KB | DESIGN | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Block size (one full segment) | lb | 4 | MB | DERIVED (s × lf = 16 × 256 KB) | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Maximum segment size on wire | — | 14 | MB | DERIVED (n × lf = 56 × 256 KB) | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Storage efficiency (data / raw) | s/n | 28.57% | — | DERIVED (16/56) | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Target annual file loss rate | LossRate | < 10⁻¹⁵ | per file per year | DESIGN | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Computed LossRate at V2 params | — | ~10⁻²⁵ | per file per year | DERIVED (Giroire Formula 3) | [Paper 10](../research/paper-10-giroire-lazy.md) |

### 1.2 Provider MTTF and bandwidth

| Name | Symbol | Value | Unit | Status | Source |
|---|---|---|---|---|---|
| Minimum acceptable provider MTTF | — | 180 | days | DESIGN | [ADR-010](../decisions/ADR-010-desktop-only-v2.md) |
| Target provider MTTF (planning) | — | 300 | days | DESIGN | [ADR-003](../decisions/ADR-003-erasure-coding.md) |
| Maximum observed provider MTTF (NAS) | — | 380 | days | DESIGN | [ADR-010](../decisions/ADR-010-desktop-only-v2.md) |
| Background bandwidth budget per provider | — | 100 | Kbps | DESIGN | [ADR-009](../decisions/ADR-009-background-execution.md), [Paper 06](../research/paper-06-blake-rodrigues.md) |
| Background CPU budget per provider | — | ≤ 5 | % | DESIGN | [ADR-009](../decisions/ADR-009-background-execution.md) |
| Steady-state repair BWavg at MTTF=300d | BWavg | ~39 | Kbps/peer | DERIVED (Giroire Formula 1, N=1,000, 50 GB/peer) | [Paper 10](../research/paper-10-giroire-lazy.md) |
| Steady-state repair BWavg at MTTF=180d | BWavg | ~65 | Kbps/peer | DERIVED (Giroire Formula 1) | [Paper 10](../research/paper-10-giroire-lazy.md) |
| Steady-state repair BWavg at MTTF=90d (mobile floor) | BWavg | ~130 | Kbps/peer | DERIVED (Giroire Formula 1) | [Paper 10](../research/paper-10-giroire-lazy.md) |
| Peak network transfer per failure event at N=1,000 | Qpeek | ~793 | GB | DERIVED (Giroire Formula 2, 50 GB/peer) | [Paper 10](../research/paper-10-giroire-lazy.md) |
| Time to complete repair at N=1,000 | θ | ~8 | hours | DERIVED (Qpeek / aggregate bandwidth) | [Paper 10](../research/paper-10-giroire-lazy.md) |
| Repair safety window | — | 12 | hours | DESIGN (must complete before second failure likely) | [ADR-004](../decisions/ADR-004-repair-protocol.md) |
| Real repair bandwidth std deviation | — | 22× | × independent model σ | MEASURED (simulation, 5,000 peers) | [Paper 36](../research/paper-36-dalle-failure-correlation.md) |

### 1.3 Audit system

| Name | Symbol | Value | Unit | Status | Source |
|---|---|---|---|---|---|
| Audit frequency per chunk | — | 1 | per 24 h | DESIGN | [ADR-006](../decisions/ADR-006-polling-interval.md) |
| Challenge timeout model | RTO | AVG + 4 × VAR | ms | DESIGN | [ADR-006](../decisions/ADR-006-polling-interval.md), [Paper 28](../research/paper-28-rhea-dht-churn.md) |
| New-provider initial RTO | — | 2,000 | ms | DESIGN | [ADR-006](../decisions/ADR-006-polling-interval.md) |
| Audit deadline (5 Mbps provider) | — | ~614 | ms | DERIVED ((256 KB / 500 KB/s) × 1.5) | [ADR-014](../decisions/ADR-014-adversarial-defences.md) |
| Audit receipt row size (raw fields) | — | ~297 | bytes | DERIVED (sum of schema fields) | [ADR-017](../decisions/ADR-017-audit-receipt-schema.md) |
| Audit receipt row size (Postgres on-disk est.) | — | ~450 | bytes | ASSUMED (raw + Postgres page overhead ~50%) | — |
| Postgres single-instance INSERT ceiling | — | 5,000–10,000 | rows/sec | ASSUMED | See note below |

*Note on Postgres INSERT ceiling:* This value is taken from general Postgres benchmark
literature, not from a measurement on the `audit_receipts` schema. The `audit_receipts`
table contains two 64-byte Ed25519 signatures (`provider_sig` and `service_sig`) plus a
32-byte nonce and multiple timestamp fields, making rows significantly wider than typical
benchmark workloads. The actual ceiling on this schema is expected to be lower than
5,000–10,000 rows/sec. **This value must be benchmarked on the real schema before V2 GA.**
Run a sustained INSERT-only load test against a production-equivalent Postgres instance
with row security policy enabled and measure the point at which INSERT latency exceeds 50 ms
at p99. Update this table with the measured value before closing the V2 launch checklist.
This is a dependency for NFR-021.

### 1.4 Provider storage engine

| Name | Symbol | Value | Unit | Status | Source |
|---|---|---|---|---|---|
| vLog entry size | — | 262,212 | bytes | DERIVED (32+4+262,144+32) | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| RocksDB index entry size | — | 44 | bytes | DERIVED (32-byte chunk_id + 8-byte offset + 4-byte size) | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |
| Write amplification at lf=256 KB | — | ~1.0 | × | MEASURED (WiscKey Figure 10) | [Paper 27](../research/paper-27-wisckey.md) |
| Single-thread audit lookup on SSD | — | ~1 | ms | MEASURED (Samsung 840 EVO, WiscKey Figure 3) | [Paper 27](../research/paper-27-wisckey.md) |
| Single-thread audit lookup on 7200 RPM HDD | — | ~12–15 | ms | DERIVED (10 ms seek + 256 KB transfer) | [Paper 27](../research/paper-27-wisckey.md) |
| Bloom filter false-positive rate | — | ~1 | % | DESIGN (10 bits per key) | [ADR-023](../decisions/ADR-023-provider-storage-engine.md) |

### 1.5 Network and connectivity

| Name | Symbol | Value | Unit | Status | Source |
|---|---|---|---|---|---|
| Hole-punch success rate (DCUtR) | — | 70 | % | MEASURED (4.4M attempts, 85k networks) | [Paper 30](../research/paper-30-trautwein-dcutr-nat.md) |
| Relay-dependent provider fraction (global) | — | ~30 | % | MEASURED (Paper 30 global) | [Paper 30](../research/paper-30-trautwein-dcutr-nat.md) |
| Relay-dependent fraction (Indian CGNAT est.) | — | 30–45 | % | ASSUMED (global floor; CGNAT prevalent in India) | [Paper 30](../research/paper-30-trautwein-dcutr-nat.md), Q20-1 |
| Relay slots per relay node | — | 128 | concurrent | DESIGN | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| Relay nodes at launch | — | 3 | nodes | DESIGN | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| Total relay slots at launch | — | 384 | concurrent | DERIVED (3 × 128) | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) |
| Relay overhead (Indian cloud-hosted) | — | < 50 | ms RTT | DESIGN | [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md), [Paper 30](../research/paper-30-trautwein-dcutr-nat.md) |
| Median DHT lookup latency (IPFS post-v0.5) | — | 622 | ms | MEASURED | [Paper 20](../research/paper-20-trautwein-ipfs.md) |
| DHT k-bucket size | k | 16 | nodes | DESIGN | [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| DHT parallel lookups | α | 3 | concurrent | DESIGN | [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| DHT record republication interval | — | 12 | hours | DESIGN | [ADR-006](../decisions/ADR-006-polling-interval.md) |
| DHT record expiry | — | 24 | hours | DESIGN | [ADR-006](../decisions/ADR-006-polling-interval.md) |
| Gossip rate (microservice replicas) | — | 1 | peer/sec | DESIGN | [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md), [Paper 12](../research/paper-12-dynamo_1.md) |

### 1.6 Timing and polling

| Name | Symbol | Value | Unit | Status | Source |
|---|---|---|---|---|---|
| Audit polling interval | t | 24 | hours | DESIGN | [ADR-006](../decisions/ADR-006-polling-interval.md) |
| Silent departure threshold | — | 72 | hours | DESIGN | [ADR-006](../decisions/ADR-006-polling-interval.md), [ADR-007](../decisions/ADR-007-provider-exit-states.md) |
| Heartbeat interval | — | 4 | hours | DESIGN | [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |
| Heartbeat payload size | — | ~200 | bytes | DESIGN | [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |
| Escrow rolling hold window | — | 30 | days | DESIGN | [ADR-024](../decisions/ADR-024-economic-mechanism.md) |
| Vetting period hold window | — | 60 | days | DESIGN | [ADR-024](../decisions/ADR-024-economic-mechanism.md) |
| Vetting period duration | — | 4–6 | months | DESIGN | [ADR-005](../decisions/ADR-005-peer-selection.md) |
| Vetting threshold (audit pass count) | — | 80 | consecutive passes | DESIGN (Jeffrey's prior, Paper 05) | [ADR-005](../decisions/ADR-005-peer-selection.md) |

### 1.7 Minimum viable network

| Name | Value | Unit | Status | Source |
|---|---|---|---|---|
| Minimum active vetted providers | 56 | providers | DESIGN | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) |
| Minimum distinct ASNs | 5 | ASNs | DESIGN | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) |
| Minimum distinct Indian metro regions | 3 | regions | DESIGN | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) |
| Minimum relay nodes | 3 | nodes | DESIGN | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) |
| Minimum Razorpay Linked Accounts (cooling complete) | 56 | accounts | DESIGN | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) |

### 1.8 Indian ISP context (planning assumptions)

| Name | Value | Unit | Status | Source |
|---|---|---|---|---|
| Target Indian residential upload speed | 100 | Mbps symmetrical | ASSUMED (₹600/month tier) | [Paper 06](../research/paper-06-blake-rodrigues.md) |
| Minimum expected provider upload for audit | 5 | Mbps | ASSUMED (minimum viable) | [ADR-014](../decisions/ADR-014-adversarial-defences.md) |
| Expected provider storage allocation | 50–500 | GB per device | ASSUMED (idle NAS / desktop disk) | [ADR-010](../decisions/ADR-010-desktop-only-v2.md) |
| DHCP lease rotation period | 24 | hours | ASSUMED (BSNL / Airtel residential) | [ADR-028](../decisions/ADR-028-provider-heartbeat.md) |

---

## 2. Storage Capacity

### 2.1 Raw storage and usable data at network scale

Each shard stored by a provider is 262,212 bytes of raw vLog space. Because n=56 shards are
required per file segment and only s=16 of those are data (the rest are parity), the usable
data fraction is:

```
Storage efficiency = s / n = 16 / 56 = 28.57%
```

For every 3.5 GB a provider contributes to the network, approximately 1 GB is stored in
user-accessible file data. The remainder is parity — the cost of the 40-fragment fault
tolerance. This is not waste; it is what allows the simultaneous departure of any 40 of 56
shard-holders to be survivable.

**Network-wide usable storage at representative scales:**

| Active providers | Storage per provider | Raw network storage | Usable file data |
|---|---|---|---|
| 56 (floor) | 50 GB | 2.8 TB | 800 GB |
| 56 (floor) | 500 GB | 28 TB | 8 TB |
| 500 | 50 GB | 25 TB | 7.1 TB |
| 500 | 500 GB | 250 TB | 71 TB |
| 1,000 | 50 GB | 50 TB | 14.3 TB |
| 1,000 | 500 GB | 500 TB | 143 TB |
| 10,000 | 50 GB | 500 TB | 143 TB |
| 10,000 | 500 GB | 5 PB | 1.4 PB |

*Note:* These are upper bounds assuming all providers are fully loaded. At any given time,
providers will be partially loaded during the vetting ramp. Actual usable storage is lower
until the provider pool matures past the vetting period.

### 2.2 Per-provider storage engine sizing

The vLog and RocksDB index scale linearly with chunk count, which scales linearly with
storage allocation.

**At 50 GB declared storage per provider:**
```
Chunks stored     = 50 GB / 262,212 bytes ≈ 200,700 chunks
RocksDB index     = 200,700 × 44 bytes    ≈ 8.8 MB  (trivially fits in RAM)
vLog on disk      = 200,700 × 262,212     ≈ 52.7 GB (slightly above declared due to metadata overhead)
```

**At 500 GB declared storage per provider:**
```
Chunks stored     = 500 GB / 262,212 bytes ≈ 2,007,000 chunks
RocksDB index     = 2,007,000 × 44 bytes   ≈ 88.3 MB (fits comfortably in RocksDB block cache)
vLog on disk      = 2,007,000 × 262,212    ≈ 527 GB
```

The RocksDB index at any realistic provider storage allocation (up to several TB) is under
1 GB and stays fully cached in memory, meaning audit lookup is a memory operation plus one
vLog read. This is the WiscKey design paying off at our value size — see
[Paper 27](../research/paper-27-wisckey.md) Figure 10.

**Bloom filter memory (10 bits per key):**
```
At 200,700 chunks:   200,700 × 10 bits = 2.51 MB  (negligible)
At 2,007,000 chunks: 2,007,000 × 10 bits = 25.1 MB (negligible)
```

The Bloom filter false-positive rate (1%) means 1 in 100 audit challenges for a chunk not
assigned to this provider requires a RocksDB lookup before returning FAIL. At 2M chunks,
this is not a meaningful cost.

### 2.3 Audit receipt table growth in Postgres

The audit receipt table is INSERT-only and grows permanently. It is the largest table in
the system by row count and requires explicit archival planning.

**Daily insert rate:**
```
Rows/day = N_providers × chunks_per_provider
```

**Annual storage at representative scales:**

| Providers | Chunks/provider | Rows/day | Rows/year | Storage/year (@ 450 B/row) |
|---|---|---|---|---|
| 56 | 200,000 | 11.2 M | 4.1 B | 1.8 TB |
| 500 | 200,000 | 100 M | 36.5 B | 16.4 TB |
| 1,000 | 10,000 | 10 M | 3.65 B | 1.6 TB |
| 1,000 | 200,000 | 200 M | 73 B | 32.9 TB |
| 10,000 | 10,000 | 100 M | 36.5 B | 16.4 TB |

*Note on chunk count per provider:* A provider storing 50 GB holds ~200,000 chunks at
256 KB each. A provider storing 2.5 GB holds ~10,000 chunks. The planning row "10,000
providers × 10,000 chunks" represents the conservative low-storage scenario; "10,000
providers × 200,000 chunks" is the high-storage scenario.

**The key finding:** At even 56 providers each storing 50 GB, the audit receipt table
accumulates ~1.8 TB per year. Postgres with table partitioning by month is mandatory from
launch, not a V3 concern. Archive partitions older than the reliability score windows
(30 days) can be moved to cold object storage without affecting operational queries.

---

## 3. Bandwidth Budget

### 3.1 Steady-state repair bandwidth (BWavg)

The Giroire Formula 1 ([Paper 10](../research/paper-10-giroire-lazy.md)) gives:

```
BWavg ≈ (B × α) / (N × ln((s+r)/(s+r0)) × τ) × (s + r - r0 - 1) × lf

where:
  ln(56/24) = ln(2.333) ≈ 0.848
  s + r - r0 - 1 = 47
  τ = 1 hour
  α = 1 / (MTTF in hours) = 1 / (MTTF_days × 24)
  B = D / lb = total data / 4 MB per block
```

**Sensitivity to MTTF (at N=1,000 providers, 50 GB each, D=50 TB):**

| MTTF | α (per hour) | BWavg | Within 100 Kbps budget? |
|---|---|---|---|
| 90 days (mobile floor) | 4.63 × 10⁻⁴ | ~130 Kbps/peer | **No — exceeds budget** |
| 180 days (desktop minimum) | 2.31 × 10⁻⁴ | ~65 Kbps/peer | Yes — 35 Kbps headroom |
| 300 days (planning target) | 1.39 × 10⁻⁴ | ~39 Kbps/peer | Yes — 61 Kbps headroom |
| 380 days (NAS tier) | 1.10 × 10⁻⁴ | ~31 Kbps/peer | Yes — 69 Kbps headroom |

**Key result:** The 180-day floor for V2 providers ([ADR-010](../decisions/ADR-010-desktop-only-v2.md)) is the minimum MTTF at which the
bandwidth budget holds. Below 180 days the network consumes more bandwidth on repair than
providers can sustain without impacting foreground I/O.

BWavg scales as D/N — it is proportional to per-provider data load, not to total providers.
Increasing N while holding per-provider storage constant does not increase BWavg. This is
what makes the formula scale-invariant for a fixed per-provider storage allocation.

**Sensitivity to per-provider storage (at N=1,000, MTTF=300 days):**

| Storage/provider | D total | BWavg |
|---|---|---|
| 10 GB | 10 TB | ~7.8 Kbps/peer |
| 50 GB | 50 TB | ~39 Kbps/peer |
| 200 GB | 200 TB | ~156 Kbps/peer — **exceeds budget** |
| 130 GB (approx.) | 130 TB | ~100 Kbps/peer — **budget ceiling** |

*Warning:* At MTTF=300 days, a provider must not be loaded beyond approximately 130 GB of
chunk data to stay within the 100 Kbps background bandwidth budget. At MTTF=180 days (worst
case in V2) the ceiling drops to about 70 GB per provider. This is a **DESIGN constraint
that must appear in provider onboarding limits**; it is not currently stated in any ADR.

### 3.2 Burst repair bandwidth (Qpeek)

When one provider fails, all chunks they held (simultaneously) enter the repair queue.
The Giroire Formula 2 ([Paper 10](../research/paper-10-giroire-lazy.md)) gives:

```
Qpeek ≈ B × (s+r) / (N × (s+r0+1) × ln((s+r)/(s+r0))) × (s+r-r0-1) × lf
       = B × 56 / (N × 25 × 0.848) × 47 × 256 KB
```

At N=1,000, 50 GB/provider (B = 12.5M blocks):

```
Qpeek ≈ 12,500,000 × 56 / (1,000 × 25 × 0.848) × 47 × 256 KB
       ≈ 700,000,000 / 21,200 × 47 × 262,144 bytes
       ≈ 793 GB total network transfer per failure event
```

**Repair window at 100 Kbps/peer (N=1,000, aggregate 100 Mbps):**
```
θ = Qpeek / (N × bandwidth_per_peer)
  = 793 GB / (1,000 × 100 Kbps)
  = 793 × 10⁹ × 8 / (10⁵)
  ≈ 63,440 seconds ≈ 8 hours
```

The 12-hour safety budget ([ADR-004](../decisions/ADR-004-repair-protocol.md)) provides a 4-hour margin.

**Qpeek and θ at multiple scales:**

| N providers | Storage/provider | Qpeek | θ at 100 Kbps/peer | Within 12h window? |
|---|---|---|---|---|
| 56 | 50 GB | 44 GB | 22 hours (aggregate 5.6 Mbps) | **No — exceeds window** |
| 200 | 50 GB | 222 GB | 25 hours (aggregate 20 Mbps) | **No — marginal** |
| 500 | 50 GB | 396 GB | 18 hours (aggregate 50 Mbps) | **Borderline** |
| 1,000 | 50 GB | 793 GB | 8 hours (aggregate 100 Mbps) | Yes |
| 1,000 | 25 GB | 397 GB | 4 hours | Yes |
| 5,000 | 50 GB | 793 GB | 1.6 hours (aggregate 500 Mbps) | Yes |
| 10,000 | 50 GB | 793 GB | 0.8 hours | Yes |

*Critical finding:* **At very small provider counts (56–200), the Qpeek repair window
exceeds 12 hours.** This means a second failure during an ongoing repair is possible and
the system is more vulnerable shortly after launch than the steady-state numbers suggest.
The `repair_queue_depth` metric (defined in [architecture.md Section 24](./architecture.md#24-observability))
is the primary monitoring signal; the alert threshold fires when depth > 1,000. At N < 500,
operators must additionally verify manually that no repair job is outstanding before
accepting a new provider departure within a 24-hour window — the automated alert alone is
insufficient at this scale because individual failure events can push the queue below the
threshold while still exceeding the 12-hour repair window. The risk decreases monotonically
as N increases past 500.

*Note on Dalle et al. ([Paper 36](../research/paper-36-dalle-failure-correlation.md)):* The real standard deviation of repair bandwidth is
22× larger than the independent model predicts. The Qpeek values above are mean burst
estimates. A correlated failure event (multiple providers on the same ISP failing
simultaneously) produces a burst whose magnitude scales with the number of simultaneous
departures. The 20% ASN cap ([ADR-014](../decisions/ADR-014-adversarial-defences.md)) limits this burst to at most 20% × N simultaneous
departures from a single correlated group.

### 3.3 Heartbeat ingress to microservice

Each provider sends a ~200-byte heartbeat every 4 hours.

```
Daily heartbeat volume = N × 200 bytes × 6 heartbeats/day
```

| N providers | Daily volume | Microservice ingress rate |
|---|---|---|
| 56 | 67 KB | < 1 byte/sec |
| 1,000 | 1.2 MB | ~14 bytes/sec |
| 10,000 | 12 MB | ~139 bytes/sec |
| 100,000 | 120 MB | ~1.4 KB/sec |

Heartbeat ingress is negligible at all realistic V2 scales. The concern is not bandwidth but
connection storms: if the microservice restarts, all N providers reconnect within 4 hours.
At N=10,000 this is ~2,500 reconnects/hour = ~0.7/sec average, but with bursty reconnection
after restart. The load balancer should be placed in front of the heartbeat endpoint separately
from the audit challenge endpoints, as [ADR-028](../decisions/ADR-028-provider-heartbeat.md) notes.

### 3.4 Economic Capacity
> **Note:** All figures in this section use ₹1.50/GB/month as an OQ-001 placeholder only. This rate has not been decided. Substitute the product-decided rate when OQ-001 is resolved before private beta (see [requirements.md OQ-001](./requirements.md/#12-open-questions)).

This section estimates the fiat cost floor at each scale milestone — the minimum payment
volume the system must process and the Razorpay fee load it incurs.

**Razorpay payout fee model (from [Paper 35](../research/paper-35-razorpay-upi-docs.md) / Razorpay documentation):**

Fees are deducted from the Razorpay master account, not from the provider payout. The gross
payout to the provider is exactly the amount specified.

- IMPS payouts: approximately ₹3–5 per transaction (varies by bank partnership)
- UPI payouts to VPA: typically ₹0 on the UPI rail, but Razorpay charges a service fee of
  ₹2–4 per transaction on programmatic payout calls
- Use ₹4 per payout as the conservative planning estimate

**Minimum viable payout volume — monthly (to keep the network solvent):**

For a provider storing 50 GB with MTTF=300 days to earn more than the cost of participation
(electricity, minimal bandwidth consumed by 39 Kbps background), a rough minimum storage
rate of ₹1.50 per GB per month is assumed. This is deliberately conservative and will be
revised once the storage rate is set as a product decision (OQ-001 in requirements.md).

```
Minimum monthly earnings per provider = 50 GB × ₹1.50/GB = ₹75/month
Monthly payout volume at N providers = N × ₹75
Razorpay payout fees at N providers = N × ₹4 (one payout per provider per month)
Net revenue to providers = monthly payout volume - fees = N × ₹71
```

**Monthly payout volume and fee load at scale milestones:**

| N providers | Monthly payout (gross, ₹) | Razorpay fees (₹) | Fee as % of payout |
|---|---|---|---|
| 56 | 4,200 | 224 | 5.3% |
| 300 | 22,500 | 1,200 | 5.3% |
| 1,000 | 75,000 | 4,000 | 5.3% |
| 5,000 | 375,000 | 20,000 | 5.3% |
| 10,000 | 750,000 | 40,000 | 5.3% |

*Note:* Fee percentage is constant because both scale linearly with N. At ₹4/payout, fees
consume 5.3% of payout volume at the ₹75/provider/month floor rate. If the actual storage
rate is higher (e.g. ₹5/GB/month → ₹250/provider/month), fees drop to 1.6%.

**Minimum data owner deposit volume to sustain the network at each scale:**

Data owners must collectively deposit enough escrow to cover provider payouts each month.
A provider storing 50 GB per file represents 56 providers × 1 chunk each — but in practice,
data owners share the provider pool. For planning, assume each provider's storage is filled
by 10 independent data owners on average, each depositing ₹7.50/month for their share.

```
Required data owner deposit volume = provider payout volume + Razorpay fees + Vyomanaut margin
Minimum (at zero margin) = N × ₹75 + N × ₹4 = N × ₹79/month
```

The storage rate product decision (OQ-001) sets the Vyomanaut margin above this floor.
At the ₹1.50/GB planning rate, the network is margin-negative — this rate is a floor for
provider viability, not a target price.

**Fee sustainability check:** The 5.3% Razorpay fee load at ₹75/month per provider is
non-trivial. If the storage rate is set at ₹2/GB/month (₹100/provider/month), fees drop
to 4%. At ₹5/GB/month (₹250/provider/month), fees drop to 1.6%. The storage rate must
be set high enough that fees do not materially erode provider earnings. The product
decision on pricing (OQ-001) must account for this explicitly.

---

## 4. Audit System Throughput

### 4.1 Challenge issuance rate

```
Challenges/day  = N_providers × chunks_per_provider
Challenges/sec  = Challenges/day / 86,400
```

| Providers | Chunks/provider | Challenges/day | Challenges/sec |
|---|---|---|---|
| 56 | 10,000 | 560,000 | 6.5 /sec |
| 56 | 200,000 | 11.2 M | 130 /sec |
| 1,000 | 10,000 | 10 M | 116 /sec |
| 1,000 | 200,000 | 200 M | 2,315 /sec |
| 10,000 | 10,000 | 100 M | 1,157 /sec |
| 10,000 | 200,000 | 2 B | 23,148 /sec |
| 100,000 | 10,000 | 1 B | 11,574 /sec — **approaches Postgres ceiling** |

### 4.2 Postgres INSERT ceiling for audit_receipts

As noted in Section 1.3, the INSERT ceiling on the `audit_receipts` schema has not been
benchmarked and must be measured before V2 GA. The general Postgres benchmark ceiling of
5,000–10,000 rows/sec is used as a planning estimate. The actual ceiling may be lower due
to row width (two 64-byte Ed25519 signatures per row).

```
Estimated Postgres ceiling at 10,000 rows/sec:
  Max challenges/day = 10,000 × 86,400 = 864 M
  → ~100,000 providers × 10,000 chunks/provider (0.25 GB/provider of 256 KB chunks)
  → ~5,000 providers × 200,000 chunks/provider (50 GB/provider)
```

**Based on the planning estimate, Postgres INSERT is the first architectural ceiling in the
audit pipeline. It binds at approximately 100,000 providers × 10,000 chunks, or 5,000
providers × 200,000 chunks. This estimate must be replaced with a measured value before
V2 GA.**

At V2 launch (hundreds of providers × tens of thousands of chunks), the INSERT rate is
~tens of rows/sec — orders of magnitude below any reasonable ceiling. This is not a V2
concern but is the hard limit before the audit system must be re-architected (batching,
partitioned writes, write-ahead log bypassing, or probabilistic sampling).

*If probabilistic sampling is introduced to extend this ceiling*, the SHELBY
incentive-compatibility conditions (Theorem 1, [Paper 37](../research/paper-37-shelby-incentive-compatibility.md)) must be re-verified at the
new per-chunk audit frequency (pau) before the change is deployed. The V2 design relies
on pau ≈ 1 (daily full audit per chunk) to trivially satisfy Condition (i).

### 4.3 Audit receipt storage ceiling

Connecting audit throughput to Postgres storage:

```
Storage/day  = Challenges/day × ~450 bytes/row
Storage/year = Storage/day × 365
```

At the Postgres INSERT planning ceiling (864 M rows/day):
```
Storage/day  = 864M × 450 bytes = 389 GB/day
Storage/year = 389 × 365 = 142 TB/year
```

At V2 launch (10M rows/day, ~1,000 providers × 10,000 chunks):
```
Storage/day  = 10M × 450 bytes = 4.5 GB/day
Storage/year = 4.5 × 365       = 1.6 TB/year
```

**Implication:** Monthly table partitioning is not optional infrastructure — it is the only
way to manage audit receipt storage without the table scan growing unbounded. Partitions older
than 30 days can be archived because the reliability scorer only queries the trailing 30-day
window ([ADR-008](../decisions/ADR-008-reliability-scoring.md)).

---

## 5. Network and DHT Scaling

### 5.1 DHT hop count and record capacity at scale

Kademlia lookup takes O(log₂ N / α) round trips with α=3 parallel lookups.

| N providers | log₂(N) | Approx. hops (α=3) | Est. DHT lookup latency |
|---|---|---|---|
| 56 | 5.8 | ~2 | ~1.2 s |
| 500 | 9.0 | ~3 | ~1.9 s |
| 1,000 | 10.0 | ~3–4 | ~2.5 s |
| 10,000 | 13.3 | ~5 | ~3.1 s |
| 100,000 | 16.6 | ~6 | ~3.7 s |
| 300,000 (IPFS scale) | 18.2 | ~6 | ~3.7 s (IPFS measured median 622 ms) |

*Note:* The IPFS measurement ([Paper 20](../research/paper-20-trautwein-ipfs.md)) observed 622 ms median for content retrieval, which
terminates on first record found. Full publication (finding k=20 closest peers) has a median
of 33.8 seconds. Vyomanaut's DHT uses FIND_VALUE (terminates early) so the retrieval latency
benchmark is the relevant one.

DHT lookup latency is a logarithmic function of N. Adding providers makes lookups marginally
slower — this is not a scaling bottleneck.

**DHT record count and provider RAM requirement:**

Each assigned chunk on each provider generates one DHT provider record. libp2p's Kademlia
implementation stores provider records in memory on every DHT server node. Record count
therefore scales as:

```
DHT records per provider node (routing table + provider records it holds)
  ≈ chunks_per_provider (provider records for chunks it owns)
    + k-bucket entries (routing state, bounded at k=16 per bucket × log₂(N) buckets)
```

The dominant term at any realistic scale is the provider records. At 200 bytes per record
(estimated from libp2p's record encoding — chunk_id hash + provider multiaddr + expiry):

```
DHT record memory at 200,000 chunks/provider:
  200,000 × 200 bytes = 40 MB per DHT server node

DHT record memory at 2,000,000 chunks/provider (500 GB at 256 KB/chunk):
  2,000,000 × 200 bytes = 400 MB per DHT server node
```

At current allocation targets (50–500 GB per provider), DHT record memory is 40–400 MB —
within the operating range of any desktop machine. However, this establishes a concrete
**provider hardware requirement**: a provider daemon running as a DHT server must have at
least the following free RAM available for the DHT layer, in addition to the OS and other
processes:

| Provider storage | Chunks | DHT record memory |
|---|---|---|
| 10 GB | ~40,000 | ~8 MB |
| 50 GB | ~200,000 | ~40 MB |
| 200 GB | ~800,000 | ~160 MB |
| 500 GB | ~2,000,000 | ~400 MB |

**This is a provider hardware requirement that must appear in the daemon's minimum system
specification.** A provider attempting to run the daemon with less RAM than their chunk
allocation requires will cause the DHT layer to evict records, producing lookup failures
and audit TIMEOUT results that are indistinguishable from provider absence.

At 10,000 providers each storing 200,000 chunks: total DHT record count across the network
is 2 billion, distributed across all DHT server nodes. Each node holds a locally relevant
subset (determined by XOR distance), not the full 2 billion — so per-node memory remains
bounded at ~40–400 MB regardless of total network size. DHT record memory does not become
a network-level bottleneck at V2 or V3 scale.

### 5.2 Relay infrastructure scaling

Every provider behind symmetric NAT requires a Circuit Relay v2 slot for every concurrent
connection. Assume relay-dependent fraction = 30% (global baseline; Indian CGNAT may be
higher, tracked in Q20-1).

```
Relay-dependent providers = N × 0.30
Concurrent relay slots needed ≈ relay-dependent providers × concurrent_connections_per_peer
```

Each relay-dependent provider needs approximately 1–2 concurrent relay connections (one for
the microservice audit channel, possibly one for data transfer). Assume 1.5 concurrent slots
on average.

```
Required relay slots = N × 0.30 × 1.5 = N × 0.45
Available slots = relay_nodes × 128
Headroom formula: add relay node when (N × relay_fraction × 1.5) > (relay_nodes × 128 × 0.80)
```

**Relay headroom at launch (3 nodes, 384 slots):**

| N providers | Relay-dependent (30%) | Slots needed (×1.5) | Slots available | Headroom |
|---|---|---|---|---|
| 56 | 17 | 26 | 384 | **92% spare** |
| 300 | 90 | 135 | 384 | **65% spare** |
| 500 | 150 | 225 | 384 | **41% spare** |
| 850 | 255 | 383 | 384 | **~0% — trigger upgrade** |
| 1,000 | 300 | 450 | 384 | **Oversubscribed** |

**Relay infrastructure upgrade trigger: ~850 providers.** At this point, add a fourth relay
node (512 total slots) or scale existing nodes. Each additional relay node buys headroom for
an additional ~280 providers.

*If Indian CGNAT pushes the relay-dependent fraction to 45%:*

```
Required slots at 45% = N × 0.45 × 1.5 = N × 0.675
Oversubscription at 3 nodes = 384 / 0.675 = ~570 providers
```

Under the pessimistic Indian CGNAT assumption, relay becomes the binding constraint at
approximately **570 providers**, not 850. Add relay capacity before reaching 400 providers
as a conservative operational rule.

### 5.3 Microservice gossip overhead

Each of the 3 microservice replicas contacts 1 random peer per second for membership
reconciliation. This produces:

```
Gossip messages/sec per replica = 1
Gossip messages/sec total = 3
```

At 3 replicas this is trivial. [Paper 12](../research/paper-12-dynamo_1.md) notes that full-membership gossip (each node aware of all
others) does not scale past a few hundred nodes. The microservice cluster will never exceed
5 nodes in the current architecture — full-membership gossip is the right model here. The
DHT solves the provider-scale discovery problem independently.

---

## 6. Payment System Throughput

### 6.1 Monthly payout release compute

On the 23rd of each month, the release computation processes every active provider:

```
Providers processed = N_active
Operations per provider = 
  1 × SELECT (30-day score)
  1 × SELECT (7-day score, for dual-window check)
  1 × SELECT (escrow_events balance)
  1 × PATCH Razorpay (on_hold_until update)
  1 × INSERT escrow_events (RELEASE event)
  = ~5 operations per provider
```

At N=10,000 providers, all processed within a few-hour window:
```
Total operations = 10,000 × 5 = 50,000
Operations/hour  = 50,000 / 4 hours ≈ 3.5/sec (sustained)
```

This is well within Postgres's capacity. The Razorpay Route PATCH calls are the rate-limiting
step — Razorpay's standard API rate limit is 500 requests/minute = 8.3/sec, sufficient for
N up to approximately 120,000 providers before the 4-hour window becomes a constraint.

### 6.2 Escrow_events table growth

```
Events per day = 
  N_providers × 1 (daily RELEASE event on payout cycle: ~1/30 per provider/day on average)
  + seizure events (proportional to churn rate)
  + deposit events (data owner frequency)
```

Simplified: approximately 1 escrow event per provider per month plus seizure and deposit events.

```
At N=10,000: ~10,000 events/month = ~333/day = negligible
```

The escrow_events table grows slowly. It is not a scaling concern at any realistic V2 scale.

### 6.3 Razorpay API constraint for seizures

When a provider departs silently, the seizure flow requires a Route reversal API call. If
there is a mass departure event (e.g., ISP outage affecting 11 providers simultaneously,
which is the maximum the 20% ASN cap allows):

```
Simultaneous seizure calls = 11
Razorpay rate limit = 500 req/minute
→ 11 seizures completes in < 2 seconds — not a constraint
```

---

## 7. Minimum Viable Floor

The network is inert — unable to store any data — until all seven bootstrap conditions
defined in [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) are simultaneously satisfied. This section expresses those
conditions in terms of what can be stored at the exact floor.

**At exactly 56 active providers, one file segment's shard assignment is the only possible operation:**

```
Max simultaneous uploads = floor(N / n) = floor(56 / 56) = 1 file segment at a time
```

The assignment service cannot begin a second file segment's upload until all 56 shard slots of
the first are confirmed — the same 56 providers are needed for both. In practice, a second
upload must wait for the first to complete at the provider level, not just the microservice
level. This constraint relaxes linearly: at N=112, 2 simultaneous uploads are possible; at
N=560, up to 10 simultaneous uploads are possible. During private beta, the product team must
communicate this concurrency limit to data owner testers — a queued second upload is not a
bug, it is the expected behaviour at minimum network size.

**First usable file size (at 56 providers, each with capacity):**
```
One segment = 14 MB of raw wire data (56 × 256 KB)
One segment = 4 MB of user data (16 × 256 KB, the systematic shards)
```

The minimum file that can be fully stored and retrieved is 4 MB of plaintext, occupying
one complete segment.

**ASN constraint at 56 providers across exactly 5 ASNs:**
```
Max shards from one ASN = floor(56 × 0.20) = 11
Providers per ASN (evenly distributed) = 56 / 5 = 11.2 ≈ 11 per ASN
→ Even distribution satisfies the cap; uneven distribution may not
```

With exactly 5 ASNs and an uneven distribution (e.g., 20 / 18 / 10 / 5 / 3), the cap fails
immediately: ASN-1 holds 20 providers = 35.7% — over the 20% limit. The assignment service
must enforce the cap at the provider level, not just the ASN count level.

---

## 8. Hard Ceilings — What Breaks First

The following table lists every identified architectural constraint, the scale at which it
binds, and the action needed to push past it. They are ordered by the scale at which each
constraint binds.

| Constraint | Binds at | Symptom | Action to extend |
|---|---|---|---|
| Relay slot exhaustion (45% fraction, Indian CGNAT) | ~570 providers | Audit TIMEOUT rate rises for relay-dependent providers | Add relay nodes; each adds capacity for ~280 more providers |
| Relay slot exhaustion (30% fraction, 3 nodes × 128 slots) | ~850 providers | Same | Same; plan for more aggressive relay scaling if Q20-1 confirms high CGNAT fraction |
| Repair window exceeds 12h (small network) | < 500 providers | Qpeek repair takes > 12 hours; second failure risk during repair; `repair_queue_depth` alert (threshold 1,000) may not fire before window is exceeded | Accept risk at launch; monitor `repair_queue_depth` (see [architecture.md §24](./architecture.md#24-observability)); manually verify no concurrent repairs outstanding before accepting new departures |
| Postgres audit INSERT rate | ~100,000 providers × 10,000 chunks OR ~5,000 providers × 200,000 chunks (planning estimate — must be benchmarked on actual schema before V2 GA) | Audit challenge processing queue grows; INSERT latency rises | Monthly partitioning (mandatory from day 1); probabilistic sampling when ceiling approached (re-verify SHELBY conditions first) |
| Postgres audit receipt storage | ~1.6 TB/year at V2 launch scale | Disk fills; query latency rises on unpartitioned table | Monthly partition archiving to object storage |
| Per-provider bandwidth (50 GB/provider at MTTF=180d) | ~70 GB/provider at MTTF=180d | BWavg exceeds 100 Kbps; repair traffic visibly impairs provider foreground I/O | Enforce per-provider chunk count ceiling in onboarding; reduce allocation at lower MTTF |
| Provider RAM for DHT records | ~400 MB at 500 GB/provider allocation | DHT record eviction; audit TIMEOUT false positives | Enforce minimum free RAM in provider hardware requirements; document in daemon minimum spec |
| Microservice quorum (3-node gossip) | > 5 replicas | Gossip overhead grows; operational complexity exceeds value of hand-rolled solution | Migrate to etcd or Consul ([ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md) open constraint) |
| Razorpay Route payout release | ~120,000 providers in 4-hour window | Release window must extend past midnight | Distribute release across longer window or upgrade Razorpay rate limit tier |

**The most likely first constraint in the V2 growth trajectory is relay slot exhaustion,
not compute or database throughput.** Relay infrastructure is cheap to add incrementally
(one node per ~280 providers) but must be provisioned ahead of the trigger, not in response
to audit TIMEOUT alerts.

The second constraint likely to bind is the per-provider bandwidth ceiling at the MTTF=180d
floor — a provider storing more than 70 GB of chunks who churns frequently will push peers
toward their background bandwidth limit. This ceiling must be enforced in the provider
onboarding flow, not discovered operationally.

---

## 9. Scale Milestones

These milestones are the operational waypoints between the minimum viable floor and the
first architectural ceiling.

### Milestone 1 — Network open (56 providers)
- First uploads permitted
- Simultaneous uploads: 1 at a time (all 56 slots occupied by one segment); second upload queues until first completes
- Relay: 17 relay-dependent providers, 26 slots needed, 384 available — no action needed
- Repair window: ~22 hours per failure — monitor `repair_queue_depth` ([architecture.md §24](./architecture.md#24-observability)); acceptable if no concurrent failures occur; manually verify no outstanding repair jobs before accepting any new provider departure within a 24-hour window

### Milestone 2 — 300 providers
- Simultaneous uploads: 5 independent file segments in parallel (300 / 56 ≈ 5)
- Relay: 90 relay-dependent, 135 slots needed — 65% headroom remaining at 30% CGNAT fraction; if Q20-1 telemetry shows Indian CGNAT fraction exceeds 30%, add 4th relay node before reaching 400 providers
- Repair window: ~8 hours per failure (within the 12-hour safety window)
- BWavg at 50 GB/provider, MTTF=300d: ~39 Kbps/peer — comfortable
- **Relay capacity is healthy; no infrastructure action needed at 30% CGNAT; watch Q20-1 telemetry**

### Milestone 3 — 570 providers (relay warning at 45% CGNAT fraction)
- Relay slots needed: ~386 vs 384 available — **add the 4th relay node now**
- A 4th relay node provides headroom to ~860 providers at 45% CGNAT

### Milestone 4 — 850 providers (relay warning at 30% fraction)
- Relay slots needed: ~383 vs 384 — **add 4th relay node if not already done**

### Milestone 5 — 1,000 providers
- Repair window: ~8 hours — comfortable
- Audit challenge rate: up to 200M/day (200,000 chunks/provider) — 2,315/sec — still below
  Postgres ceiling (planning estimate)
- Monthly audit receipt storage: ~375 GB/month at 200,000 chunks/provider — archiving needed
- **First time Giroire formula is comfortably within all parameters simultaneously**

### Milestone 6 — 5,000 providers
- Audit INSERT rate at 200,000 chunks/provider: ~11,574/sec — approaching Postgres planning ceiling
- At 10,000 chunks/provider: 579/sec — comfortable
- **Storage allocation ceiling per provider becomes the binding parameter at this scale;
  providers storing > 50 GB at MTTF=300d push BWavg toward the 100 Kbps budget**
- DHT record memory: ~40 MB per provider at 200,000 chunks — within range; document in hardware spec

### Milestone 7 — 10,000 providers
- Audit receipts: 16.4 TB/year (10,000 chunks/provider) — partitioning and archiving mandatory
- Heartbeat ingress: 12 MB/day — negligible
- Monthly payout computation: 50,000 operations at 3.5/sec — within Razorpay rate limits
- Monthly Razorpay fees at ₹75/provider/month: ₹40,000 — monitor fee-to-payout ratio
- **Postgres audit INSERT rate remains the primary ceiling; benchmark actual schema INSERT rate; evaluate probabilistic sampling**

---

## 10. Summary: Floor and Ceiling

| | Value | Binding constraint |
|---|---|---|
| **Minimum viable floor** | 56 providers, 5 ASNs, 3 metro regions, 3 relay nodes, 56 Razorpay accounts | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) — all conditions must be simultaneously true |
| **Simultaneous uploads at floor** | 1 at a time | Assignment service requires 56 distinct providers per segment |
| **Usable storage at floor** | ~800 GB user data (56 providers × 50 GB, 28.57% efficiency) | s/n ratio |
| **BWavg at floor (MTTF=300d)** | ~39 Kbps/peer | Giroire Formula 1, [Paper 10](../research/paper-10-giroire-lazy.md) |
| **Repair window at floor** | ~22 hours (exceeds 12h safety target) | Small N; monitor `repair_queue_depth`; manual oversight required |
| **First relay upgrade trigger** | ~570–850 providers | CGNAT fraction uncertainty; Q20-1 telemetry resolves |
| **Per-provider chunk ceiling (MTTF=180d)** | ~70 GB | 100 Kbps bandwidth budget |
| **Per-provider chunk ceiling (MTTF=300d)** | ~130 GB | 100 Kbps bandwidth budget |
| **Provider RAM required for DHT (500 GB allocation)** | ~400 MB free | DHT record memory at 2M chunks per provider |
| **Audit throughput ceiling (current architecture)** | ~100,000 providers × 10,000 chunks (planning estimate) | Postgres INSERT rate — must be benchmarked on actual schema before V2 GA |
| **Architecture change required above** | 100,000 providers (at 10,000 chunks each) | Probabilistic audit sampling or write sharding |
| **Razorpay fee load** | ~5.3% of payout at ₹75/provider/month floor rate | Drops to 1.6% at ₹250/provider/month; pricing decision (OQ-001) must account for this |