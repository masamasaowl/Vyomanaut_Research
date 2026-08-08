# Paper 65 — A Bi-level Programming Model to Enhance the Encoding Efficiency of Erasure Codes (BiLP-BX)

**Authors:** Jiexin Chen, Zhongbo Hu, Chunhua Wang, Quan Zhou, Xinyi Zhang (Yangtze University)
**Venue / Year:** Expert Systems With Applications 331 (2026) 133334, Elsevier
**Citations:** not yet indexed (published June 2026)
**Topics:** #3 Erasure Coding, #11 Background OS Execution
**ADRs produced:** ADR-009 (revised), ADR-003 (evidence added)
**Findings addressed:** F-27 (the 5% CPU budget rests on one paper and no measurement)
**Reading list:** Band 0 #65; Domain F / R-20

---

## Problem Solved

ADR-009 asserts a ≤5% CPU budget for the provider daemon on the strength of Bolosky (2000), which measured *ambient* Microsoft-campus desktop load and not Vyomanaut's workload at all. F-27 named this the thinnest evidence base of all 58 ADRs. The reading list's instruction for this paper was explicit: **read it for the comparison table, not the optimisation method.**

The paper itself is about a real but narrow question. XOR-based erasure coding is conventionally optimised along two independent tracks — make the generator bit-matrix sparser (fewer ones), or schedule the XOR operations better (reuse intermediates via common-subexpression elimination). The authors show these two objectives are not aligned. Among Cauchy generator matrices sharing the *identical minimum* Hamming weight, the resulting XOR schedule count varies substantially; and a matrix with a *higher* Hamming weight can produce a *lower* XOR count, because its non-zeros are arranged to admit more intermediate reuse. Their worked example over GF(2³) at `(k,m) = (3,2)`: a minimum-weight matrix at HW = 25 needs 25 XOR steps, while a matrix at HW = 32 needs 23.

Sparsity is therefore a proxy, not the objective. BiLP-BX reformulates the problem as bi-level programming — matrix structure at the upper level constraining the feasible XOR schedules at the lower level — and solves it with an iterative greedy search using a penalty mechanism and tabu memory (IG-PTS).

For Vyomanaut, the method is not adoptable and the reading list already said so. What we came for is the measured throughput of a production erasure-coding library on commodity hardware, which no document in the corpus contains.

---

## Key Findings

### The extractable data — Jerasure Cauchy-RS throughput

The baseline is standard Cauchy Reed–Solomon in Jerasure with its default matrix generation and standard XOR scheduling. Conditions: Intel Core i5-13600KF, 32 GB DDR5, Ubuntu 22.04, gcc 11.3.0 `-O3`, **single-threaded**, **no SSE or AVX**, GF(2⁸), 10 MB dataset, 1,000 runs averaged.

Absolute Jerasure throughputs are not tabulated directly; they are recoverable from Fig. 10 (absolute BiLP-BX throughput) divided by Fig. 8/9 (percentage improvement). Derived, and flagged as derived:

| `(k, m)` | Jerasure (MB/s, derived) | BiLP-BX (MB/s) | Improvement |
| --- | --- | --- | --- |
| (4, 2) | 2,412 | 2,916 | 20.9% |
| (4, 3) | 1,417 | 1,958 | 38.2% |
| (4, 4) | 951 | 1,145 | 20.4% |
| (6, 2) | 1,211 | 4,560 | 276.4% |
| (6, 3) | 749 | 1,331 | 77.8% |
| (6, 4) | 521 | 869 | 66.7% |
| (8, 2) | 1,107 | 3,711 | 235.2% |
| (8, 3) | 594 | 1,406 | 136.6% |
| (8, 4) | 376 | 885 | 135.4% |
| (10, 2) | 680 | 3,693 | 442.8% |
| (10, 3) | 390 | 1,381 | 254.4% |
| **(10, 4)** | **258** | **788** | **205.2%** |

The headline 442.8% is at `(10, 2)`, not at the largest configuration. The relevant number for us is the last row: **258 MB/s single-threaded scalar Cauchy-RS at `(k=10, m=4)` on a current mid-range desktop CPU.**

### The shape of the curve

Throughput falls superlinearly in the coding work `k·m`. A log-log fit over all twelve points gives `T ≈ 40,335 · (k·m)^(−1.357)` with R² = 0.985 — a strong fit *within the tested range*.

### The XOR-count result

Against a two-stage sparsity-first baseline, BiLP-BX reduces XOR scheduling count by 0–11.4%, with several configurations at exactly 0%. Yet throughput improves 20–443%. **A 0–11% instruction-count reduction cannot produce a 205% throughput gain.** The paper attributes the gap to cache behaviour and instruction scheduling and does not decompose it. This is the single largest reason to treat the throughput numbers as an upper bound on what matrix optimisation buys, and to treat the *baselines* rather than the improvements as the reliable content — which is what we are using them for.

### Scope, stated by the authors

Single coding family (Cauchy RS), single scheduling model (Uber-CSHR), single-threaded, no GPU/FPGA/SIMD, no formal optimality or convergence guarantee for the bi-level formulation. All tested configurations satisfy `k ≤ 10` and `m ≤ 4`.

---

## Substitution at Vyomanaut's parameters — and why it stops short

Vyomanaut is `(k=16, m=40)`, so `k·m = 640` against the largest tested value of 40. That is **16× outside the fitted range**. Extrapolating the fit:

```
T(640) = 40,335 × 640^(−1.357) ≈ 6.3 MB/s
```

**This number should not be used for anything.** A power-law fit extrapolated 16× beyond its support, with no mechanistic basis and an unexplained 20× discrepancy between its own instruction-count and throughput axes, is not an estimate. It is recorded here only to establish a direction: the evidence runs unfavourably, and by an amount that matters.

What the paper *does* support is narrower and still useful:

1. A production XOR-based RS library, single-threaded and without SIMD, delivers **258 MB/s at `(10,4)`** on a CPU better than most Indian home desktops will have.
2. Throughput falls faster than linearly in `k·m` across every configuration tested.
3. Vyomanaut's coding work is 16× the largest configuration in that set.

Together those are sufficient to say the following, which is the actual deliverable for F-27: **ADR-009's ≤5% CPU budget has never been evaluated against the actual codec, the first real measurement in the corpus points the wrong way, and the gap is large enough that it cannot be assumed away.**

### The budget must be decomposed by role before it can be defended

The most useful thing this paper does is force the question of *whose* CPU. ADR-009 states one budget for "the desktop daemon" and the three costs land on three different machines at three different rates.

| Workload | Runs on | Rate | Assessment |
| --- | --- | --- | --- |
| RS(16,56) **encode** | data owner's client, at upload | once per uploaded byte | Bursty, user-initiated, user-visible. Not a background budget item at all — it is upload latency, and no NFR covers it. |
| SHA-256 over stored chunks | provider, daily full audit | 70 GB/day at the conservative tier | **Cheap.** 46.7 s/day (0.05% of a core) with SHA-NI; 467 s/day (0.54% of a core) scalar. Comfortably inside 5%. |
| ChaCha20-Poly1305 | provider, transfer path only | proportional to traffic | Bounded by the 100 Kbps background budget; negligible. |
| RS(16,56) **decode + re-encode** | repairing provider | 32 fragments reconstructed per repair event (ADR-004, `r0=8`) | **Unmeasured and unbudgeted.** The only place the extrapolation above could bite. |

The audit-hashing calculation confirms F-40's framing from the other side: the daily full audit's cost to a provider is **disk seeking, not CPU**. Hashing 70 GB/day costs half a percent of a core; seeking to 286,720 random chunks costs about an hour of drive time. ADR-009's CPU budget was never the binding constraint on the audit path, and the ADR should stop implying it is.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Bi-level formulation (matrix constrains schedule) | Weighted single-level multi-objective | Captures the nested dependency the paper's two phenomena demonstrate; NP-hard, non-convex, no optimality guarantee |
| Pessimistic bi-level model | Optimistic | Robustness under the worst-case scheduling response; a more conservative and less flattering objective |
| Tabu list (hard exclusion) plus penalty (soft diversification) | Either alone | The ablation shows neither alone suffices; adds two tunable parameters with no stated selection procedure |
| Cauchy RS over GF(2^w), Uber-CSHR scheduling | Vandermonde RS, other schedulers | Matches Jerasure's deployed configuration and makes the comparison fair; confines every result to XOR-decomposable codes |
| No SIMD, single-threaded | AVX2/AVX-512, multi-threaded | Isolates the algorithmic contribution honestly; puts every absolute figure well below what a modern library achieves |

---

## Breaks in Our Case

- **Vyomanaut does not use Cauchy RS, and probably should not start.** The paper's entire object is the Cauchy bit-matrix, whose encoding decomposes into pure XOR. Vyomanaut's codec (Go, GF(2⁸), RS(16,56)) is a Vandermonde-style construction using table-driven or SIMD-accelerated finite-field multiplication. BiLP-BX's optimisation has no point of attachment. The reading list's instruction — read for the table, not the method — is confirmed correct.

- **No SIMD means every absolute number is a floor, not an estimate.** Modern Go RS libraries use AVX2 table lookup and reach GB/s at datacentre parameters. The measured 258 MB/s at `(10,4)` is what you get with the optimisation deliberately switched off. Using it as a prediction understates real throughput by an unknown but large factor; using it as a *floor* is defensible and is how it is used above.

- **`(16,40)` is 16× outside the tested envelope, and the failure mode of the fit is unknown.** Larger `m` may degrade gracefully (more parity rows over the same cached input, good locality) or catastrophically (working set exceeds L2, table thrashing). The paper's own unexplained gap between XOR count and throughput says cache effects dominate, which means the direction cannot be inferred from instruction counts. There is no substitute for measuring Vyomanaut's actual codec at `(16,40)`.

- **The improvement percentages are not reproducible from the paper's own instruction counts.** 0–11.4% fewer XOR operations producing 20–443% more throughput is a 20× discrepancy the paper acknowledges as cache and scheduling effects and does not decompose. Anything cited from this paper should be the *baseline* column, which is a straightforward measurement of an unmodified library, and not the improvement column.

- **A 10 MB working set is not a storage workload.** Encoding 10 MB repeatedly measures a mostly cache-resident case. Vyomanaut encodes 4 MB stripes drawn from files that are not in cache, on machines simultaneously serving audits and running a WebView2 GUI.

- **The CPU class is favourable.** An i5-13600KF with DDR5 is well above the min-spec Indian home desktop ADR-009 is budgeting for. The numbers are optimistic in hardware and pessimistic in instruction-set usage, and the two do not cancel in any way that can be quantified from what is published.

---

## Decisions Influenced

- **[ADR-009](../decisions/ADR-009-background-execution.md) [#11 Background OS Execution]** `REVISED`
  Replaces "one paper from 2000 measuring ambient load" with a decomposition by role, a measured floor for erasure-coding throughput on commodity hardware, and a computed figure showing that the audit path's CPU cost is negligible while its disk cost is not. Downgrades the ≤5% claim from an assertion to an open constraint with a named measurement task.
  *Because:* the derived baseline column is a direct measurement of an unmodified production library under stated conditions, and the SHA-256 arithmetic is checkable from first principles. Neither depends on the paper's contested improvement claims.

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `EVIDENCE ADDED`
  First data point in the corpus on what `(k, m)` actually costs to encode. Records that encode cost grows superlinearly in `k·m` and that `(16,40)` sits far outside every published benchmark, which is a consequence of `r = 40` that the ADR's parameter derivation never considered — Giroire's `∂BWavg/∂r = 0` optimises repair bandwidth and is silent on compute.

---

## Disagreements

- **Against ADR-009's evidence base, not its conclusion.** The ≤5% budget may well hold; the audit-path arithmetic here suggests the parts that were worried about are fine. But it was asserted from a 2000 measurement of a different workload, and the one part nobody costed — repair-time decode and re-encode of 32 fragments at `(16,40)` — is the part the only available benchmark data points away from. F-27 stands; this paper narrows it rather than closing it.

- **Against the paper's own headline framing.** "442.8% encoding throughput improvement" is a `(10,2)` result, is not reproducible from the paper's XOR-count table, and is not the reason to read this paper. The reading list was right to specify the comparison table.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q65-1.
