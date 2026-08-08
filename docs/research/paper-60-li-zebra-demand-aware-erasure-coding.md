# Paper 60 — Zebra: Demand-Aware Erasure Coding for Distributed Storage Systems

**Authors:** Jun Li, Baochun Li (University of Toronto)
**Venue / Year:** IEEE/ACM IWQoS 2016 (conference); extended journal version in IEEE Transactions on Cloud Computing, 2019
**Citations:** widely cited foundational demand-aware EC paper; cited directly by HALO (Paper 59, ref [28])
**Topics:** #5 Peer Selection (storage type), #3 Erasure Coding
**ADRs produced:** ADR-018 (revised, jointly with Paper 59)

---

## Problem Solved

Choosing a single erasure-coding rank for an entire storage fleet forces a bad trade-off: a low rank (small `k`) wastes storage on files nobody reads often; a high rank (large `k`) makes every reconstruction expensive, including for the small number of files that get read constantly. Zebra formulates per-file rank assignment as a geometric program — solve for the redundancy level `k_i` for each file `i` given its measured demand, under one global storage-overhead budget — and picks Cauchy Reed-Solomon codes specifically so that re-ranking a file later costs a handful of XOR operations, not a full re-encode. This is the algorithm ADR-018 has been missing: not just a motivation for hot/cold tiering, but a way to actually solve for the numbers.

---

## Key Findings

- Reconstruction overhead reduced up to 73.6% versus uniform Reed-Solomon coding, evaluated on real Facebook cluster traces.
- Migration/re-ranking overhead stays under 13% of demand traffic, because Cauchy RS matrices are constructed so the first `k` rows of a rank-`(k+x)` code are identical to the rank-`k` code's rows — re-ranking upward only requires generating the additional `x` parity pieces, not touching the existing ones.
- The rank-assignment problem is convex under a log transform (standard geometric-programming technique), so it solves efficiently even at fleet scale — this is a real algorithm with a solver, not a heuristic.
- An LRC variant of the same demand-aware assignment is also given, for deployments that want local (not just global) repair.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Cauchy RS specifically (not arbitrary generator matrices) | A generic RS library with no migration-friendly structure | Cheap re-ranking; constrains the encoder implementation to Cauchy-matrix construction specifically |
| Global storage-budget optimization across the whole fleet | Per-file independent parameter choice | Provably good aggregate storage/latency trade-off; requires centralized visibility into all files' demand, which needs a coordinator |
| Continuous demand monitoring driving the optimization | A one-time classification at write time | Adapts as access patterns genuinely shift; requires a metering system tracking per-file request rate over time |

---

## Breaks in Our Case

- **Zebra optimizes rank jointly across the entire file population against one global storage budget** ≠ **Vyomanaut's Hot/Cold choice is a per-file decision made once by the data owner at upload time (ADR-018's enrolment flow), with no fleet-wide optimizer**
  → Zebra's full geometric program is not directly deployable. The usable subset is the *per-file* sub-problem: given a target retrieval-latency SLA for "Hot" and one for "Cold," solve for the two corresponding `k` values once, offline — not a live per-file solver.

- **Zebra's demand signal is continuous per-file request-rate telemetry visible to a central controller** ≠ **Vyomanaut's coordination microservice does not currently meter per-file read frequency — it audits chunk possession, not access rate**
  → If Vyomanaut ever wants Zebra's live re-ranking (rather than a one-time static band choice), new metering infrastructure is required first. Not needed for the current static-band model.

---

## Decisions Influenced

- **[ADR-018](../decisions/ADR-018-hot-cold-storage-bands.md) [#5 Peer Selection]** `REVISED`
  Confirms the Cauchy RS construction as the concrete mechanism behind ADR-018's existing "RS monotonicity" migration claim (Paper 05 §6.1) and supplies the actual method (geometric program, solved once offline against Vyomanaut's target SLAs, not live) for deriving Hot-band `k`.
  *Because:* this is the paper HALO itself cites as its own precedent for demand-aware redundancy — reading it directly, rather than relying on HALO's secondhand description, confirms the mechanism generalizes beyond HALO's specific HDFS implementation.

---

## Disagreements

None — Zebra and HALO agree on the core mechanism (smaller `k` for hot data); they differ on adaptation cadence (Zebra: continuous re-optimization; HALO: coarse tier buckets), which is a deployability trade-off, not a factual disagreement. See Q59-1.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q59-1 (shared with Paper 59).
