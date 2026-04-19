# Paper 22 — Minimum Storage Regenerating Codes For All Parameters

**Authors:** Sreechakra Goparaju, Arman Fazeli, Alexander Vardy — UC San Diego
**Venue / Year:** arXiv:1602.04496 | February 2016 (presented at Allerton 2015)
**Topics:** #3, #17
**ADRs produced:** none — ADR-026 updated in Decisions Influenced

---

## Problem Solved

Prior MSR code constructions existed only for restricted parameter ranges: either low coding
rate (k/n ≤ 0.5) or maximum repair connectivity (d = n − 1). No explicit construction
existed for high-rate codes (k/n > 0.5) with d < n − 1 — which are the practical parameters
for any real deployment where contacting all n − 1 surviving nodes to repair one failed
node is infeasible. This paper provides the first explicit construction of systematic-repair
MSR codes for all (n, k, d) parameter combinations, proving existence was not enough —
the construction must be explicit and finite-field. For Vyomanaut, the paper directly answers
Q19-2: it gives the sub-packetization formula (α = ρ^(k·C(r,ρ))) that determines whether
Clay-class MSR codes are computationally feasible at our (n=56, k=16) parameters.

---

## Key Findings

**The core result (Theorem 7):**
Construction 2 gives a systematic-repair [n, k, d] MSR code for any set of d helper nodes,
for large enough field size. The construction requires repair bandwidth:

```
β = α / (d − k + 1)     per helper node
γ = d × β               total repair bandwidth
```

where α is the sub-packetization level (chunk split into α sub-chunks). For comparison,
standard RS repair requires downloading k full chunks (k × α symbols total) to reconstruct
one failed chunk. MSR reduces this to d × α/(d−k+1) symbols — strictly less than k × α
for any d ≥ k.

**Sub-packetization formula (Construction 2):**

```
α = ρ^(k × C(r, ρ))

where:
  ρ = d − k + 1     (number of parity nodes in the helper set)
  r = n − k         (total number of parity chunks)
  C(r, ρ)           (binomial coefficient "r choose ρ")
```

This is the critical formula for Q19-2. The paper notes that α is not optimised — this is
a theoretical existence result, and the actual minimum α is an open problem. However, even
this construction gives the order of magnitude.

**Applying to Vyomanaut's parameters (n=56, k=16, r=40):**

For d = n − 1 = 55 (maximum connectivity, minimum repair bandwidth):
```
ρ = 55 − 16 + 1 = 40 = r
C(40, 40) = 1
α = 40^(16 × 1) = 40^16 ≈ 4.3 × 10^25 sub-chunks per chunk
```

For d = k + 1 = 17 (minimum connectivity, t=1 parity in helper set):
```
ρ = 2
C(40, 2) = 780
α = 2^(16 × 780) = 2^12480   (astronomically large)
```

For any practically relevant d between k+1 and n-1:
```
C(40, ρ) grows rapidly as ρ moves away from 0 or 40.
The minimum of C(40, ρ) is at ρ=40 giving α = 40^16 ≈ 10^25.
```

Every parameter combination yields α that is computationally intractable for any hardware.
A 256 KB chunk divided into 40^16 sub-chunks has sub-chunks of size essentially zero.

**Repair bandwidth saving (even if α were feasible):**

For d = n − 1 = 55:
```
β = α / (d − k + 1) = α / 40
γ_MSR = 55 × (α/40) = 55α/40 = 1.375α
γ_RS  = k × α = 16α
```
MSR repair bandwidth saving: 16α / (55α/40) = 640/55 ≈ 11.6× less than RS.

For d = k + 1 = 17:
```
β = α / 2
γ_MSR = 17 × α/2 = 8.5α
γ_RS  = 16α
```
MSR repair bandwidth saving at minimum d: ~1.9× less than RS.

The bandwidth savings are real and significant. The sub-packetization level is what makes
MSR codes inapplicable at our parameters.

**Interference alignment mechanism (Section II-A):**
The MSR repair property relies on interference alignment: each surviving node sends α/r
sub-chunk symbols, arranged so that the "interference" from non-failed systematic nodes
occupies a known subspace that can be subtracted out. This works because each chunk is
sub-packetised into α sub-chunks, and the repair involves specific sub-chunk indices (the
Y_j sets). Without sub-packetisation into α parts, the interference alignment conditions
(equations 5-6) cannot be satisfied.

**Clay codes connection:**
The paper describes the family of codes that Clay codes (EC Survey Paper 19) implement in
practice. Clay codes are the state-of-the-art access-optimal MSR implementation (both
repair bandwidth and repair I/O are minimised). The sub-packetisation parameter for Clay
codes is α = r^(k/(r-d+k)), which for d = n-1 simplifies to α = r^k — same order as this
paper's result at d=n-1. Clay codes are therefore not an exception to the sub-packetisation
problem; they are its best known implementation.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Explicit code construction for all (n,k,d) | Asymptotic existence proof | Proves feasibility in finite field arithmetic; but α grows as ρ^(k·C(r,ρ)) — impractical at large (n,k) |
| Systematic-repair only (optimal repair for data chunks) | Optimal repair for all nodes | Parity node failures repaired by full reconstruction, not sub-optimal; simplifies proof |
| Any d helper nodes from {k ≤ d ≤ n-1} | d = n-1 only | Flexible repair connectivity; minimum repair bandwidth at d=n-1; minimum α at d=k (but C(r,1)=r still large) |
| Large field size q > qANY + qMDS | Smaller field | Guarantees MDS property via Schwartz-Zippel lemma; field size itself can be large in practice |

---

## Breaks in Our Case

- **Construction assumes sub-packetisation α = ρ^(k·C(r,ρ)) is feasible** ≠ **our n=56, k=16,
  r=40 produces α ≥ 40^16 ≈ 10^25 sub-chunks per 256 KB fragment**
  → This is computationally impossible. A 256 KB fragment divided into 10^25 sub-chunks gives
    sub-chunks of size 0 bytes. MSR codes with this sub-packetisation cannot be implemented on
    any hardware. Q19-2 is now answered definitively: Clay-class MSR codes are not applicable
    to Vyomanaut's (n=56, k=16) parameters.

- **Paper proves existence for any (n,k,d) but optimising α is left as an open problem**
  ≠ **practical deployment requires α ≤ chunk_size / (minimum addressable unit)**
  → Even if a future lower-α construction is found, the known lower bound on α for MSR codes
    (reference [19] in the paper, Goparaju et al. IEEE ToIT 2014) shows that α must be at least
    exponential in k for d = n − 1. Our k=16 is large enough that no polynomial-α MSR code can
    exist for our parameters at d = n − 1.

- **Repair bandwidth savings are real (up to 11.6× less than RS at d=n-1)** ≠ **irrelevant if
  the code cannot be implemented**
  → The bandwidth argument for MSR is valid in theory. At smaller (n,k) — say n=14, k=10 as
    in Hitchhiker codes — sub-packetisation is manageable. At n=56, k=16, even Hitchhiker's
    two-sub-stripe approach (which limits α=2) only applies because Hitchhiker does NOT achieve
    the full MSR repair bandwidth bound; it achieves a partial reduction (25–45%) while keeping
    α low. This is the correct V3 candidate.

---

## Decisions Influenced

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `CONFIRMED`
  RS(s=16, r=40) is the only feasible code at our parameters. MSR codes are theoretically
  superior in repair bandwidth but require sub-packetisation levels that are computationally
  intractable at (n=56, k=16). The RS baseline is not a compromise — it is the only option.
  *Because:* Construction 2's sub-packetisation formula gives α ≥ 40^16 ≈ 10^25 for our
  parameters at d = n − 1. No known lower-bound exception exists for general k/n > 0.5.

- **[ADR-026](../decisions/ADR-026-repair-bw-optimisation.md) [#17 Repair BW Optimisation]**
  `CANDIDATE NARROWED — MSR/CLAY REJECTED, HITCHHIKER REMAINS`
  Q19-2 is now answered. MSR codes (including Clay codes) are not feasible at (n=56, k=16)
  due to intractable sub-packetisation. The only viable V3 repair bandwidth optimisation
  candidate is **Hitchhiker codes** (piggybacking framework), which limit sub-packetisation
  to α = 2 (two sub-stripes per stripe) and achieve 25–45% bandwidth reduction without
  violating the MDS storage property. ADR-026 should be updated to remove Clay codes as a
  candidate and retain Hitchhiker codes only. ADR-026 remains Proposed pending Phase 2A #5
  (cold data erasure coding) and production telemetry to confirm whether the 25–45% saving
  justifies implementation complexity at V2 scale.
  *Because:* Sub-packetisation formula applied to (n=56, k=16, r=40) yields α ≥ 40^16 for
  any d in {k+1, ..., n-1}. Hitchhiker's α = 2 avoids this entirely.

- **Q19-2 [Answered]:**
  Clay codes' sub-packetisation at (n=56, k=16) is computationally intractable (α ≥ 40^16).
  MSR codes in general cannot be implemented at our stripe width. The sub-packetisation lower
  bound (exponential in k for d = n − 1) closes this question definitively. Hitchhiker codes
  with α = 2 remain the only practical V3 candidate.

---

## Disagreements

- **EC Survey (Paper 19):** Identifies Clay codes as the state-of-the-art MSR implementation
  and the Pareto-optimal V3 candidate. Clay codes are correct for standard deployment scales
  (n ≤ 20 as in all surveyed production systems). At n=56 they are not applicable.
  *Implication for us:* The EC survey's recommendation of Clay codes is correct for n ≤ 20
  but does not extend to our wide-stripe configuration. This is a new finding not in the survey.

- **Dimakis et al. (2010, cited as [4]):** The foundational paper proving the MSR repair
  bandwidth bound. This paper (Goparaju et al.) builds on Dimakis but proves explicit
  constructions for all parameters. The bandwidth bound from Dimakis (equation 2 in this
  paper: β = α/(d−k+1)) is correct and achievable — the obstacle is only sub-packetisation,
  not the bandwidth formula itself.
  *Implication for us:* Dimakis's bandwidth formula confirms that Hitchhiker's 25–45%
  reduction is real and below the MSR optimal — it is a useful but not information-
  theoretically optimal repair bandwidth reduction.

---

## Open Questions

See [open-questions.md](open-questions.md) — Q19-2 is now answered. No new questions added.
