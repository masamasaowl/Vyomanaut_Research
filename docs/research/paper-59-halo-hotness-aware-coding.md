# Paper 59 — HALO: A Scalable Framework for Hotness-Aware Coding and Transformation-Efficient Placement

**Authors:** Junmei Chen, Ne Wang, Zongpeng Li, Zhiquan Liu, Dan Xiang (Guangzhou Maritime University / Wuhan University / Tsinghua University / Jinan University)
**Venue / Year:** Future Generation Computer Systems 183 (2026) 108562
**Citations:** not yet indexed (published April 2026)
**Topics:** #5 Peer Selection (storage type), #3 Erasure Coding
**ADRs produced:** ADR-018 (revised)

---

## Problem Solved

Static erasure-coded redundancy configurations are tuned for average-case access patterns, but real storage workloads are Zipfian and non-stationary — the top 5% of blocks in the MSR trace serve over 75% of accesses, and which blocks are hot shifts within minutes (Fig. 2). A system that fixes redundancy per file once, at write time, either over-protects cold data (wasting storage) or under-protects data that later turns hot (slow repair when it matters). HALO is a middleware control loop — hotness predictor, non-uniform LRC template engine, graph-guided placement — that keeps redundancy and layout aligned with *current* access patterns without requiring full re-encoding on every shift. For Vyomanaut, this is the first paper found that gives a working mechanism for ADR-018's still-open Hot-band question, rather than just motivating the need for one.

---

## Key Findings

- Two-dimensional hotness classifier: `W(bi) = writes in Tw / Tw`, `R(bi) = t_current − t_last_read`. Hot = high write intensity OR recent read; Cold = both low. Runs online, O(1) metadata per block, no offline profiling or ML training required.
- Three-tier NU-LRC templates (Table 3): Hot `(k=6, r=2, g=1)`, Warm `(k=8, r=2, g=2)`, Cold `(k=12, r=3, g=3)`. Smaller stripe width `k` on hot data directly reduces the number of shards a reconstruction must read.
- vs. seven baselines (static RS, U-LRC, Static NU-LRC, Dynamic NU-LRC, Dynamic Replication, AERS, Oracle-ideal) on MSR + Alibaba traces: up to 50% lower repair latency, 2.3× faster repair than Static NU-LRC, 27% higher repair throughput, 74% lower cross-rack repair traffic vs. plain RS, over 50% lower migration overhead than DynRep/AERS, 91% hotness-prediction accuracy at 5.2 ms inference cost.
- Repair fan-in at `n=9, k=6`: HALO touches 2.0 failure domains per repair vs. 6.0 for plain RS/AERS (Fig. 14) — the mechanism is smaller `k`, not a smarter repair algorithm.
- Reconfiguration is throttled and executed as background updates so redundancy switching never blocks foreground I/O — the same "stay off the critical path" principle Vyomanaut already applies via its ≤5% CPU/bandwidth provider budget (ADR-009).
- HALO explicitly does not propose a new erasure code — it is a control loop that selects among pre-generated NU-LRC templates (Fan et al., ACM ICPP 2024) and switches via constant-time lookup, avoiding coding-matrix regeneration on every transition.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Fixed, pre-generated per-tier templates (Hot/Warm/Cold) selected by lookup | Per-file dynamically-solved parameters | Fast, constant-time switching; loses the theoretically-optimal fit an online solver (like Zebra's) would give per file |
| Smaller `k` for hot tiers | Fixed `k`, tier differentiated only by parity count | Genuine reconstruction-latency win; breaks RS-monotonic "add pieces only" migration between tiers |
| Two-dimensional heuristic predictor (write intensity + read recency) | A trained ML admission/eviction model | Near-zero inference cost and no training data requirement; caps prediction accuracy at 91% instead of chasing marginal gains |
| Graph-guided rack placement to bound migration cost | Free redundancy switching with no placement constraint | Empirically over 80% of single-block repairs stay within one stripe; requires a controllable placement layer (rack assignment) the paper's authors control |

---

## Breaks in Our Case

- **HALO assumes a NameNode/DataNode/rack topology it can place blocks onto (Algorithms 3–4, Fig. 9)** ≠ **Vyomanaut has no controllable placement layer — providers self-select and join via vetting, the coordination microservice cannot dictate which physical machine holds a shard**
  → The graph-guided placement optimizer (§4.4) does not transfer. Only the predictor (§4.2) and template-switching mechanism (§4.3) are portable to Vyomanaut.

- **HALO's redundancy tier is assigned automatically and continuously by the system, re-classified as hotness drifts (Fig. 5, "closed-loop control")** ≠ **ADR-018's existing enrolment flow has the data owner declare Hot or Cold once, at upload time — there is no runtime re-classification mechanism in Vyomanaut today**
  → HALO's predictor (§4.2) is not directly deployable without first deciding whether Vyomanaut wants automatic dynamic re-tiering at all. This is a real open architectural question — see Q59-1 below. Until resolved, only HALO's static template *structure* (fixed per-tier `(k,r,l,g)` config, looked up not computed per request) is usable.

- **HALO's baseline stripe widths (`n=9,k=6` in the fan-in experiment; templates up to `k=12`) are tuned for HDFS block sizes and datacenter repair topologies** ≠ **Vyomanaut's Cold band already runs RS(16,56) — larger `k`, different scale entirely**
  → HALO's exact `(k,r,l,g)` numbers cannot be copied verbatim. Only the *shape* of the solution (hot tier gets smaller `k` for lower fan-in, cold tier gets larger `k` for storage efficiency) transfers; the actual numbers must be re-derived for Vyomanaut's stripe size and target SLA (see ADR-018 open constraints).

---

## Decisions Influenced

- **[ADR-018](../decisions/ADR-018-hot-cold-storage-bands.md) [#5 Peer Selection]** `REVISED`
  Confirms the mechanism ADR-018 needs for Hot band (smaller stripe width `k`, not just added parity) and supplies the per-tier template pattern (fixed, cached `(k,r,l,g)` configs switched by lookup) as the deployable shape for Vyomanaut's static, upload-time-declared band model. Does not supply Vyomanaut-specific numeric parameters — see ADR-018's updated open constraints.
  *Because:* HALO's own ablation study (E8) shows the hotness predictor and redundancy adapter are each independently load-bearing — removing either sharply degrades responsiveness or increases storage/repair cost — confirming this isn't a single trick but a real methodology worth adopting in part.

---

## Disagreements

- **HALO vs. ADR-018's original theoretical basis (RS monotonicity, Paper 05 §6.1):** ADR-018 previously assumed Hot-band scaling means adding parity on top of Cold band's same `k=16` — cheap migration, but does not reduce reconstruction cost. HALO's own baselines (and Zebra's, Paper 60) show the actual latency win comes from a *smaller* `k` for hot tiers, which breaks that monotonic-subset property.
  *Implication for us:* ADR-018 cannot deliver both "cheap migration" and "genuinely faster Hot-band retrieval" with the same mechanism. This is now an explicit fork in ADR-018 rather than an implicit assumption — see its Decision section.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q59-1.
