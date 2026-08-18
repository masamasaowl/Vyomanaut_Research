# ADR-076 Addendum A — F-69's Disjunction Is Not Exhaustive

**Status:** Accepted
**Track:** LTS
**Topic:** #3 Confidentiality / #4 Repair Protocol
**Amends:** ADR-076 (the gate-then-topology decision stands unchanged; the framing of F-69 as a
closed structural result does not)
**Research source:** Domain P literature session, August 2026 — Papers 76, 78, 80 (Goparaju et al.;
Huang & Bruck; Rawat)

---

## The claim being narrowed

ADR-076 states, as an upgrade of F-69 from finding to structural result: *"No assignment of the
repair role under RS + AONT-RS leaves every party blind... either the code family changes (Domain P)
or some party is explicitly trusted with plaintext."*

That disjunction has exactly two branches. Domain P's Band 1 reading found a third: **a protocol-only
change that leaves the code family untouched and trusts no party**, at a quantified, moderate storage
cost. F-69's own text is not wrong about the current implementation — under RS(16,56) exactly as
deployed, with naive whole-shard repair, every party genuinely is disqualified as ADR-076's table
shows. But the disjunction as stated forecloses a real third option that exists in the literature and
should not have been foreclosed.

## What the third branch is

Huang & Bruck (Paper 78) give a two-round, dealer-free repair protocol for any linear code carrying
an information-theoretic secrecy margin `z ≥ 1`, proved within a factor of ~2 of optimal bandwidth.
Goparaju et al. (Paper 76) and Rawat (Paper 80) independently derive the exact secrecy capacity a
regenerating code can offer against a repair-observing eavesdropper, and their construction achieves
it at a buildable sub-packetization. **Both routes require the code to be adjusted to carry the
secrecy margin the guarantee needs — not replaced with a different code family, and not requiring any
party to be trusted with the plaintext.**

Quantified cost, single-repairer blindness (`ℓ2=1` / `z=1`), cross-validated by two independent
derivations:

| Source | Method | Expansion at `ℓ2=1` |
|---|---|---|
| Paper 76 (Goparaju et al.) | subspace-intersection secrecy capacity | `3.83×` |
| Paper 78 (Huang & Bruck) | information-theoretic rate identity `k'=n−r−z` | `3.73×` |

Two proofs, two techniques, agreement to within 2.5%, against RS(16,56)'s current `3.5×`. **The
honest cost of closing F-69's exposure without trusting any party is roughly +7–9% storage — not a
code-family replacement, and not free.**

## Why this was missed

ADR-076's Council 2 evaluated the three *parties* who could execute repair under the code as
deployed and correctly found all three disqualified. It did not evaluate whether the *code itself*
could be adjusted to make the question moot — that is a Domain P question, and Domain P's reading had
not yet happened when ADR-076 was drafted. This is not a defect in Council 2's reasoning; it is
sequencing. The addendum exists because the sequencing is now complete for this branch.

## What remains open

This addendum does **not** select the third branch over ADR-076's status quo (elected-provider
repair, exposure rotated but real) or over the other candidates Domain P's reading surfaced (secure
locally-repairable codes, Paper 81; secure determinant codes, Paper 82). Three genuine open items
block a selection:

1. **Whether Huang & Bruck's protocol composes with AONT-RS's *computational* security** (their
   proofs are information-theoretic; AONT-RS is not) — Q79-2, unresolved.
2. **Whether any of these constructions survive ADR-076's own `r0`-gated batch repair** (up to 32
   simultaneous shard losses). Goparaju's and Rawat's constructions require `d=n−1`; the `r0` gate
   caps live helpers at 24. Paper 82's determinant-code family is the only candidate proved
   compatible with the gate as built. This is the single largest open item across the whole band —
   see Paper 76's note.
3. **Whether a single batched repair cycle can itself exceed the secrecy-capacity wall** (`ℓ2 ≥ k`)
   in one event, independent of lifetime accumulation — Q79-1, raised by Paper 76's reading of its own
   Theorem 3 precondition against ADR-076's batch size.

These three items, plus the priced menu across all Domain P Band 1/2 papers, are carried forward to
**ADR-079 (new, Proposed)**, which this addendum does not substitute for.

## Decision

1. **F-69's text in ADR-076 stands as an accurate description of the current implementation.** No
   correction is made to ADR-076's own decision (build the gate, move execution provider-side) — that
   decision is sound regardless of which Domain P branch is eventually chosen, since a rare repair
   event is a precondition for every candidate's economics, not just the trusted-party ones.
2. **The disjunction is corrected going forward.** Any future document restating F-69 should state it
   as: *"under the code and topology as currently built, every repairer is disqualified; whether a
   protocol-only or codec-adjustment change removes this without trusting a party is an open Domain P
   question, not a closed impossibility."*
3. **ADR-076's own "Open constraints" list is extended** with a pointer to ADR-079 rather than
   restating Domain P's status inline — ADR-076 should not carry research-tracking detail that will
   go stale as Domain P progresses.

## Consequences

**Positive.** F-69 is no longer mis-cited as foreclosing a solution space that in fact has live
candidates. Future council sessions on this topic start from an accurate premise.

**Negative.** None — this addendum corrects a framing, not a decision. ADR-076's actual engineering
work (the gate, the elected-repairer protocol) is unaffected and remains the right immediate step
regardless of which Domain P branch is eventually selected.

**Open constraints:** unchanged from ADR-076 except as narrowed above; the repairing provider under
the current implementation still obtains plaintext until ADR-079 selects and a follow-on ADR
implements a replacement.

## References

- ADR-076 — the decision this addendum narrows, not reverses
- Paper 76 (Goparaju, El Rouayheb, Calderbank & Poor) — exact secrecy capacity, `d=n−1`
- Paper 78 (Huang & Bruck) — dealer-free protocol, matching lower bound
- Paper 80 (Rawat) — optimality proof for Paper 76's construction
- ADR-079 (new) — the priced menu and the council decision this addendum defers to
