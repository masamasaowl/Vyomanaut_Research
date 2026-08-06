# ADR-053 — Confidence-Gated Variable Vetting Duration

**Status:** Proposed
**Topic:** #5 Peer Selection Algorithm
**Supersedes:** — (revises the duration clause of ADR-005; ADR-005's identity-gate, filtering, and preference subsystems are unaffected)
**Superseded by:** —
**Research source:** Papers 05, 08, 46; reuses the Jeffrey's-prior machinery ADR-005 already accepted

---

## Context

ADR-005 fixes vetting at 4–6 months (120–180 days), calibrated so a provider accumulates at least 80 consecutive clean audits at the 24-hour polling interval — the point at which a Beta(0.5,0.5) posterior mean crosses 99% confidence in the provider's audit-pass probability. That number is a single fixed calendar window applied to every provider identically, regardless of whether they are demonstrably excellent from day one or barely adequate the whole way through. It also has no ceiling: a provider who never quite reaches 80 clean audits has no defined exit from vetting other than being filtered out by ADR-008's sustained-failure ejection. Two problems follow. First, onboarding friction is the same for a flawless provider as for a mediocre one, which does nothing to reward the providers the network most wants to keep. Second, an open-ended window makes it impossible to promise a provider anything about when — or that — they will start earning full rate, which blocks the earnings-ramp design in ADR-054.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Keep ADR-005's fixed 120–180 day window | No change, no new failure modes | Does not differentiate providers by demonstrated reliability; blocks a bounded earnings ramp |
| Fixed audit-count threshold only (graduate at N clean audits, whenever that happens) | Simple; already close to ADR-005's stated calibration | Still open-ended for poor performers; does not compress the *typical* case, only removes the ceiling entirely |
| **Confidence-gated window: fluid graduation day bounded to [45, 90] days, driven by a rolling Bayesian confidence score** | Rewards demonstrated reliability with faster graduation; bounds worst-case exposure to 90 days; reuses existing Jeffrey's-prior and 30-day-window infrastructure | Requires a live per-provider confidence computation running throughout vetting, not just a pass/fail counter checked once |

## Decision

Vetting duration is bounded to **[45, 90] days** and determined per provider by a **rolling 30-day confidence score**, not a fixed calendar length:

- **Prior:** Beta(0.5, 0.5) — the same Jeffrey's prior ADR-005 already uses.
- **Window:** the trailing 30 days of audit outcomes (reuses ADR-008's existing 30-day long window; no new state is introduced). At any check-in on day *t*, let `passes` and `fails` be the pass/fail counts among audits in `[t-30, t]`.
  `C(t) = (0.5 + passes) / (1 + passes + fails)`
- **Graduation rule:** a provider graduates to ACTIVE on the first day `t ≥ 45` where `C(t) ≥ 0.97`. If no such day occurs, graduation is **forced at t = 90** regardless of score, provided the provider has not already been ejected under ADR-008's sustained-failure filtering.
- **Full-history vs. rolling window — this is not a cosmetic choice.** An earlier full-history design (confidence computed over *all* audits since registration, target 0.99 to match ADR-005 exactly) was tested and rejected: under that design, a single failed audit anywhere in the window requires **148 subsequent clean audits** to recover to 0.99 confidence — mathematically impossible inside a 90-day ceiling. A single blip would permanently lock a provider to the worst-case ceiling regardless of everything else they did right. The rolling 30-day window lets old blips age out, which is what makes a *short, capped* window survivable at all. The target confidence was correspondingly recalibrated from 0.99 to 0.97 (see Monte Carlo results below for why 0.99 remains infeasible even with the rolling window at realistic pass rates).
- **Forced graduation at day 90 is not a clean pass.** A provider forced to the ceiling with confidence meaningfully below 0.97 (i.e., no consecutive-audit streak strong enough to clear the bar) graduates to ACTIVE for assignment purposes, but should carry a **flag for continued enhanced monitoring** — e.g., initial preference-subsystem weighting per ADR-005's Power-of-Two-Choices treated as if still partially vetted for a further period. This ADR proposes the flag; the exact post-graduation monitoring policy is left to ADR-008's scoring implementation (see Open Questions).
- **Nothing about chunk assignment changes.** Vetting continues to use non-critical, high-redundancy chunks exactly as ADR-005 already specifies; this decision only changes *when* graduation happens, not what a vetting-stage provider holds.
- **Disclosure:** following the precedent in ADR-045, the provider is never shown the confidence score, the 0.97 threshold, or a countdown. The app may show a plain-language state ("in review") and, once graduated, a plain notice that full rate has started — consistent with ADR-045's "amounts and category, not formula" policy applied to a new event type.

### Monte Carlo validation (5,000 simulated providers per tier, Beta(0.5,0.5) prior, 30-day rolling window, target 0.97)

| True reliability | p10 grad day | p50 grad day | p90 grad day | % forced to 90 | Mean confidence at graduation |
| --- | --- | --- | --- | --- | --- |
| Flawless (99.5%) | 45 | 45 | 53 | 0.4% | 0.984 |
| Good (97%) | 45 | 53 | 90 | 15.3% | 0.976 |
| Borderline (90%) | 59 | 90 | 90 | 77.4% | 0.904 |
| Poor (75%) | 90 | 90 | 90 | 99.7% | 0.744 |

The design differentiates cleanly: a genuinely reliable provider graduates at or near the 45-day floor almost every time; a borderline provider is held to the 90-day ceiling more often than not; a poor provider is essentially always held to the ceiling and flagged. This is the behavior the fixed 120–180 day window cannot produce, since it applies the same wait to every provider regardless of measured quality.

## Consequences

**Positive:**

- Demonstrably reliable providers reach full trust in as little as 45 days instead of 120–180 — directly lowers intake friction for the providers the network most wants
- Worst-case time-to-decision is now bounded at 90 days; ADR-005's current design has no such bound
- Reuses existing infrastructure (Jeffrey's prior, 30-day scoring window) — no new subsystem, only a new read pattern against data ADR-008 already stores

**Negative / trade-offs:**

- A forced-graduation provider at day 90 carries materially lower confidence (as low as ~0.74 for a genuinely poor performer) than any provider under the old 99%-calibrated standard — this is a deliberate trade of statistical confidence for bounded onboarding time, not a free improvement, and should be reviewed once real provider-tier data exists (see Open Questions)
- Even a flawless provider graduating at the 45-day floor carries ~98.9% confidence under full-history math, or ~98.4% under the 30-day window — both below ADR-005's original 99.4% (80-audit) standard. The floor buys speed at a small, quantified cost in certainty.
- The rolling window means a provider's status can, in principle, oscillate near the boundary if performance is noisy right around day 45–50; this ADR does not specify hysteresis and treats it as a known rough edge

**Open constraints:**

- Post-graduation monitoring policy for forced-ceiling providers is not yet defined (flag exists, response does not)
- Sybil-farming exposure from restarting identities to repeatedly collect the low-tenure ramp is bounded by identity-creation cost (ADR-005 Subsystem 1: phone/KYC gate), not by this ADR — see the shared open question in ADR-054
- This ADR assumes desktop-only V2/V3 scope (ADR-010); mobile providers are out of scope

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): source of the Jeffrey's-prior / 80-audit calibration this ADR builds on and partially revises
- [Paper 08 — Bhagwan et al.](../research/paper-08-bhagwan-availability.md): two-component churn model motivating why a short vetting window still needs a real confidence signal, not just a countdown
- [Paper 46 — Storj node operator docs](../research/paper-46-storj-node-operator-docs.md): vetting-period friction observed in a comparable production network
- [ADR-005](ADR-005-peer-selection.md): the ADR this decision revises (duration clause only)
- [ADR-008](ADR-008-reliability-scoring.md): source of the reused 30-day rolling window
- [ADR-054](ADR-054-progressive-earnings-ramp.md): the earnings-ramp decision this graduation signal also drives

---
