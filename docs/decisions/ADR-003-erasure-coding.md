# ADR-003 — Use Reed-Solomon Erasure Coding: s=16, r=40, r0=8, lf=256 KB

**Status:** Accepted
**Topic:** #3 Erasure Coding
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 06, 10, 19, 22, 23, 34, 36, 38

---

## Context

The system needs a redundancy scheme that survives provider churn without consuming excessive repair bandwidth. Simple replication was evaluated and rejected (Paper 06). Erasure coding provides the same durability at a fraction of the storage overhead. Three schemes were evaluated: Reed-Solomon (RS), Minimum Storage Regenerating (MSR), and Locally Repairable Codes (LRC). MSR and LRC are deferred to V3 after Phase 2A research.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Simple replication (3× copies) | Simple implementation | 3× storage overhead; 3× repair bandwidth per failure |
| **Reed-Solomon RS(s=16, r=40)** | Proven; well-understood; 2.5× overhead; 8× BW savings over replication | Not bandwidth-optimal for repair (Dimakis 2010); full repair requires contacting all s surviving fragments |
| MSR regenerating codes | Bandwidth-optimal repair | Not yet proven for consumer-device networks; deferred to V3 (reading-list Phase 2A #15) |

## Decision

Reed-Solomon erasure coding with the following parameters, derived to meet LossRate < 10⁻¹⁵ per year (Giroire formulas, Paper 10):

| Parameter | Value | Meaning | Storj mapping |
| --- | --- | --- | --- |
| s | 16 | Minimum fragments needed for reconstruction | k in Storj notation |
| r | 40 | Total redundancy fragments (total shards = s + r = 56) | — |
| r0 | 8 | Repair trigger threshold — repair fires when available fragments drop to s + r0 = 24 | m in Storj notation (Paper 05, Section 4.7) |
| lf | 256 KB | Fragment size | — |
| BWavg | ≈ 39 Kbps/peer | Steady-state repair bandwidth at MTTF=180 days | Confirmed by Storj Table 7.2 at MTTF=6 months |

Lazy repair: repair is deferred until redundancy falls below r0, not triggered immediately on each failure. This produces a ~3× bandwidth saving over eager repair for a gap of r − r0 = 32 ([paper-10](../research/paper-10-giroire-lazy.md))

r=40 is not an arbitrary choice. [paper-10](../research/paper-10-giroire-lazy.md)  derives the optimality condition ∂BWavg/∂r = 0, which for s=16, r0=8 has the unique solution r=40. This is the analytically optimal redundancy level for minimising repair bandwidth at our reconstruction threshold. Raising or lowering r both increase bandwidth consumption.


Desktop-only V2 uses a single parameter set — no per-tier differentiation (see [ADR-010](ADR-010-desktop-only-v2.md)). If mobile providers are added in V3, parameters must be recalibrated for MTTF~30 days.

## Consequences

**Positive:**
- LossRate < 10⁻¹⁵ per year at target MTTF
- BWavg ≈ 39 Kbps/peer — well within 100 KB/s background budget
- 2.5× storage overhead vs 3× for simple replication

**Negative / trade-offs:**
- CRITICAL CO-REQUISITE: The LossRate < 10⁻¹⁵ per year guarantee is derived under the independent-failure Markov assumption. [Paper 38](../research/paper-38-nath-correlated-failures.md) proves that for wide-stripe erasure systems (ERASURE(16,56) in their notation), correlated failures can reverse the superiority ordering entirely. This guarantee only holds jointly with the 20% ASN cap in [ADR-014](./ADR-014-adversarial-defences.md). The erasure parameters alone are insufficient. An engineer deploying ADR-003 without ADR-014 does not achieve the stated durability.
- Full reconstruction requires contacting all 16 surviving fragments — not bandwidth-optimal
- MSR codes would reduce repair bandwidth further but require more research

**Open constraints:**
- Parameters calibrated for desktop-tier MTTF (180–380 days). If mobile is introduced, these values must change.
- Encryption-erasure interaction order (encrypt-then-code vs code-then-encrypt) is still TBD — see [ADR-022](ADR-022-encryption-erasure-order.md)
- BWavg ≈ 39 Kbps/peer is a mean under the independent-failure assumption. [Paper 36](../research/paper-36-dalle-failure-correlation.md) proves the real standard deviation is 22× higher due to correlated burst failures. The peak bandwidth during an ASN-level outage significantly exceeds this mean. The 20% ASN cap (ADR-014) bounds the worst-case burst size.
- Upload optimality threshold (deferred): Storj uses a fourth parameter o — once o pieces confirm, the slowest n-o uploads are cancelled (Paper 05, Section 4.7). This reduces P99 upload latency but requires the microservice to issue cancel signals. Not specified for V2; add as a V3 enhancement.

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): four-parameter RS scheme (k=s, m=r0, o=upload threshold, n=total); r0=8 is Storj's production-validated m parameter; Table 7.2 empirically confirms Giroire BWavg formula at MTTF=6 months
- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): erasure coding confirmed over replication
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): all four closed-form formulas with V2 parameter derivations; r=40 is analytically optimal (Formula 4: ∂BWavg/∂r = 0); BWavg = 39 Kbps/peer (Formula 1); Qpeek = 793 GB (Formula 2); LossRate ≈ 10^-25/year (Formula 3)
- [Paper 38 — Nath et al.](../research/paper-38-nath-correlated-failures.md): proves LossRate guarantee requires ADR-014 as co-requisite; ERASURE(16,56) is in the strongest diminishing-return regime for correlated failures
- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): BWavg is mean only; real σ is 22× higher; ASN cap is the correct peak mitigation
- [Paper 19 — EC Survey](../research/paper-19-ec-survey.md): RS confirmed as the only MDS code family in production; n=56 is wide-stripe territory (3× larger than any surveyed production system)
- [Paper 22 — Goparaju et al.](../research/paper-22-goparaju-msr-codes.md): MSR/Clay codes computationally intractable at (n=56, k=16); RS(16,56) is the only feasible code at these parameters
- [Paper 23 — Yin et al.](../research/paper-23-yin-cold-erasure-codes.md): EC confirmed as optimal for cold data; 256 KB fragment size throughput penalty is acceptable for write-once workload
- [Paper 34 — ELECT](../research/paper-34-elect-ec-tiering.md): empirically confirms repair is 93.3% network-bound; validates Giroire bandwidth model's foundational assumption