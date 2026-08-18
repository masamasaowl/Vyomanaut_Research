# Paper 79 — Repairing Reed-Solomon Codes (Guruswami & Wootters; Dau, Duursma, Kiah & Milenkovic)

**Authors (paper A):** Venkatesan Guruswami, Mary Wootters (Carnegie Mellon University)
**Venue / Year (A):** STOC 2016 / full version cited as such
**Authors (paper B):** Hoang Dau (UIUC → Monash), Iwan M. Duursma (UIUC), Han Mao Kiah (Nanyang
Technological University), Olgica Milenkovic (UIUC)
**Venue / Year (B):** IEEE Transactions on Information Theory, Vol. 64, No. 10, October 2018 (extends
a 2017 ISIT paper)
**Track:** LTS
**Reading list:** Domain P / R-28 — Band 1, my addition — accept criterion was originally "a helper
contributes a function of its shard; no single party holds a decodable set"; **this note corrects
that framing before applying it** — see below.
**ADRs produced:** none — see "What This Paper Rules Out"
**Findings raised:** none for Domain P proper — retasks these two papers to a different domain
**Questions closed:** none · **Questions raised:** none
**Triage score:** paper A 5/10, paper B 4/10 (parameter reach 1 · trust model 0 · evidence 2 ·
actionability 1 · corpus delta 1, averaged) — **both mine-the-introduction under Domain P's own
scorecard once correctly scored on the "trust model" axis, which they score 0 on because neither
paper has a trust model at all.**

---

## Provenance

Read: Guruswami & Wootters' STOC 2016 conference paper in full, including the proofs of Theorems 1,
3, and 4 and the worked Facebook HDFS-RAID (14,10) example (§6). Read Dau, Duursma, Kiah &
Milenkovic's TIT 2018 journal version in full through §III (the two- and three-erasure trace-collection
constructions); the higher-erasure generalization sections were read for structure, not
line-by-line-verified.

## The correction this note makes before doing anything else

**Both papers are exclusively about repair bandwidth. Neither says anything about confidentiality.**
Guruswami & Wootters' entire motivation is stated in their own introduction: RS repair traditionally
requires downloading `k` full symbols to recover one lost symbol, which is *"wasteful: we have to
read `k` symbols from `F` when we only want one"* `[P-A §1]` — the problem is bandwidth, and the
paper's abstract promises `O(k)` bits instead of `Ω(k log k)` bits, again a bandwidth claim with no
adversary in the model at all. Dau et al. extend the same trace-collection technique to two and three
*simultaneous* erasures, again purely for bandwidth, with replacement nodes **collaborating with each
other** to complete repair — no secrecy claim anywhere in either paper.

This domain's earlier informal pass filed these two papers as confidentiality candidates ("Option A′
— repair RS(16,56) itself... bandwidth up 72%, information learned down to one symbol") and that
framing was wrong on the central point: in an exact-repair scheme, the replacement node is *supposed*
to learn the missing symbol exactly — that is the definition of exact repair, not a confidentiality
leak. What these papers reduce is how much data must be **downloaded** to compute that reconstruction,
not how much the reconstructing party is **allowed to know**. **Reclassifying both papers out of
Domain P's confidentiality menu and into a bandwidth-engineering role, feeding the `W` parameter in
Paper 78's Theorem 4 (Huang & Bruck's secure-repair bandwidth bound).**

## Problem Solved (bandwidth, as actually scoped)

`[P-A §1]` Guruswami & Wootters ask: instead of querying whole evaluations `f(α)` of a degree-`(k−1)`
polynomial, what if each queried node returns only a **partial** evaluation — a few elements of a
small subfield `B ≤ F`, rather than a full element of `F`? They show `O(k)` bits suffice to recover a
missing evaluation this way, against `Ω(k log k)` bits for full-symbol downloads.

`[P-B §I]` Dau et al. extend the same trace-collection idea from one erasure to **two and three**
simultaneous erasures, with replacement nodes exchanging additional information with each other in a
second ("collaboration") phase after an initial download phase.

## Key Findings

### Guruswami–Wootters' headline construction is scoped to a high-rate regime we are not in

`[P-A Thm 1]` The paper's main new theorem requires `k = (1 − 1/|B|)|F|` **and** the evaluation set
`A = F` (the full field is used as evaluation points). This forces a **high-rate** code: rate
`= 1 − 1/|B|`, which for any subfield size `|B| ≥ 2` is at least `1/2`. Vyomanaut's rate is
`k/n = 16/56 ≈ 0.286`. `[DERIVED]` Solving `1 − 1/|B| = 16/56` for `|B|` gives `|B| = 1/(1−0.286) ≈
1.4` — not an integer, let alone a prime power `≥2`. **Theorem 1 cannot be instantiated at
Vyomanaut's rate at all.** The paper is explicit that this is its chosen regime, not an oversight:
*"Our main focus is constant-rate RS codes with `A=F`... t is small compared to `n−k`"* `[P-A §2.4]`
— high rate, low sub-packetization, by design.

### The paper's Regime 1 (large `t`) is the classical cut-set bound, not new to this paper

`[P-A §2.4]` The paper separately discusses a second regime, "large `t`," where the correct bandwidth
is the well-known cut-set formula `b = td/(d+1−k)` — but attributes the lower bound to prior work
(Dimakis et al., already Paper 55 in the corpus) and the matching upper bound to other prior work,
not to this paper's own contribution. **This domain's earlier informal pass used exactly this
formula, mislabelled as a Guruswami–Wootters result — it is not; it is the standard MSR/regenerating-code
trade-off, already documented in Paper 55.** Corrected here.

### Dau et al. cap out at three simultaneous erasures, with collaboration required

`[P-B Fig. 1, §I]` The two-erasure worked example requires replacement nodes to exchange information
with each other after their initial download — a genuinely different (and more complex) protocol
shape than single-erasure repair. The paper extends to three erasures; nothing in the abstract or
introduction claims a general-`t`-erasure result.

## Substitution at Vyomanaut's Parameters

`[DERIVED]` **Theorem 1 instantiation check, worked explicitly:** `k=(1−1/|B|)|F|` requires, at our
`k=16`, either `|F|` chosen to make this exact for some prime-power `|B|`. Trying `|B|=2`:
`k=0.5|F|` → `|F|=32`, giving `n=|F|=32 ≠ 56`. Trying `|B|=4`: `k=0.75|F|` → `|F|=21.33`, not an
integer. **No subfield size makes Theorem 1's construction land on `(n,k)=(56,16)` exactly** — this
is not a rounding inconvenience, it is a structural mismatch between a high-rate-only construction and
a low-rate code.

`[DERIVED]` **The batching mismatch, for Dau et al.:** ADR-076's lazy `r0` gate triggers repair of up
to `r − r0 = 32` simultaneous shard losses. Dau et al.'s furthest-extended result covers **three**.
`32/3 ≈ 10.7×` beyond what has been published in this line of work. **No construction in either paper
addresses our actual batch size**, at any rate.

## What This Paper Rules Out

- **Rules out citing either paper for a confidentiality claim of any kind.** Corrected above; this is
  the note's central finding.
- **Rules out the specific `(n=56,t=2,q=16)` GF(256)-as-degree-2-over-GF(16) instantiation this
  domain's earlier informal pass proposed**, on the rate-mismatch grounds shown in Substitution.
- **Rules out treating "GW-style repair" as a drop-in bandwidth fix for the batched 32-shard lazy
  repair scenario** — the multi-erasure line of work (Dau et al. and its citation descendants) tops
  out at 3, not 32, in the corpus as currently held.
- **Adjacent-not-this, precisely as `reading-list-LTS.md`'s own screen predicted:** these are
  "MSR bandwidth-optimality results. They minimise bytes moved, not who can decode" — the domain's own
  pre-written reject line for this exact class of paper. The earlier informal pass should have
  screened against this line before drafting and did not; corrected now.

## Trade-offs

| Role (corrected) | Not this role |
|---|---|
| Bandwidth-efficiency input to Huang & Bruck's `W` parameter (Paper 78, Theorem 4), IF a construction at our actual `(n,k)` and batch size existed | A standalone Domain P confidentiality candidate |
| A proof-of-concept that sub-symbol repair *can* beat naive whole-shard download for RS, motivating a future search for a construction at our parameters | A citable number for Vyomanaut's repair bandwidth today |

## Breaks in Our Case

- **Theorem 1 requires high rate (`k=(1−1/|B|)|F|`, `A=F`)** ≠ **Vyomanaut's rate is 0.286, low**
  → **Fatal for direct instantiation.** No adaptation salvages this without a different construction;
    flagged as an open engineering question for whoever eventually designs a bandwidth-efficient
    RS(16,56) repair scheme, not something this literature note can resolve.

- **Dau et al.'s constructions cover 2–3 simultaneous erasures** ≠ **our lazy-repair batch is up to 32**
  → **Fatal at our batch size**, useful only as a demonstration that the general technique (trace
    collection with inter-node collaboration) scales *conceptually* past one erasure — not as a
    number we can cite.

## Decisions Influenced

None directly. Neither paper produces an ADR for Domain P. Their correct role — bandwidth input to
whichever secure-repair protocol Domain P eventually adopts — is noted in `ADR-079`'s context section
as a flagged future dependency, not a decision made now.

## Falsifiers

- **A measurement / literature gap.** If a future search finds a bandwidth-efficient RS repair
  construction stated for low-rate codes (`k/n < 0.3`) or for erasure counts approaching 32, this
  note's "Fatal" classification for direct application is void and the construction should be read
  next, feeding `W` into Paper 78's bound.

## Disagreements

**With this domain's own prior triage pass**, recorded as a self-correction rather than a third-party
disagreement: the earlier framing of these papers as "Option A′" providing confidentiality is
withdrawn. See the correction section above for the full reasoning.

## Corpus Delta

**Corrects paper-55's territory boundary**: the classical cut-set formula `b=td/(d+1−k)` belongs to
Paper 55 (Dimakis et al.), not to this note's two papers — no edit needed to Paper 55 itself, but any
future citation of "GW-style repair bandwidth" in an ADR should cite Paper 55 for the classical
formula and these two papers only for the specific high-rate sub-symbol trace technique, which does
not apply at Vyomanaut's parameters. **Withdraws** the informal "Option A′" framing from this domain's
prior triage pass; no standing document asserted it as an ADR-level claim, so no ADR requires
correction.

## Open Questions

None raised. This note's function is a correction and a reclassification, not a new open question —
the two papers are demoted from Domain P's confidentiality menu and are not carried forward into
`ADR-079`.
