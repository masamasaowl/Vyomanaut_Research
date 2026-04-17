# Paper 10 — Peer-to-Peer Storage Systems: A Practical Guideline to Be Lazy

**Authors:** Frédéric Giroire, Julian Monteiro, Stéphane Pérennes, MASCOTTE joint project INRIA / I3S (CNRS, Univ. of Nice-Sophia Antipolis)
**Venue / Year:** IEEE GlobeCom 2010
**Topics:** #3, #4, #6, #17

---

## Problem Solved

1. Provides actual mathematical formulas for the entire DSN
2. Helps decide the delay in repair triggers and erasure parameters
3. Formally proves that lazy repair outperforms eager repair under bandwidth constraints

---

## Key Formulas

**Average bandwidth consumption:**
```
BWavg = ((B·α) / (N · ln((s+r) / (s+r0)) · τ)) · (s + r − r0 − 1) · lf
```

**Data loss rate:**
```
LossRate = B / ((s+r0+1) · ln((s+r)/(s+r0)) · τ) · (s+r0)! / (s-1)! · (α/γ)^(r0+2)
```

**Best bandwidth consumption for given s and r0 (optimality condition):**
```
r0 − s − r + (s + r) · ln((s + r) / (s + r0)) = 0
```

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Lazy repair (repair only when redundancy falls below r0) | Eager repair | BW savings grow logarithmically with gap r−r0 (gap of 8 reduces BWavg by ~3× vs gap of 1), but data loss probability increases |
| Markov chain model with independent peer failures | Correlated failure model | We use the diversity cap from SoK to make this assumption work |
| Fixed timeout for peer loss | Diurnal churn model | Doesn't capture nighttime vs weekend absence patterns — supplement with Bolosky bimodal |

---

## Breaks in Our Case

- **Assumes homogenous peer distribution** ≠ **our tier-wise population with different MTTFs**
  → Paper 10 parameters are calibrated for a uniform peer set; tier-specific r0 is deferred to V3

- **Assumes peer re-enters with no stored data** ≠ **our promised downtime allows peers to return with their data intact**
  → Promised downtime is a significant advantage over the paper's assumptions

- **N (number of peers) and D (total network storage capacity) kept constant** ≠ **our growing network**
  → Parameters must be revisited periodically as the network grows

---

## Decisions Influenced

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding — DECIDED, parameters]:**
  Target: LossRate < 10⁻¹⁵ per year
  Fixed parameters:
  - Initial fragments: s = 16
  - Fragment size: lf = 256 KB
  - Repair threshold: r0 = 8
  - Redundancy fragments: r = 40
  - Result: BWavg ≈ 39 Kbps/peer

- **[ADR-006](../decisions/ADR-006-repair-protocol.md) [#4 Repair Protocol — DECIDED]:** r0 is the repair trigger threshold. Derived from the formulas in this paper.

- **[ADR-017](../decisions/ADR-017-repair-bw-optimisation.md) [#17 Repair BW — DECIDED]:** Peak bandwidth formula Qpeek is the key input for sizing the repair bandwidth budget. For each provider failure event, the network must absorb Qpeek/θ bandwidth over reconstruction window θ.

---

## Disagreements

- **Dimakis et al. (2007):** Shows that standard RS repair is not bandwidth-optimal.
  *Implication for us:* MSR/LRC regenerating codes are candidates for V3 optimisation — reading-list Phase 2A #15.

---

## Quantitative Results (from engineering questions)

**Repair bandwidth spike Qpeek for one mobile provider failure:**
For N=1000, B=25M, s=16, r=40, r0=8, lf=256 KB:
- Qpeek = 793 GB per peer
- At 100 Kbps per peer: takes 8 hours to absorb the loss of one peer
- 8 h < 12 h reconstruction window θ → MTTF must not fall below current levels

**Optimal lazy repair for mixed-tier network:**
- *Simpler approach (V2):* Use a single r0 calibrated to the most volatile tier (mobile), giving desktops more buffer than they need but simplifying the repair state machine
- *Sophisticated approach (V3):* Separate shard pools per tier
  - 20 shards to Tier 1 (desktop) with r0_desktop = 12
  - 36 shards to Tier 3 (mobile) with r0_mobile = 8
  - Optimal bandwidth per tier but requires repair scheduler to track two independent Markov chains per block

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q10-1 through Q10-2.
