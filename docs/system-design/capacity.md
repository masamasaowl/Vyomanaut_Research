# Vyomanaut V2 — Capacity Estimation

**Status:** Reference document — recompute when any value in the dimensions table changes.  
**Audience:** Platform and infrastructure engineers sizing hardware, setting operational
alert thresholds, and evaluating whether a proposed scale target is safe.  
**Method:** Every ceiling is computed from values already present in the ADRs and research
log. No new assumptions are introduced. Where a value is uncertain or assumed, it is
flagged explicitly. Every quoted source is linked.

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
| Postgres single-instance INSERT ceiling | — | 5,000–10,000 | rows/sec | MEASURED (optimised single instance) | [Paper 11](../research/paper-11-bailis-coordination.md) |

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
Operators should monitor repair queue depth carefully in the first months before N > 500.
The risk decreases monotonically as N increases.

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
| 100,000 | 10,000 | 1 B | 11,574 /sec — **Postgres ceiling** |

### 4.2 Postgres INSERT ceiling for audit_receipts

The Bailis paper ([Paper 11](../research/paper-11-bailis-coordination.md)) and operational Postgres benchmarks establish a single-instance
optimised INSERT ceiling of 5,000–10,000 rows/sec.

```
Postgres ceiling at 10,000 rows/sec:
  Max challenges/day = 10,000 × 86,400 = 864 M
  → ~100,000 providers × 10,000 chunks/provider (0.25 GB/provider of 256 KB chunks)
  → ~5,000 providers × 200,000 chunks/provider (50 GB/provider)
```

**Postgres INSERT is the first architectural ceiling in the audit pipeline. It binds at
approximately 100,000 providers × 10,000 chunks, or 5,000 providers × 200,000 chunks.**

At V2 launch (hundreds of providers × tens of thousands of chunks), the INSERT rate is
~tens of rows/sec — orders of magnitude below the ceiling. This is not a V2 concern but
is the hard limit before the audit system must be re-architected (batching, partitioned
writes, write-ahead log bypassing, or probabilistic sampling).

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

At the Postgres INSERT ceiling (864 M rows/day):
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

### 5.1 DHT hop count at scale

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
upload must wait for the first to complete at the provider level, not just the microservice level.

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
| Relay slot exhaustion (30% fraction, 3 nodes × 128 slots) | ~850 providers | Audit TIMEOUT rate rises for relay-dependent providers | Add relay nodes; each adds capacity for ~280 more providers |
| Relay slot exhaustion (45% fraction, Indian CGNAT) | ~570 providers | Same | Same; plan for more aggressive relay scaling |
| Repair window exceeds 12h (small network) | < 500 providers | Qpeek repair takes > 12 hours; second failure risk during repair | Accept risk at launch; monitor repair queue; ensure repair completes before MTTF second-failure probability window |
| Postgres audit INSERT rate | ~100,000 providers × 10,000 chunks OR ~5,000 providers × 200,000 chunks | Audit challenge processing queue grows; INSERT latency rises | Monthly partitioning (mandatory from day 1); probabilistic sampling when ceiling approached (re-verify SHELBY conditions first) |
| Postgres audit receipt storage | ~1.6 TB/year at V2 launch scale | Disk fills; query latency rises on unpartitioned table | Monthly partition archiving to object storage |
| Per-provider bandwidth (50 GB/provider at MTTF=180d) | ~70 GB/provider at MTTF=180d | BWavg exceeds 100 Kbps; repair traffic visibly impairs provider foreground I/O | Enforce per-provider chunk count ceiling in onboarding; reduce allocation at lower MTTF |
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
- Simultaneous uploads: 1 at a time (all 56 slots occupied by one segment)
- Relay: 17 relay-dependent providers, 26 slots needed, 384 available — no action needed
- Repair window: ~22 hours per failure — monitor; acceptable if no concurrent failures

### Milestone 2 — 300 providers
- Simultaneous uploads: 5 independent file segments in parallel (300 / 56 ≈ 5)
- Relay: 90 relay-dependent, 135 slots needed — 65% headroom remaining
- Repair window: ~8 hours per failure (within the 12-hour safety window)
- BWavg at 50 GB/provider, MTTF=300d: ~39 Kbps/peer — comfortable
- **Relay capacity is healthy; no infrastructure action needed**

### Milestone 3 — 570 providers (relay warning at 45% CGNAT fraction)
- Relay slots needed: ~386 vs 384 available — **add the 4th relay node now**
- A 4th relay node provides headroom to ~860 providers at 45% CGNAT

### Milestone 4 — 850 providers (relay warning at 30% fraction)
- Relay slots needed: ~383 vs 384 — **add 4th relay node if not already done**

### Milestone 5 — 1,000 providers
- Repair window: ~8 hours — comfortable
- Audit challenge rate: up to 200M/day (200,000 chunks/provider) — 2,315/sec — still below
  Postgres ceiling of 5,000–10,000/sec
- Monthly audit receipt storage: ~375 GB/month at 200,000 chunks/provider — archiving needed
- **First time Giroire formula is comfortably within all parameters simultaneously**

### Milestone 6 — 5,000 providers
- Audit INSERT rate at 200,000 chunks/provider: 11,574/sec — at Postgres ceiling
- At 10,000 chunks/provider: 579/sec — comfortable
- **Storage allocation ceiling per provider becomes the binding parameter at this scale;
  providers storing > 50 GB at MTTF=300d push BWavg toward the 100 Kbps budget**

### Milestone 7 — 10,000 providers
- Audit receipts: 16.4 TB/year (10,000 chunks/provider) — partitioning and archiving mandatory
- Heartbeat ingress: 12 MB/day — negligible
- Monthly payout computation: 50,000 operations at 3.5/sec — within Razorpay rate limits
- **Postgres audit INSERT rate remains the primary ceiling; evaluate probabilistic sampling**

---

## 10. Summary: Floor and Ceiling

| | Value | Binding constraint |
|---|---|---|
| **Minimum viable floor** | 56 providers, 5 ASNs, 3 metro regions, 3 relay nodes, 56 Razorpay accounts | [ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) — all conditions must be simultaneously true |
| **Usable storage at floor** | ~800 GB user data (56 providers × 50 GB, 28.57% efficiency) | s/n ratio |
| **BWavg at floor (MTTF=300d)** | ~39 Kbps/peer | Giroire Formula 1, [Paper 10](../research/paper-10-giroire-lazy.md) |
| **Repair window at floor** | ~22 hours (exceeds 12h safety target) | Small N; monitor closely |
| **First relay upgrade trigger** | ~570–850 providers | CGNAT fraction uncertainty |
| **Per-provider chunk ceiling (MTTF=180d)** | ~70 GB | 100 Kbps bandwidth budget |
| **Per-provider chunk ceiling (MTTF=300d)** | ~130 GB | 100 Kbps bandwidth budget |
| **Audit throughput ceiling (current architecture)** | ~100,000 providers × 10,000 chunks | Postgres INSERT rate, [Paper 11](../research/paper-11-bailis-coordination.md) |
| **Architecture change required above** | 100,000 providers (at 10,000 chunks each) | Probabilistic audit sampling or write sharding |