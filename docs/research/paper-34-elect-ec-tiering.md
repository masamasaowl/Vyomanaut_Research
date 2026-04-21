## Paper 34 — ELECT: Enabling Erasure Coding Tiering for LSM-tree-based Storage

**Authors:** Yanjing Ren, Yuanming Ren — CUHK; Xiaolu Li, Yuchong Hu — HUST; Jingwei Li — UESTC; Patrick P. C. Lee — CUHK
**Venue / Year:** USENIX FAST 2024
**Topics:** #3, #16
**ADRs produced:** none — ADR-023 and ADR-018 confirmed; see Decisions Influenced

---

### Problem Solved

Distributed KV stores that use LSM-trees (Cassandra and similar) default to full replication across all nodes for all data, regardless of access frequency. For resource-constrained hot tiers (edge nodes with limited storage), this 3× overhead is prohibitive. Erasure coding reduces storage overhead but introduces reconstruction penalty on reads. The paper demonstrates that the LSM-tree's own level structure already partitions data by temperature — the last level holds the vast majority of data but a tiny fraction of accesses — making it the natural target for EC while leaving hotter lower levels replicated. For Vyomanaut, the paper's primary value is an empirical confirmation that write-once cold storage is the ideal operating regime for EC, and that reconstruction cost in EC systems is overwhelmingly network-bound rather than compute-bound.

---

### Key Findings

**Last-level data is overwhelmingly cold (Figure 2):**
In a Cassandra cluster loaded with 100M KV pairs and 10M Zipf-distributed reads (Zipfian constant 0.99), the last LSM-tree level L4 holds 56.2% of all SSTables but accounts for only 10.2% of all accesses. Within L4, only 18.2% of SSTables are ever accessed during the workload. The intermediate levels (L2, L3) hold 39.7% of SSTables and absorb 89.8% of reads. This measurement validates that LSM-tree level is a reliable cold proxy.

**Reconstruction is 93.3% network-bound (Table 2):**
For full-node recovery of 30 GiB of data with (n=6, k=4) RS coding, the total recovery time breaks down as: copy replicated SSTables 13.5 s, retrieve k data/parity SSTables for EC decoding 374.0 s, decode 13.3 s. Network retrieval dominates at 93.3% of recovery time. Decode is 3.3% — erasure coding compute is negligible compared to moving the data.

**Normal-mode performance overhead is minimal (Figures 5–6):**
ELECT achieves 56.1% edge storage savings versus full 3× replication, while maintaining within 3% throughput parity across all six YCSB workloads in normal mode. Scan-intensive workloads actually improve by 2.84× because ELECT's decoupled LSM-tree management reduces I/O amplification in compaction. Write performance is unaffected — redundancy transitioning happens entirely in the background.

**Degraded reads are significantly more expensive (Figure 6b, Table 1):**
When a node fails, reads that land on erasure-coded SSTables must retrieve k=4 SSTables from surviving nodes for reconstruction. This produces a 5.32× increase in average read latency in degraded mode. The 99th-percentile latency for reads, however, remains similar (~1.7 ms) because most reads still hit live nodes — it is the tail of reads that encounters degraded paths.

**Offline encoding is the right pattern for mixed hot/cold workloads (Section 3, Q2):**
Encoding KV pairs inline (on the write path) forces all writes through the EC overhead and cannot distinguish hot from cold data. Offline encoding — writing with replication first, then converting in background — decouples the write path from the encoding decision. This allows hotness to be measured before encoding is applied, limiting degraded read overhead to genuinely cold data.

---

### Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Cross-SSTable encoding (one SSTable = one chunk) | Per-KV-pair self-encoding | No per-KV metadata overhead; matches LSM-tree's natural I/O granularity; degraded reads must retrieve k entire SSTables (4 MiB each) |
| Offline encoding (background conversion) | Inline encoding (write path) | Hotness data is available before encoding; write path is unaffected; brief window where data is neither fully replicated nor fully EC-protected |
| EC applied to last LSM-tree level only | EC across all levels | Hot data in lower levels stays fast; EC benefits are concentrated where data is coldest; levels L0–L(ℓ−1) still incur 3× replication overhead |
| Storage saving target α (user-configurable) | Fixed coding parameters | Single knob balances storage vs performance across deployment contexts; α > 0.5 begins offloading to cold tier, increasing degraded read latency significantly |
| Decoupled replication management (R separate LSM-trees per node) | All replicas in one LSM-tree | Enables per-replica compaction and targeted EC conversion; increases per-node memory overhead by R-fold for metadata |

---

### Breaks in Our Case

- **ELECT starts from a replicated baseline — each node holds R copies of every KV pair it owns** ≠ **Vyomanaut providers each hold exactly one RS shard of each file; there is no local replication to convert**
→ ELECT's "redundancy transitioning" (replication → EC) has no direct equivalent in Vyomanaut. Our RS encoding happens client-side before any data is transmitted to providers. A Vyomanaut provider's vLog is already an EC data store from the moment a chunk arrives. ELECT's encoding architecture is not adopted.
- **ELECT's coding group spans n consecutive nodes in a distributed hash ring** ≠ **Vyomanaut's coding group is the full 56-provider network, distributed by the coordination microservice**
→ ELECT treats a group of n physically adjacent nodes as the fault domain. Vyomanaut's fault domain is the entire provider set, with the 20% ASN cap (ADR-014) enforcing diversity. The hash-ring locality assumption in ELECT would concentrate correlated failures — exactly what ADR-014 prevents.
- **ELECT's access pattern is Zipf-distributed with 81.8% of L4 SSTables never accessed during a workload** ≠ **Vyomanaut's access pattern is uniform and microservice-driven: each chunk is audit-challenged approximately once per 24h regardless of owner behaviour**
→ ELECT's hotness-aware encoding exploits the natural skew in user-driven access. Vyomanaut's audit challenges are issued by the microservice on a randomised schedule (ADR-014 Defence 3), deliberately producing near-uniform access across all stored chunks. ELECT's hotness metric (access frequency + lifetime) would classify all Vyomanaut chunks as equally cold, making the tiering distinction meaningless at the provider level.
- **ELECT uses 4 MiB SSTables as the EC chunk granularity** ≠ **our lf=256 KB (ADR-003)**
→ Vyomanaut's 256 KB fragments are 16× smaller than ELECT's encoding unit. This is not a concern — Vyomanaut's encoding granularity was fixed by the lazy repair formula (Giroire, Paper 10) and validated by AONT-RS (Paper 16). ELECT's use of 4 MiB SSTables is a by-product of Cassandra's SSTable sizing.
- **ELECT's degraded reads require cross-node network retrieval of k SSTables (374 s for 30 GiB)** ≠ **Vyomanaut's audit challenge is a single-chunk point read from local vLog, with no cross-provider retrieval during normal auditing**
→ ELECT's degraded read penalty (5.32×) is exactly the scenario Vyomanaut avoids by giving each provider one self-contained shard. A Vyomanaut provider does not perform reconstruction — it reads its local 256 KB shard and returns a hash. Reconstruction only occurs during repair (ADR-004), triggered after a provider departs. ELECT's recovery breakdown (93.3% network, 3.3% decode) directly validates Vyomanaut's Giroire-derived claim that bandwidth is the binding repair resource.

---

### Decisions Influenced

- **ADR-023 [#16 Provider-Side Storage Engine]** `CONFIRMED`
ELECT's empirical finding that 56.2% of SSTables reside in the last LSM-tree level while receiving only 10.2% of reads is the production-scale confirmation that Vyomanaut's storage workload is cold. Vyomanaut chunk storage is write-once (one chunk per provider per file upload) and read only for audit challenges (once per 24h, uniformly distributed). This is a strictly colder access pattern than even ELECT's L4. WiscKey (ADR-023) is the correct design because its write amplification at 256 KB approaches 1.0, and its single random vLog read for each audit challenge completes in ≤15 ms on HDD — well inside the audit deadline. ELECT independently validates that EC for cold storage is operationally sound when the encoding unit matches the natural I/O granularity of the storage engine.
*Because:* ELECT's measurement confirms that the last-level data (which Vyomanaut's entire chunk store resembles) has access frequency close to zero per SSTable, making reconstruction overhead a tail event rather than a routine cost.
- **ADR-018 [#5 Hot/Cold Storage Bands]** `CONFIRMED`
ELECT's core thesis — that hot and cold data require different redundancy schemes even within the same physical node — independently validates Vyomanaut's two-tier network design. Vyomanaut's Cold Storage Band (V2, accepted) stores write-once, rarely-retrieved archive data exactly as ELECT's L4: the encoding overhead is invisible to normal operation because reads are infrequent. The Hot Storage Band (V3, deferred) would mirror ELECT's lower-level replication approach: replicated or low-latency access paths for frequently-retrieved data. ELECT's experimental confirmation that EC for cold data matches replication performance for normal reads (within 3%) while saving 56% storage strengthens the economic case for the V3 Hot band.
*Because:* Figure 5 shows ELECT achieves near-identical throughput to Cassandra across all workloads in normal mode, with storage savings of 56.1%. This is the empirical evidence that a cold-EC / hot-replica split is operationally viable at production scale.
- **ADR-004 [#4 Repair Protocol]** `CONFIRMED`
ELECT's recovery breakdown (Table 2: 374 s for retrieval vs 13.3 s for decode, for 30 GiB) is the clearest empirical statement yet that EC reconstruction bandwidth is the binding resource. This is the exact premise of Vyomanaut's lazy repair model: the 72h departure threshold and r0=8 trigger exist specifically to batch reconstruction events and avoid saturating provider upload bandwidth. ELECT runs on a 3 Gbps intra-cluster network; Vyomanaut providers operate on 100 Mbps home ISP connections. The bandwidth asymmetry is 30×, making reconstruction bandwidth management even more critical in Vyomanaut's environment than in ELECT's.
*Because:* Table 2 shows decode is 3.3% of recovery cost and network retrieval is 93.3%. This is independent empirical confirmation of the Giroire model's foundational assumption.

---

### Disagreements

- **CassandrEAS (Cadambe et al., 2020):** Applies EC inline on the write path for all KV pairs. ELECT's evaluation notes CassandrEAS incurs much higher read and write latencies than baseline Cassandra and high storage overhead for small values.
*Implication for us:* Inline EC during write is the wrong approach for any system where data hotness is unknown at write time. Vyomanaut's client-side AONT-RS transform (ADR-022) is applied before upload — analogous to ELECT's offline approach but at the network level rather than the storage-engine level.
- **EC-Cache (Rashmi et al., 2016):** Applies self-encoding (splitting one large object into k chunks) to achieve load balancing across cache servers, treating EC as a read-performance tool rather than a storage-reduction tool.
*Implication for us:* The encoding granularity choice matters. ELECT's cross-encoding (one SSTable = one chunk) and Vyomanaut's RS coding (one 256 KB fragment = one chunk) both use cross-encoding. Self-encoding would require Vyomanaut to split each 256 KB shard further — adding metadata overhead with no benefit at our value size.

---

### Open Questions

See open-questions.md — questions Q34-1 and Q34-2.