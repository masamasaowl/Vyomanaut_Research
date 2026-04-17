# ADR-004 — Lazy Repair with r0=8 and 72-Hour Departure Threshold

**Status:** Accepted
**Topic:** #4 Replication / Repair Protocol
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 06, 09, 10

---

## Context

When a provider fails or goes offline, the system must decide when and how to trigger repair (re-encoding and redistribution of missing fragments). Two strategies exist: eager repair (trigger immediately on any failure) and lazy repair (defer until redundancy falls below a threshold r0). The repair process itself must be P2P — no central entity fetches and re-encodes on behalf of others.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Eager repair (trigger immediately) | Maximum durability at all times | Catastrophic bandwidth waste: every nightly absence triggers a full repair cycle |
| **Lazy repair (trigger at r0=8)** | ~3× bandwidth savings vs eager at gap=32 (Giroire) | Slightly higher data loss probability during the repair window |

## Decision

Lazy repair: repair is triggered when available fragments for a file drop to s + r0 = 24 (i.e. 8 fragments above the reconstruction floor of s=16).

**Repair trigger flow:**
1. Audit scheduler detects a provider has been absent for t=24 h
2. Fragment count for that provider's chunks is decremented in the repair state tracker
3. If any chunk's available count drops to ≤ 24: repair job is queued
4. Repair job contacts the k=16 surviving fragment holders, reconstructs the missing fragments, and pushes them to newly selected providers

**Departure threshold (when to declare a provider gone):**
Based on Bolosky's bimodal distribution (nightly µ=14 h, weekend µ=64 h):
- Accidental/temporary absence (0–24 h): no repair, wait, decrease reliability score
- Promised downtime (provider declared in advance): wait for promised period; penalise if broken
- Permanent silent departure: after **72 h** without contact → trigger repair, seize escrow
- Announced departure: trigger repair immediately on announcement

The 72-hour threshold safely exceeds the weekend absence peak (99.7% of weekend absences resolve within 70 h per Bolosky) without waiting for a true departure.

**Peak bandwidth (Qpeek) per failure event:**
For N=1000, s=16, r=40, r0=8, lf=256 KB: Qpeek ≈ 793 GB total.
At 100 Kbps per peer, repair completes in ≈ 8 h — within the 12-hour reconstruction window θ.

## Consequences

**Positive:**
- ~3× bandwidth savings over eager repair
- 72-hour threshold eliminates false-positive repair triggers from nightly and weekend absences
- Repair window (8 h) fits within the 12-hour safety budget

**Negative / trade-offs:**
- Data availability is slightly reduced during the lazy window (r0=8 below full redundancy)
- A provider returning during the lazy window finds their slot already re-filled — storage credit must be handled carefully

**Open constraints:**
- r0=8 and 72 h threshold are calibrated for desktop-only V2. Both must be revisited for mobile (V3).
- If MTTF falls below 180 days at scale, these parameters must be recalculated.

## References

- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): lazy repair bandwidth savings
- [Paper 09 — Bolosky](../research/paper-09-bolosky-feasibility.md): bimodal absence distribution; 72 h threshold
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): Qpeek formula and repair window calculation
