# Paper 62 — LESS is More for I/O-Efficient Repairs in Erasure-Coded Storage

**Authors:** Keyun Cheng, Guodong Li, Xiaolu Li, Sihuang Hu, Patrick P. C. Lee (CUHK / Shandong University / HUST)
**Venue / Year:** USENIX FAST 2026 (24th Conference on File and Storage Technologies), pp. 631–641
**Citations:** not yet indexed (published February 2026)
**Topics:** #3 Erasure Coding, #17 Repair Bandwidth Optimisation, #4 Repair Protocol
**ADRs produced:** ADR-003 (revised), ADR-004 (revised), ADR-026 (revised — decision escalated)
**Findings addressed:** F-29 (wide stripes), F-43 (Clay sub-packetisation — **confirmed**), F-44 (Hitchhiker single-source)
**Reading list:** Band 0 #62; Domain C / R-09

---

## Problem Solved

Repair-friendly erasure codes have been optimising the wrong metric. The MSR line — Clay, Butterfly, PM-RBT — minimises *repair I/O*, the bytes read from local disk during a single-block repair, and pays for it with exponential sub-packetisation `α`, which shatters each block into thousands of sub-blocks scattered across the platter. Minimising bytes read while multiplying the count of non-contiguous reads is a bad trade on real hardware. The paper's central observation is that repair performance depends on bytes read *and* seek count together, that driving the first to its information-theoretic minimum provably requires an exponential `α`, and that the resulting seek count erases the gain.

LESS makes I/O seeks a first-class metric. It layers `α+1` *extended sub-stripes* of Vandermonde-based Reed–Solomon over a regular stripe, arranged so that every block's sub-blocks live entirely inside one extended sub-stripe. A single-block repair is then an RS decode within that one sub-stripe, at a small, bounded, operator-chosen `α` — as low as two — rather than an exponential one. The construction stays MDS, systematic and defined for general `(n,k)`, so it drops into an existing RS stack rather than replacing the algebra.

For Vyomanaut this is the first primary wide-stripe repair source in the corpus (F-29). Its most valuable contribution, however, is a negative one: its multi-block fallback rule, combined with ADR-004's `r0 = 8`, shows that the entire single-block repair-optimisation family — including the family ADR-026 has already committed V3 to — delivers nothing at Vyomanaut's stated repair policy.

---

## Key Findings

### The construction

- Parameters `(n, k, α)` with `k < n` and `2 ≤ α ≤ n−k`. Each block splits into `α` sub-blocks; the `nα` sub-blocks are organised into `α+1` extended sub-stripes, each an `(|X_z|, |X_z|−n+k)` Vandermonde-based RS stripe tolerating any `n−k` sub-block failures.
- The `n` blocks partition into `α+1` block groups whose sizes differ by at most one: `|G_z| = ⌈n/(α+1)⌉` or `⌊n/(α+1)⌋`. Extended sub-stripe size is `|X_z| = n + (α−1)|G_z|`.
- Each sub-block belongs to exactly two extended sub-stripes. The `(α+1)`-th sub-stripe needs no explicit encoding — it satisfies its parity-check equation automatically once the first `α` are encoded, which is why the scheme adds no storage.
- Every RS property survives: MDS, general `(n,k)`, systematic. This is the "adoptable, not merely interesting" property Band 0 flagged.

### Repair cost — Equation (7)

```
IO_i = k + (α−1)·⌈n/(α+1)⌉   if g_i ≤ n mod (α+1)
     = k + (α−1)·⌊n/(α+1)⌋   otherwise          [units: sub-blocks]

I/O seeks = k + α − 1                            [uniform across all n blocks]
```

Repair cost is near-identical for any data *or* parity block, differing by at most `α−1` sub-blocks, and each of the `k+α−1` helpers contributes exactly one seek. This balance across all `n` shard positions is what separates LESS from Hitchhiker and HashTag — and at `n−k = 40` it is decisive. See Paper 64.

### Reported results at `(14,10)` (paper Table 2)

| Code | α | Avg repair I/O (blocks) | Avg I/O seeks |
| --- | --- | --- | --- |
| RS | 1 | 10.00 | 10.00 |
| Clay | 256 | 3.25 | 286.00 |
| Hitchhiker | 2 | 7.50 | 10.86 |
| HashTag | 4 | 6.04 | 12.14 |
| ET | 4 | 5.86 | 14.29 |
| **LESS** | **4** | **4.64** | **13.00** |

At α = 4, LESS cuts average repair I/O by 53.6% against RS, 38.1% against Hitchhiker, 23.1% against HashTag and 20.7% against ET, while cutting Clay's average seek count by 95.5%.

Testbed — 15 machines, HDFS 3.3.4 + OpenEC, 7200 RPM SATA HDD, 64 MiB blocks, 256 KiB packets, 1 Gbps — single-block repair time falls 50.8% against RS and 33.9% against Clay; full-node recovery falls 48.3% and 36.6%. The advantage over Clay *widens* as network bandwidth rises (83.3% at 10 Gbps) and as packet size falls (50.4% at 128 KiB), because both changes make seeks a larger share of repair time.

Wide stripes: at `(124,120)`, LESS at α = 4 cuts RS's repair I/O by 59.5% for a few extra seeks.

### Encoding cost

Single-threaded, 256 KiB packets, `(14,10)`: RS reaches 2.8 GiB/s, LESS at α = 4 reaches 1.6 GiB/s — a 43% cut from sub-packetisation. At `(124,120)` the figures are 2.6 GiB/s and 1.1 GiB/s. This is an input to F-27 alongside Paper 65.

### Clay sub-packetisation — **F-43 independently confirmed**

The paper gives Clay's sub-packetisation as `α = (n−k)^⌈n/(n−k)⌉`. Substituting Vyomanaut's parameters, working shown per the standing rule:

```
n = 56,  k = 16,  n − k = 40
⌈n/(n−k)⌉  = ⌈56/40⌉ = 2
α          = 40² = 1,600
sub-chunk  = 256 KB / 1,600 = 163.8 bytes
```

ADR-026 asserts `α ≥ 40^16 ≈ 4.29 × 10²⁵`. The correct value is **1,600**, smaller by a factor of 2.7 × 10²². This is a second, independent confirmation of F-43, reached through a different formula than the interrogation used (Goparaju's `d = n−1` form), and the two agree. Sanity check against the paper's own table: at `(14,10)`, `4^⌈14/4⌉ = 4⁴ = 256`, exactly the α it reports for Clay. ✓

---

## Substitution at Vyomanaut's parameters

Equation (7) evaluated at `(n=56, k=16)`. The model was first validated by reproducing the paper's own Table 2 at `(14,10)` — average, minimum, maximum and seek count for α = 2, 3 and 4, twelve figures in all, matching to published precision.

| α | Sub-block on a 256 KB fragment | Avg repair I/O (blocks) | vs RS = 16 | I/O seeks (fan-in) |
| --- | --- | --- | --- | --- |
| 2 | 128.0 KB | 17.34 | **−8.4% (worse)** | 17 |
| 3 | 85.3 KB | 14.67 | +8.3% | 18 |
| **4** | **64.0 KB** | **12.41** | **+22.4%** | **19** |
| 5 | 51.2 KB | 10.69 | +33.2% | 20 |
| 6 | 42.7 KB | 9.33 | +41.7% | 21 |

**LESS is a net loss at α = 2 for Vyomanaut.** The paper states its sufficiency argument for `k > ⌈n/3⌉`, describing that as the common case. The general form is `k > ⌈n/(α+1)⌉`; at `k=16, n=56` that requires `α ≥ 3`, with usable margin only from `α = 4`.

The reason is code rate, not stripe width. Every configuration LESS evaluates sits at `k/n ≥ 0.71` — `(14,10)` is 0.714, `(124,120)` is 0.968. Vyomanaut is at `k/n = 0.286`, the opposite end of the parameter space. The parenthetical "common in practice" carries real weight. **The corpus should stop calling `n=56` the anomaly. `r=40` is.**

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Small, configurable `α` (2–4) at near-minimum repair I/O | Exponential `α` at provably minimum repair I/O (Clay) | 95.5% fewer seeks for 1.4× the bytes read; wins on wall-clock on HDDs, and the margin grows as packets shrink and networks widen |
| Balanced reduction across all `n` blocks | Data-block-only reduction (Hitchhiker, HashTag) | Every shard position benefits — decisive at `n−k = 40`, where 71% of shards are parity |
| Layering RS sub-stripes | A new algebraic code family | MDS, systematic, general `(n,k)`, drops into an RS stack; costs 43% encode throughput and a larger Galois field |
| Repair confined to one extended sub-stripe | Repair across the whole stripe | Fan-in rises from `k` to `k+α−1`; each helper does strictly less work, but more helpers must be simultaneously reachable |
| Vandermonde parity-check formulation, multiplicatively generated coefficients | Free choice of coding coefficients | Coefficients must be brute-force searched for MDS; that search has only ever been run for `n−k ≤ 4` |

---

## Breaks in Our Case

- **LESS optimises single-block repair; ADR-004 does not perform single-block repair.** For multi-block failures outside one block group, the paper prescribes falling back to conventional repair — retrieve `k` blocks. Its intra-group multi-block path requires every failed block to sit in the same block group. ADR-004 defers repair until a chunk drops to `s + r0 = 24` available fragments: **32 fragments missing, in one batch, spread across all `α+1 = 5` block groups**. Two ways to cost it, both bad:
  - *Conventional fallback* (what the paper prescribes): read `k = 16` blocks, reconstruct all 32. Saving = **0%**.
  - *Per-group repair*: `5 × ~12.4 ≈ 62` blocks to reconstruct the same 32 fragments, versus 16 for plain RS — **~3.9× worse**. RS's conventional repair amortises one `k`-block read across every missing shard; LESS's per-group repair does not.

  → **At ADR-004's stated repair policy, LESS delivers zero benefit.** This is not a defect in LESS. It is a collision between two optimisations that are each correct in isolation: LESS prices savings per repair *event*, and lazy repair is a deliberate choice to have very few, very large events. See ADR-026 and Q62-1.

- **The 98.08% single-block statistic is a property of eager repair, not of the failure process.** This whole literature line — Hitchhiker included — rests on Facebook's measurement that ~98% of degraded stripes are missing exactly one block. HDFS repairs promptly, so failures are observed one at a time. Vyomanaut batches by construction. ADR-026's open constraint Q19-3 asks whether the 98% figure holds for P2P consumer networks; the question is mis-posed. With `r0 = 8`, single-fragment repair events are ~0% of repair events **by policy**, whatever the underlying failure process does.

- **The MDS field-size condition is unverified at `n−k = 40`.** The paper's sufficient condition is `2^w ≥ nα + (n−k−1)·C(n−1,k)`. Substituting:

  ```
  C(55,16)            = 29,749,251,314,475        (≈ 2^44.8)
  (n−k−1)·C(n−1,k)    = 39 × 2.975e13 ≈ 1.16e15   (≈ 2^50.0)
  nα at α = 4         = 224 distinct coefficients
  ```

  The condition demands `w ≥ 50`; GF(2⁸) and GF(2¹⁶) both fail it. It is only *sufficient*, and the paper notes smaller fields work for typical parameters — but its table of feasible primitive elements covers only `n−k ≤ 4` and `2 ≤ α ≤ 4`. Vyomanaut's `n−k = 40` has never been searched by anyone. Independently, α = 4 needs 224 distinct non-zero coefficients and GF(2⁸) holds only 255, so GF(2¹⁶) is the realistic floor — a codec change, not a parameter change.

- **Block size is off by 256×.** LESS evaluates 64 MiB blocks with 256 KiB packets. Vyomanaut's *entire fragment* is 256 KB (`lf`, ADR-003). At α = 4 the sub-block is 64 KB. On a 7200 RPM HDD a ~12 ms seek dwarfs the ~0.5 ms needed to stream 64 KB, so the seek term dominates absolutely and the byte reduction buys almost nothing in local disk time. The paper's seek accounting implicitly assumes seek and transfer terms are comparable. What the reduction *does* buy at our fragment size is **network bytes** — which, given Paper 34's finding that repair is 93.3% network-bound, is the term that actually matters. State the benefit as bandwidth, not I/O.

- **Fan-in moves the wrong way for F-29.** F-29 objects that repair fan-in scales with `k` — one shard loss touches 16 providers. LESS at α = 4 makes it **19**. It trades fan-in for bytes. That trade is probably right for Vyomanaut, but it does not answer the concern F-29 raised, and under ADR-021's NAT and relay constraints three more simultaneously-reachable peers per repair is not free.

- **The audit unit becomes ambiguous.** ADR-002's challenge is `SHA-256(chunk_data ‖ nonce)` over a whole 256 KB fragment. Under LESS a fragment is `α` sub-blocks with internal coding structure. Whether the challenge should address the fragment or a sub-block is the same structural question ADR-026 already raises for Clay, at a gentler magnitude — but it is the same question, and it lands on ADR-002, ADR-017 and ADR-020 together.

- **One internal inconsistency to resolve before implementing.** §3.4 describes single-block repair "for a failed block `B_i` in `G_z` (`1 ≤ z ≤ α`)", but the construction defines `α+1` block groups and Equation (7) ranges over all `1 ≤ i ≤ n`. `X_{α+1}` is an RS stripe containing all of `G_{α+1}`, so the `(α+1)`-th group is repairable and the §3.4 range reads as a typo. Confirm against the released artifact (`github.com/adslabcuhk/less`, v1.0.0) rather than assuming — exactly the substitution-shown discipline F-43 earned.

---

## Decisions Influenced

- **[ADR-026](../decisions/ADR-026-repair-bw-optimisation.md) [#17 Repair BW Optimisation]** `REVISED — decision escalated to design council`
  Supplies three things: a second independent confirmation that Clay's `α` at our parameters is 1,600 and not `40^16`, closing F-43; a third candidate family whose reduction is balanced across data *and* parity, which at `(56,16)` beats Hitchhiker outright (see Paper 64); and the finding that under ADR-004's `r0 = 8`, every member of the single-block repair-optimisation family delivers **zero** saving. The V3 code-family question is downstream of a repair-policy question nobody has asked.
  *Because:* the paper's own multi-block fallback rule and Equations (3), (4) and (7) are sufficient to establish this exactly. No further literature is required.

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `REVISED`
  First primary wide-stripe source in the corpus (F-29). Confirms `n=56` sits comfortably inside the range modern repair-friendly codes are constructed for, and that wide stripes are where they pay best. It also reframes the anomaly: `k/n = 0.286` is what puts Vyomanaut outside the tested envelope, not `n = 56`.

- **[ADR-004](../decisions/ADR-004-repair-protocol.md) [#4 Repair Protocol]** `REVISED`
  `r0 = 8` now carries a cost that was never priced: it forecloses every single-block repair optimisation in the literature. Giroire's 38× eager-vs-lazy saving is far larger than the 22–47% these codes offer, so lazy repair almost certainly still wins — but the two are **not** orthogonal, which is what ADR-026 currently asserts, and ADR-004 should record the interaction.

---

## Disagreements

- **This paper's Table 1 vs ADR-026's characterisation of Hitchhiker.** ADR-026 describes Hitchhiker's structural burden as "two coupled sub-stripes, sequential I/O preserved" and its benefit as a flat 25–45%. Table 1 classifies Hitchhiker's reduced repair I/O for data/parity as **No** — the reduction is data-blocks-only. At `(14,10)` that forfeits 4 of 14 positions; at `(56,16)` it forfeits **40 of 56**. ADR-026 never states the restriction. Full computation in Paper 64.

- **ADR-026 vs this paper on *why* Clay is rejected.** ADR-026 rejects Clay as computationally intractable. LESS rejects it on **I/O seeks** — 286 average versus RS's 10 at `(14,10)` — while explicitly conceding it is I/O-optimal. F-43 predicted this outcome: the rejection survives, but on the I/O argument, and that argument has a shelf life. Intractability never improves; a seek-cost argument weakens as providers move to SSDs, and this paper's own bandwidth and packet-size sweeps show exactly which way that trend runs.

- **This paper vs Paper 39 (Silberstein) on lazy-repair orthogonality.** ADR-026 cites Paper 39 for "lazy recovery and bandwidth-efficient codes are orthogonal … Xorbas+LAZY outperforms either alone by more than 2×." Xorbas is an LRC; its local groups repair small numbers of losses independently and therefore compose with accumulation differently from a single-block-optimised MDS code. The arithmetic above shows the orthogonality claim does **not** transfer to Hitchhiker or LESS at `r0 = 8`. Whether Paper 39's degradation regime is comparable to ours has never been checked, and should be, before ADR-026's orthogonality sentence is relied on again.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q62-1 and Q62-2.
