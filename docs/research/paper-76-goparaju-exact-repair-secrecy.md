# Paper 76 — Data Secrecy in Distributed Storage Systems Under Exact Repair

**Authors:** Sreechakra Goparaju, Salim El Rouayheb, Robert Calderbank (Princeton), H. Vincent Poor (Princeton)
**Venue / Year:** IEEE Transactions on Information Theory (submitted version read; also appears as
arXiv:1207.3335) | conference precursor at Allerton 2012
**Track:** LTS
**Reading list:** Domain P / R-27 — Band 1 — accept criterion *"states secrecy as a function of
(n,k,d) with the storage cost of the guarantee"*
**ADRs produced:** ADR-079 (Domain P Menu, jointly with the rest of this band)
**Findings raised:** none new · sharpens F-LTS-15
**Questions closed:** none · **Questions raised:** Q79-1 (batch-repair disclosure ceiling)
**Triage score:** 9/10 (parameter reach 2 · trust model 2 · evidence 2 · actionability 1 · corpus delta 2)

---

## Provenance

Read: the full IEEE Xplore version (11 pages including proofs), all sections including the proof of
Theorem 3 in §V. The paper's own related-work section places it against Pawar/El Rouayheb/Ramchandran
(2011) and Shah/Rashmi/Kumar/Ramchandran's product-matrix results; both are treated here as cited
context rather than re-read, per `reading-list-LTS.md`'s note that this paper displaces Pawar as the
domain's exact-repair anchor and Pawar demotes to a forward-citation seed.

## Problem Solved

Every prior secrecy-capacity result for regenerating codes restricted the eavesdropper to observing
the repair of **at most two** compromised nodes `[P §I, Related Work]`. This paper removes that cap:
it gives an exact, closed-form secrecy capacity for **any** number of compromised nodes, at the
`d = n−1` repair degree, and a new upper bound for `d < n−1`.

The paper opens with the exact scenario Domain P exists to address, stated as its own motivating
example: a `(n,k,d)=(4,2,2)` system, secure against one eavesdropped node, where **repairing a
failed node leaks the file** — the replacement node downloads two packets, an eavesdropper watching
that download recovers the file, even though the system was perfectly secure before the failure
`[P §I, Fig. 1]`. This is F-69 stated four years before Vyomanaut's Council 2 rediscovered it
independently by reading the codebase.

## Key Findings

### The exact secrecy capacity at d = n−1

`[P Thm 3]` For an `(n,k,d)`-DSS with `d = n−1`, employing an optimal-bandwidth (MSR) linear code
with exact repair of systematic nodes, against an eavesdropper who statically observes `ℓ1` nodes'
stored content and observes the repair-time download for `ℓ2` systematic nodes (`ℓ1+ℓ2 < k`
required — Eq. 4), the secrecy capacity is

```
C_s(α) = (k − ℓ1 − ℓ2) · (1 − 1/(n−k))^ℓ2 · α                              [P Thm 3, eq. 8]
```

achievable for **all** `(ℓ1, ℓ2)` satisfying the precondition, via a construction that precodes an
`(n,k)` zigzag code `[P §III, Thm 17]` with a maximum-rank-distance (MRD) code, at storage
`α = (n−k)·k` `[P Thm 3]`.

### The precondition is a hard wall, not a soft degradation

`[P Thm 2, eq. 4]` The result requires `ℓ1 + ℓ2 < k` strictly. At `ℓ1+ℓ2 = k` the theorem does not
apply, and the construction's own structure (§IV's subspace-intersection argument) makes clear why:
each additional repair-observed node adds a genuinely new linear constraint on the eavesdropper's
knowledge, and once `k` such constraints accumulate, the eavesdropper's view spans the same space a
legitimate data collector's would.

### The mechanism: subspace intersection, not raw byte count

`[P §IV, Lemma 1]` The paper's key technical tool bounds the **dimension of the sum of subspaces**
an eavesdropper accumulates across repeated repair observations, not a naive count of downloaded
bytes. For the two-parity toy case (`β = α/2`), each additional repair subspace reveals only the
*new* information in it — `Vi,1 + Vi,2` cannot exceed `α/4` of overlap `[P §IV]` — giving the
geometric-decay factor `(1 − 1/(n−k))^ℓ2` that appears in the closed form. This is the formal version
of the "accumulation" argument this domain's earlier informal pass made loosely; here it is a proved
lower bound on leaked dimension, not an estimate.

## Substitution at Vyomanaut's Parameters

Standing parameters: `n = 56, k = 16, d = n−1 = 55` (unconstrained repair pool; the `r0`-gated floor
of `s+r0 = 24` live shards is addressed separately below, since Theorem 3 assumes `d = n−1` exactly).

`[DERIVED]` At `ℓ1 = 0` (no static-node compromise), `ℓ2 = 1` (blindness against a single repair
event — the minimal, load-bearing target: does the party performing one repair learn the package?):

```
C_s(α) = (16 − 1) · (1 − 1/40)^1 · α = 15 · 0.975 · α = 14.625α
```

Naive (non-secure) MDS capacity at the same `α` is `M = kα = 16α`. Expansion needed to store the
**same plaintext** `M` securely:

```
expansion = n·α_unit / C_s(α_unit) = 56 / 14.625 = 3.829×
```

against RS(16,56)'s existing baseline of `3.5×`. **Cost of blindness against one repair event: +9.4%
storage over the status quo.** Working shown: `56/14.625 = 3.8291...`, rounds to `3.83×`.

`[DERIVED]` At `ℓ2 = 15` (one below the wall, `ℓ1=0`):

```
C_s(α) = (16−15)·(1−1/40)^15 · α = 1 · 0.6786 · α = 0.6786α
expansion = 56/0.6786 = 82.5×
```

`[DERIVED]` At `ℓ2 = 16` (the wall — `ℓ1+ℓ2 = k`): the precondition `ℓ1+ℓ2 < k` fails and the
theorem gives no guarantee at all; `C_s → 0` is the honest reading, not an extrapolation of the
formula past its domain.

### The batching hazard, made precise

`[DERIVED]` This domain's earlier informal accumulation argument (219 regenerations/segment/3yr,
73/year) reasoned about *how many times* a segment is repaired, not *how many of those repairs one
observer sees at once*. Theorem 3's precondition answers the question that actually matters: **a
single observer who watches ℓ2 = 16 or more repair events for the same segment — whether spread over
years or, worse, batched into one lazy-repair cycle — exceeds the wall and the secrecy guarantee
provides nothing.** Under `ADR-076`'s lazy `r0` gate (`s+r0=24` live, i.e. up to `r−r0=32` shards
repaired in one batched event), **a single batched lazy-repair cycle can expose up to 32 repair
events to whichever party executes it — twice past the wall at `ℓ2=16`.** This is sharper and more
alarming than the earlier informal argument: it is not that repairs *accumulate* toward a threshold
over a segment's lifetime; it is that **one lazy-repair cycle alone can exceed the threshold outright**,
independent of lifetime totals, if a single party executes all 32 repairs in that cycle. Raised as
**Q79-1**.

## What This Paper Rules Out

- **Rules out treating `d < n−1` with the same closed form.** `[P §V, "For the other cases"]` the
  paper gives only an upper bound for `d < n−1`, not an achievability result — meaning any repair
  topology that keeps live-helper count below `n−1` (which `ADR-076`'s `r0` gate always does, since
  `s+r0=24 < 55`) is **not** covered by Theorem 3's achievability claim. The `α = (n−k)k` construction
  is proved *only* for the full-pool case. Applying it under the `r0` gate is an extrapolation this
  paper does not license — flagged as a genuine gap, not resolved here.
- **Rules out any hope that repair rarity substitutes for a per-event bound.** The precondition is
  per-cumulative-observer, not per-unit-time. `reading-list-LTS.md`'s own substitution instruction
  ("evaluate against a repair event that is rare... it changes what a partial mitigation is worth")
  is now falsified by this paper's own theorem: rarity does not relax the `ℓ1+ℓ2<k` wall, because the
  wall counts *observations*, not *rate*.
- **Near-miss class avoided:** this is not a bandwidth paper (contrast Papers 79's Guruswami–Wootters
  and Dau et al., which prove nothing about secrecy at all — see that note's own "What This Paper
  Rules Out"). Do not cite this paper for repair bandwidth numbers; its `β = α/(d−k+1)` is the
  standard MSR trade-off, not a contribution of this work.

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Precoded zigzag + MRD (this paper) | Rawat's Gabidulin + Ye–Barg (Paper 80) | Same secrecy capacity formula, achievable at `α=(n−k)k=640` — a real, implementable sub-packetization, vs Rawat's `α=(n−k)^(n−1)=40^55` for the *general* construction. This paper is the one to build, not merely cite. |
| Exact repair | Functional repair | Required — Vyomanaut's systematic property and chunk-ID stability (ADR-073) depend on exact repair; functional repair would change stored content identity on every repair, breaking `chunk_id = SHA-256(chunk_data)`. |
| `ℓ2 = 1` target (single-repairer blindness) | `ℓ2 = 11` (ASN-cap-scale blindness) | `82.5×` vs `3.83×` expansion. F-34's ASN-collusion adversary is unaffordable under this construction; only single-elected-repairer blindness is economically viable. Confirms this domain's prior scoping decision to drop ADR-022/F-34 from Domain P's header. |

## Breaks in Our Case

- **Theorem 3 assumes `d = n−1` (full-pool repair, all 55 survivors participate)** ≠ **ADR-076's `r0`
  gate caps live helpers at `s+r0 = 24`**
  → **Fatal, as stated.** The achievability proof is not shown to hold at `d=24`. Either the `r0`
    gate must be raised to `d=n−1` for any segment adopting this construction (defeating the gate's
    bandwidth purpose), or the construction must be re-derived at `d<n−1`, which the paper explicitly
    leaves open (§V, "characterization... for `d<n−1`... remains an open problem" for general systems,
    though it does give an upper bound). **This is the single largest open item this note raises.**

- **The precondition `ℓ1+ℓ2<k` counts observations, not calendar time** ≠ **ADR-076's batched
  lazy-repair executes up to 32 shard-regenerations in one cycle, all visible to whichever party runs
  the batch**
  → **Costed, quantified above as Q79-1.** If repair execution is centralized (as `F-LTS-08` finds it
    currently is — the microservice), a single lazy-repair cycle can expose `ℓ2` up to 32, twice past
    the `ℓ2<16` wall. If repair execution is distributed one-shard-per-election (`ADR-075`), each
    election is a separate observer and the relevant count becomes *per-provider cumulative elections*
    — the `Poisson(3.9)` argument from this domain's earlier pass, which stays valid **only if no
    single party executes the batch**. This makes the election-distribution question (already flagged
    as needing to run in parallel with this reading) load-bearing for whether the batching hazard or
    the lifetime-accumulation hazard is the binding constraint.

## Decisions Influenced

No ADR is closed by this paper alone. It is a primary input to **ADR-079 — Domain P Menu** (new,
Proposed), which records this construction as the tightest-known achievable point and cannot itself
select among the domain's options — that remains a council decision per this domain's own prior
scoping (`reading-list-LTS.md` §5, council needed for "genuine tradeoff... no dominant answer").

## Falsifiers

- **A proof obligation.** The `d=n−1` restriction is falsified as a blocker if someone shows Theorem
  3's zigzag+MRD construction (or an equivalent) achieves the same `C_s(α)` formula at `d = s+r0 = 24`.
  Absent that proof, adopting this construction under the `r0` gate is unverified.
- **A parameter change.** If `ADR-075`'s replacement-provider selection is shown to concentrate
  repair execution on a small recurring set of providers (rather than uniform election), the
  `Poisson(3.9)` mitigation for lifetime accumulation fails and the batching hazard (Q79-1) becomes
  the sole binding constraint regardless of election policy.

## Disagreements

None found in the corpus at the time of this reading — no existing Vyomanaut document asserts a
secrecy-capacity claim this paper contradicts.

## Corpus Delta

New to the corpus: the first exact (not merely bounded) secrecy-capacity result at general `ℓ`,
displacing Pawar (2011) as this domain's anchor for the `d=n−1` regime. Cross-validates with Paper
80 (Rawat 2017) — both report the identical `C_s(α) = (k−ℓ1−ℓ2)(1−1/(n−k))^ℓ2 α` formula via
independent constructions (zigzag+MRD here; Gabidulin+Ye–Barg there); see Paper 80's note for the
cross-check working. No existing note is corrected by this one.

## Open Questions

**Manual step for the reader:** add to `docs/research/open-questions.md`:

```
### Q79-1 — Does a single batched lazy-repair cycle (up to 32 shards under ADR-076's r0 gate)
expose ℓ2 up to 32 to whichever party executes it, exceeding Goparaju et al.'s secrecy-capacity
wall at ℓ2 = k = 16 in one cycle rather than accumulating toward it over a segment's lifetime?

**Raised by:** Paper 76 (Goparaju, El Rouayheb, Calderbank & Poor) Theorem 3's precondition
ℓ1+ℓ2 < k, evaluated against ADR-076's batch size.
**Status:** open — blocks ADR-079's council decision
**Blocked on:** whether repair execution is centralized (F-LTS-08 says currently yes) or
distributed one-shard-per-election (ADR-075's intent). If centralized, Q79-1 is answered "yes,
every batch" and the accumulation argument from the prior triage pass is superseded by this
sharper, worse bound.
```

Also note in Q74-1's existing entry: this paper's Theorem 3 does not resolve Q74-1 (it assumes a
"perfectly secure" MSR construction is *adopted*; it does not analyse whether AONT-RS-as-deployed
satisfies that construction's preconditions — see the Paper 16 LTS addendum for that half).
