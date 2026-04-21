## Paper 38 — Subtleties in Tolerating Correlated Failures in Wide-area Storage Systems

**Authors:** Suman Nath (Microsoft Research), Haifeng Yu, Phillip B. Gibbons (Intel Research Pittsburgh), Srinivasan Seshan (Carnegie Mellon University)
**Venue / Year:** USENIX NSDI 2006
**Topics:** #3, #4, #19
**ADRs produced:** none — ADR-003 and ADR-014 gain critical quantitative constraints; see Decisions Influenced

---

### Problem Solved

Distributed storage systems are almost universally evaluated under the assumption of independent failures. This paper demonstrates, using three real-world failure traces (PlanetLab, web servers, RON testbed), that this assumption produces system designs that are simultaneously overly pessimistic in some availability regimes and dangerously optimistic in others. Four specific subtleties are identified: failure patterns are not reliably predictable from history; single-failure-size models lead to designs that waste 240% of resources or miss availability targets by two nines; additional fragments yield strongly diminishing returns under correlated failures for systems with large m; and a design superior under independent failures (ERASURE(8,16)) can be two nines worse than a simpler design (ERASURE(1,4)) under real-world correlated failures. For Vyomanaut, the paper's critical value is quantifying the diminishing-return risk in our RS(s=16, r=40, n=56) wide-stripe configuration — the largest m of any system in the related literature at the time — and confirming that the 20% ASN cap (ADR-014) is the correct primary mitigation rather than simply increasing n.

---

### Key Findings

**Finding 1 — Failure pattern prediction provides negligible availability improvement:**
Even when failure clusters are stable across training and test periods (confirmed by mutual information scores), placing fragments to avoid historically correlated node groups does not improve availability. The reason: large failure events (the top 1% by size) dominate unavailability but are drawn from an exponential rather than Pareto distribution — meaning their pairwise failure patterns are memoryless and therefore unpredictable. The 99% of failures that are small (and predictable) contribute almost nothing to unavailability because data redundancy and regeneration absorb them.

**Finding 2 — Single-failure-size models (Glacier-style) cause 2-nine errors and 240% resource waste:**
Glacier assumes a single maximum failure fraction f. Because real failure size distributions are bi-exponential (two exponential regions with different slopes), a single value of f cannot match the real curve at all availability targets simultaneously. Depending on f chosen: at high availability targets, Glacier over-estimates by 2 nines (recommending ERASURE(6,32) when only ERASURE(6,10) is needed, wasting 240% of bandwidth and storage); at low targets, Glacier under-estimates by 2 nines. The bi-exponential model G(α, ρ₁, ρ₂) avoids both errors.

**Finding 3 — Diminishing returns under correlated failures scale with m:**
Under independent failures, increasing n in ERASURE(m,n) gives exponentially decreasing unavailability. Under real-world correlated failures with bi-exponential failure size distributions, the gain diminishes strongly once n exceeds a threshold — and the effect is stronger for larger m. For ERASURE(n/2, n) (large m), increasing n from 20 to 60 provides less than half a nine of improvement. For ERASURE(4,n) (small m), the improvement is near-linear. This diminishing return arises not from correlation per se but from the combination of failures of different sizes: failures of each size contribute independently to unavailability, and the combined curve is not a straight line in availability nines.

**Finding 4 — Correlated failures can reverse the superiority ordering of designs:**
ERASURE(8,16) achieves 1.5 more nines than ERASURE(1,4) under independent failures. Under real-world correlated failures matching the WS trace, ERASURE(8,16) achieves 2 fewer nines than ERASURE(1,4). The inversion is caused by the diminishing-return effect: ERASURE(8,16) has larger m and therefore suffers more from correlated failures than the simpler design. The correct intuition is that larger m concentrates the reconstruction requirement on more simultaneously-needed fragments, making a single large correlated failure event more likely to take out enough fragments to prevent reconstruction.

**IRISSTORE deployment result:**
Running a 7-replica read/write storage layer over 450+ PlanetLab nodes for 8 months, the system achieved 99.927% availability (a preconfigured 99.9% target). During a 62-hour stress period before SOSP'05, when available node fraction dropped sharply multiple times, IRISSTORE object availability stayed consistently above 98%.

---

### Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Bi-exponential failure size distribution model | Single maximum failure size (Glacier model) | Avoids 2-nine over/under-estimation errors; requires fitting α, ρ₁, ρ₂ to traces |
| Small m (ERASURE(4,n) or replication ERASURE(1,n)) | Large m (ERASURE(n/2, n)) | Resists diminishing returns under correlated failures; lower storage efficiency or reconstruction parallelism |
| Tolerating correlated failures through parameter choice | Avoiding them through pattern-aware placement | Pattern prediction fails for large failures; correct parameter choice is always effective |
| Regeneration systems with explicit repair | Static redundancy | Steady-state availability depends on repair speed vs. failure rate |

---

### Breaks in Our Case

- **Paper studies non-P2P environments with largely homogeneous software (PlanetLab, web servers, RON)** ≠ **Vyomanaut providers are heterogeneous home desktops and NAS devices behind different ISPs across India**
→ The paper explicitly notes that P2P failure correlation can be dramatically different because many P2P failures are voluntary departures rather than software crashes. Vyomanaut's failure correlation is driven by ISP-level events (CGNAT failures, power grid outages) and the voluntary departure pattern characterised by Bolosky (Paper 09), not by software homogeneity. The bi-exponential model's large-failure component (dominated by DDoS and infrastructure events) applies directly; the small-failure component (dominated by software bugs in homogeneous deployments) may not.
- **Paper's ERASURE(m,n) notation uses m as the reconstruction threshold (our s=16) and n as total fragments** ≠ **our notation uses s=16 (reconstruction threshold), r=40 (redundancy fragments), n=s+r=56 (total)**
→ Direct translation: Vyomanaut is ERASURE(16,56) in the paper's notation. Our m=16 is among the largest in any system studied — the paper's results on diminishing returns are most severe for large m, so our configuration faces the strongest form of this risk.
- **Paper's diminishing return finding concerns increasing n while keeping the m/n ratio fixed (ERASURE(n/2,n) style)** ≠ **our parameters are fixed at RS(16,56) rather than being tuned dynamically**
→ We do not intend to increase n beyond 56. The relevant question is whether our fixed (s=16, r=40) configuration sits in the diminishing-return regime for its primary defence against correlated failures. The 20% ASN cap (ADR-014) limits the maximum correlated failure to 20% of 56 = ~11 shards simultaneously — keeping the surviving shard count at 45, well above s=16. The erasure parameters alone are insufficient without this structural cap.
- **Paper uses a bi-exponential model G(α, ρ₁, ρ₂) derived from enterprise and university network traces** ≠ **Vyomanaut's failure distribution for Indian home ISPs is unknown at V2 launch**
→ The bi-exponential structure (small predictable failures + large unpredictable failures) is likely to hold for ISP-correlated events, but the parameters α, ρ₁, ρ₂ must be measured empirically at launch. The Dalle et al. paper (Paper 36) provides the same warning from a different angle. Until provider telemetry exists, the conservative design assumption is that large correlated failures occur and that the ASN cap bounds their impact.
- **Paper finds that ERASURE(8,16) can be two nines worse than ERASURE(1,4) under real correlated failures** ≠ **Vyomanaut has no replication alternative — the Cold Storage Band requires EC for economic viability at 256 KB fragments**
→ Simple replication at 3× overhead would require 768 KB per 256 KB shard across n providers — economically infeasible for cold storage at the India-first target price point. The correct response is not to switch to replication but to structurally bound the maximum correlated failure through the ASN cap, which limits any single correlated group to below the reconstruction threshold s=16 regardless of m.

---

### Decisions Influenced

- **ADR-003 [#3 Erasure Coding]** `CONSTRAINT ADDED — DIMINISHING RETURN RISK QUANTIFIED`
Finding 3 establishes that our RS(s=16, r=40) — ERASURE(16,56) in the paper's notation — sits in the large-m regime where correlated failures produce the strongest diminishing-return effects. The paper shows that for ERASURE(n/2,n) systems (same ratio as ours at approximately k/n = 16/56 ≈ 0.29), increasing n from 20 to 60 provides less than half a nine of improvement. This confirms that adding more providers beyond n=56 would not meaningfully improve availability under realistic correlated failures. The erasure parameters are correct for the independent-failure regime; ADR-014's 20% ASN cap is what provides the correlated-failure resilience that the erasure parameters alone cannot.
*Because:* Figure 5(a) directly shows the diminishing-return effect for ERASURE(n/3,n) and ERASURE(n/2,n) configurations. Our m/n ratio falls between these, placing us in the most affected regime.
- **ADR-014 [#19 Adversarial Defences]** `CONFIRMED AND ELEVATED`
Finding 4 proves that ERASURE(8,16) is two nines worse than ERASURE(1,4) under real-world correlated failures — even though ERASURE(8,16) is 1.5 nines better under independent failures. The mechanism is exactly the diminishing-return effect: larger m suffers more from correlated failures. This finding elevates the 20% ASN cap from an adversarial mitigation (Honest Geppetto attack) to a reliability requirement: without the cap, a correlated ISP-level failure could take out 11 of 56 shards simultaneously, which is within the RS(16,56) reconstruction budget but damages the safety margin materially. The cap ensures that no single correlated provider group can exceed 20% × 56 ≈ 11 shards — keeping all correlated failure events well below the reconstruction threshold of s=16.
*Because:* Section 8 and Figure 6 show that the ranking of designs reverses under correlated failures, and the reversal is proportional to m. The ASN cap structurally limits the maximum correlated failure size in Vyomanaut's network, preventing this reversal from occurring.
- **ADR-004 [#4 Repair Protocol]** `CONFIRMED`
The paper's IRISSTORE deployment demonstrates that for regeneration systems, a 12-minute repair convergence time achieves almost identical availability to a 5-minute convergence time, because repair only needs to complete before the next failure arrives. Vyomanaut's 72-hour departure threshold (calibrated for Bolosky's bimodal absence distribution) provides a far larger repair budget than the inter-failure interval — the lazy repair parameters are conservative by this standard. The paper also confirms that Paxos-merging and admission-control on the repair scheduler are necessary at scale to prevent the positive feedback problem (repair load causing failure detection errors). The microservice's repair queue (ADR-004) should implement burst admission control for large correlated failure events.
*Because:* Section 10.4 shows that without admission control on the repair scheduler, the 2478-object repair after a 42-node failure does not converge. This is the operational manifestation of the Dalle et al. variance finding (Paper 36).

---

### Disagreements

- **OceanStore and CFS (ERASURE(n/2,n) configurations):** Both systems use large m/n ratios and rely on increasing n for availability. Finding 3 shows this approach hits strong diminishing returns under correlated failures — adding more nodes provides less than half a nine per doubling.
*Implication for us:* Our fixed RS(16,56) is not iterated upward in response to failures; the ASN cap is the scaling mechanism. This is the architecturally correct response to the diminishing-return finding.
- **Glacier's single-failure-size model:** Glacier provides a closed-form formula for erasure parameters under a fixed maximum failure fraction f. Finding 2 shows this produces 2-nine errors in practice because real failure size distributions are bi-exponential.
*Implication for us:* The Giroire formulas (Paper 10) that derive our erasure parameters assume Markov chain independent failure rates, not a bi-exponential distribution. The ASN cap bounds the maximum correlated failure event size, which effectively converts the bi-exponential distribution into a truncated one where the tail (large correlated failures) cannot exceed 20% of shards. Within this truncated distribution, the Giroire-derived parameters remain applicable.
- **Weatherspoon et al. (maintenance cost finding):** Weatherspoon et al. show that pattern-aware fragment placement has almost identical maintenance costs to random placement. This paper extends the finding to availability: pattern-aware placement also has no availability benefit.
*Implication for us:* Vyomanaut's random-within-ASN-cap placement (ADR-005) is confirmed as the correct strategy. Sophisticated placement algorithms that try to exploit historical correlation data would add complexity without benefit.

---

### Open Questions

See open-questions.md — questions Q38-1 and Q38-2.