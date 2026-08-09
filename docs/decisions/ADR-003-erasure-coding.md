# ADR-003 — Use Reed-Solomon Erasure Coding: s=16, r=40, r0=8, lf=256 KB

**Status:** Accepted
**Topic:** #3 Erasure Coding
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 06, 10, 19, 22, 23, 34, 36, 38, **62, 63, 65**

> **Revision note (Papers 62, 63, 65).**
>
> 1. **F-07 closed.** The "2.5× storage overhead vs 3× for simple replication" claim is removed. RS(16,56) costs **3.5×**, which is *more* than 3× replication. Paper 63 supplies the replacement argument, and it is a strong one.
> 2. **F-29 partially closed.** First primary wide-stripe sources in the corpus. The finding reframes the anomaly: `n = 56` is unremarkable; `k/n = 0.286` is what puts us outside the literature's envelope.
> 3. **New:** a departure-based durability number to sit alongside Giroire's failure-based LossRate, because the two model different processes and only one was ever recorded.
> 4. **New open constraint:** encode/decode cost at `(16,40)` is unmeasured and sits far outside every published benchmark.

---

## Context

The system needs a redundancy scheme that survives provider churn without consuming excessive repair bandwidth. Simple replication was evaluated and rejected (Paper 06). Erasure coding provides far higher durability per unit of storage. Three schemes were evaluated: Reed-Solomon (RS), Minimum Storage Regenerating (MSR), and Locally Repairable Codes (LRC). MSR and LRC are deferred to V3 — see [ADR-026](./ADR-026-repair-bw-optimisation.md), whose candidate decision is currently under council review.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Simple replication (3× copies) | Simple implementation; marginally *cheaper* in storage than our RS parameters | Departure tolerance an order of magnitude worse (Paper 63: governing quantity 3 vs 41); 3× repair bandwidth per failure |
| **Reed-Solomon RS(s=16, r=40)** | Proven; well-understood; 8× BW savings over replication; 13.7× the departure tolerance of 3× replication at 1.17× its storage cost | 3.5× storage overhead; not bandwidth-optimal for repair (Dimakis 2010); full repair requires contacting all s surviving fragments |
| MSR regenerating codes | Bandwidth-optimal repair | Sub-packetisation `α = 1,600` at our parameters (Papers 22, 62 — **not** `40^16`, see ADR-026); 164-byte sub-chunks are pathological on desktop HDDs; deferred to V3 |

## Decision

Reed-Solomon erasure coding with the following parameters, derived to meet LossRate < 10⁻¹⁵ per year (Giroire formulas, Paper 10):

| Parameter | Value | Meaning | Storj mapping |
| --- | --- | --- | --- |
| s | 16 | Minimum fragments needed for reconstruction | k in Storj notation |
| r | 40 | Total redundancy fragments (total shards = s + r = 56) | — |
| r0 | 8 | Repair trigger threshold — repair fires when available fragments drop to s + r0 = 24 | m in Storj notation (Paper 05, Section 4.7) |
| lf | 256 KB | Fragment size | — |
| BWavg | ≈ 39 Kbps/peer | Steady-state repair bandwidth at MTTF=180 days | Confirmed by Storj Table 7.2 at MTTF=6 months |

Lazy repair: repair is deferred until redundancy falls below r0, not triggered immediately on each failure. This produces a ~3× bandwidth saving over eager repair for a gap of r − r0 = 32 ([paper-10](../research/paper-10-giroire-lazy.md)). **Note (Papers 62, 64):** this choice has a cost that was not priced when it was made — it forecloses every single-block repair optimisation in the literature. See ADR-004 and ADR-026.

r=40 is not an arbitrary choice. [paper-10](../research/paper-10-giroire-lazy.md) derives the optimality condition ∂BWavg/∂r = 0, which for s=16, r0=8 has the unique solution r=40. This is the analytically optimal redundancy level for minimising **repair bandwidth** at our reconstruction threshold. Raising or lowering r both increase bandwidth consumption. **It optimises bandwidth only — it is silent on storage cost, on encode/decode compute, and on the data/parity ratio that determines whether repair-friendly codes help us at all.** All three consequences are recorded below.

Desktop-only V2 uses a single parameter set — no per-tier differentiation (see [ADR-010](ADR-010-desktop-only-v2.md)). If mobile providers are added in V3, parameters must be recalibrated for MTTF~30 days.

### Durability: two numbers, two failure processes

The corpus has until now carried one durability figure and quoted it for both purposes. They are not interchangeable.

| Failure process | Model | Result at our parameters |
| --- | --- | --- |
| Disk/node **failure**, with repair running | Giroire ([Paper 10](../research/paper-10-giroire-lazy.md)) | LossRate ≈ 10⁻²⁵/year at MTTF=300 d — steady state |
| Provider **departure with erasure**, no repair | Friedman/Kapelko ([Paper 63](../research/paper-63-friedman-kapelko-data-persistency.md)) | `E[X]` departures before the first file is unrecoverable |

Paper 63's `REC(p, p+q, r)` maps to Vyomanaut as `p=16, q=40, r=1`; everything scales in `r(q+1) = 41`. Vyomanaut places one shard per distinct provider, which is the paper's **sequential** strategy, for which expected persistency does not depend on the number of documents stored.

| Providers `N` | Departures tolerated before first file loss |
| --- | --- |
| 56 (ADR-029 bootstrap gate) | **41** — 73.2% of the entire network |
| 560 | 350 — 62.6% |
| 5,600 | 3,137 — 56.0% |

This is a **floor**, not a prediction: it assumes repair never runs. It answers "how much of the network can walk away during an extended microservice outage," which is a real operational scenario with no prior number.

**Two cautions carried from Paper 63.** (a) The asymptotic form (its Theorem 11) underestimates by 36% at `N = 56` — the exact Theorem 9 integral must be used anywhere near the bootstrap regime. (b) The model assumes independent uniform departure, which is the assumption F-28 disputes; it does not close F-28, it gives F-28 an exact baseline.

## Consequences

**Positive:**

- LossRate < 10⁻¹⁵ per year at target MTTF (Giroire, failure-with-repair)
- 41 of 56 providers may depart before first file loss at the bootstrap gate, with no repair at all (Paper 63, departure-without-repair)
- BWavg ≈ 39 Kbps/peer — well within the 100 Kbps background budget
- Any 3 of 5 simultaneously-failing ASNs are survivable at the bootstrap gate (derivation in ADR-029)

**Negative / trade-offs:**

- **CORRECTED (F-07) — storage overhead is 3.5×, and it is worse than 3× replication.**

  | Scheme | Total stored / data | Parity / data |
  | --- | --- | --- |
  | RS(16,56) | **3.5×** | 2.5× |
  | 3× replication | 3.0× | 2.0× |

  The previous text compared RS's *parity* ratio (2.5) against replication's *total* ratio (3.0) and reported a win that does not exist. Paper 63 states erasure-code storage cost as `(p+q)/p` — 56/16 = 3.5 — independently. **Any unit economics built on 2.5× are 40% off and must be recomputed**: provider cost-per-GB, owner price-per-GB, and the ADR-030 synthetic-chunk 10% cap all take storage expansion as an input.

  The honest justification for r=40 is durability and repair bandwidth, not storage efficiency, and Paper 63 makes it quantitative:

  | Scheme | Storage | Governing quantity `r(q+1)` |
  | --- | --- | --- |
  | 3× replication | 3.0× | 3 |
  | 4× replication | 4.0× | 4 |
  | **RS(16,56)** | **3.5×** | **41** |

  **RS(16,56) costs 17% more storage than 3× replication and buys 13.7× the departure tolerance.** Matching 41 by replication alone would require 41 copies. State it this way.

- **CRITICAL — Co-requisite:** The LossRate < 10⁻¹⁵ per year guarantee holds only jointly with the 20% ASN cap in [ADR-014](./ADR-014-adversarial-defences.md). The independent-failure Markov formula correctly predicts the mean; [Paper 38](../research/paper-38-nath-correlated-failures.md) (Nath et al.) proves that large-m erasure schemes can have their superiority ordering reversed under real-world correlated failures. With the 20% ASN cap, the maximum correlated failure event is bounded at 11 shards, leaving 44 — 28 above the s=16 floor. An engineer deploying ADR-003 without ADR-014 does not achieve the stated LossRate guarantee.

- **The one-shard-per-provider-per-file invariant is load-bearing for durability, not just a mechanical consequence of needing 56 holders.** Paper 63's random strategy (collisions permitted) versus sequential (one piece per node) at `D = 10⁶` documents: 226 vs 350 tolerated departures at `N=560` (**55% worse**), 2,254 vs 3,137 at `N=5,600` (**39% worse**). `architecture.md` already requires 56 distinct providers per file segment; ADR-005 should record *why*.

- **`k/n = 0.286` is what puts us outside the literature, not `n = 56`.** Paper 62 (LESS) evaluates only `k/n ≥ 0.71`; Paper 64 (Hitchhiker) at 0.71; the surveyed production systems similarly. This is a direct consequence of `r = 40`, and it has two concrete effects: repair-friendly codes assume most shards are systematic (they are not — 71% of ours are parity), and their sufficiency conditions can fail outright at low rate (LESS at `α=2` is a *net loss* for us). The corpus should stop describing `n = 56` as the anomaly.

- Full reconstruction requires contacting all 16 surviving fragments — not bandwidth-optimal.

**Open constraints:**

- Parameters calibrated for desktop-tier MTTF (180–380 days). If mobile is introduced, these values must change.
- Encryption-erasure interaction order is still TBD — see [ADR-022](ADR-022-encryption-erasure-order.md). **Also now: F-69 — RS repair of one lost shard assembles `k=16`, which is exactly AONT-RS's disclosure threshold. Every repair event reconstructs plaintext. This is a property of the *code parameters chosen here*, and Domain P may return a requirement to change them.**
- RS(16,56) is the outer code in the proof-of-retrievability sense (Paper 67 §3.3): the audit primitive supplies detection, and this code supplies retrievability. The erasure code is therefore load-bearing for the security argument of the audit path, not only for durability. Any future code-family change (ADR-026, F-44) must preserve that property.
- BWavg ≈ 39 Kbps/peer is a mean under the independent-failure assumption. [Paper 36](../research/paper-36-dalle-failure-correlation.md) proves the real standard deviation is 22× higher due to correlated burst failures.
- Upload optimality threshold (deferred): Storj's fourth parameter `o` — not specified for V2.
- **NEW — encode/decode cost at `(16,40)` is unmeasured and unbudgeted.** [Paper 65](../research/paper-65-bilp-bx-xor-scheduling.md) is the first throughput data in the corpus: a production XOR-based RS library reaches 258 MB/s single-threaded without SIMD at `(k=10, m=4)` on a current mid-range desktop, and throughput falls superlinearly in `k·m`. Vyomanaut's `k·m = 640` is **16× the largest configuration ever benchmarked in that set**. `∂BWavg/∂r = 0` chose `r = 40` for bandwidth and never looked at compute. The client-side encode cost and the repair-time decode cost are separate numbers on separate machines and neither exists. → ADR-009, F-27, Q65-1.
- **NEW — small-file penalty.** `s=16 × 256 KB` makes the minimum efficiently-coded object 4 MB; at 3.5× (not 2.5×) the expansion for sub-4 MB files is worse than F-30 recorded. No file-size distribution is stated anywhere. → Domain C / R-11.

## References

- [Paper 62 — LESS](../research/paper-62-less-io-efficient-repair.md): first primary wide-stripe repair source (F-29); establishes that low code rate, not stripe width, is what puts our parameters outside the tested envelope; confirms Clay `α = 1,600`
- [Paper 63 — Friedman, Kapelko & Marchwicki](../research/paper-63-friedman-kapelko-data-persistency.md): departure-with-erasure persistency; independent confirmation of the 3.5× storage ratio (F-07); the `r(q+1) = 41` vs 3 comparison against replication; quantifies the one-shard-per-provider invariant
- [Paper 65 — BiLP-BX](../research/paper-65-bilp-bx-xor-scheduling.md): first erasure-coding throughput measurement in the corpus; 258 MB/s at `(10,4)` scalar; superlinear falloff in `k·m`
- [Paper 05 — Storj](../research/paper-05-storj.md): four-parameter RS scheme; r0=8 is Storj's production-validated m parameter; Table 7.2 confirms Giroire BWavg at MTTF=6 months
- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): erasure coding confirmed over replication
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): all four closed-form formulas; r=40 analytically optimal for bandwidth (Formula 4)
- [Paper 38 — Nath et al.](../research/paper-38-nath-correlated-failures.md): proves the LossRate guarantee requires ADR-014 as co-requisite
- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): real σ is 22× the mean; ASN cap is the correct peak mitigation
- [Paper 19 — EC Survey](../research/paper-19-ec-survey.md): RS confirmed as the only MDS family in production. **Its "n=56 is wide-stripe territory" note is superseded by Paper 62** — the width is ordinary, the rate is not
- [Paper 22 — Goparaju et al.](../research/paper-22-goparaju-msr-codes.md): MSR sub-packetisation. **The `α ≥ 40^16` figure previously drawn from here was a substitution error — see ADR-026 Correction 1**
- [Paper 23 — Yin et al.](../research/paper-23-yin-cold-erasure-codes.md): EC optimal for cold data; 256 KB fragment throughput penalty acceptable for write-once
- [Paper 34 — ELECT](../research/paper-34-elect-ec-tiering.md): repair is 93.3% network-bound — the reason Paper 62's benefit should be stated as bandwidth, not disk I/O
