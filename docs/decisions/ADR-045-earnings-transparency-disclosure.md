# ADR-045 — Earnings Transparency: Disclose Amounts and Reason Category, Not the Scoring Formula

**Status:** Proposed
**Topic:** #20 Client-Facing UX & Copy
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 47 (Rosenblat & Stark)

---

## Context

The ADR-016 addendum already records, per release, the gross amount and the exact multiplier applied — the data needed to explain a reduced payout. Paper 47 documents that Uber drivers' distrust stems specifically from opaque fare/earnings calculation, while also showing that platforms have reason to keep some algorithmic detail undisclosed to prevent gaming. This decision fixes how much of that data actually reaches the provider, and in what form — a question Q47-1 left open.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Full opacity — show only the final payout amount | Simplest to build; scoring logic fully protected from gaming | Directly reproduces the opacity Paper 47 identifies as the mechanism of driver distrust |
| Full disclosure — show the exact reliability-score formula and weights | Maximum transparency | Makes the score straightforwardly gameable, undermining the audit/reliability system's own purpose |
| **Disclose gross amount, multiplier applied, and a plain-language reason category (e.g., "reliability score dropped below 95% this period") — withhold the exact formula and weights** | Directly answers "why was my payout smaller," using data already captured by the ADR-016 addendum, without exposing the mechanics a bad actor would need to game the score | Requires maintaining a mapping from internal score thresholds to plain-language reason categories, kept in sync as scoring logic evolves |

## Decision

The Provider app's earnings screen shows, for any release where the multiplier was below 100%: the gross amount, the amount actually released, and a plain-language reason category drawn from a small, fixed set (e.g., "reliability score below threshold," "uptime below threshold" — exact set to be defined alongside the scoring implementation). The underlying formula, exact score, and exact thresholds are not disclosed. This is rendered through the same IC §14 copy contract used for error/status messaging, not a separate ad hoc earnings-specific text system.

## Consequences

**Positive:** directly answers the "why was my payout smaller" question using data the ledger already captures (ADR-016 addendum), without exposing gameable scoring internals; consistent with the existing error/status copy infrastructure rather than a new parallel system.

**Negative:** the reason-category mapping is a new piece of state to define and maintain, separate from both the raw score computation and the copy contract, and must be kept honest as scoring logic changes.

**Open constraints:** the exact set of reason categories is not yet defined — this ADR fixes the disclosure *policy* (amounts + category, not formula), not the final category list, which should be defined alongside the scoring/reliability implementation itself.

## References

- Paper 47 — Algorithmic Labor and Information Asymmetries: A Case Study of Uber's Drivers (Rosenblat & Stark, IJoC 2016)
- ADR-016 addendum — Gross Amount and Release Multiplier on RELEASE Events (supplies the data this ADR governs disclosure of)
