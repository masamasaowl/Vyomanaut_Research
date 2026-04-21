## Paper 36 — Analysis of Failure Correlation in Peer-to-Peer Storage Systems

**Authors:** Olivier Dalle, Frédéric Giroire, Julian Monteiro, Stéphane Pérennes — MASCOTTE, INRIA / I3S / CNRS / Univ. Nice Sophia
**Venue / Year:** INRIA Research Report RR-6771 | December 2008
**Topics:** #3, #4, #17
**ADRs produced:** none — ADR-003 and ADR-004 gain a critical variance constraint; ADR-014 confirmed as the correct structural mitigation; see Decisions Influenced

---

### Problem Solved

Every prior analytical model of P2P storage systems — including the Markov Chain Model (MCM) that underlies Giroire's lazy repair formulas (Paper 10) — models each data block as failing independently. This correctly predicts average bandwidth consumption and average data loss probability. But disk crashes in real systems cause tens of thousands of blocks to fail *simultaneously*, creating catastrophic variance that independent models cannot capture. The paper proves that provisioning repair bandwidth based on mean + 5σ of the independent model still causes large data loss, because the real standard deviation is 22× higher than the model predicts. For Vyomanaut, this paper directly quantifies the risk that the Giroire-derived BWavg ≈ 39 Kbps/peer understates peak demand, and validates why the 20% ASN cap (ADR-014) is the correct structural mitigation rather than a bandwidth headroom solution.

---

### Key Findings

**The independence assumption correctly predicts the mean but catastrophically underestimates variance (Section 4.3):**
In a 5,000-peer simulation, the independent block model and the correlated model both produce a mean bandwidth of ~5.5 Mbits/s. But the standard deviation is 2.23 Mbits/s in the correlated model versus only 0.10 Mbits/s in the independent model — a 22× gap. The two models are indistinguishable at the mean and completely different in practice.

**Provisioning at mean + 5σ of the independent model causes thousands of lost blocks (Section 4.5):**
Figure 4 shows that limiting bandwidth to the MCM's µ + 5σ (6.04 Mbits/s) causes 196 dead blocks over 400 simulated years. At the MCM mean (5.5 Mbits/s), 4,300 blocks are lost. The independent model's µ + 5σ has a theoretical exceedance probability of 5.8 × 10⁻⁷ — which sounds safe — but the real system exceeds this threshold regularly due to correlated burst failures. Bandwidth provisioning based solely on the independent model is therefore dangerously wrong.

**The correlation effect persists even in very large networks (Section 4.4):**
At 50,000 peers, the correlated standard deviation is still 5× higher than the independent model predicts. The two only converge when the number of peers approaches 3 million — far larger than any realistic Vyomanaut deployment. For any practical V2 scale, correlated burst failures dominate variance.

**The Fluid Model (FM) correctly captures variance (Section 5):**
The paper introduces a Fluid Model that treats the entire system state as a vector of block proportions per redundancy level, updated by random matrix transitions. This model correctly reproduces the mean (matching the MCM) and correctly reproduces the standard deviation (matching simulation within 20–40%). The key mechanism: when a disk fails, all its blocks lose a fragment simultaneously — modelled as a single matrix transition affecting the whole state vector.

**The Refined Fluid Model (RFM) adds disk age heterogeneity (Section 6):**
New disks enter empty after a crash and fill gradually. Old disks hold far more data than young disks, following a geometric distribution of disk ages. When an old disk fails, it causes a proportionally larger burst than when a young disk fails. The RFM models this by drawing the failed disk's fill ratio from the geometric age distribution. The RFM matches simulation within 2–4% on standard deviation across all parameter combinations.

**The number of dead blocks decreases exponentially with the threshold value r0 (Section 3.2):**
From the simplified chain analysis: `#deads ∝ ρ^(r0+1)`, where `ρ = 1 / (1 + γ/g × (1−g))`. The number of reconstructions is approximately proportional to `1 / (r − r0)`. These are the same formulas underlying ADR-003's parameter derivation from Paper 10.

---

### Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Fluid Model (whole-system state vector) | Per-block Markov Chain | Captures correlated variance correctly; requires solving an n² × n² linear system for covariances |
| Refined Fluid Model (stochastic disk fill ratio) | Basic Fluid Model (uniform disk fill assumption) | Reduces standard deviation error from 20–40% to 2–4%; adds geometric distribution sampling per failure event |
| Bandwidth provisioning at real σ (correlated model) | Provisioning at independent model µ + 5σ | Prevents data loss; requires 3–5× more headroom than the independent model implies |
| Shuffling algorithm (periodic re-randomisation of fragments) | Biased reconstruction (preferential routing to younger disks) | Shuffling lowers correlation and reduces variance at the cost of extra network traffic; biased reconstruction is cheaper but creates new within-cohort correlation |

---

### Breaks in Our Case

- **Paper models backup-style storage where all peers are always online (no churn)** ≠ **Vyomanaut providers have MTTF 180–380 days with daily and weekend absence patterns**
→ The paper's model assumes disk failures are the only disruption. Vyomanaut's lazy repair threshold (r0=8, ADR-004) is calibrated for departure events, not just disk crashes. The correlation effect in this paper is actually stronger in Vyomanaut because a provider departure causes all its chunks to become simultaneously unavailable — exactly the burst event the paper models. The correlation risk is therefore at least as severe as the paper describes, not less.
- **Paper uses s=9, r=6, r0=3 (small parameters) for simulation tractability** ≠ **our RS(s=16, r=40, r0=8) wide-stripe configuration**
→ Our parameters are wider (more fragments per block, more parity). The number of blocks simultaneously affected by one disk crash is `(s+r)×B/N = 56×B/N` for Vyomanaut versus `15×B/N` for the paper's default. The correlation effect scales with fragments-per-disk, so Vyomanaut's burst magnitude is roughly 4× larger per failure event for equivalent disk occupancy.
- **Paper proposes the Fluid Model as the solution to correctly size bandwidth** ≠ **Vyomanaut's primary mitigation is the 20% ASN cap (ADR-014), not bandwidth headroom**
→ The paper's conclusion is "provision more bandwidth than the independent model suggests." Vyomanaut's architectural answer is different: prevent correlated burst failures from happening in the first place by ensuring no single correlated provider group holds more than 20% of any file's shards. This is the structurally correct response — it bounds the worst-case burst magnitude rather than trying to absorb arbitrary bursts with bandwidth.
- **Paper's bandwidth variance is driven by single-disk crashes affecting B×(s+r)/N fragments simultaneously** ≠ **our ASN-level correlated failures (ISP outage, power grid event) could take out N×ASN_fraction providers simultaneously**
→ This is a strictly harder failure mode than the paper models. The paper's disk crash affects one peer's fragments. An ISP outage could affect 20% of all providers simultaneously. The 20% ASN cap (ADR-014) limits this to 20% of any single file's shards, but the aggregate repair demand across all files in those providers would be a burst event several orders of magnitude larger than any single disk crash. The Giroire BWavg formula does not capture this scenario.
- **Paper's Refined Fluid Model requires knowing the distribution of disk ages** ≠ **Vyomanaut cannot observe provider storage fill ratios without invasive monitoring**
→ The practical implication is that Vyomanaut's repair burst prediction must use the conservative independent model with a large safety margin, since the exact RFM inputs are unavailable. The ASN cap provides a tighter bound on worst-case burst size than any bandwidth-headroom calculation.

---

### Decisions Influenced

- **ADR-003 [#3 Erasure Coding]** `CONSTRAINT CLARIFIED`
The Giroire-derived BWavg ≈ 39 Kbps/peer is the mean bandwidth under the independent failure assumption. This paper proves that the real standard deviation is 22× larger than the independent model predicts. The ADR-003 parameters remain correct for the mean — they guarantee LossRate < 10⁻¹⁵ per year under the independent assumption — but they do not guarantee this under correlated burst failures. The 20% ASN cap (ADR-014) is the mechanism that keeps the correlated burst scenario bounded. Without the ASN cap, the erasure parameters alone are insufficient. The two decisions are jointly necessary, not independently sufficient.
*Because:* Section 4.5 shows that provisioning based on the independent model's µ + 5σ still produces thousands of data losses over the simulation lifetime. The variance from correlated failures cannot be absorbed by bandwidth headroom at any practical cost.
- **ADR-004 [#4 Repair Protocol]** `CONFIRMED WITH CAVEAT`
Lazy repair with r0=8 and 72-hour departure threshold correctly manages single-provider departure events. The paper confirms that the MCM (which underlies the lazy repair bandwidth formula) correctly predicts average repair demand. The caveat is that the repair bandwidth budget must account for the real variance, not the independent model's variance. At V2 scale (hundreds of providers), individual provider departures produce manageable burst events. The dangerous scenario is simultaneous multi-provider departure from the same ASN, which the ASN cap prevents. The lazy repair parameters do not need to change; the ASN cap must remain enforced.
*Because:* Table 2 confirms that average bandwidth from the MCM matches simulation closely. The problem is variance, not the mean — and variance is bounded by the ASN cap, not by changing r0.
- **ADR-014 [#19 Adversarial Defences]** `CONFIRMED AND ELEVATED`
The 20% ASN cap's reliability justification is now strengthened beyond its adversarial origin (Honest Geppetto attack). This paper provides the reliability mathematics: a correlated failure across a group holding >20% of any file's shards creates a repair burst whose magnitude scales directly with the group's shard fraction. The cap directly limits the worst-case correlated burst to 20% of any file's fragments — keeping the remaining 80% of fragments (44 of 56) well above the s=16 reconstruction floor even if the entire correlated group fails simultaneously. This is information-theoretically safe at any repair bandwidth level.
*Because:* The paper's key insight — that a single correlated failure event can simultaneously degrade all blocks that share fragments on the affected disk(s) — applies equally to ASN-correlated provider groups. The 20% cap is the correct bound, and this paper explains why mathematically.
- **ADR-026 [#17 Repair Bandwidth Optimisation]** `NEW CONSTRAINT`
Repair bandwidth optimisation (Hitchhiker codes, V3) must account for correlated burst events, not just average bandwidth. The Giroire BWavg formula that motivates ADR-026's current "acceptable at V2 scale" conclusion is the mean under independent failures. The real peak bandwidth during a correlated burst event (ASN outage, simultaneous multi-provider failure) is substantially higher. Before evaluating whether Hitchhiker's 25–45% bandwidth reduction is economically worthwhile, the burst-to-average ratio at V3 scale must be estimated using the Fluid Model framework from this paper, not just BWavg. The paper's conclusion that variance remains 5× the independent prediction at 50,000 peers is directly relevant to the V3 scale question.
*Because:* Section 4.4 shows that at 50,000 peers the correlated standard deviation is still 5× the independent model. At V3 scale, repair bandwidth optimisation targeting the mean may be insufficient if the burst demand is the binding constraint.

---

### Disagreements

- **Giroire et al. (Paper 10, and this paper's own MCM):** The MCM correctly predicts the average. This paper is the same research group acknowledging the MCM's limitation — it is not a contradiction but an extension.
*Implication for us:* ADR-003's use of the Giroire formulas is valid for the mean. The correction factor is the ASN cap, not a revision of the erasure parameters.
- **Weatherspoon & Kubiatowicz (cited as [17]):** Proved that erasure coding requires 8× less bandwidth than replication for equivalent durability, under the independent failure assumption.
*Implication for us:* Under correlated failures, the advantage of erasure coding over replication is preserved for the mean but reduced at peak. Replication would also produce correlated bursts (an entire replicated chunk group disappears when its host fails). Erasure coding remains the correct choice; the relative advantage at peak is smaller than 8× but still positive.

---

### Open Questions

See open-questions.md — questions Q36-1 and Q36-2.