# ADR-054 — Progressive Earnings Ramp During Vetting

**Status:** Proposed
**Topic:** #18 Economic Mechanism, #13 Escrow & Payment Basis
**Supersedes:** — (revises ADR-024 §6 only; the 30-day hold window and post-vetting release tiers are unaffected)
**Superseded by:** —
**Research source:** Paper 56 (Lazear); reuses ADR-053's confidence signal

---

## Context

ADR-024 §6 currently holds vetting-stage providers to a flat 50% release-multiplier cap for the entire 4–6 month window, with a doubled 60-day hold, and seizes all held earnings on 72-hour silent departure. This gives a provider nothing left to lose on day one that they didn't already have to lose on day 89 — the cap is flat, so there is no growing stake in staying. It also pays nothing extra for demonstrably good behavior partway through vetting; a provider performing at 99% reliability on day 40 earns exactly the same rate as one who just barely avoided ejection. ADR-053 now produces a live, per-provider confidence score throughout vetting. This decision is about what to do with it on the payment side.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Keep ADR-024's flat 50% cap | No change | No incentive gradient during vetting; a provider has equal reason to leave on any given day |
| Pay 100% from day one, no ramp | Maximum motivational effect, zero perceived friction | Removes the deferred-compensation bond entirely (Lazear) — nothing distinguishes a provider planning to leave immediately from one planning to stay |
| **Continuous ramp tied to ADR-053's confidence score, starting at the existing 50% and rising to 100% at graduation; no seizure of vetting-stage earnings** | Directly implements the deferred-compensation mechanism Lazear identifies as necessary for a rising stake to deter early departure; day-zero value matches ADR-024 exactly, so this is a refinement, not a reset | Removes a security mechanism (seizure) that existed for a reason — must be justified separately |

## Decision

**Release multiplier is defined continuously as `RampMultiplier(t) = C(t)`**, using the exact confidence score ADR-053 already computes for graduation timing. No separate formula, no separate state. Because `C(0)` under a fresh Beta(0.5,0.5) prior is exactly 0.5, day-one pay under this design is **identical to ADR-024's current 50% cap** — this is not a new number, it is the existing number re-expressed as the starting point of a curve instead of a constant held for the whole window. As audits accumulate, the multiplier rises with the same signal that governs graduation, reaching effectively 100% at the moment of graduation (mean confidence at graduation was 0.984 for the flawless tier and 0.976 for the good tier in ADR-053's simulation; both round to full rate in practice).

**Payment basis is unchanged from ADR-012** — providers are paid per audit passed, not per GB or per transfer. This decision only multiplies that existing per-pass amount by `RampMultiplier(t)`.

**Vetting-stage earnings are not seized on departure.** ADR-024's 72-hour silent-departure seizure applies to real customer data risk; during vetting, providers hold only synthetic, non-critical chunks under ADR-005's existing design, so a departing vetting-stage provider is not creating the real-data-loss exposure the seizure rule exists to prevent. A provider who leaves during vetting simply stops earning further ramp — what has already been released stays released. Post-graduation seizure-on-departure (ADR-024's core mechanism, protecting real customer assignments) is entirely unchanged by this decision.

**Framing to the provider**, following ADR-045's precedent exactly: the app shows amount released and, for any release below 100%, the same plain-language reason category infrastructure ADR-045 already defines (e.g., a category meaning "still building trust" rather than a specific score or day count). No formula, no threshold, no countdown — consistent with, not a new exception to, ADR-045's disclosure policy.

### Cost comparison — does this increase or reduce vetting-period cash outflow?

Using ADR-053's simulated graduation days and this ADR's ramp multiplier, expected "rate-days" (mean multiplier × days spent in vetting) compare to a V2 baseline of a flat 0.50 held for the full 150-day midpoint of ADR-005's 120–180 day window:

| True reliability | Mean rate-days (V3 ramp) | V2 baseline (flat 0.50 × 150d) | Change |
| --- | --- | --- | --- |
| Flawless (99.5%) | 45.2 | 75.0 | **−39.8%** |
| Good (97%) | 56.5 | 75.0 | **−24.7%** |
| Borderline (90%) | 73.0 | 75.0 | **−2.7%** |
| Poor (75%) | 66.2 | 75.0 | **−11.7%** |

Under this specific formulation, the ramp does **not** increase expected subsidy cost for any tier — the compression of vetting duration from ADR-053 (45–90 days vs. 120–180) dominates the increase in average payout rate, for every tier tested. The original framing of this feature as "accepting extra cash burn as an acquisition cost" does not hold up numerically under this design; the honest finding is closer to the opposite — faster, tier-differentiated graduation reduces expected subsidy exposure per provider relative to the current flat-cap baseline, while paying every provider *something* from day one instead of holding half their earnings for months.

**Caveat on this comparison:** it assumes ADR-005's current implementation holds every provider for the full calendar window regardless of audit performance, since the ADR text describes a calibrated *duration*, not a documented early-exit rule. If an early-exit path already exists in the current implementation that this research did not surface, this comparison overstates the saving and should be re-run against the true current behavior before being used to justify the change.

## Consequences

**Positive:**

- Implements a real deferred-compensation gradient (Lazear) instead of a flat cap with no rising stake
- Day-zero pay is unchanged from today (50%) — no provider is worse off on day one than under the current design
- Under the tested formulation, expected vetting-period payout is lower, not higher, for every reliability tier — reframes this from "accept more cash burn" to "pay out less, faster, more fairly"

**Negative / trade-offs:**

- Removing seizure during vetting removes a deterrent against a specific attack: an actor registering many fake identities purely to collect the rising ramp on synthetic vetting chunks before departing. This is bounded (see Open constraints) but not eliminated by this decision alone.
- The reason-category UI work this requires is additive to, not a superset of, the work ADR-045 already scoped — a new category meaning "still building trust" needs to be added to that fixed set

**Open constraints:**

- **Sybil-farming exposure is shared with ADR-053** and not fully closed here: because vetting-stage assignments are synthetic and volume-bounded by declared storage, and identity creation requires phone/KYC gating (ADR-005 Subsystem 1), the maximum extractable value per fake identity is bounded — but this ADR does not compute that bound in currency terms, because it depends on the per-audit rate figures and declared-storage caps, which are implementation parameters not sourced during this research pass. This must be computed before launch, not assumed safe by analogy.
- The comparison above uses a 150-day midpoint as the V2 baseline; if the true current average vetting duration differs materially, the cost comparison should be rerun

## References

- [Paper 56 — Lazear (1979)](../research/paper-56-lazear-deferred-compensation.md): deferred-compensation theory this design implements
- [ADR-012](ADR-012-payment-basis.md): per-audit-pass basis this decision multiplies, unchanged
- [ADR-024](ADR-024-economic-mechanism.md): the ADR this decision revises (§6 vetting-period multiplier only)
- [ADR-045](ADR-045-earnings-transparency-disclosure.md): disclosure precedent this decision follows rather than reopens
- [ADR-053](ADR-053-confidence-gated-vetting-duration.md): source of the shared confidence signal

---
