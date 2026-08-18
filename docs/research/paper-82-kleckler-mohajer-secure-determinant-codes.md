# Paper 82 — Secure Determinant Codes: A Class of Secure Exact-Repair Regenerating Codes

**Authors:** Michelle Kleckler, Soheil Mohajer (University of Minnesota)
**Venue / Year:** conference version (venue not stated on the PDF's own pages beyond author/affiliation
block; page-header numbering — pp. 211–216 — and reference style match an ISIT-class proceedings.
**Provenance flag: not confirmed against IEEE Xplore metadata in this reading.**)
**Track:** LTS
**Reading list:** Domain P / R-27 — Band 1, must-read table item, requested introduction supplied by
Karma — accept criterion *"states secrecy as a function of (n,k,d) with the storage cost of the
guarantee"*
**ADRs produced:** contributes to ADR-079 (Domain P Menu)
**Findings raised:** none new **Questions closed:** none · **Questions raised:** Q79-3 (artefact
version mismatch)
**Triage score:** 7/10 (parameter reach 2 · trust model 2 · evidence 1 · actionability 1 · corpus
delta 1)

---

## Provenance

Read: the full PDF, 6 pages including proof sketches (§V is a security proof for the general
construction; read for structure, individual steps not independently re-verified). Theorem 1's
formula (eq. 6) was **misread on first pass** from `pdftotext`'s linearized extraction — nested
binomial coefficients rendered ambiguously — and was re-verified by rendering the source page as an
image and reading the typeset formula directly. The corrected formula is used throughout this note;
see Substitution for the working.

**Version mismatch, disclosed per §3.2:** `reading-list-LTS.md`'s must-read table cites this work as
Elmahdy, Kleckler & Mohajer, *IEEE Transactions on Information Theory*, Vol. 69, No. 3, March 2023 —
a **three**-author journal paper covering **both** Type-I and Type-II eavesdroppers. The PDF supplied
and read here is a **two**-author (Kleckler, Mohajer only) version covering **Type-I only**, with a
conjectured-but-unproved optimality claim (*"we conjecture that the upper bound derived here is
optimal"* `[P §II]`) rather than the journal version's presumably-proved result. This is almost
certainly the ISIT-class conference precursor to the 2023 journal paper. **The Type-II analysis this
domain specifically wanted — the eavesdropper model that most closely matches an ASN-scale or
repair-observing adversary — is not in this artefact and was not obtained.** Recorded as Q79-3;
Type-II content should not be assumed present when this paper is cited.

## Problem Solved

Determinant codes `[P, citing Elyasi & Mohajer 2016]` are an exact-repair regenerating code family for
`(n, k=d, d)` systems achieving a piecewise-linear storage/bandwidth trade-off curve with `d` corner
points (`mode m ∈ [d]`), matching known-optimal points where those are known. This paper adds a
**Type-I secrecy guarantee** to the family: an eavesdropper with static access to up to `ℓ < k` nodes'
stored content learns nothing, at every mode `m` simultaneously — the whole trade-off curve, not just
one point on it.

## Key Findings

### The corrected achievability formula

`[P Thm 1, eq. 6]`, verified against the typeset page image:

```
F_s^(m) = (d−ℓ)·C(d,m) − C(d,m+1) + C(ℓ,m+1)
α^(m)   = C(d,m)
β^(m)   = C(d−1,m−1)
```

achievable for all `m ∈ [d]`. **This is not the formula this note's own earlier pass used before the
image re-check** — the earlier pass applied `(d−ℓ)` as a multiplier across the entire bracketed
difference rather than to `C(d,m)` alone. The corrected formula is used throughout Substitution below;
every number differs from the earlier pass.

### The construction requires k = d, sacrificing the ability to choose repair degree independently

`[P §II]` The whole paper is scoped to `k=d` systems. Repairing a node under this construction always
contacts exactly `k` helpers, never fewer and never the full `n−1` pool — a fixed, non-adjustable
repair degree, unlike the MSR options in Papers 76/80 (`d=n−1`) or the LRC options in Paper 81
(`d=r`, tunable).

## Substitution at Vyomanaut's Parameters

`[DERIVED]` At `d=k=16, ℓ=1` (single-repairer blindness). Working shown for `m=1` and `m=8`; the full
table follows the same method.

`m=1`: `F_s = 15·C(16,1) − C(16,2) + C(1,2) = 15·16 − 120 + 0 = 120`. `α=C(16,1)=16`, `β=C(15,0)=1`.
Storage expansion `= n·α/F_s = 56·16/120 = 896/120 = 7.467×`.

`m=8`: `F_s = 15·C(16,8) − C(16,9) + C(1,9) = 15·12870 − 11440 + 0 = 181,610`. `α=C(16,8)=12870`,
`β=C(15,7)=6435`. Expansion `= 56·12870/181610 = 720720/181610 = 3.968×`.

Full table, `d=k=16, ℓ=1`, repair-download fraction given as `dβ/(kα)` (bytes moved during one repair
as a fraction of a full naive re-fetch of the segment — note `β/α = m/d` exactly, a clean identity
verified independently below):

| `m` | `F_s` | `α` | `β` | expansion `56α/F_s` | repair fraction `m/d` |
|---|---|---|---|---|---|
| 1 | 120 | 16 | 1 | **7.467×** | 6.25% |
| 2 | 1,240 | 120 | 15 | **5.419×** | 12.5% |
| 4 | 22,932 | 1,820 | 455 | **4.446×** | 25% |
| 8 | 181,610 | 12,870 | 6,435 | **3.968×** | 50% |
| 15 | 239 | 16 | 15 | **3.749×** | 93.75% |
| 16 | 15 | 1 | 1 | **3.733×** | 100% |

`[DERIVED]` The `β/α = m/d` identity: `C(d−1,m−1)/C(d,m) = m/d` is a standard binomial identity,
confirmed numerically at every row above (e.g. `m=8`: `6435/12870 = 0.5 = 8/16` ✓). This means **the
mode `m` is literally the fraction of a full re-fetch that repair costs**, a clean and checkable
design dial.

### Where this sits against the rest of the menu

`[DERIVED]` Every mode in this table costs **more** than Papers 76/80's plain-MSR construction at
`3.83×`, except modes `m≥15` (essentially the MSR-point-equivalent modes, which converge to `3.73–
3.75×` — matching Papers 76/80 to within 3%, as expected since large-`m` determinant codes approach
the MSR point). **The determinant-code family is dominated by the plain-MSR construction at every
mode except the two modes closest to its own MSR limit**, where it is roughly equivalent. Its genuine
advantage is elsewhere: at low `m`, repair bandwidth is very cheap (`6.25%` at `m=1`) in exchange for
high storage — a real corner of the trade-off space the MSR-only construction does not offer.

## What This Paper Rules Out

- **Rules out picking a low `m` (cheap repair) without a matching storage-cost acceptance.** The
  trade-off is explicit and steep: `m=1`'s `6.25%` repair cost comes with `7.467×` storage, roughly
  double the MSR-point cost.
- **Adjacent-not-this, avoided correctly this time:** this is genuinely a Domain P candidate (states
  secrecy as a function of the trade-off, with quantified storage cost), unlike Paper 79.

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Determinant code at `m≈16` (near-MSR) | Plain MSR (Papers 76/80) | Within 3% of the MSR construction's cost, at a real, buildable sub-packetization (`α=C(16,16)=1`, trivial) — genuinely competitive, not dominated, at this specific mode |
| Determinant code at low `m` (e.g. `m=1–2`) | Any other option in the menu | Only sensible if repair bandwidth is the binding constraint and storage is cheap relative to it — not Vyomanaut's stated cost profile (ADR-024, storage priced per GB-month, bandwidth not separately metered to data owners) |

## Breaks in Our Case

- **Requires `k=d`** ≠ **nothing in Vyomanaut's architecture currently fixes repair degree equal to
  reconstruction threshold**
  → **Costed.** Adopting this family means committing to `d=16` for repair, always — losing the
    flexibility the MSR options preserve (repair degree independent of `k`). Not evaluated here
    against ADR-076's `r0`-gate floor of 24 live shards, which is **above** `d=16` — meaning this
    construction is **compatible** with the `r0` gate as stated (unlike Papers 76/80's `d=n−1=55`
    requirement, which is not). **Correction, see ADR-079:** Paper 81's LRC family (`d=r ∈ {2,4,8}`)
    is also `r0`-gate-compatible on the same grounds; this construction is not unique in that
    respect, only in requiring a *fixed* `d=k=16` rather than a tunable one.

## Decisions Influenced

Input to **ADR-079 — Domain P Menu**, contributing the one candidate in the menu that is `r0`-gate-
compatible as proved (not merely as hoped), at a cost of `3.97–4.45×` depending on mode chosen.

## Falsifiers

- **A measurement / documentation gap.** Q79-3 — obtaining the actual TIT 2023 journal version
  (Elmahdy, Kleckler & Mohajer) would confirm whether the Type-I bound proved here is proved optimal
  (not merely conjectured) in the later version, and would add the Type-II result this domain
  originally wanted. Until obtained, cite this paper only for the Type-I achievability formula, not
  for any optimality or Type-II claim.

## Disagreements

**With this note's own first-pass reading**, recorded as a self-correction: the formula transcription
error (see Provenance) produced a materially different, incorrect table in an earlier informal pass.
Corrected here via direct page-image verification.

## Corpus Delta

New to the corpus. **Corrects this domain's own earlier informal substitution table** for this paper
(wrong formula, see Provenance) — no other document in the corpus cited the earlier wrong numbers, so
no downstream correction is required. Adds the `r0`-gate-compatibility finding, which is new and not
derivable from any other paper in this band (Papers 76/80 are explicitly incompatible with the gate;
Paper 81's LRC family's compatibility was not evaluated).

## Open Questions

**Manual step for the reader:** add to `docs/research/open-questions.md`:

```
### Q79-3 — Obtain the TIT 2023 journal version (Elmahdy, Kleckler & Mohajer, Vol. 69, No. 3) of
secure determinant codes. Confirm whether Theorem 1's optimality is proved (not conjectured) and
extract the Type-II eavesdropper result, which the conference-precursor version read for Paper 82
does not contain.

**Raised by:** Paper 82 — artefact version mismatch against reading-list-LTS.md's must-read table.
**Status:** open, low urgency — Paper 82's Type-I result is independently useful and this domain's
current target (ℓ2=1, single-repairer blindness) is a Type-I-shaped question, not Type-II.
**Blocked on:** obtaining the journal PDF.
```
