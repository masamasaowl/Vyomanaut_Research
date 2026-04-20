# ADR-006 — Polling Interval 24 Hours; Departure Threshold 72 Hours

**Status:** Accepted
**Topic:** #6 Availability & Polling
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 06, 08, 09

---

## Context

The system must periodically check whether providers are still online. Two parameters must be set: the polling interval t (how often we check), and the departure threshold (how long we wait before declaring a provider gone and triggering repair). Getting t wrong in either direction is expensive: too short and nightly absences trigger unnecessary repair; too long and true departures go undetected for too long.

## Decision

**Polling interval t = 24 hours.**
Rationale from Blake & Rodrigues (Paper 06): a 24-hour timeout reduces maintenance bandwidth by 30× compared to an instant timeout. Bhagwan (Paper 08) confirms that the minimum safe value of t is ≥ 8 hours (diurnal absence window). 24 hours is within the safe range and provides the bandwidth saving.

**Departure threshold = 72 hours (desktop).**
Rationale from Bolosky (Paper 09): the bimodal absence distribution shows:
- Nightly: µ=14 h, σ=1.9 h → 99.7% of providers return within 20 h
- Weekend: µ=64 h, σ=2.1 h → 99.7% of providers return within 70 h

A 72-hour threshold safely exceeds the weekend peak without triggering repair for routine absences.

**DHT record refresh interval = 12 hours**
Change noted after paper 20

**Lower bound (emergency repair):**
A minimum redundancy floor is maintained regardless of t. If available fragments drop to s (reconstruction floor = 16), repair is triggered immediately without waiting for the timeout.

**Mobile tier:** t and departure threshold are not set for mobile — mobile providers are deferred to V3.

## Consequences

**Positive:**
- 30× bandwidth saving vs instant timeout
- 72 h threshold eliminates false-positive repair triggers for weekend absences
- Emergency floor ensures data is never at genuine loss risk regardless of t

**Negative / trade-offs:**
- A truly departed provider can go undetected for up to 72 h — this reduces redundancy during that window
- t is not adaptive (Q08-4 is still open — adaptive t is a V3 consideration)

**Open constraints:**
- t=24 h is a starting value; production telemetry may justify tuning (Q06-3)
- These values assume desktop-only V2

## References

- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): 30× bandwidth saving at 24 h
- [Paper 08 — Bhagwan](../research/paper-08-bhagwan-availability.md): minimum safe t ≥ 8 h
- [Paper 09 — Bolosky](../research/paper-09-bolosky-feasibility.md): bimodal distribution; 72 h threshold
