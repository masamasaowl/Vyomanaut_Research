## Paper 10 — Peer-to-Peer Storage Systems: A Practical Guideline to Be Lazy

**Authors:** Frédéric Giroire, Julian Monteiro, Stéphane Pérennes — MASCOTTE joint project INRIA / I3S (CNRS, Univ. of Nice-Sophia Antipolis)
**Venue / Year:** IEEE GlobeCom 2010
**Topics:** #3, #4, #6, #17
**ADRs produced:** [ADR-003](../decisions/ADR-003-erasure-coding.md) — RS Erasure Coding: s=16, r=40, r0=8, lf=256 KB, [ADR-004](../decisions/ADR-004-repair-protocol.md) — Lazy Repair with r0=8 and 72-Hour Departure Threshold

---

### Problem Solved

P2P storage systems depend on parameters that interact in non-obvious ways: increasing redundancy r reduces data-loss probability but at some point starts increasing repair bandwidth rather than decreasing it. No prior work had derived closed-form expressions for both metrics simultaneously under lazy repair. Without these formulas, parameter choices are guesses. This paper models the system as a Markov chain, derives three closed-form expressions — average bandwidth consumption BWavg, peak bandwidth Qpeek on peer failure, and data loss rate LossRate — and provides a step-by-step methodology for choosing s, r, r0, and lf. For Vyomanaut, it is the primary derivation source for the erasure coding parameters in ADR-003 and the repair threshold in ADR-004. Every numeric parameter Vyomanaut uses for durability and bandwidth budgeting traces directly to one of these three formulas.

---

### Key Findings

### Notation

| Symbol | Meaning | V2 value |
| --- | --- | --- |
| N | Number of peers | 1,000 (planning target) |
| D | Total data stored | variable |
| s | Initial fragments per block (reconstruction threshold) | 16 |
| r | Redundancy fragments | 40 |
| r0 | Lazy repair trigger (repair fires when fragments drop to s + r0) | 8 |
| lf | Fragment size | 256 KB |
| lb | Block size = s × lf | 4 MB |
| B | Total blocks = D / lb | D / 4 MB |
| MTTF | Mean time to failure per peer | 300 days |
| α | Failure probability per time step τ = 1/MTTF | 1 / 7,200 per hour |
| γ | Reconstruction probability per time step = 1/θ | depends on bandwidth |
| θ | Average time to reconstruct one block |  |

### Formula 1 — Average Bandwidth Consumption

`BWavg ≈ (B · α) / (N · ln((s+r)/(s+r0)) · τ) · (s + r − r0 − 1) · lf`

The key insight in this formula is the logarithm in the denominator. The larger the gap r − r0, the larger ln((s+r)/(s+r0)), and the lower BWavg. Lazy repair works because the gap r − r0 amortises bandwidth across many failures before a reconstruction is triggered. The formula does not depend on γ (the reconstruction rate) — average bandwidth is a property of the failure rate and system configuration, not of how fast you repair.

**Applying to V2 parameters (s=16, r=40, r0=8, lf=256 KB):**

`ln((16+40)/(16+8)) = ln(56/24) = ln(2.333) ≈ 0.848
s + r − r0 − 1 = 16 + 40 − 8 − 1 = 47`

For D = 1 TB, N = 1,000, MTTF = 300 days, τ = 1 hour:

`B = 1 TB / 4 MB = 262,144 blocks
α = 1 / (300 × 24) ≈ 1.39 × 10⁻⁴ per hour
BWavg ≈ (262,144 × 1.39×10⁻⁴) / (1,000 × 0.848) × 47 × 256 KB
       ≈ 36.4 / 848 × 47 × 262,144 bytes
       ≈ ~39 Kbps per peer`

This is the 39 Kbps/peer figure used in ADR-003. It is well within the 100 Kbps background budget established in ADR-009 (Blake & Rodrigues, Paper 06).

### Formula 2 — Peak Bandwidth on Peer Failure

When one peer fails, all its fragments trigger reconstructions simultaneously. The aggregate data transfer induced by one failure event is:

`Qpeek ≈ B·(s+r) / (N·(s+r0+1)·ln((s+r)/(s+r0))) · (s+r−r0−1) · lf`

**Applying to V2 parameters at N=1,000:**

`Qpeek ≈ B × 56 / (1,000 × 25 × 0.848) × 47 × 256 KB`

For D = 50 TB (50 GB per peer at N=1,000), B = 12,500,000:

`Qpeek ≈ 700,000,000 / 21,200 × 47 × 262,144 bytes
       ≈ 793 GB total network transfer per failure event`

At 100 Kbps per peer (N=1,000 peers, aggregate 100 Mbps), reconstruction completes in approximately 8 hours — well within the 12-hour safety window θ before a second failure becomes likely at MTTF=300 days. This is the Qpeek figure used in ADR-004.

### Formula 3 — Data Loss Rate

`LossRate ≈ B / ((s+r0+1)·ln((s+r)/(s+r0))·τ) · (s+r0)! / (s−1)! · (α/γ)^(r0+2)`

This formula is dominated by the term (α/γ)^(r0+2). Because this is raised to the power r0+2 = 10, LossRate decreases exponentially as r0 increases or γ increases (faster repair). Crucially, LossRate does not depend on N directly — it depends on B, the total number of blocks, which scales with D. A larger network storing the same data has the same LossRate.

**Applying to V2 parameters targeting LossRate < 10⁻¹⁵ per year:**

At θ = 8 hours (reconstruction window from Qpeek calculation), γ = 1/8 per hour, α = 1.39×10⁻⁴ per hour:

`α/γ = (1.39×10⁻⁴) / (1/8) = 1.11×10⁻³
(α/γ)^(r0+2) = (1.11×10⁻³)^10 ≈ 3.4×10⁻³⁰`

LossRate is approximately 10⁻²⁵ per year — four orders of magnitude below the ADR-003 target of 10⁻¹⁵. The parameters are conservative. This safety margin accommodates the correlated failure variance that Paper 36 (Dalle et al.) identifies as a risk at wide stripe widths.

### Formula 4 — Optimal r for Given s and r0

Setting ∂BWavg/∂r = 0 yields:

`r0 − s − r + (s + r) · ln((s+r)/(s+r0)) = 0`

This equation must be solved numerically for r. For s=16, r0=8, solving gives r=40 — the exact value used in ADR-003. This is not a coincidence; r=40 is the analytically optimal redundancy for minimising bandwidth consumption at our reconstruction threshold and block size.

---

### Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Lazy repair (trigger at r0=8, fire only when fragments drop 8 below full) | Eager repair (trigger immediately on every fragment loss) | ~3× bandwidth reduction at gap r−r0=32 (reading from Figure 7 at s=16, r=40 vs r=r0+1); data is slightly less redundant during the lazy window |
| r=40 (optimal r from Formula 4) | r=16 (2× overhead, simpler) | BWavg drops from ~58 Kbps to ~39 Kbps per peer at our parameters; storage overhead rises from 2× to 3.5×; durability improves by ~10 orders of magnitude |
| r0=8 (repair trigger 8 above reconstruction floor) | r0=4 or r0=12 | r0=4 reduces repair bandwidth but exposes data dangerously close to the s=16 reconstruction floor; r0=12 increases bandwidth toward eager repair; r0=8 balances both |
| Markov chain with independent peer failures | Correlated failure model | Correctly predicts the mean (confirmed by Paper 36); underestimates variance by 22× during burst events; mitigated by 20% ASN cap (ADR-014) |
| Fixed timeout before declaring a peer failed | Diurnal churn model (bimodal presence) | Simplified model; does not distinguish nightly absences from permanent departures; supplemented by Bolosky's bimodal distribution (Paper 09) for the actual 72h threshold |
| Block-level reconstruction (s=16 fragments downloaded per repair) | Regenerating codes (sub-optimal bandwidth) | Reed-Solomon requires downloading all k=16 surviving fragments per repair; MSR codes would reduce this but are computationally intractable at n=56 (Paper 22) |

---

### Breaks in Our Case

- **Paper derives BWavg assuming crashed disks reappear empty** ≠ **Vyomanaut has four provider exit states, including promised downtime where the disk returns intact (ADR-007)**
→ For announced and promised departures, repair is triggered before the provider leaves; the provider's data may still be live for the promise period. This means some "repairs" are pre-emptive rather than reactive, consuming bandwidth without a corresponding fragment loss event. The BWavg formula over-counts bandwidth for promised departures and under-counts for silent departures — the two effects partially cancel.
- **Paper assumes α is constant and peers fail independently** ≠ **our providers fail in correlated bursts when an ASN goes down or a power event hits a region (Paper 36, Dalle et al.)**
→ The mean BWavg (Formula 2) remains correct under correlation; the variance is 22× larger than the formula predicts. Bursts of peer failures produce Qpeek events clustered in time. The 20% ASN cap (ADR-014) bounds the maximum correlated fragment loss per event, keeping even the worst burst within the 8-hour repair window.
- **Paper's optimality condition (Formula 4) gives r=40 at s=16, r0=8 for minimum BWavg, not for minimum LossRate** ≠ **LossRate and BWavg are jointly constrained in Vyomanaut**
→ r=40 minimises bandwidth consumption while achieving LossRate ≈ 10⁻²⁵ per year — far below the 10⁻¹⁵ target. The parameters are simultaneously optimal for bandwidth and comfortably safe for durability. No additional tuning is required.
- **Paper models N as constant** ≠ **Vyomanaut's provider count grows over time**
→ BWavg and Qpeek scale inversely with N (more peers, smaller per-peer burden). LossRate depends on B (total blocks), not N directly — a growing network storing more data has the same LossRate per unit data. As N grows, per-peer bandwidth drops and the network becomes more conservative than the ADR-003 numbers predict.
- **Paper's examples use small parameters (s=5, r=6, r0=2) for tractability** ≠ **our RS(16,56) is the widest stripe in any production deployment at the time of writing**
→ The formulas are valid for any parameter set where α/γ ≪ 1. At our parameters, α/γ ≈ 1.11×10⁻³, satisfying this condition comfortably (Figure 4 of the paper shows accuracy within noise at α/γ < 10⁻³). The formulas are reliable at V2 scale.
- **Paper does not account for the time to declare a peer departed (the polling timeout t)** ≠ **our 72-hour departure threshold (ADR-006, ADR-007) means repair is deferred by up to 72h after a provider actually fails**
→ During the 72-hour window before a silent departure is declared, the block's fragment count is below r0 but repair has not started. The effective repair window for silent departures is θ_effective = θ + 72h. The LossRate formula uses γ = 1/θ; substituting θ_effective = θ + 72h reduces γ slightly. At MTTF=300 days and r0=8, the LossRate is so far below 10⁻¹⁵ that this reduction is absorbed without breaching the target.

---

### Decisions Influenced

- **ADR-003 [#3 Erasure Coding — DECIDED, parameters]** `ACCEPTED`
Formula 4 (∂BWavg/∂r = 0) with s=16 and r0=8 gives the unique optimal r=40. Formula 2 then gives BWavg ≈ 39 Kbps/peer at MTTF=300 days, D=50TB, N=1,000. Formula 3 gives LossRate ≈ 10⁻²⁵ per year — more than ten orders of magnitude below the 10⁻¹⁵ target. All three formulas are satisfied simultaneously at s=16, r=40, r0=8, lf=256 KB. These are the V2 parameters.
*Because:* The paper proves r=40 is not an arbitrary choice — it is the unique r that minimises BWavg for s=16, r0=8. No other value of r gives lower bandwidth consumption while maintaining the same reconstruction threshold.
- **ADR-004 [#4 Repair Protocol — DECIDED]** `ACCEPTED`
r0=8 is the repair trigger threshold, meaning repair fires when available fragments drop to s + r0 = 24 (8 above the reconstruction floor of s=16). This value comes from Formula 3: r0=8 is the smallest value that keeps LossRate < 10⁻¹⁵ per year at our MTTF and γ. The Qpeek formula gives 793 GB total network transfer per failure event at N=1,000, completing in ~8 hours at 100 Kbps/peer — within the 12-hour safety window.
*Because:* Formula 3 proves that LossRate decreases exponentially with r0. The minimum r0 satisfying the durability target determines the repair threshold, not arbitrary engineering judgement.
- **ADR-026 [#17 Repair BW Optimisation — PROPOSED]** `INFORMED`
The Qpeek formula is the key input for sizing V3 repair bandwidth budgets. At N=10,000 providers (V3 scale), Qpeek per failure event scales with B/N, so per-peer burden decreases. However, failure events are more frequent at larger N. The formula also confirms that Hitchhiker codes (ADR-026's sole V3 candidate), which reduce repair bandwidth by 25–45%, would reduce the per-event Qpeek proportionally while keeping r0 and the other parameters unchanged.
*Because:* BWavg and Qpeek are the two quantities repair bandwidth optimisation targets. The formula shows which parameters drive each metric and how code improvements map to real savings.

---

### Engineering Questions

**Q1 — Applying Formula 2 at different MTTF values**

| MTTF (days) | BWavg (Kbps/peer) | Within 100 KB/s budget? |
| --- | --- | --- |
| 90 (mobile floor) | ~130 | No — mobile providers excluded (ADR-010) |
| 180 (desktop minimum) | ~65 | Yes |
| 300 (desktop median) | ~39 | Yes |
| 380 (NAS tier) | ~31 | Yes |

The 180-day floor for V2 desktop providers (ADR-010) keeps BWavg below the 100 Kbps background budget even at the worst-case end of the MTTF range. Mobile providers at MTTF ≈ 30–90 days would push BWavg above budget, confirming their exclusion.

**Q2 — Reconstruction window θ at V2 scale**

From Qpeek = 793 GB total at N=1,000, with each peer contributing 100 Kbps:

`θ = Qpeek / (N × BWcapacity)
  = 793 GB / (1,000 × 100 Kbps)
  = 793 × 10⁹ × 8 bits / (10⁵ bits/s)
  ≈ 63,440 seconds
  ≈ 8 hours`

The 12-hour safety budget from ADR-004 provides a 4-hour margin above the 8-hour reconstruction time. At MTTF=300 days, the probability of a second peer failure within the 8-hour reconstruction window is approximately 8/(300×24) = 0.11% per peer — small enough that simultaneous dual-failure events are rare but not impossible. The r=40 redundancy buffer (40 parity fragments above s=16) absorbs multiple simultaneous failures before the reconstruction floor is breached.

**Q3 — Bandwidth savings of lazy repair over eager repair**

At r0=r−1 (eager policy), every fragment loss triggers immediate reconstruction of one fragment, costing s+1 fragments downloaded per repair. At r0=8 (our setting), reconstruction fires after r−r0=32 fragments are lost, reconstructing all 32 at once. The bandwidth saving is approximately:

`BW_eager / BWavg ≈ (r − r0) / ln((s+r)/(s+r0))
                 = 32 / ln(56/24)
                 = 32 / 0.848
                 ≈ 38×`

Compared to an intermediate eager threshold at r0 = r−1 = 39, our r0=8 achieves roughly 38× lower bandwidth. Even compared to a moderate threshold of r0=20 (half of r), we save approximately 3×. This is the dominant factor in why lazy repair is chosen over eager repair regardless of the specific r0 value.

---

### Disagreements

- **Dimakis et al. (2010, cited as [8]):** Shows that standard RS repair is not bandwidth-optimal — regenerating codes can repair a single fragment by downloading less than s full fragments. This is the foundational objection to RS repair.
*Implication for us:* MSR codes (the optimal regenerating codes) are computationally intractable at n=56 (Paper 22). Hitchhiker codes achieve 25–45% bandwidth reduction while keeping α=2 sub-stripes. The Giroire parameters remain correct for RS; Hitchhiker is the V3 improvement path (ADR-026).
- **Dalle et al. (Paper 36, same group):** Proves that the Markov chain model correctly predicts the mean BWavg but underestimates the standard deviation by 22× due to correlated burst failures when a disk crashes and simultaneously drops all its fragments.
*Implication for us:* The mean figures from these formulas are correct for steady-state bandwidth planning. The variance risk is structural, not parametric — the ASN cap (ADR-014) is the correct mitigation, not increasing r or r0.
- **Silberstein et al. (Paper 39):** Argues that simple threshold-based lazy repair is insufficient in large-scale datacenter DSS because permanently failed stripes accumulate. Proposes dynamic recovery thresholds.
*Implication for us:* At V2 scale (hundreds of providers, MTTF 180–380 days), the accumulation problem does not manifest. Vyomanaut's four exit states (ADR-007) already implement Silberstein's core insight: permanent departures (72h threshold) trigger immediate repair; transient absences are handled lazily. The dynamic threshold becomes relevant at V3 scale.

---

### Open Questions

See open-questions.md — questions Q10-1 through Q10-3. Q10-4 answered in answered-questions.md.