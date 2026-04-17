# ADR-003 — Use Reed-Solomon Erasure Coding: s=16, r=40, r0=8, lf=256 KB

**Status:** Accepted
**Topic:** #3 Erasure Coding
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 06, 10

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

| Parameter | Value | Meaning |
|---|---|---|
| s | 16 | Minimum fragments needed for reconstruction |
| r | 40 | Total redundancy fragments (total shards = s + r = 56) |
| r0 | 8 | Repair trigger threshold — repair fires when available fragments drop to s + r0 = 24 |
| lf | 256 KB | Fragment size |
| BWavg | ≈ 39 Kbps/peer | Steady-state repair bandwidth at MTTF=180 days |

Lazy repair: repair is deferred until redundancy falls below r0, not triggered immediately on each failure. This produces a ~3× bandwidth saving over eager repair for a gap of r − r0 = 32 (Giroire Section 3).

Desktop-only V2 uses a single parameter set — no per-tier differentiation (see [ADR-010](ADR-010-desktop-only-v2.md)). If mobile providers are added in V3, parameters must be recalibrated for MTTF~30 days.

## Consequences

**Positive:**
- LossRate < 10⁻¹⁵ per year at target MTTF
- BWavg ≈ 39 Kbps/peer — well within 100 KB/s background budget
- 2.5× storage overhead vs 3× for simple replication

**Negative / trade-offs:**
- Full reconstruction requires contacting all 16 surviving fragments — not bandwidth-optimal
- MSR codes would reduce repair bandwidth further but require more research

**Open constraints:**
- Parameters calibrated for desktop-tier MTTF (180–380 days). If mobile is introduced, these values must change.
- Encryption-erasure interaction order (encrypt-then-code vs code-then-encrypt) is still TBD — see [ADR-022](ADR-022-encryption-erasure-order.md)

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): starting parameters (k=29, n=80)
- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): erasure coding confirmed over replication
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): parameter derivation formulas
