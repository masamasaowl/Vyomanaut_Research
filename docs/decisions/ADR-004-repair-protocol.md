# ADR-004 — Lazy Repair with r0=8 and 72-Hour Departure Threshold

**Status:** Accepted
**Topic:** #4 Replication / Repair Protocol
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 06, 09, 10, 19, 34, 36, 39

---

## Context

When a provider fails or goes offline, the system must decide when and how to trigger repair (re-encoding and redistribution of missing fragments). Two strategies exist: eager repair (trigger immediately on any failure) and lazy repair (defer until redundancy falls below a threshold r0). The repair process itself must be P2P — no central entity fetches and re-encodes on behalf of others.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Eager repair (trigger immediately) | Maximum durability at all times | Catastrophic bandwidth waste: every nightly absence triggers a full repair cycle |
| **Lazy repair (trigger at r0=8)** | Bandwidth saving over eager repair: at our parameters, the lazy threshold produces a ~38× improvement over eager repair (Giroire([paper-10](../research/paper-10-giroire-lazy.md)): BW_eager / BWavg ≈ (r − r0) / ln((s+r)/(s+r0)) = 32 / 0.848 ≈ 38). Even compared to a moderate threshold of r0=20, the saving is approximately 3×. This is the dominant reason lazy repair is chosen.| Slightly higher data loss probability during the repair window |

## Decision

Lazy repair: repair is triggered when available fragments for a file drop to s + r0 = 24 (i.e. 8 fragments above the reconstruction floor of s=16).

**Repair trigger flow:**
1. Audit scheduler detects a provider has been absent for t=24 h
2. Fragment count for that provider's chunks is decremented in the repair state tracker
3. If any chunk's available count drops to ≤ 24: repair job is queued
4. Repair job contacts the k=16 surviving fragment holders, reconstructs the missing fragments, and pushes them to newly selected providers

Scheduler priority ordering: when the repair queue contains both jobs triggered by confirmed permanent departures (72h threshold crossed, ADR-007) and jobs triggered by the redundancy pre-warning (fragment count approaching r0), the scheduler must service permanent-departure jobs first. Transient-absence jobs may wait behind permanent-departure jobs. This prevents permanently degraded stripes from accumulating while routine churn is being processed. ([Paper 39](../research/paper-39-silberstein-lazy-recovery.md))

**Departure threshold (when to declare a provider gone):**
Based on Bolosky's bimodal distribution ([paper-09](../research/paper-09-bolosky-feasibility.md)) (nightly µ=14 h, weekend µ=64 h):
- Accidental/temporary absence (0–24 h): no repair, wait, decrease reliability score
- Promised downtime (provider declared in advance): wait for promised period; penalise if broken
- Permanent silent departure: after **72 h** without contact → trigger repair, seize escrow
- Announced departure: trigger repair immediately on announcement

The 72-hour threshold safely exceeds the weekend absence peak (99.7% of weekend absences resolve within 70 h per Bolosky) without waiting for a true departure.

**Peak bandwidth (Qpeek) per failure event:**
For N=1000, s=16, r=40, r0=8, lf=256 KB: Qpeek ≈ 793 GB total.
At 100 Kbps per peer, repair completes in ≈ 8 h — within the 12-hour reconstruction window θ. At 100 Kbps per peer background budget (N=1000 peers, aggregate 100 Mbps), reconstruction completes in approximately 8 hours — within the 12-hour safety window θ. The 12-hour window provides a 4-hour margin above the reconstruction time.

## Consequences

**Positive:**
- 38× bandwidth savings over eager repair
- 72-hour threshold eliminates false-positive repair triggers from nightly and weekend absences
- Repair window (8 h) fits within the 12-hour safety budget

**Negative / trade-offs:**
- Data availability is slightly reduced during the lazy window (r0=8 below full redundancy)
- A provider returning during the lazy window finds their slot already re-filled — storage credit must be handled carefully

**Open constraints:**
- r0=8 and 72 h threshold are calibrated for desktop-only V2. Both must be revisited for mobile (V3).
- If MTTF falls below 180 days at scale, these parameters must be recalculated.
- At V3 scale, the repair scheduler must implement burst admission control. [Paper 38](../research/paper-38-nath-correlated-failures.md) shows that without it, a large correlated failure event (equivalent to an ASN outage affecting multiple providers simultaneously) produces a repair backlog that itself degrades audit performance.

## References

- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): lazy repair bandwidth savings
- [Paper 09 — Bolosky](../research/paper-09-bolosky-feasibility.md): bimodal absence distribution; 72 h threshold
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): Qpeek formula; 793 GB peak per failure event; 8-hour reconstruction window at 100 Kbps/peer; 38× bandwidth saving over eager repair; BWavg table at different MTTFs confirming 180-day floor
- [Paper 19 — EC Survey](../research/paper-19-ec-survey.md): lazy repair confirmed as established industry technique; cites TotalRecall (P2P context) and Silberstein (DSS context)
- [Paper 34 — ELECT](../research/paper-34-elect-ec-tiering.md): Table 2 confirms repair is 93.3% network-bound; validates lazy repair as the correct bandwidth-optimisation axis
- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): Giroire formula correct for mean repair bandwidth; variance 22× higher under correlated failures; lazy repair parameters unchanged but burst admission control needed at V3 scale
- [Paper 39 — Silberstein et al.](../research/paper-39-silberstein-lazy-recovery.md): scheduler priority ordering — permanent departures before transient absences; lazy recovery validated; confirms permanent/transient distinction already in ADR-007 is the critical structural requirement

