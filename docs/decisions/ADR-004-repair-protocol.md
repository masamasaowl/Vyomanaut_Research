# ADR-004 — Lazy Repair with r0=8 and 72-Hour Departure Threshold

**Status:** Accepted
**Topic:** #4 Replication / Repair Protocol
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 06, 09, 10, 19, 34, 36, 39, **62, 63, 64**

> **Revision note (Papers 62, 64).** `r0 = 8` has a cost that was never priced: **it forecloses every
> single-block repair optimisation in the erasure-coding literature.** Hitchhiker, LESS, Clay,
> HashTag and ET all optimise the repair of one lost block and all fall back to plain RS when more
> than one is missing. Lazy repair at `r0 = 8` means 32 fragments are missing at every repair event,
> so none of them ever engages. Giroire's 38× eager-vs-lazy saving is far larger than the 13–33%
> those codes offer, so **lazy repair almost certainly still wins** — but the two are *not*
> orthogonal, which is what ADR-026 asserted, and the interaction belongs here. A specification
> ambiguity in the trigger flow is also flagged below (Q26-4).

---

## Context

When a provider fails or goes offline, the system must decide when and how to trigger repair (re-encoding and redistribution of missing fragments). Two strategies exist: eager repair (trigger immediately on any failure) and lazy repair (defer until redundancy falls below a threshold r0). The repair process itself must be P2P — no central entity fetches and re-encodes on behalf of others.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Eager repair (trigger immediately) | Maximum durability at all times; **single-block repair optimisations (ADR-026) become applicable** | Catastrophic bandwidth waste: every nightly absence triggers a full repair cycle |
| **Lazy repair (trigger at r0=8)** | Bandwidth saving over eager repair: at our parameters, the lazy threshold produces a ~38× improvement over eager repair (Giroire([paper-10](../research/paper-10-giroire-lazy.md)): BW_eager / BWavg ≈ (r − r0) / ln((s+r)/(s+r0)) = 32 / 0.848 ≈ 38). Even compared to a moderate threshold of r0=20, the saving is approximately 3×. This is the dominant reason lazy repair is chosen. | Slightly higher data loss probability during the repair window; **forecloses every repair-friendly code family in ADR-026 (Papers 62, 64)** |

## Decision

Lazy repair: repair is triggered when available fragments for a file drop to s + r0 = 24 (i.e. 8 fragments above the reconstruction floor of s=16).

**Repair trigger flow:**

1. Audit scheduler detects a provider has been absent for t=24 h
2. Fragment count for that provider's chunks is decremented in the repair state tracker
3. If any chunk's available count drops to ≤ 24: repair job is queued
4. Repair job contacts the k=16 surviving fragment holders, reconstructs the missing fragments, and pushes them to newly selected providers

Scheduler priority ordering: when the repair queue contains both jobs triggered by confirmed permanent departures (72h threshold crossed, ADR-007) and jobs triggered by the redundancy pre-warning (fragment count approaching r0), the scheduler must service permanent-departure jobs first. Transient-absence jobs may wait behind permanent-departure jobs. This prevents permanently degraded stripes from accumulating while routine churn is being processed. ([Paper 39](../research/paper-39-silberstein-lazy-recovery.md))

> **⚠ Specification ambiguity flagged (Q26-4), not resolved here.** Steps 3–4 gate *all* repair on
> `available count ≤ 24`. The scheduler-priority paragraph above, however, refers to
> permanent-departure jobs as a distinct class already in the queue, which reads as though crossing
> the 72-hour threshold enqueues a repair job on its own. These cannot both be true. Which one holds
> determines whether single-fragment repair events exist at all in Vyomanaut, and therefore whether
> ADR-026's entire code-family question has any value. **This must be settled before ADR-026 goes to
> council.** Flagged rather than resolved, per project convention.

**Departure threshold (when to declare a provider gone):**
Based on Bolosky's bimodal distribution ([paper-09](../research/paper-09-bolosky-feasibility.md)) (nightly µ=14 h, weekend µ=64 h):

- Accidental/temporary absence (0–24 h): no repair, wait, decrease reliability score
- Promised downtime (provider declared in advance): wait for promised period; penalise if broken
- Permanent silent departure: after **72 h** without contact → trigger repair, seize escrow
- Announced departure: trigger repair immediately on announcement

The 72-hour threshold safely exceeds the weekend absence peak (99.7% of weekend absences resolve within 70 h per [Bolosky](../research/paper-09-bolosky-feasibility.md)) without waiting for a true departure. *(F-06/F-15 remain open on the staleness of this source — Domain D.)*

Repair chunk assignment: replacement chunk assignment during repair uses the identical Power of Two Choices algorithm from [ADR-005](./ADR-005-peer-selection.md), weighted by provider reliability score and free capacity. No special-case assignment logic is required for repair vs upload. [Giroire](../research/paper-10-giroire-lazy.md) BWavg = 39 Kbps/peer is far enough below the 100 Kbps budget that 3× launch-phase variance remains safe.

**Peak bandwidth (Qpeek) per failure event:**
For N=1000, s=16, r=40, r0=8, lf=256 KB: Qpeek ≈ 793 GB total. At 100 Kbps per peer across N=1000 peers, reconstruction completes in approximately 8 hours — within the 12-hour safety window θ, a 4-hour margin. *(F-08 remains open on the reproducibility of this margin.)*

### What `r0 = 8` costs, beyond the loss-probability trade (Papers 62, 64)

Every repair event under this ADR reconstructs **32 missing fragments at once**. That is the operation the repair-optimisation literature does not address. Both primary sources state their multi-block behaviour explicitly:

| Code family | Behaviour at >1 missing block | Saving at `r0 = 8` |
| --- | --- | --- |
| Hitchhiker ([Paper 64](../research/paper-64-hitchhiker-piggybacking.md)) | Reconstructs exactly as RS, reading `k` entire units | **0%** |
| LESS ([Paper 62](../research/paper-62-less-io-efficient-repair.md)) | Optimised only if all failed blocks share one block group; otherwise conventional repair | **0%** |
| Clay (MSR) | Single-failure construction | **0%** |

Worse, forcing LESS's per-group path anyway costs `5 × 12.4 ≈ 62` fragments against RS's 16 to reconstruct the same 32 — **3.9× worse** — because RS's conventional repair amortises one `k`-fragment read across every missing shard and a per-group repair does not.

**Two things follow.**

1. **Lazy repair and repair-friendly codes are alternatives, not complements.** ADR-026 previously asserted additive savings, citing Paper 39's Xorbas+LAZY result. Xorbas is an LRC whose local groups compose with accumulation differently from a single-block-optimised MDS code. That orthogonality claim is withdrawn in ADR-026 pending Q26-3.

2. **The comparison still favours lazy repair, by a wide margin.** 38× against 13–33%. Paper 64 itself considers accumulate-then-repair and rejects it — but on read-latency grounds appropriate to a warm data warehouse, not to a write-once cold archive. Vyomanaut's workload makes the trade it declined, and on the numbers that is the right call. **This should be stated as a deliberate, priced decision rather than left implicit.**

### Durability with repair switched off ([Paper 63](../research/paper-63-friedman-kapelko-data-persistency.md))

Repair is the mechanism that turns a departure-tolerance budget into a steady state. Paper 63 gives the number when it stops running — providers leaving and erasing, no reconstruction:

| Providers `N` | Departures before first file loss (no repair) |
| --- | --- |
| 56 | 41 (73.2% of the network) |
| 560 | 350 (62.6%) |
| 5,600 | 3,137 (56.0%) |

This is the floor a prolonged microservice outage falls back to, and it is generous. It is not a substitute for Giroire's steady state, and it assumes independent uniform departure (F-28 unaffected).

## Consequences

**Positive:**

- 38× bandwidth savings over eager repair
- 72-hour threshold eliminates false-positive repair triggers from nightly and weekend absences
- Repair window (8 h) fits within the 12-hour safety budget
- Even with repair fully stopped, 56–73% of the provider population must depart before the first file is lost (Paper 63)

**Negative / trade-offs:**

- Data availability is slightly reduced during the lazy window (r0=8 below full redundancy)
- A provider returning during the lazy window finds their slot already re-filled — storage credit must be handled carefully
- **`r0 = 8` forecloses every single-block repair optimisation in ADR-026.** The V3 repair-bandwidth saving from Hitchhiker, LESS or Clay is 0% under this repair policy
- **Each repair event reconstructs 32 fragments from 16 — an unmeasured compute burst on the repairing provider.** Not covered by ADR-009's ≤5% CPU budget, and at `(16,40)` far outside every published erasure-coding benchmark (Paper 65). → Q65-1
- **Every repair event assembles `k=16` shards, which is exactly AONT-RS's disclosure threshold (F-69).** The repairing provider holds a decodable package. This is a correctness problem in the repair path, not an efficiency one, and it outranks everything in ADR-026. → Domain P

**Open constraints:**

- r0=8 and 72 h threshold are calibrated for desktop-only V2. Both must be revisited for mobile (V3).
- If MTTF falls below 180 days at scale, these parameters must be recalculated.
- At V3 scale, the repair scheduler must implement burst admission control ([Paper 38](../research/paper-38-nath-correlated-failures.md)).
- **NEW (Q26-4):** the trigger-flow ambiguity above — does permanent departure enqueue repair independently of the `≤ 24` gate? Blocks ADR-026's council session.
- **NEW (Q62-1):** if the answer to Q26-4 is that single-fragment repair events do occur, what fraction of repair events are they, and does that fraction justify a code-family change? Depends on measured churn (Domain D), which is itself stale.

## References

- [Paper 62 — LESS](../research/paper-62-less-io-efficient-repair.md): multi-block fallback rule; per-group repair costs 3.9× conventional at `r0=8`; the 98.08% single-block statistic is a property of eager repair, not of the failure process
- [Paper 63 — Friedman, Kapelko & Marchwicki](../research/paper-63-friedman-kapelko-data-persistency.md): departure-without-repair persistency floor
- [Paper 64 — Hitchhiker (Rashmi et al.)](../research/paper-64-hitchhiker-piggybacking.md): explicit multi-block fallback to RS; explicit consideration and rejection of accumulate-then-repair, on grounds that do not transfer to a write-once archive
- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): lazy repair bandwidth savings
- [Paper 09 — Bolosky](../research/paper-09-bolosky-feasibility.md): bimodal absence distribution; 72 h threshold *(stale — Domain D)*
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): Qpeek formula; 793 GB peak; 38× saving over eager repair
- [Paper 19 — EC Survey](../research/paper-19-ec-survey.md): lazy repair as an established technique
- [Paper 34 — ELECT](../research/paper-34-elect-ec-tiering.md): repair is 93.3% network-bound
- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): variance 22× higher under correlated failures; burst admission control needed at V3
- [Paper 39 — Silberstein et al.](../research/paper-39-silberstein-lazy-recovery.md): scheduler priority ordering. **The additivity result cited from here in ADR-026 is withdrawn pending Q26-3** — it was obtained for an LRC, not for a single-block-optimised MDS code
