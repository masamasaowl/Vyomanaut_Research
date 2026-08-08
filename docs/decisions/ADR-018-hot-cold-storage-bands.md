# ADR-018 — Explicit Hot and Cold Provider Storage Bands

**Status:** Accepted (Cold band only in V2; Hot band mechanism now specified, exact parameters still open — deferred to V3)
**Topic:** #5 Peer Selection (storage type)
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 23, 34, 59 (HALO), 60 (Zebra)

---

## Context

Providers have different hardware profiles. Some have large disks and stable connections; others have fast connections but limited storage. Data owners also have different needs: some data is accessed frequently (hot), some is archived (cold). Matching provider profile to data access pattern improves both economics and performance.

This ADR previously stated that Hot-band scaling would reuse RS monotonicity — the first `n` pieces of RS(k,n+x) equal the first `n` pieces of RS(k,n) — to add Hot-band redundancy on top of Cold band's existing pieces without re-encoding. Papers 59 and 60 (this revision's research source) show that mechanism, on its own, cannot deliver what this ADR's own Decision section already promised for Hot band: "low latency... smaller chunks for retrieval speed." Adding parity on top of a fixed `k=16` improves durability, not reconstruction speed — reconstruction still needs 16 shards either way. This revision makes that tension explicit and resolves the *mechanism*; it does not yet resolve the *exact numbers*.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **A — Keep `k=16` fixed, Hot band only adds parity (original ADR-018 assumption)** | Cheapest possible migration (Cauchy-monotonic subset, near-zero re-encode cost per Paper 60) | Does not actually reduce retrieval/reconstruction latency — a Hot-band file still needs 16 shards to read, same as Cold. Fails the ADR's own stated Hot-band goal. |
| **B — Hot band uses a smaller `k` than Cold's 16, chosen via demand-driven optimization (Paper 60's method, applied statically since Vyomanaut's band choice is one-time, not continuously re-solved)** — chosen | Genuinely reduces reconstruction bandwidth and latency for Hot data — the actual mechanism both papers validate empirically (HALO: 2.0 vs 6.0 failure domains touched per repair at comparable stripe scale) | Breaks the monotonic-subset migration property. Upgrading an existing Cold-band file to Hot band later requires a full re-encode, not an incremental add — a cost this ADR did not previously account for |
| **C — Adopt HALO's fully dynamic, continuously-reclassified hotness tracking** | Best theoretical fit per-file, adapts as access patterns drift (HALO: up to 50% lower repair latency vs. static schemes) | Requires new infrastructure Vyomanaut does not have today: per-file access-rate metering at the coordination microservice, and a runtime re-tiering trigger. Contradicts the existing enrolment flow, where the data owner declares Hot or Cold once at upload time. Deferred — see Open constraints. |

## Decision

Hot band adopts **Option B**: a smaller stripe width `k_hot < 16`, distinct from Cold band's `k=16`, chosen once via Zebra's demand-driven geometric-programming method against Vyomanaut's own target Hot-band retrieval-latency SLA and storage-overhead budget — not computed per-request, and not continuously re-solved (that is Option C, explicitly deferred). Vyomanaut's existing enrolment flow is unchanged: the data owner declares Hot or Cold once at upload time; there is no automatic runtime re-classification in V2 or the current V3 scope.

The redundancy templates are pre-generated and cached, selected by constant-time lookup at upload time — following HALO's template-switching pattern (§4.3.2) rather than solving the optimization live for every file. This keeps the V3 implementation cost bounded: one offline parameter-derivation exercise, not a new always-on optimizer service.

Cauchy Reed-Solomon construction (Paper 60) is adopted explicitly for both bands, superseding the previous ADR's implicit "any RS library" assumption. This keeps the cheap, near-zero-cost migration path available *within* a band (e.g., a Cold-band file gaining more Cold-band parity later still costs almost nothing) even though it does not eliminate the re-encode cost of moving *between* bands under Option B.

**Cross-band migration cost is now an explicit, named trade-off of this ADR** — see Consequences below. It was not accounted for in the previous version.

## Consequences

**Positive:**

- Hot band now has a real mechanism (smaller `k`) that actually delivers the latency/bandwidth improvement this ADR always claimed, rather than only a cheap-migration property that didn't deliver it
- The template-lookup pattern (not a live solver) keeps V3 implementation scope bounded — no new always-on prediction service required
- Cauchy RS construction is now explicit in both bands' encoders, closing an implementation ambiguity the original ADR left open

**Negative / trade-offs:**

- Upgrading a file from Cold to Hot band after initial upload now costs a full re-encode, not a cheap incremental add — this did not exist as a named cost in the original ADR and must be priced into any future "upgrade my file to Hot" feature
- The exact value of `k_hot` is not yet derived — see Open constraints. Implementers cannot build against this ADR yet without that number
- Dynamic, continuously-adapting tiering (Option C) is explicitly out of scope. If access patterns genuinely drift after a one-time Hot/Cold declaration, V3 will not adapt automatically — the data owner must manually reclassify

**Open constraints:**

- **`k_hot` is not yet numerically derived.** This requires solving Paper 60's geometric program with Vyomanaut-specific inputs: target Hot-band P99 retrieval latency, the home-provider bandwidth distribution (not a datacenter's), and the maximum acceptable storage-overhead delta versus Cold band's existing 40-parity overhead. This is a well-defined follow-up calculation, not open-ended research — flagged for the next session that has real target-SLA numbers to work from.
- **Whether Vyomanaut ever wants Option C (dynamic re-tiering)** is a genuine open architectural question, not a research gap — see Q59-1. This ADR takes no position beyond "not now."

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): Section 6.1 RS monotonicity property — still governs within-band redundancy scaling, no longer assumed sufficient for cross-band Hot/Cold differentiation
- [Paper 23 — Yin et al.](../research/paper-23-yin-cold-erasure-codes.md): write-once cold storage is EC's optimal regime; 256 KB fragment throughput penalty is negligible for single-upload workloads
- [Paper 34 — ELECT](../research/paper-34-elect-ec-tiering.md): hot/cold tiering empirically validated at production scale; cold-EC band matches replication performance in normal mode within 3%
- [Paper 59 — HALO](../research/paper-59-halo-hotness-aware-coding.md): confirms smaller-`k` mechanism and supplies the deployable static-template pattern
- [Paper 60 — Zebra](../research/paper-60-li-zebra-demand-aware-erasure-coding.md): supplies the geometric-programming method for deriving `k_hot` and the Cauchy RS migration mechanism
