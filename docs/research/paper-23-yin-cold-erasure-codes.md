# Paper 23 — Erasure Codes for Cold Data in Distributed Storage Systems

**Authors:** Chao Yin, Zhiyuan Xu, Wei Li, Tongfang Li, Sihao Yuan, Yan Liu — Jiujiang University / Jiangxi University of Finance and Economics
**Venue / Year:** Applied Sciences (MDPI), Vol. 13, No. 4 | February 2023
**Topics:** #3, #5
**ADRs produced:** none

---

## Problem Solved

Cold data — data written once and rarely or never read again — is routinely stored
with replication (typically 3×) despite the fact that it does not benefit from replication's
read performance advantage. The paper argues that erasure coding is strictly better for
cold data on both storage efficiency and write throughput, and validates this empirically
by comparing NewLib code (a minor optimisation of Liberation code), Liberation code,
RDP code, EVENODD code, and triple replication across block sizes of 4 KB to 1024 KB.
For Vyomanaut the paper's primary value is a single benchmark result: confirming the
block size at which EC write throughput matches or exceeds replication, and validating
that cold data's characteristics (write-once, sequential, rarely read) are exactly the
conditions under which RS-class erasure codes are appropriate. This partially feeds
ADR-018 (hot/cold storage bands) but does not change any existing parameter.

---

## Key Findings

**Write throughput crossover at ≥512 KB (Figure 14):**
At block sizes below 64 KB, triple replication has higher write throughput than any
tested erasure code (310 MB/s vs ~230–260 MB/s for EC codes). At 64 KB the gap
narrows. At 512 KB and above, all EC codes match replication write throughput (~540–580
MB/s for EC vs ~540 MB/s for replication). The crossover is at approximately 512 KB.
At block sizes above 64 KB, triple replication's write penalty (generating 2× more
network bandwidth than data written) dominates, eliminating its throughput advantage.

**Read performance: replication always wins for small blocks (Figures 9–12):**
For all block sizes, triple replication has higher read IOPS than EC codes (~25% gap at
4 KB). This is expected — replication allows local reads without involving other nodes,
while EC requires either local data chunks or k-node reconstruction. The gap closes at
large block sizes (512 KB+) where metadata overhead dominates.

**Liberation code has fewest XOR operations among tested codes:**
The proportion of ones in the Liberation code parity matrix is 16%, vs 28% for RDP and
21% for CRS. XOR operation count scales with ones in the parity matrix. Liberation code
requires k−1 XOR operations per element (Equation 1), which is the theoretical minimum
for RAID-6 class codes and is ~1.5× fewer than EVENODD (Equation 2). This makes
Liberation code the correct choice for write-optimised cold data where encoding speed
is the key constraint.

**NewLib code contribution is marginal:**
The paper's proposed NewLib code aligns data requests after stripping to eliminate
non-aligned read-modify-write overhead. In benchmarks, NewLib code performs
indistinguishably from Liberation code at block sizes ≥ 64 KB (within measurement noise).
The improvement is visible only at 4 KB blocks in write IOPS, where it shows ~5%
improvement over Liberation code. This is not relevant for Vyomanaut's 256 KB
fragment size.

**N-Schedule consistent hashing:**
The paper proposes a consistent hash ring with virtual nodes for load balancing. Each
physical node is abstracted into multiple virtual nodes proportional to storage and
compute capacity. When a physical node fails, only its hash space is redistributed
to adjacent nodes, not the entire ring. This is a well-known technique (Amazon Dynamo)
presented without new insight.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| EC for cold data (write-once, high durability) | Replication for cold data | Storage efficiency 2.5× better; write throughput equal at ≥512 KB blocks; read IOPS lower (acceptable since cold data is rarely read) |
| Liberation code (fewest XOR ops) | EVENODD, RDP, CRS for encoding | 16% ones in parity matrix vs 28% (RDP) and 21% (CRS); ~1.5× faster encoding than EVENODD |
| Large block sizes (≥512 KB) | Small blocks (4–64 KB) for EC cold data | Throughput stable and equal to replication; IOPS lower but irrelevant for cold data |

---

## Breaks in Our Case

- **Paper tests RAID-6 style codes (n-k=2 fault tolerance) with small k (k=7, n=9)** ≠ **our
  RS(s=16, r=40, n=56) with 40 parity shards and high fault tolerance**
  → The XOR-based codes (Liberation, EVENODD, RDP) tested are RAID-6 variants — they tolerate
    exactly 2 failures. Our RS(16,56) tolerates 40 simultaneous failures. The paper's code
    comparisons do not apply at our parameters. RS code (which uses Galois Field arithmetic,
    not XOR-only) is the only feasible code at n-k=40.

- **Paper's write crossover is at 512 KB block size** ≠ **our fragment size is 256 KB (lf=256 KB,
  ADR-003)**
  → Our fragment size sits just below the crossover. At 256 KB, EC write throughput is still
    slightly below replication write throughput (reading the trend between 64 KB and 512 KB
    in Figure 14). This is acceptable for cold storage since write happens once and providers
    store the data for months; the transient write performance penalty is insignificant.

- **Paper assumes a centralised master/slave GFS-style architecture** ≠ **our pure P2P provider
  network with microservice coordination**
  → The N-Schedule consistent hashing scheme is designed for a datacenter with collocated
    nodes and reliable high-speed intra-rack networking. In our P2P model, providers are
    geographically distributed with varying latency. Consistent hashing for chunk placement
    is already handled by our Kademlia DHT (ADR-001); N-Schedule adds nothing new.

- **Cold data definition used in this paper (write-once, rarely read) matches Vyomanaut's
  primary use case exactly** = **our Cold Storage Band (ADR-018)**
  → This is a match, not a break. The paper confirms that the cold data model — sequential
    write, high durability requirement, low read frequency — is precisely where erasure coding
    outperforms replication on every dimension that matters (storage efficiency and write
    throughput at large block sizes). ADR-018's Cold Storage Band design is validated.

---

## Decisions Influenced

- **[ADR-018](../decisions/ADR-018-hot-cold-storage-bands.md) [#5 Peer Selection — Hot/Cold Bands]** `CONFIRMED`
  Cold data's access pattern (write-once, sequential, rarely read) is exactly where EC
  provides its strongest advantage over replication: lower storage overhead, equal write
  throughput at block sizes ≥ 512 KB, and no meaningful read performance penalty since cold
  data is read infrequently. Our 256 KB fragment size is slightly below the write throughput
  crossover, but the gap is marginal and irrelevant for a storage system where writes happen
  once at upload time. The Cold Storage Band (V2 only, ADR-018 accepted) is the correct
  design for Vyomanaut's primary use case.
  *Because:* Figure 14 directly shows EC write throughput equals replication at 512 KB and
  above; the cold data argument is validated empirically, not just theoretically.

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `CONFIRMED`
  RS is the correct code for our parameters. The XOR-only RAID-6 codes (Liberation, EVENODD,
  RDP) are not applicable at n-k=40. The paper's encoding speed comparison between
  Liberation and EVENODD is irrelevant for RS — RS uses Galois Field multiplication, not
  XOR-only arithmetic, and is the only MDS code family that supports arbitrary (n,k). The
  fragment size of 256 KB is appropriate for cold storage based on the write throughput
  data — the performance is acceptable even slightly below the crossover point.

---

## Disagreements

- **Paper claims NewLib code improves encoding performance:** The benchmark data shows NewLib
  code performs within noise of Liberation code at all block sizes above 4 KB. The claimed
  improvement is not reproducible from the presented figures at relevant block sizes.
  *Implication for us:* NewLib code is not a candidate for any reason. Liberation code itself
  is not applicable at our (n-k=40) parameters. This disagreement is moot for Vyomanaut.

- **Paper does not cite or compare against the RSC (Cauchy Reed-Solomon) code at larger n-k:**
  The CRS code tested (k=7, 21% ones) is a RAID-6 variant. Standard RS codes used in
  production (Google Colossus, Facebook f4) use larger (n,k) parameters not studied here.
  *Implication for us:* The paper's performance conclusions apply only to n-k=2 RAID-6 codes
  and do not extend to our parameters.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q23-1.
