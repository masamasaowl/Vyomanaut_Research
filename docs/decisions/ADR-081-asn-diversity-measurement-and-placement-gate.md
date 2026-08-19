# ADR-081 — ASN Diversity Is Sufficient in Count and Insufficient in Distribution

**Status:** Proposed
**Track:** LTS
**Topic:** #3 Confidentiality / #5 Peer Selection / #14 Placement
**Amends:** ADR-022 Addendum A §4 (the gate condition); ADR-014 (cap enforcement granularity);
ADR-029 (the "grows into safety" reading)
**Research source:** M-01 — `measurement-01-asn-diversity-india.md`
**Data:** `docs/research/data/R-17/` — `asn_in_fixed.csv`, `analyse.py`, `analysis_output.txt`

---

## Context

ADR-022 Addendum A withdrew the confidentiality claim made for the 20% ASN cap and established
Council 4's general ruling that **a placement cap cannot repair a collusion threshold**. It then
deferred one question, and made the whole of Domain K wait on it:

> R-17 becomes the feasibility gate […] how many genuinely independent ASNs an Indian consumer
> provider pool can span, treating CGNAT. **If the answer is < 8, cap-tuning is off the table
> entirely and R-30 is the only instrument.**

M-01 measured it. The answer is that **the gate as written passes and the property it was written to
protect fails**, which is a result the binary could not express. This ADR records the arithmetic in
full so that no future session re-derives it, re-argues it, or — the specific failure Q66-2 records —
lifts a number out of it without its substitution.

Everything below is reproducible from the three files named above. Where a number is measured it is
tagged `[P-num]`; where it is computed here the working is shown.

---

## The measurement, in one table

| Quantity | Value | Source |
| --- | --- | --- |
| Indian fixed-line residential ASNs above the ~200k-user floor | 55 | M-01 §3.3 |
| Distinct operators after ownership collapse | **49** | M-01 §3.3 |
| ASN : operator ratio | 1.122 | M-01 §3.3 |
| Total active networks in India (all classes) | 2,957 `[P-num]` | APNIC, Jul 2026 |
| IRINN member organisations | 4,649 `[P-num]` | IRINN, Jan 2026 |
| **Effective independent domains `N_eff = 1/HHI`** | **5.161** | M-01 §3.4 |
| HHI (operator level) | 0.19375 | M-01 §3.4 |
| Largest single fixed-line domain (Reliance Jio, AS55836) | 32.68% `[P-num]` | TRAI, Jun 2026 |
| Top-3 combined | 71.63% | M-01 §3.4 |
| Domains already at or above the 20% cap | 2 | M-01 §3.4 |
| P(a 56-provider proportional draw breaches the cap) | **0.9954** | M-01 §3.5 |
| Tail over-recruitment factor to reach `f < 0.143` | **2.203×** | M-01 §5.3 |
| Untargeted enrolments to reach 8 usable domains | **427** (7.6× the ADR-029 gate) | M-01 §5.4 |
| RPKI ROA coverage, India | 88.04% IPv4 / 97.91% IPv6 `[P-num]` | APNIC Labs, Jul 2026 |

---

## The arithmetic, written out

Kept here in full because this ADR is the citable record. Vyomanaut parameters are the standing set:
`n = 56`, `k = 16`, `f = 0.20`, ADR-029 gate `≥56` providers across `≥5` ASNs.

### A. The cap allowance and the existing break (reproduces F-34)

```
cap allowance       = ⌊n × f⌋ = ⌊56 × 0.20⌋ = ⌊11.2⌋ = 11 shards/ASN
two-ASN coalition   = 2 × 11 = 22 shards
disclosure threshold k = 16
margin              = 16 − 22 = −6                          DISCLOSES
```

### B. The cap fraction that would fix two-domain collusion

```
require   2 × ⌊56f⌋ < 16
        →     ⌊56f⌋ ≤ 7
        →         f < 8/56 = 0.142857
placement ceiling per domain = 7 shards
minimum domains              = ⌈56/7⌉ = 8
required placement weight    = 7/56 = 0.125 = 12.50% per domain
```

Note the two distinct quantities, which are easy to conflate: **`f < 0.142857` is the cap fraction
bound; `12.50%` is the placement weight ceiling.** They differ because `⌊·⌋` discards the remainder.
Citing 0.143 where 0.125 is meant overstates the achievable margin by 14%.

### C. Concentration of the recruitable population

`[P-num]` TRAI wireline base, June 2026 — 47.82 M total:

```
Reliance Jio     15,625,342 / 47,820,000 = 0.32676
Bharti Airtel    11,990,437 / 47,820,000 = 0.25074
BSNL              6,637,405 / 47,820,000 = 0.13880
Vodafone Idea       783,861 / 47,820,000 = 0.01639
residue          12,782,955 / 47,820,000 = 0.26731   (decomposed in APNIC proportion — see Limits)
```

```
HHI  = Σ sᵢ²
     = 0.32676² + 0.25074² + 0.13880² + … over 51 domains
     = 0.10677  + 0.06287  + 0.01927  + 0.00484 (tail)
     = 0.19375
N_eff = 1/HHI = 1/0.19375 = 5.161
```

**Hand check.** Head only, tail discarded entirely:
`HHI₃ = 0.3268² + 0.2507² + 0.1388² = 0.18891`, `1/HHI₃ = 5.293`.
The full 51-domain model returns 5.161. Forty-eight additional real operators move the effective
count by **−0.13**. Two routes, one conclusion: `N_eff` is set by the head. ✓

### D. Breach probability under proportional recruitment

Exact binomial over 56 independent draws, `P(X > 11) = Σ_{i=12}^{56} C(56,i) sⁱ (1−s)^{56−i}`:

```
Reliance Jio  (s = 0.3268)  p = 0.977103
Bharti Airtel (s = 0.2507)  p = 0.780286
BSNL          (s = 0.1388)  p = 0.080311
ACT / Atria   (s = 0.0465)  p = 0.000008

P(≥1 of top-6 breaches) = 1 − Π(1 − pᵢ) = 0.995373
```

### E. Over-recruitment required to hold every domain at ≤ 12.50%

Water-fill: clip every share above 0.125 to 0.125, redistribute the excess to the tail in proportion,
iterate to fixpoint.

```
head (3 operators)  0.7163 → 0.3750     suppression 0.524×
tail (48 operators) 0.2837 → 0.6250     over-recruitment 2.203×
per-tail-operator factor                              2.203×
Reliance Jio                                          0.38×
Bharti Airtel                                         0.50×
BSNL                                                  0.90×
```

**Hand check by conservation of mass.**
`freed = 0.7163 − 0.3750 = 0.3413`; `tail must absorb → 0.2837 + 0.3413 = 0.6250`;
`0.6250 / 0.2837 = 2.203`. ✓

### F. Recruitment depth

```
enrolments for 7 providers from domain i = 7 / sᵢ

  Reliance Jio    7 / 0.3268 =  21
  BSNL            7 / 0.1388 =  50
  ACT / Atria     7 / 0.0465 = 151
  Vodafone Idea   7 / 0.0164 = 427    ← binding: the 8th domain

fleet multiplier vs ADR-029's 56-provider gate = 427 / 56 = 7.6×
```

### G. The residual after all of the above

```
2 domains × 7 = 14  vs k = 16  → margin +2   safe
3 domains × 7 = 21  vs k = 16  → margin −5   DISCLOSES
4 domains × 7 = 28  vs k = 16  → margin −12  discloses
```

**Cap-tuning buys exactly one coalition size, at a cost of 2.2× recruitment distortion and a 7.6×
fleet multiplier.** Council 4 stated this symbolically. This is the price.

### H. Multi-ASN ownership defeats the cap without collusion

```
ACT / Atria Convergence holds 4 ASNs: AS24309, AS55577, AS18209, AS131269
admissible under a per-ASN 20% cap = 4 × 11 = 44 shards = 78.6% of the stripe
margin vs k = 16                   = 16 − 44 = −28
```

No conspiracy is required. One company, four ASNs, one honest-but-curious operator.

### I. Cost of the self-declared ASN field (F-LTS-10, costed)

```
identities to saturate the cap from one fabricated ASN     = 11
identities to fake ADR-029's gate (5 ASNs × 11)            = 55
identities to fake 8 domains × 7 shards                    = 56
real broadband lines required                              = 1
```

---

## Decision

**1. Restate the Domain K gate over `N_eff`, and record that it fails.**

ADR-022 Addendum A §4's condition — *"if the answer is < 8 [ASNs], cap-tuning is off the table"* — is
**superseded**. It is not wrong so much as unmeasurable-in-the-useful-direction: the count is 49 and
answers nothing.

The replacement gate:

> Cap-tuning is available only where the effective number of independent placement domains
> `N_eff = 1/Σsᵢ²`, computed over the *recruitable* provider population weighted by actual
> recruitment share, satisfies `N_eff ≥ ⌈n / ⌊n·f_target⌋⌉`.

At the current parameter set this is `N_eff ≥ 8`. **Measured: `N_eff = 5.161`. The gate fails.**

The consequence ADR-022 Addendum A attached to failure stands: **R-30 (raising the privacy threshold
above the reconstruction threshold) is the primary instrument for Domain K.** Cap-tuning is not
prohibited — §E and §F price a route to `f < 0.143` — but it is now a *costed* option with a named
price rather than an assumed remedy, and it does not reach three-domain collusion at any price.

**2. The ASN cap is enforced per organisation, not per ASN.**

Per §H. `NetworkProfile`'s cap field is reinterpreted as a fraction of the stripe admissible to one
*organisation*, and the placement path requires an ASN→organisation mapping as an input. Until that
mapping exists the cap is not merely a weak control (ADR-022 Addendum A) but an **unbinding** one
against the fourth-largest fixed-line operator in the country.

Q-R17-4 owns whether such a mapping is maintainable. **This ADR does not schedule it**, and the
`[UNDERIVED]` tag under ADR-077 applies to the cap fraction until it is.

**3. Extend `TestProfileConfidentialityMarginHolds` with a distribution assertion.**

ADR-022 Addendum A landed a compiler-enforced invariant that fails today on the production profile,
deliberately. It tests the *cap*. It cannot test the *distribution*, which §D shows is where the
failure actually occurs. Add a second assertion, in the same intentionally-failing style, over the
recruited population's measured `N_eff` once provider registrations carry a validated ASN.

**4. Withdraw the "grows into safety" framing.**

ADR-029's ≥5-ASN bootstrap gate sits at `N_eff = 5.161` — the *steady state*, not a floor to be
outgrown. Per §C's hand check, `N_eff` is flat past about twenty tail operators. Every document
asserting that confidentiality exposure is a bootstrap transient is corrected by this clause. The
≥5-ASN gate remains valid as a **durability** floor under ADR-014, which is what it was sized for.

**5. Adopt RPKI-backed origin attribution as the detection mechanism for F-LTS-10.**

`[P-num]` 88.04% of Indian IPv4 route objects and 97.91% of IPv6 route objects carry a valid ROA.
Detection is buildable now: observe the source address at the microservice, map to origin AS, validate
against a ROA. Residual per 56-provider stripe: 6.70 providers unsigned on IPv4, 1.17 on IPv6.

Registrations whose origin cannot be validated are **not rejected** — that would exclude 12% of a
market Vyomanaut is trying to enter — but are placed in an `asn_unverified` pool that counts as a
**single shared domain** for cap purposes. This makes the unverifiable residue costly to the attacker
without being costly to the honest provider on an unsigned prefix.

**6. Plan fixed-line attribution against the IPv4 figure, not the national IPv6 figure.**

`[P-num]` India's ~79% IPv6 capability is, per APNIC, concentrated in mobile. Vyomanaut recruits from
fixed-line. Using the national IPv6 number to size a fixed-line control is the Q66-2 error in a new
place. **The planning figure is 88.04%** (F-LTS-19).

**7. Record the eligibility dependency.**

Every number in this ADR rests on providers being fixed-line desktops. Extending eligibility to
always-on devices on mobile-carrier links re-admits AS55836 and AS45609 — 73.58% of Indian Internet
users in two ASNs — and collapses `N_eff` toward 2.6. **Provider-class eligibility is hereby a
confidentiality-relevant parameter**, not a product-scope one, and changing it re-opens this ADR.

---

## Consequences

**Immediate.** Domain K's research sequence is settled: R-30 first, R-31 second, cap-tuning as a
costed fallback with a known ceiling of three-domain collusion. One superseded gate condition, one
withdrawn framing in ADR-029, one enforcement-granularity change in ADR-014, one new required system
input (ASN→org mapping), one new registration pool (`asn_unverified`).

**For the product.** §E and §F are the first quantified statement of a **recruitment constraint** in
the project. Meeting the confidentiality property requires declining Jio customers at 62% and
over-recruiting small-ISP customers at 2.2×, and doing so *before* the provider is useful. This is a
go-to-market constraint discovered in a cryptography investigation, and it belongs in the
product-market-fit work rather than being left in Domain K.

**What this does not settle.** Nothing here addresses F-69 or F-LTS-08 — every routine repair still
assembles `k = 16` shards by design (ADR-076). A perfect placement policy and a perfect ASN oracle
leave that untouched. Domain K constrains who *holds* shards; it does not constrain who *assembles*
them, and the second is the larger exposure.

**Open constraints.** The ASN→organisation mapping of Decision 2 is unscheduled. The `N_eff`
assertion of Decision 3 requires validated ASNs, which requires Decision 5, which is also
unscheduled. This ADR creates two dependencies and closes neither; that is deliberate, per
`reading-list.md` §3.6 — a finding is allowed to remain a finding.

---

## Limits of this decision

- **§C's residue decomposition is a substitution, not a measurement.** The 26.73% non-big-four
  wireline residue is split in APNIC user proportion. Defensible only because `N_eff` is insensitive
  to tail shape (§C hand check). **Not to be cited for any claim about a specific small operator.**
  Q-R17-3 replaces it.
- **Head shares are June 2026; ASN structure is March 2024.** Two years of consolidation is
  unaccounted. Bias direction: consolidation lowers operator count, so true `N_eff` is likely below
  5.161. Conservative for the gate.
- **F-LTS-16 (JioFiber inside AS55836) is `[INFERRED]`, not `[P-num]`.** No decision clause above
  rests on it alone — Decision 1 follows from `N_eff` regardless of which ASN Jio's fixed line sits
  in, since Jio is one *organisation* either way, and Decision 2 makes the organisation the unit.
  Q-R17-1 confirms it.
- **The binomial in §D assumes independent recruitment.** ISP-partnership recruitment makes draws
  positively correlated within a cohort, which makes breaches *deterministic per cohort* rather than
  probabilistic. The 0.9954 figure is therefore a floor, not a point estimate, under any bulk
  acquisition channel.

---

## References

- **M-01** — `../research/measurement-01-asn-diversity-india.md` — the measurement, its provenance,
  threats to validity, and falsifiers
- **R-17** — `../research/literature_research_results/R-17.md` — source record
- ADR-014 — where the ASN cap lives as a durability control; enforcement granularity amended here
- ADR-022 Addendum A — the claim withdrawn, the council ruling confirmed, the gate condition superseded
- ADR-029 — the bootstrap gate; "grows into safety" withdrawn here
- ADR-070 — records that real ASN detection is unimplemented (F-LTS-10); costed here
- ADR-076 — F-LTS-08: repair assembles `k` shards by design; unaffected by this ADR and larger than it
- ADR-077 — `[UNDERIVED]` / `[VENDOR-DEFAULT]` triage tags; the cap fraction remains `[UNDERIVED]`
- `reading-list-LTS.md` §5 Domain K — R-17 answered; R-30 promoted by elimination
