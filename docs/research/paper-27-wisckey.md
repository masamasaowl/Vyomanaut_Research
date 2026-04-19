# Paper 27 — WiscKey: Separating Keys from Values in SSD-Conscious Storage

**Authors:** Lanyue Lu, Thanumalayan Sankaranarayana Pillai, Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau — University of Wisconsin–Madison
**Venue / Year:** USENIX FAST 2016
**Topics:** #16
**ADRs produced:** [ADR-023 — Provider-Side Storage Engine: WiscKey Key-Value Separation](../decisions/ADR-023-provider-storage-engine.md)

---

## Problem Solved

Standard LSM-tree implementations (LevelDB, RocksDB) couple key sorting with value storage, forcing compaction to read, sort, and rewrite both keys and values together. At 256 KB value sizes this produces write amplification of 10–14×, consuming the storage device's write bandwidth for background compaction that competes with foreground workloads. The paper demonstrates that compaction needs to sort only keys — values can be stored separately in an append-only log — reducing write amplification to approximately 1.14 for 16B keys and 1 KB values, and to effectively 1.0 at 256 KB. For Vyomanaut, this paper closes ADR-023: the WiscKey architecture (LSM for chunk index, append-only value log for 256 KB chunk data) is the correct storage engine for a write-once cold storage workload at our specific value size, working well on both SSD and HDD provider hardware.

---

## Key Findings

**Write amplification at 256 KB values (Figure 10):**
WiscKey write amplification reaches approximately 1.0 at 256 KB values. LevelDB at the same size is still at ~3×. The formula: for a 16B key and 256 KB value with key-tree write amplification of 10, effective WiscKey amplification is `(10 × 16 + 262144) / (16 + 262144) ≈ 1.0006`. This is the theoretical optimum.

**Random read performance at 256 KB on SSD (Figure 3):**
Single-thread random reads at 256 KB achieve ~250 MB/s on a Samsung 840 EVO SSD, roughly half of sequential throughput. With 32 concurrent threads, random read throughput matches sequential at request sizes above 16 KB. For our audit challenge — one 256 KB chunk read per challenge — a single-thread read at 250 MB/s completes in approximately 1 ms, well within the 614 ms audit deadline for a 5 Mbps provider.

**Random lookup performance (Figure 11):**
WiscKey is 1.6×–14× faster than LevelDB across all value sizes. At 256 KB, WiscKey random lookup throughput is device-bandwidth-limited because the small LSM index fits in cache — the only disk operation is the single value-log read for the 256 KB chunk.

**Write throughput at 256 KB values (Figure 9):**
WiscKey reaches peak device write throughput (~400 MB/s on the test SSD) for values above 4 KB. LevelDB plateaus at ~4 MB/s at all value sizes because compaction continuously consumes write bandwidth and throttles foreground writes.

**Garbage collection overhead (Figure 13):**
With 100% invalid data in the value log, GC reduces foreground throughput by only ~10%. For mixed workloads with 25–75% free space, the drop is approximately 35%. For a write-once cold storage workload where chunks are deleted only on provider exit or file owner action, GC is a maintenance operation rather than a continuous background cost.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| LSM index (keys only) + append-only vLog (values) | Standard LSM (keys + values together) | Write amplification ≈ 1 at 256 KB; audit lookup = one index lookup + one vLog random read; GC required on chunk deletion |
| Sequential vLog appends for writes | Random per-chunk disk placement | HDD-optimal write path; chunks from different files are interleaved by upload time, not grouped |
| RocksDB Bloom filters for negative lookup | Full LSM scan on absent key | Absent-chunk audit challenges fail at the memory check with no disk I/O |
| Online GC with head/tail pointers in LSM | Offline compaction-based GC | Low-overhead background reclamation proportional to deletion rate; negligible for cold storage |
| Append-only vLog with crash recovery via tail scan | Journal-based crash recovery | No garbage data after crash (file system append property); recovery scans only the unconfirmed tail |

---

## Breaks in Our Case

- **WiscKey assumes mixed read/write/delete workloads with frequent updates** ≠ **our chunks are write-once — a 256 KB chunk is stored once and never overwritten in place**
  → GC is triggered only on explicit chunk deletion. A provider holding 50 GB at 256 KB/chunk stores ~200,000 chunks; deletions occur only on provider exit or file owner action. The vLog grows monotonically during normal operation and GC is a rare maintenance sweep, not a continuous background process. This simplifies the WiscKey design for our use case.

- **WiscKey's vLog stores values sequentially by append time** ≠ **the IRON non-contiguous placement requirement (Paper 25) — spread chunks across non-adjacent disk regions to reduce correlated spatial media failure risk**
  → Sequential vLog layout means chunks appended in the same time window are physically adjacent. However, different files' chunks are interleaved by upload time, not grouped by file. A provider holds one shard from each of many files, not all 56 shards of any one file. The 20% ASN cap (ADR-014) bounds the impact of any one provider's burst disk failure at the network level. Accept sequential vLog layout for V2; defer sparse vLog placement (via fallocate pre-allocation) to V3 pending empirical measurement of within-provider spatial failure rates (Q27-2).

- **WiscKey benchmarked on SSD (Samsung 840 EVO)** ≠ **Indian desktop providers at V2 launch will include a significant fraction of HDD-based machines**
  → On HDD, vLog appends are sequential — HDD-optimal. The only HDD-unfriendly operation is the single random vLog read per audit challenge. At 256 KB and a 10 ms HDD seek, this completes in approximately 12–15 ms — well within the 614 ms audit deadline. WiscKey's design is adequate for both HDD and SSD at our value size. Q26-2 is answered.

- **WiscKey supports range queries with parallel SSD prefetch using 32 background threads** ≠ **audit challenges are point lookups — one chunk_id to one 256 KB read**
  → Range query complexity is irrelevant. The audit path is a pure point lookup: RocksDB for the offset, then one random vLog read. The 32-thread prefetch machinery is not adopted.

---

## Decisions Influenced

- **[ADR-023](../decisions/ADR-023-provider-storage-engine.md) [#16 Provider-Side Storage Engine]** `ACCEPTED`
  WiscKey-style key-value separation is the correct architecture. The provider daemon uses: (1) a RocksDB instance storing `chunk_id → (vlog_offset, chunk_size)` with built-in Bloom filters; (2) an append-only vLog file storing raw 256 KB chunk data plus a per-entry `content_hash = SHA256(chunk_data)` for integrity verification on read. Write amplification ≈ 1 at 256 KB. Audit lookup = one RocksDB point query (index typically cached) + one 256 KB sequential vLog read (~1 ms SSD, ~15 ms HDD). GC triggered only on chunk deletion events.
  *Because:* Figure 10 directly shows write amplification ≈ 1 at 256 KB. Figure 11 shows lookup throughput is device-bandwidth-limited at 256 KB — the LSM index is small enough to stay cached. Our write-once cold storage pattern is WiscKey's optimal regime.

- **Q26-1 [Answered]:** The Five Minute Rule analysis from Paper 26 asked whether the chunk index is in the hot, warm, or cold regime. WiscKey answers this differently: at 256 KB values, write amplification ≈ 1 regardless of the index regime because compaction overhead is dominated by value movement — and with WiscKey, values are never moved during compaction. The access regime of the 44-byte index entries is immaterial.

- **Q26-2 [Answered]:** WiscKey is correct for both SSD and HDD providers. vLog writes are sequential (HDD-optimal). Single random vLog reads at 256 KB complete in ~1 ms on SSD and ~12–15 ms on HDD — both well within the 614 ms audit deadline. The SSD/HDD mix among Indian desktop providers does not change the storage engine design.

---

## Disagreements

- **RocksDB (Facebook):** Presented in the paper as an SSD-optimised LevelDB with multiple compaction threads, but still exhibits write amplification above 10× because it does not separate keys from values. WiscKey outperforms RocksDB in all six YCSB workloads.
  *Implication for us:* The provider daemon should use RocksDB as the LSM implementation (mature, well-maintained, Bloom filters built-in) but with WiscKey-style separation applied at the application level — chunk_id and vLog address in RocksDB; the actual 256 KB data in the separate vLog file.

- **Standard LSM advocates:** For small values (64B), WiscKey performs 12× worse than LevelDB on sequential range queries because the random vLog reads for small values do not utilise SSD internal parallelism. This worst-case does not apply to Vyomanaut — our values are 256 KB and we have no range queries over chunk data.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q27-1 and Q27-2.
