# Paper 19 — A Survey of the Past, Present, and Future of Erasure Coding for Storage Systems

**Authors:** Zhirong Shen, Yuhui Cai, Keyun Cheng, Patrick P. C. Lee, Xiaolu Li, Yuchong Hu, Jiwu Shu — Xiamen University, CUHK, HUST, Tsinghua University
**Venue / Year:** ACM Transactions on Storage, Vol. 21, No. 1, Article 4 | January 2025
**Citations:** 285 articles surveyed (2002–2023)
**Topics:** #3, #4, #17
**ADRs produced:** [ADR-026 — V3 Repair Bandwidth Optimisation: Clay (MSR) and Hitchhiker Candidates](../decisions/ADR-026-repair-bw-optimisation.md)

---

## Problem Solved

Erasure coding has a three-way tension between storage efficiency, repair performance, and fault
tolerance that has no single optimal solution — every code construction trades one for another.
Practitioners deploying erasure coding in production have chosen different points on this curve
without a unified framework for comparing them. This survey maps the full landscape of erasure
code constructions (RS, regenerating codes, LRCs, SD codes, piggybacking codes), algorithmic
techniques (repair parallelisation, lazy repair, update efficiency), and emerging architectures
(flash, disaggregated memory, programmable switches, edge-cloud) that the field has developed
from 1988 through 2023. For Vyomanaut V2, the paper's principal value is threefold: it confirms
that systematic Reed-Solomon is the correct production baseline, it identifies Clay codes and
Hitchhiker codes as the V3 repair-bandwidth reduction candidates, and it reveals that our n=56
stripe configuration is "wide-stripe" territory — studied separately and with distinct design
considerations from the n ≤ 20 configurations used in every surveyed production system.

---

## Key Findings

**Table 1 — Production deployment parameters:**
Every surveyed production system (Google Colossus, Facebook f4, Microsoft Azure, Yahoo, Backblaze,
Baidu) uses n between 9 and 20. The maximum n in any production deployment is 20 (Backblaze
Vaults, n=20, k=17). Our (n=56, k=16) is the widest stripe of any system in this survey by a
factor of ~3. The survey explicitly notes that n is "typically upper-bounded to limit the repair
penalty."

**RS codes — why they dominate (Section 2.1):**
RS codes are Maximum Distance Separable (MDS): any k of n chunks suffice for reconstruction
with minimum storage overhead n/k. They support arbitrary (n, k) parameters, are available in
systematic form (first k chunks are unmodified data), and their arithmetic is over GF(2^w) with
w=8 (byte-level, standard in all libraries). No other code family achieves MDS with this
generality. All production systems except Azure use RS codes.

**Repair bandwidth tradeoff (Figure 8):**
For fixed fault tolerance (three or four failures) and fixed storage overhead, the repair bandwidth
hierarchy is: RS codes (highest) > Azure-LRC and Hitchhiker codes (similar, moderate) > Clay codes
(lowest). Clay codes reach the Pareto frontier of repair bandwidth vs storage overhead, but at
the cost of high sub-packetization (each chunk split into many small sub-chunks, causing
non-sequential I/O).

**Regenerating codes — MSR and MBR (Section 3.2):**
Dimakis et al. (2010) proved the information-theoretic lower bound on repair bandwidth for
single-chunk failures. Two extremes: MSR codes minimise repair bandwidth subject to MDS
storage overhead; MBR codes minimise repair bandwidth subject to some storage overhead penalty.
For single-chunk failures (>98% of all failure events in a Facebook warehouse cluster), MSR codes
reduce repair bandwidth of RS codes by up to 50% at n-k=2. Clay codes are the only implemented
MSR construction supporting general (n, k); all others (Butterfly, FMSR, HashTag) require n-k=2
or n ≥ 2k-1.

**Piggybacking codes — Hitchhiker (Section 3.5):**
Hitchhiker codes couple information from two sub-stripes of RS codes to reduce repair bandwidth
by 25–45% while remaining MDS (no storage overhead penalty). Sub-packetization is limited to
two sub-stripes per stripe, so I/O remains sequential. Implemented on Hadoop HDFS.

**LRCs — local repair (Section 3.3):**
Azure LRC (k, l, g) adds local parity chunks within groups of k/l data chunks. Repairing any
failed data or local-parity chunk requires only k/l chunks (not k). This reduces repair I/Os
by a factor of l. LRCs are non-MDS — they store more redundancy than RS for the same fault
tolerance. ECWide extends LRCs to wide stripes (large k, small n-k) with combined parity
locality and rack-level topology locality.

**Lazy repair (Section 4.2):**
Silberstein et al. (SYSTOR 2014) show that lazy repair — deferring repair until a threshold number
of chunks are unavailable — significantly reduces the amount of repair bandwidth. TotalRecall
(NSDI 2004) applies lazy repair in peer-to-peer networks. The survey lists lazy repair as an
established technique for reducing peak bandwidth in distributed storage.

**Reliability evaluation (Section 4.6):**
Markov modelling assumes failure and repair times are independently and exponentially distributed.
Ford et al. (OSDI 2010) show erasure coding has higher MTTDL than replication. Weatherspoon &
Kubiatowicz (IPTPS 2002) quantify the advantage: same reliability as 3-way replication at n/k
storage overhead. The survey also notes Greenan's critique (HotStorage 2010) that MTTDL as a
metric is misleading without careful modelling.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| RS codes (MDS, systematic) | Replication (3×) | Storage overhead n/k vs 3×; repair bandwidth amplified by k; universally deployed |
| Systematic form (first k chunks = data) | Non-systematic Rabin IDA | Reads in normal operation require no decoding; AONT-RS (ADR-022) exploits this property |
| Clay codes (MSR, general n,k) | Butterfly / FMSR (n-k=2 only) | Clay is the only general-parameter MSR implementation; high sub-packetization is the cost |
| Hitchhiker piggybacking (25–45% BW reduction) | Full MSR (Clay) | Lower sub-packetization cost; simpler I/O; gives up some BW reduction vs Clay |
| LRC local repair | RS global repair | Repair I/O reduced by factor l; non-MDS (extra storage); rack-topology assumption breaks in P2P |
| Lazy repair (threshold-triggered) | Eager repair (immediate) | Significant BW saving confirmed by Silberstein et al.; Giroire's formula already applied in ADR-003 |
| Wide-stripe RS (n=56, ECWide-class) | Standard-stripe RS (n ≤ 20) | Maximum production n is 20; our n=56 is 3× larger; ECWide studied combined locality for this regime |
| MDS property (RS) | Non-MDS (LRC, MBR) | Storage-optimal; repair bandwidth is higher than non-MDS codes; correct for a consumer network where storage space is the scarce resource |

---

## Breaks in Our Case

- **All production deployments use n ≤ 20 (Table 1, survey finding)** ≠ **our n=56 (s=16, r=40)**
  → Our stripe is 3× wider than any surveyed production system. We are in wide-stripe territory,
    studied separately under ECWide (Section 3.3). The repair penalty (contact k=16 nodes) is
    the same as a (n=20, k=16) deployment but the provider diversity benefit is larger — 56
    distinct providers per segment reduces correlated failure risk. Whether the wide stripe's
    repair I/O profile creates observable degradation on desktop providers must be benchmarked.

- **MSR codes (Butterfly, FMSR, HashTag) restrict n-k=2 or n ≥ 2k-1** ≠ **our n-k=40**
  → None of the production-deployed MSR codes support our parameter space. Clay codes are the
    only general-parameter MSR implementation; this is the V3 repair BW candidate. Reading
    Dimakis (Phase 2A #4) before evaluating Clay codes is mandatory.

- **LRCs assume rack-based topology for local repair groups** ≠ **our P2P provider model with no
  concept of physical rack**
  → LRC's local repair benefit (k/l nodes instead of k) requires that all nodes in a local group
    are simultaneously reachable and fast. In a P2P desktop network with independent providers
    behind NAT, group co-locality cannot be guaranteed. LRCs are not a V2 candidate.

- **Survey's Markov reliability model assumes independent exponential failure rates** ≠ **our
  correlated failure model (ADR-014, Honest Geppetto attack)**
  → The MTTDL formulas from the survey's Section 4.6 apply under independent failure assumption.
    Correlated failures (same ASN, same ISP outage) are not captured. Our 20% ASN cap
    (ADR-014) is the practical mitigation; the Markov model gives a lower bound on MTTDL
    under the independence assumption.

- **RS-Paxos trades liveness for storage in consensus protocols (Section 4.4)** ≠ **our
  microservice metadata store uses (3,2,2) quorum with full replication (ADR-025)**
  → RS-Paxos is not applicable. Our audit receipts and chunk location records are small
    metadata, not the bulk data RS-Paxos targets. The (3,2,2) quorum with full replication
    (ADR-025) is the correct design for our metadata store size and throughput.

---

## Decisions Influenced

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `CONFIRMED`
  Systematic Reed-Solomon is the universally correct baseline. Every production system uses RS
  or a derivative. The systematic property (which AONT-RS relies on — ADR-022) eliminates
  encoding overhead for the first k chunks and improves partial-recovery decode performance.
  Parameters s=16, r=40, r0=8, lf=256 KB remain unchanged.
  *Because:* Survey Table 1 shows RS is the only MDS code family deployed in production at
  scale; no surveyed system has moved to MSR codes in production. Clay codes are implemented
  on Ceph and Hadoop HDFS in research settings only.

- **[ADR-004](../decisions/ADR-004-repair-protocol.md) [#4 Repair Protocol]** `CONFIRMED`
  Lazy repair is validated as the standard approach for reducing peak bandwidth in distributed
  storage. The survey's Section 4.2 explicitly cites Silberstein et al. (SYSTOR 2014) as
  showing that deferring repair until a threshold is crossed "significantly reduces the amount
  of repair bandwidth." TotalRecall (NSDI 2004) applies this pattern in peer-to-peer networks.
  The 72-hour trigger threshold and r0=8 remain correct.
  *Because:* The survey confirms lazy repair is a well-understood technique with production
  precedent, not a novel research idea. Our Giroire-derived parameters already optimise the
  lazy repair threshold.

- **[ADR-026](../decisions/ADR-026-repair-bw-optimisation.md) [#17 Repair Bandwidth Optimisation]** `PROPOSED — CANDIDATES IDENTIFIED`
  Two V3 candidates are now identified from the survey's tradeoff analysis (Figure 8):
  (1) **Clay codes** — state-of-the-art MSR, supports general (n,k), access-optimal (minimises
  both repair bandwidth and I/Os); up to 50% repair BW reduction over RS. High sub-packetization
  overhead is the cost: each chunk is split into many sub-chunks, causing non-sequential I/O
  on desktop disks. (2) **Hitchhiker codes** — piggybacking construction over RS, 25–45% BW
  reduction, MDS, limited sub-packetization (two sub-stripes). Simpler to implement than Clay.
  ADR-026 stays Proposed pending Dimakis (Phase 2A #4) which provides the information-theoretic
  bounds needed to evaluate whether the BW reduction justifies the sub-packetization cost at
  our (n=56, k=16) parameters.
  *Because:* Survey Figure 8 quantifies the tradeoff. Clay is Pareto-optimal but complex;
  Hitchhiker gives a simpler partial win. Neither has been deployed in a consumer P2P network.

---

## Disagreements

- **Wide-stripe LRCs (ECWide, Kadekodi et al.):** Wide-stripe RS with combined locality (parity
  locality + rack topology locality) is deployed at Google for k ≈ 100. The paper argues wide
  stripes should use LRCs for repair efficiency.
  *Implication for us:* LRC's local repair requires group co-locality that P2P cannot guarantee.
  Wide-stripe RS without LRC-style locality is our correct design, accepting the higher repair
  BW as the cost of provider diversity.

- **Greenan (HotStorage 2010):** "Mean Time to Meaningless" — argues MTTDL from Markov models
  is a misleading metric without accounting for the actual amount of data at risk.
  *Implication for us:* Our LossRate < 10^-15 per year target (ADR-003, Giroire formula) uses
  Markov modelling. Greenan's critique applies: the formula gives a lower bound under
  independence; correlated failures (same ASN group) can produce much higher LossRate. The 20%
  ASN cap (ADR-014) is our practical hedge against this.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q19-1 through Q19-3.
