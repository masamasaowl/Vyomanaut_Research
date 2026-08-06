# Paper 56 — Why Is There Mandatory Retirement?

**Authors:** Edward P. Lazear (Stanford GSB / NBER)
**Venue / Year:** Journal of Political Economy, 87(6):1261–1284 | 1979
**Citations:** foundational paper in personnel economics; DOI 10.1086/260835
**Topics:** #18 Economic Mechanism, #13 Escrow & Payment Basis
**ADRs produced:** ADR-054

---

## Problem Solved

Lazear asks why firms write contracts with a mandatory end date rather than paying workers their marginal product every period. His answer: a firm cannot cheaply monitor effort every period, so it instead pays new workers *below* their marginal product and pays tenured workers *above* it. The gap is an implicit bond — a worker caught shirking forfeits the future above-market wages, so the threat of losing deferred pay substitutes for costly monitoring. The paper solves the same structural problem Vyomanaut has with new providers: the audit system can verify storage, but it cannot verify *intent* — a provider who plans to depart the moment their vetting subsidy stops earning still passes every audit up to that point. A rising-then-flattening pay curve, not a flat one, is what makes staying the dominant strategy.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Upward-sloping pay curve (below-value early, at-value later) | Flat pay from day one | Requires the firm to commit credibly to the back-loaded portion, or the bond has no deterrent value |
| Implicit, reputation-based commitment | Formal contractual guarantee of the ramp | Cheaper to build, but only works if the provider trusts the platform to honor it — ties directly into the disclosure question ADR-045 already answers for a different payout event |

---

## Breaks in Our Case

- **Lazear's worker cannot easily re-enter under a new identity if fired** ≠ **a Vyomanaut provider can, in principle, re-register and restart vetting under a new phone-verified identity**
  → The bond only deters *this* identity's shirking, not re-entry. This is a Sybil-farming vector: an actor could deliberately restart identities to keep collecting the low-tenure portion of the ramp rather than ever reaching full pay. The deterrent value of the ramp is bounded by how cheap a new identity is (Subsystem 1, ADR-005) — this is not solved by the ramp itself and is called out as an open constraint in ADR-054.

- **Lazear's model assumes a single, long-tenure career with one mandatory-retirement cliff** ≠ **Vyomanaut's "career" is a short, capped 45–90 day window, and the "retirement" event (graduation) triggers *more* pay, not less**
  → The mechanics invert cleanly: Lazear's workers accept low pay early in exchange for high pay for the rest of a long career; Vyomanaut's providers accept low pay for at most 90 days in exchange for the market rate for as long as they stay active afterward. The direction of the incentive (stay to reach the better regime) is preserved even though the paper's specific "mandatory retirement" trigger has no analogue here.

- **Lazear's firm cannot observe effort directly, only output over time** ≠ **Vyomanaut's audit system observes a cryptographically verifiable, near-continuous effort signal (PoR challenge responses) rather than an inferred one**
  → This is a strictly better monitoring environment than the one Lazear assumes. It means Vyomanaut does not need as steep a deferred-compensation slope as a firm with pure output-based inference would, because the audit signal already does most of the "was this person actually working" job the wage slope does in Lazear's model. The ramp can be shallower and shorter than a pure agency-cost calculation would suggest, precisely because ADR-002's PoR design substitutes for some of the monitoring the ramp would otherwise need to buy.

---

## Decisions Influenced

- **[ADR-054](../decisions/ADR-054-progressive-earnings-ramp.md) [#18 Economic Mechanism]** `ACCEPTED — REVISES ADR-024 §6`
  Replace the flat 50% release-multiplier cap during vetting with a continuous multiplier that starts at the same 50% (day-zero value) and rises monotonically toward 100% as the provider's confidence score climbs, reaching full rate at graduation.
  *Because:* a flat cap gives a departing provider no more to lose on day 1 of vetting than on day 89 — there is no rising bond to protect. A continuous ramp makes every additional day of good behavior strictly more valuable to keep than to abandon, which is the entire mechanism Lazear identifies as necessary for deferred pay to deter shirking at all.

---

## Disagreements

- **Huck, Seltzer & Wallace (2011)** (cited via the same literature) find in laboratory tests that Lazear's full-commitment prediction holds only when firms can *credibly commit* to the back-loaded wage — without commitment, deferred compensation collapses toward the flat-pay control condition.
  *Implication for us:* the ramp only works as an incentive device if providers believe Vyomanaut will actually pay it — this is exactly why ADR-045's disclosure precedent (show amounts and category, not the formula, but show them honestly and consistently) matters here too. A ramp that providers don't trust is a flat cap with extra steps.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q56-1.
