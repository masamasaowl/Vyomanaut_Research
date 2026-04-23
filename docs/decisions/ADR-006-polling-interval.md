# ADR-006 — Polling Interval 24 Hours; Departure Threshold 72 Hours

**Status:** Accepted
**Topic:** #6 Availability & Polling
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 06, 08, 09, 20, 28

---

## Context

The system must periodically check whether providers are still online. Two parameters must be set: the polling interval t (how often we check), and the departure threshold (how long we wait before declaring a provider gone and triggering repair). Getting t wrong in either direction is expensive: too short and nightly absences trigger unnecessary repair; too long and true departures go undetected for too long.

## Decision

**Polling interval t = 24 hours.**
Rationale from Blake & Rodrigues ([Paper 06](../research/paper-06-blake-rodrigues.md)): a 24-hour timeout reduces maintenance bandwidth by 30× compared to an instant timeout. Bhagwan ([Paper 08](../research/paper-08-bhagwan-availability.md)) confirms that the minimum safe value of t is ≥ 8 hours (diurnal absence window). 24 hours is within the safe range and provides the bandwidth saving.

**Per-challenge response timeout:** the timeout used when awaiting a provider's audit challenge response must not be a fixed value. Use a per-provider TCP-style RTO: RTO = AVG + 4×VAR, where AVG and VAR are the EWMA mean and variance of that provider's recent audit response latencies. This is stored as (avg_rtt_ms, var_rtt_ms) per provider in the reliability scoring DB. New providers are initialised to the pool median. Fixed timeouts double detection latency under realistic conditions ([Paper 28](../research/paper-28-rhea-dht-churn.md), Figure 9). This formula governs the micro-level per-challenge timeout only; the 72-hour departure threshold is unchanged.

**Departure threshold = 72 hours (desktop).**
Rationale from Bolosky ([Paper 09](../research/paper-09-bolosky-feasibility.md)): the bimodal absence distribution shows:
- Nightly: µ=14 h, σ=1.9 h → 99.7% of providers return within 20 h
- Weekend: µ=64 h, σ=2.1 h → 99.7% of providers return within 70 h

A 72-hour threshold safely exceeds the weekend peak without triggering repair for routine absences.

**DHT record refresh interval = 12 hours** (not 24 hours). IPFS production measurements ([Paper 20](../research/paper-20-trautwein-ipfs.md)) show a 24-hour republication interval against a 24-hour expiry leaves zero buffer: a single delayed cycle causes silent record expiry. The 12-hour republish / 24-hour expiry gap provides the same buffer as the IPFS production design.

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
- [Paper 20 — IPFS Measurement](../research/paper-20-trautwein-ipfs.md): 12h republish / 24h expiry — zero buffer at 24h/24h; production-validated reference parameters
- [Paper 28 — Rhea et al.](../research/paper-28-rhea-dht-churn.md): per-provider RTO = AVG + 4×VAR; fixed timeouts double detection latency; periodic recovery model confirmed
