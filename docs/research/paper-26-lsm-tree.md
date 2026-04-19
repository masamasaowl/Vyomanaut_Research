# Paper 26 — The Log-Structured Merge-Tree (LSM-Tree)

**Authors:** Patrick O'Neil, Edward Cheng, Dieter Gawlick, Elizabeth O'Neil — UMass/Boston, DEC, Oracle
**Venue / Year:** Acta Informatica | 1996
**Topics:** #16
**ADRs produced:** none — ADR-023 gains quantitative constraints; decision deferred to WiscKey (Paper 27)

---

## Problem Solved

B-trees pay two random I/Os per index insert on large tables: one to read the target leaf page and one to write it back. Because leaf pages are referenced too infrequently to justify staying in buffer (the Five Minute Rule), each insert hits disk independently with no batching. For a 1000-TPS system this doubles disk arm cost. The LSM-Tree solves this by staging all inserts first into a memory-resident component C0 and later merging them into disk-resident components C1…CK in large sequential multi-page blocks, amortising I/O cost across many entries per merge step. For Vyomanaut, this paper establishes the theoretical cost model for evaluating whether an LSM-based chunk index on provider desktops is justified, and derives the exact conditions under which the write-batching advantage applies.

---

## Key Findings

**Core architecture: rolling merge between components (Section 2):**
An LSM-tree has a memory-resident C0 (AVL-tree or 2-3 tree, any node size) and one or more disk-resident components C1…CK in a B-tree-like structure with 100%-full nodes packed into multi-page blocks. All inserts go to C0. When C0 reaches threshold size, a rolling merge cursor circulates through C0 and C1 sequentially, writing merged output to new disk locations in large multi-page blocks. Old blocks are never overwritten — they are invalidated only after the merge succeeds, giving crash recovery for free.

**Two batching effects produce the cost advantage (Section 3.2):**

The B-tree insert cost is `COSTB-ins = COSTP × (De + 1)` where De ≈ 2 is effective tree depth and COSTP is cost of a random page I/O. The LSM insert cost amortised over M entries is `COSTLSM-ins = 2·COSTπ / M`. The ratio is:

`COSTLSM-ins / COSTB-ins = K1 × (COSTπ/COSTP) × (1/M)`

where K1 ≈ 0.67. The two factors: (1) sequential multi-page block I/O vs. random page I/O, COSTπ/COSTP ≈ 1/10; (2) M entries batched per leaf page during merge, where `M = (Sp/Se) × (S0/(S0+S1))`. Together these give up to two orders of magnitude cost improvement for high-insert-rate workloads.

**Optimal component sizing — geometric progression (Theorem 3.1):**
For K+1 components with fixed largest component SK and memory component S0, total I/O rate is minimised when all size ratios ri = Si/Si-1 are equal to a common value r. This is the key insight: intermediate components should be sized in geometric progression between the smallest and largest. In practice three components is the maximum useful number — additional components reduce memory cost marginally while adding CPU and buffer overhead.

**The Five Minute Rule — when LSM applies (Section 3.1):**
An index page referenced less than once every τ ≈ 60 seconds (at 1995 costs) should not be buffer-resident. The LSM advantage only materialises for warm data — indexes large enough that leaf pages are accessed infrequently enough to miss buffer. For small indexes that fit in buffer, B-trees are already optimal. The LSM is definitively the right choice when `H.COSTP >> S.COSTd`, i.e. when disk arm cost dominates disk media cost.

**LSM read penalty (Section 2.2):**
Every exact-match find must search all K+1 components. For K=1 this is one extra disk read over a B-tree. For K=2, potentially two extra reads. The paper suggests a Bloom filter approach (referencing the Differential File paper) to eliminate disk accesses for absent keys, which directly reduces the find penalty.

**The critical condition for LSM superiority:**
If M < K1 × COSTπ/COSTP, the LSM offers no advantage over B-trees — this happens when C0 is too small relative to C1, or when entries are too large to batch many per page. The condition is: LSM wins when the index is large, inserts are frequent relative to reads, and entries are small.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Memory-resident C0 with guaranteed residence | Probabilistic buffer residence | Insert I/O cost is zero until merge; no degradation under memory pressure unlike a buffered B-tree |
| Sequential multi-page block I/O for merges | Random page I/O | COSTπ/COSTP ≈ 1/10 savings; requires large contiguous free disk regions |
| 100%-full nodes in C1 | Normal B-tree fill factor (~70%) | ~30% more storage-efficient; nodes can never be in-place updated — always written to new locations |
| Rolling merge to new disk locations | In-place update | Natural crash recovery (old blocks intact until new merge succeeds); disk space recycling requires bookkeeping |
| Deferred placement in final position | Immediate placement (B-tree) | Eliminates the two random I/Os per insert; finds now require checking all components |
| Geometric component sizing | Arbitrary sizing | Minimises total merge I/O rate; intermediate components must be resized as total data grows |

---

## Breaks in Our Case

- **LSM was designed for small index entries (16 bytes) in a high-frequency insert workload (1000 entries/sec)** ≠ **Vyomanaut's storage engine manages 256 KB chunk values at a low insert rate (providers store chunks infrequently, on file upload events)**
→ The batching parameter M = (Sp/Se) × (S0/(S0+S1)) assumes entries are small relative to page size. For 256 KB chunks used as both index key and value, Sp/Se ≈ 1 — there is no batching advantage from fitting multiple entries per page. The LSM's write-amplification advantage vanishes entirely. LSM is the wrong design for the chunk value store itself. However, a small index mapping chunk_id → disk offset (16-byte entries) would fit the LSM model well.
- **LSM assumes a steady high insert rate with infrequent exact-match finds** ≠ **cold storage has write-once access (one insert per chunk, ever) followed by periodic audit reads (one read per challenge, every 24h)**
→ With one insert per chunk and a 256 KB chunk size, the insert rate R in bytes/sec is low. The disk arm cost for a B-tree index on chunk_ids will almost certainly be within the cold or warm data regime, not the hot regime that justifies LSM. The Five Minute Rule test: if the chunk index is accessed for a new insert less than once every 60 seconds per leaf page, B-tree may already be adequate. The calculation depends on the number of chunks per provider.
- **LSM's rolling merge continuously rewrites data to new disk locations** ≠ **ADR-023's non-contiguous chunk placement requirement (from Paper 25/IRON)**
→ LSM naturally writes merged output to new contiguous disk locations — this is at tension with the requirement to spread chunk placement across non-adjacent disk regions to reduce correlated media failure risk. A flat object store (one file per chunk at a random disk offset) satisfies the non-contiguous placement requirement directly. This is the core trade-off ADR-023 must resolve.
- **LSM read performance degrades with number of components** ≠ **audit challenge response deadline of (chunk_size / declared_upload_speed) × 1.5**
→ For a 256 KB chunk at 5 Mbps, the deadline is ~614 ms. A two-component LSM requires checking C0 (memory) then C1 (one disk read). For the chunk index this is fast — the index entry (16 bytes) is found quickly, then the 256 KB chunk value is read from its disk location. The extra index component lookup adds ~10 ms on HDD, well within the deadline.
- **Paper's cost model uses 1995 hardware prices (COSTP = $25/IO/s, COSTm = $100/MB)** ≠ **2025 Indian desktop hardware where SSDs have transformed random vs. sequential I/O cost ratios**
→ On modern NVMe SSDs, COSTπ/COSTP approaches 1 (sequential and random I/O performance converge). The multi-page block batching advantage that makes LSM compelling on HDDs largely disappears on SSDs. For Indian home desktop providers, storage is a mix of HDDs (where LSM still helps) and SSDs (where it does not). WiscKey (Paper 27) directly addresses this SSD case.

---

## Decisions Influenced

- **ADR-023 [#16 Provider Storage Engine]** `NEW CONSTRAINTS`
The LSM-Tree provides the theoretical basis for three concrete constraints that ADR-023 must satisfy:
(1) **LSM is the correct design for the chunk index (chunk_id → disk offset), not for chunk value storage.** Index entries for chunk_id lookup are 16–32 bytes; this is exactly the small-entry regime where LSM batching produces a 10–100× write cost reduction over B-trees. The chunk index is write-once-per-chunk and read on every audit challenge — a reasonable fit for an LSM with bloom filters eliminating disk touches for negative lookups.
(2) **Chunk value storage is a different problem.** 256 KB chunks are values, not index entries. The correct structure for value storage is a flat append-only log or object store, not an LSM. This is the WiscKey insight (Paper 27): separate the key index (LSM) from the value log (flat append-only file).
(3) **Bloom filters are mandatory for the chunk index.** With K=2 components, a find that misses all components incurs 2 disk reads before returning "not found." A per-component Bloom filter reduces this to a single memory check for absent keys, making the lookup overhead negligible for the audit challenge path.
*Because:* The batching parameter M = (Sp/Se) × (S0/(S0+S1)) is proportional to Sp/Se — entries per page. At Se=256 KB and Sp=4 KB, M < 1 and LSM provides no write advantage.
- **ADR-002 [#2 Proof of Storage]** `CONFIRMED`
The PoR challenge reads exactly one 256 KB chunk at a known offset from the provider's disk. The challenge lookup path is: (1) check Bloom filter for chunk_id in each component → (2) if present, read index entry from C1 to get disk offset → (3) read 256 KB chunk from value log. Total disk reads: 1 index entry read + 1 chunk value read = 2 reads, well within the audit deadline.
*Because:* LSM with Bloom filters reduces lookup to O(1) disk reads for present keys, satisfying the response latency requirement.

---

## Disagreements

- **B-tree advocates (Comer 1979, standard database literature):** B-trees are optimal for mixed read/write workloads. The paper concedes this — LSM degrades find performance by 1-2 extra reads per component. For Vyomanaut's read-heavy audit workload (one write per chunk ever, one read per challenge per day), the B-tree's read advantage is relevant. WiscKey's empirical data on 256 KB value sizes (Paper 27) is the deciding measurement.
- **Log-Structured File System (Rosenblum & Ousterhout, 1992):** LSM takes its name and the idea of writing to new disk locations from LFS. The key difference is scope: LFS restructures the entire file system; LSM applies the principle only to the index, leaving data placement separate. Vyomanaut's chunk value store follows the LFS model more closely than the LSM model — chunks are written once, never updated.

---

## Open Questions

See open-questions.md — questions Q26-1 and Q26-2.