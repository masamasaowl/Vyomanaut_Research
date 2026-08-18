# Paper 80 — Secrecy Capacity of Minimum Storage Regenerating Codes

**Authors:** Ankit Singh Rawat (MIT, work done at Carnegie Mellon University)
**Venue / Year:** 2017 IEEE International Symposium on Information Theory (ISIT)
**Track:** LTS
**Reading list:** Domain P / R-27 — Band 1 — accept criterion *"states secrecy as a function of
(n,k,d) with the storage cost of the guarantee"*
**ADRs produced:** contributes to ADR-079 (Domain P Menu)
**Findings raised:** cross-validates F-LTS-15's quantitative basis (jointly with Paper 76)
**Questions closed:** none · **Questions raised:** none (Q79-1's sub-packetization concern, raised
independently below, folds into the existing sub-packetization framing rather than a new ID)
**Triage score:** 9/10 (parameter reach 2 · trust model 2 · evidence 2 · actionability 1 · corpus
delta 2)

---

## Provenance

Read: the full 3-page ISIT paper including the proof of Lemma 2 and Proposition 1. ISIT papers are
short by convention; nothing was deferred to a technical report.

## Problem Solved

Prior secure-MSR constructions required the eavesdropper's repair-observed nodes to come from a
**specific subset of `k` nodes** — a real restriction, since an adversary in practice chooses which
nodes to compromise `[P §I]`. This paper removes that restriction: it presents a construction secure
against an eavesdropper observing repair of **any** `ℓ2` nodes out of all `n`, and proves this
construction **characterizes the exact secrecy capacity** of linear-repairable MSR codes at
`d = n−1` `[P §I, §III]` — not merely an achievable bound, a tight one.

## Key Findings

### The construction: Gabidulin precoding + Ye–Barg's MSR codes

`[P §III, Construction 2]` The scheme precodes the message (plus `M − M_s` random padding symbols)
with a **Gabidulin code** — a rank-metric code — then encodes the precoded vector with an explicit
MSR construction from Ye & Barg that works for **all** system parameters `(n,k,d)`, unlike earlier
MSR constructions that only supported bandwidth-efficient repair for a restricted set of nodes.

### The secrecy capacity formula — and its exact numerical match to Paper 76

`[P eq. 16]` For the construction at `d=n−1`, storage `α = (n−k)^{n−1}`, and message
`M = kα = k(n−k)^{n−1}`:

```
M_s = (k − ℓ1 − ℓ2) · (1 − 1/(n−k))^ℓ2 · (n−k)^{n−1}                        [P eq. 16]
```

**This is algebraically the identical formula to Paper 76's Theorem 3** (`C_s(α) = (k−ℓ1−ℓ2)
(1−1/(n−k))^ℓ2 α`), evaluated at a different `α`. Two independent constructions — Goparaju et al.'s
zigzag+MRD precoding (Paper 76) and this paper's Gabidulin+Ye–Barg precoding — reach the same secrecy
capacity as a function of `α`, by different routes. Rawat's own related-work section confirms this is
not coincidental: it states this paper "establishes that... the bound in (14) [Pawar et al.'s upper
bound] is the exact characterization of the secrecy capacity of an MSR code" `[P §III]` — i.e. this
paper proves **optimality**, which Paper 76 does not claim for its own construction. Together: Paper
76 gives the practical construction, this paper gives the proof that nothing beats it.

### The sub-packetization this specific construction needs is not usable

`[P §III, Construction 2]` This paper's own achievability requires `α = (n−k)^{n−1}`. `[DERIVED]` At
Vyomanaut's parameters, `n−k = 40`, `n−1 = 55`: `α = 40^55` — a number with **88 decimal digits**.
For comparison, the number of atoms in the observable universe is estimated at roughly `10^80`. This
construction is optimal in a theorem but unbuildable in a system.

## Substitution at Vyomanaut's Parameters

`[DERIVED]` **The identity that resolves the practical question.** Since both papers achieve the same
`M_s(α)` formula, and Paper 76's construction achieves it at the vastly smaller `α = (n−k)k = 640`,
**Paper 76's construction should be adopted for implementation and this paper cited for the optimality
proof.** There is no reason to build this paper's specific Gabidulin-precoded MSR code when Goparaju
et al.'s zigzag+MRD code reaches the identical secrecy capacity at a sub-packetization six orders of
magnitude smaller (`640` vs `40^55`).

`[DERIVED]` **Repeating Paper 76's headline number here for continuity, since it is this paper's
formula too:** at `ℓ1=0, ℓ2=1`: `M_s = 15 × 0.975 × α = 14.625α`, expansion `56/14.625 = 3.829×`. See
Paper 76's Substitution section for the full working; not repeated in full here to avoid duplicate
derivation under §3.6's corpus-delta discipline.

## What This Paper Rules Out

- **Rules out the possibility that some cleverer construction beats the `(1−1/(n−k))^ℓ2` decay factor**
  for linear-repairable MSR codes at `d=n−1` — this is now a proved tight bound, not merely the best
  known achievable rate. Any future paper claiming a better secrecy-capacity/storage trade-off at these
  parameters for a linear-repairable MSR code would be claiming to beat a proved optimum, and should
  be read with that scepticism.
- **Rules out treating this paper's specific construction (Gabidulin+Ye–Barg) as the one to implement**
  — ruled out on sub-packetization grounds above, not on any deficiency in the secrecy proof itself.

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Paper 76's zigzag+MRD construction, cited against this paper's optimality proof | This paper's own Gabidulin+Ye–Barg construction | Identical secrecy capacity; `α=640` (buildable) vs `α=40^55` (not a number a system can have) |
| MSR point (`d=n−1`), tight bound proved here | Product-matrix MBR constructions (Shah et al., cited in `[P §I]` related work, not read) | This paper's tightness result is specific to MSR; MBR-point secrecy capacity is a separate, unread question |

## Breaks in Our Case

- **The tightness proof (this paper) and the practical construction (Paper 76) are two different
  papers, and nothing in the corpus states explicitly that Paper 76's construction achieves what this
  paper proves is optimal** ≠ **an ADR needs a single citable source for "we are using the optimal
  achievable secrecy capacity at these parameters"**
  → **Costed, resolved by this note.** The cross-reference is now made explicit here and should be
    cited jointly (Paper 76 for construction, Paper 80 for optimality) in any ADR that adopts this
    approach.

## Decisions Influenced

Joint primary input, with Paper 76, to **ADR-079 — Domain P Menu**. This paper's specific
contribution to that ADR is the **optimality** claim attached to Paper 76's construction — without it,
Paper 76's number is "the best we found"; with it, the number is "the best that is achievable by any
linear-repairable MSR code," a materially stronger claim for a council decision to rely on.

## Falsifiers

- **A proof obligation.** If a non-linear repair scheme is later shown to beat this bound (the paper
  notes Goparaju et al. — a different Goparaju paper than Paper 76, on Pareto optimality of linear
  vs non-linear MSR codes — already address this and find linear is Pareto-optimal among MSR codes
  generally `[P §III]`, but this was not independently verified in this reading), the "tight" claim
  narrows to "tight among linear-repairable schemes," which is what is claimed here, not "tight
  among all schemes."

## Disagreements

None — this paper is consistent with, and formally strengthens, Paper 76.

## Corpus Delta

Adds the optimality half of a result whose achievability half is Paper 76. **No existing note
requires correction.** The corpus now holds, for the first time, a matched achievability/optimality
pair for MSR secrecy capacity at general `ℓ`, which is a materially stronger position than either
paper alone.

## Open Questions

None new. This note resolves rather than raises a question — it answers "is Paper 76's construction
provably optimal" in the affirmative.
