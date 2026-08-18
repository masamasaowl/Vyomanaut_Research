# Paper 81 — Optimal Locally Repairable and Secure Codes for Distributed Storage Systems

**Authors:** Ankit Singh Rawat, Onur Ozan Koyluoglu, Natalia Silberstein, Sriram Vishwanath
**Venue / Year:** IEEE Transactions on Information Theory, Vol. 60, No. 1, January 2014
**Track:** LTS
**Reading list:** Domain P / R-27 — Band 1, must-read table item — accept criterion *"states secrecy
as a function of (n,k,d) with the storage cost of the guarantee"*
**ADRs produced:** contributes to ADR-079 (Domain P Menu)
**Findings raised:** F-LTS-17 (locality trades secrecy cost non-monotonically with repair-group size)
**Questions closed:** none · **Questions raised:** none
**Triage score:** 8/10 (parameter reach 2 · trust model 2 · evidence 2 · actionability 1 · corpus
delta 1)

---

## Provenance

Read: the full 20-page journal article. Two independent results are extracted and used here:
(1) §III's secure-MSR bound and construction (`ℓ2 ≤ 2` only), and (2) §V–VI's locally-repairable-code
minimum-distance bound and its combination with secrecy (Theorem 33). Constructions in §IV
(Gabidulin-code-based LRC construction, Construction I) were read for structure and the theorem
statements that depend on them, not independently re-derived.

## Problem Solved

Two DSS design goals — small **locality** (repairing a failed node touches few other nodes) and
**secrecy** against a passive eavesdropper — had been studied separately. This paper is the first in
the corpus to design codes for both jointly, and its introduction states the reason they interact:
*"local-repairability and security against eavesdropping attack go hand in hand... a joint design of
both features can prove to be particularly useful"* `[P §I]` — smaller repair groups mean an
eavesdropper who watches one repair event learns less, an intuition this paper turns into a theorem
(and, as shown below, one this domain's earlier informal reasoning got backwards).

## Key Findings

### Part 1 — Secure MSR bound (§III), now superseded

`[P §III, Corollary 16]` For an MSR code at `d=n−1` employing interference alignment, and `ℓ2 ≤ 2`:

```
M_s ≤ (k−ℓ1−ℓ2)(α − θ(α,β,ℓ2))     where θ(α,β,1)=β,  θ(α,β,2)=2β − β/(n−k)     [P eq. 18–19]
```

This is the same secrecy-capacity question Papers 76 and 80 answer for **all** `ℓ2`, not just `ℓ2≤2`.
**This paper's own §III result is superseded by Paper 80 (Rawat, 2017, same lead author) for the MSR
question** — the paper is explicit that the `ℓ2≤2` restriction is a known limitation of the 2014
technique, later removed by the author's own 2017 follow-up. Not re-derived further here; recorded
for provenance completeness only.

### Part 2 — The LRC minimum-distance bound (§V, Theorem 21)

`[P Thm 21]` For an `(r,δ,α)` LRC of length `n` storing `M` symbols:

```
dmin(C) ≤ n − M/α + 1 − (⌈M/(rα)⌉ − 1)(δ − 1)                                    [P Thm 21, eq. 21]
```

At the scalar case `M=k, α=1`: `dmin(C) ≤ n − k + 1 − (⌈k/r⌉ − 1)(δ − 1)` — the Gopalan-family LRC
Singleton-type bound. `[P §V-B, Construction I]` The bound is **achieved** by a construction
concatenating a Gabidulin code with local MDS array codes per local group.

### Part 3 — Secrecy in locally repairable DSS (§VI, Theorem 33) — the load-bearing result

`[P §VI, Theorem 33]` For an `(r, δ=2, α, β=α, d=r)` LRC, dmin-optimal, against an `(ℓ1,ℓ2)`-
eavesdropper:

```
M_s = [μr + h − (ℓ2·r + ℓ1)]^+ · α                                                    [P eq. 47]
where μ = ⌊(n−dmin+1)/(r+1)⌋,  h = (n−dmin+1) − (r+1)μ,  and dmin from Theorem 21 with equality
```

**The structural reason this formula has `ℓ2·r`, not just `ℓ2`, in it is the paper's own stated
insight and is the finding that matters most here:** *"eavesdropping `r` nodes in a group gives the
eavesdropper all the information that the group has to offer... [an eavesdropper] that observes a
single node repair in the local group [learns] all the information in the local group"* `[P §VI]`.
**Observing one repair event inside a local group of size `r` reveals the entire group — `r` symbols'
worth, not one.** This is an all-or-nothing property at the group level, and it means the secrecy
*cost* of a single observed repair scales with `r`, the very parameter chosen to reduce repair
bandwidth.

## Substitution at Vyomanaut's Parameters

`[DERIVED]` Non-secure baseline, all three localities, verified to satisfy `μr + min(h,r) = k = 16`
by construction (confirms `[P §V-B]`'s claim that this LRC family attains rate `k/n` exactly, same
as RS(16,56)):

| `r` | `dmin` (Thm 21, `δ=2`) | `μ` | `h` | non-secure `M` | non-secure expansion |
|---|---|---|---|---|---|
| 2 | 34 | 7 | 2 | `14α+2α=16α` | `56/16 = 3.5×` |
| 4 | 38 | 3 | 4 | `12α+4α=16α` | `56/16 = 3.5×` |
| 8 | 40 | 1 | 8 | `8α+8α=16α` | `56/16 = 3.5×` |

`[DERIVED]` Secure capacity at `ℓ1=0, ℓ2=1` (single-repairer blindness, this domain's minimal target),
via Theorem 33 (eq. 47):

| `r` | `M_s = [μr+h−r]^+·α` | working | secure expansion `56α/M_s` |
|---|---|---|---|
| 2 | `[14+2−2]^+α = 14α` | `μr+h−(1·2+0)=14` | `56/14 = 4.00×` |
| 4 | `[12+4−4]^+α = 12α` | `μr+h−(1·4+0)=12` | `56/12 = 4.667×` |
| 8 | `[8+8−8]^+α = 8α` | `μr+h−(1·8+0)=8` | `56/8 = 7.00×` |

**This inverts the ordering this domain's earlier informal pass assumed.** The prior pass picked
`r=4` as "the sweet spot" on durability grounds alone (37 tolerable losses vs RS's 40, a modest `−3`
penalty) without accounting for Theorem 33's group-wide-revelation effect. Once secrecy cost is
priced in, **`r=2` is cheapest for blindness (`4.00×`) despite having the worst durability of the
three (33 tolerable losses, `−7` against RS's 40)**, and `r=8` — the *best*-durability option
(`−1` only) — is the *worst* for secrecy cost (`7.00×`, worse than the plain-MSR options in Papers 76
and 80 at `3.83×`). **Locality and secrecy genuinely trade against each other in the direction this
paper's own theorem predicts, and the earlier pass reasoned about only one side of that trade.**

## What This Paper Rules Out

- **Rules out treating "smaller repair footprint" and "cheaper blindness" as the same design axis.**
  They point in the same direction only up to a point (§I's own framing suggested they align); Theorem
  33 proves the *secrecy cost coefficient* is `r` itself, so past a certain group size the locality
  benefit (fewer nodes touched) is exactly cancelled by the secrecy penalty (more revealed per touch).
- **Rules out `r=8` as competitive with the plain-MSR options (Papers 76/80) on any axis this note has
  priced.** At `7.00×` it is worse than Papers 76/80's `3.83×` on secrecy-adjusted storage **and**
  worse than `r=2`/`r=4` on that same measure, while its durability advantage (`dmin=40`, matching RS
  exactly) is real but does not compensate.
- **Adjacent-not-this, partially:** §III's secure-MSR result is superseded by Paper 80 and should not
  be cited for the MSR-point question; only §V–VI (locality) is this paper's live contribution to
  Domain P.

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| `r=2` LRC, secure at `ℓ2=1` | `r=4` (this domain's earlier pick) | `4.00×` vs `4.667×` — cheaper blindness, but repair now touches only 2 nodes at the cost of `dmin=34` (33 tolerable losses) vs `r=4`'s 37 |
| Plain secure MSR (Papers 76/80, `3.83×`) | Any LRC option in this paper | Plain MSR remains cheaper than every locality option evaluated here at `ℓ2=1`; the locality construction's advantage (fewer nodes touched per repair) is a **latency/liveness** benefit under churn (ADR-021), not a storage or secrecy-cost benefit |

## Breaks in Our Case

- **Theorem 33 requires `δ=2, β=α, d=r`** ≠ **Vyomanaut has not adopted an LRC layer at all; RS(16,56)
  is a plain MDS code with `d` up to `n−1`, no locality structure**
  → **Open.** Every number above describes a *candidate* architecture, not the current one. Adopting
    it requires the codec change this domain's earlier framing called "Option D/E" — a real code-family
    change, unlike the plain-MSR options.
- **The `ℓ2·r` cost is proved for a single repair event; ADR-076's lazy gate batches up to 32 shard
  losses at once** ≠ **this paper's Theorem 33 is stated for one observed repair, not a batch**
  → **Open, and worse here than for the plain-MSR case (Paper 76).** If a batched lazy-repair cycle
    touches multiple local groups simultaneously, and a single party observes the batch, the
    `ℓ2·r` term could scale with *both* the number of groups touched and `r` itself — this is not
    derived in the paper and is flagged, not resolved, here. Given Paper 76's Q79-1 already
    identifies batching as the dominant risk for the plain-MSR options, and this LRC family appears
    to make batching *worse* (each group-repair reveals `r` symbols, not 1), **LRC options are
    provisionally the most batching-exposed candidates in the menu** until Q79-1's election-
    distribution question is resolved.

## Decisions Influenced

Input to **ADR-079 — Domain P Menu**, contributing the locality-tradeoff row (`r=2/4/8`) with
corrected numbers. Does not itself select an option.

## Falsifiers

- **A proof obligation.** Whether Theorem 33 generalizes to multiple simultaneously-observed local
  groups (the batched-repair case) is unproved either way; a derivation either confirms the
  worse-than-plain-MSR concern above or bounds it more favourably.
- **A parameter change.** All secrecy-cost numbers above assume `δ=2`. Vyomanaut's actual desired
  fault tolerance per local group (`δ`) has not been decided; `δ>2` changes every number in the table
  and was not evaluated in this reading.

## Disagreements

**With this domain's own prior triage pass**, recorded as a self-correction: the earlier "`r=4`
sweet spot" pick is withdrawn as the *cheapest* option; it may still be preferred on other grounds
(a middle durability/secrecy balance) but is not cheapest on either axis alone.

## Corpus Delta

Adds **F-LTS-17**: locality and repair-time secrecy cost move in the *same* direction as group size
`r`, not opposite directions — a genuinely non-obvious result worth naming as a standing finding
since it will recur if any future locality-based code is considered elsewhere in the corpus (e.g.
Domain C, closed by Council 2, may want to revisit this if locality is ever reconsidered there).
Corrects this domain's own prior informal locality pick (see Disagreements).

## Open Questions

**Manual step for the reader:** no new question ID required. Fold the batching concern into
**Q79-1**'s existing text (raised in Paper 76's note) by appending: *"...and whether the LRC family's
`ℓ2·r` group-wide-revelation property (Paper 81, Theorem 33) makes locality-based options
strictly worse-exposed to batching than the plain-MSR options, pending a multi-group generalization
of Theorem 33."*
