# Paper 64 — A "Hitchhiker's" Guide to Fast and Efficient Data Reconstruction in Erasure-coded Data Centers

**Authors:** K. V. Rashmi, Nihar B. Shah, Dikang Gu, Hairong Kuang, Dhruba Borthakur, Kannan Ramchandran (UC Berkeley / Facebook)
**Venue / Year:** ACM SIGCOMM 2014, Chicago
**Citations:** high — the canonical piggybacking-framework systems paper (exact count not verified in this pass)
**Topics:** #17 Repair Bandwidth Optimisation, #3 Erasure Coding, #2 Proof of Storage (audit unit)
**ADRs produced:** ADR-026 (revised — decision escalated), ADR-004 (revised)
**Findings addressed:** F-44 (Hitchhiker committed on one survey figure — **now read**), F-43 (context), F-29
**Reading list:** Band 0 #64; Domain C / R-10

---

## Problem Solved

ADR-026 commits Vyomanaut's V3 chunk layout, pointer-file schema and audit challenge design to the piggybacking code family on the strength of a single band — 25–45% — read off one figure in one survey (Paper 19). F-44 objected that the primary source had never been read. This is that source.

The engineering problem Hitchhiker solves is real and well posed. Under an `(k, r)` RS code, reconstructing one lost unit requires downloading `k` whole units — `k` times the size of the thing being reconstructed. At Facebook's data-warehouse scale that was measured at a median of more than 180 TB per day crossing top-of-rack switches purely for reconstruction. Hitchhiker rides on top of an existing RS code using the piggybacking framework: pairs of RS stripes are coupled into one stripe of two sub-stripes, and functions of the first sub-stripe's data are XORed into the second sub-stripe's parities. Fault tolerance and storage overhead are provably unchanged, because the added functions are computable from data already recoverable and each occupies exactly one byte.

It also contributes a second, separable idea — *hop-and-couple* — which is the reason the network saving translates into a disk saving rather than into a storm of alternate-byte reads. That idea is more portable than the code itself.

---

## Key Findings

### The three variants

| Variant | Requires | Saving at `(10,4)` | Cost |
| --- | --- | --- | --- |
| Hitchhiker-XOR | nothing beyond RS | 35% (first six units), 30% (last four) | XOR only |
| Hitchhiker-XOR+ | underlying RS has an all-XOR parity | 35% uniformly across data units | XOR only |
| Hitchhiker-nonXOR | nothing | 35% uniformly | finite-field arithmetic |

Reconstruction is a three-step procedure: RS-decode the second sub-stripe from `k` accessed bytes, subtract the recovered components from the piggybacks, XOR (or RS-decode) out the target. All three variants have the **repair-by-transfer** property — helper nodes send stored bytes and perform no computation.

### The general-`(k,r)` formulas (Appendix)

These are the load-bearing part for us, and they are in the appendix rather than the body, which is presumably why the survey reported a band instead of a function:

- The `k` data units are partitioned into `r−1` disjoint sets of roughly equal size.
- A data unit in a set of size `s` is reconstructed from `k + s` bytes, against `2k` under RS.
- Under XOR+, the last `ℓ` units (for a chosen `ℓ ∈ {0..k}`) instead cost `k + r + ℓ − 2` bytes; `ℓ` is tuned to minimise the average or maximum.

### The restriction the survey figure omits

Hitchhiker optimises reconstruction of **data units only**. Parity units are reconstructed exactly as under RS. This is stated plainly in §3 and confirmed independently by Paper 62's Table 1, which classifies Hitchhiker's reduced repair I/O for data-and-parity as *No*. At `(10,4)` this forfeits 4 of 14 shard positions and the effect on the average is modest. At Vyomanaut's `(56,16)` it forfeits **40 of 56**.

### Multi-block behaviour

If more than one unit of a stripe is unavailable, Hitchhiker reconstructs exactly as RS does — reading `k` entire units. The paper justifies confining itself to the single-unit case with a six-month Facebook measurement: 98.08% of degraded stripes were missing exactly one unit, 1.87% two, 0.05% three or more. It also considers and rejects deliberately waiting for more failures before repairing, on the grounds that read performance requires prompt repair.

### Hop-and-couple

Coupling adjacent bytes into sub-stripes makes reconstruction read alternate bytes across nine units — the network saving is real and the disk saving is destroyed. Coupling bytes half a block apart (`hop-length = B/α`) makes every sub-stripe read contiguous. The paper notes the technique applies to any code with `α` sub-stripes, not just this one.

### Measured results at `(k=10, r=4)`, 256 MB blocks, production Facebook cluster

| Metric | vs RS-based HDFS-RAID |
| --- | --- |
| Data downloaded / read during reconstruction | 35% less |
| Median data read time | 31.8% less |
| 95th-percentile read time | 30.2% less |
| Median reconstruction computation time | 36.1% less |
| **Median encoding time** | **72.1% more** |

The encoding penalty is argued away on the grounds that encoding is a one-time background job while reconstruction is repeated and degraded reads are served in real time.

---

## Substitution at Vyomanaut's parameters

`(n=56, k=16, r=40)`, `lf = 256 KB`. The model below was validated by reproducing Paper 62's independently published Table 2 entry for Hitchhiker at `(14,10)` — average 7.50, minimum 6.50, maximum 10.00 blocks — which it matches exactly under XOR+ with `ℓ = 1`. That agreement is what licenses the extrapolation.

**Step 1 — the sets degenerate.** The construction partitions `k = 16` data units into `r − 1 = 39` sets. There are more sets available than units to place, so every set is a **singleton**, `s = 1`, and 23 of the 39 piggyback slots go unused.

**Step 2 — `ℓ = 0` is forced.** Under XOR+, the last `ℓ` units cost `k + r + ℓ − 2 = 54 + ℓ` bytes against `2k = 32`. Any `ℓ > 0` is catastrophically worse, so the optimal `ℓ` is 0 and XOR+ degenerates to plain Hitchhiker-XOR. (At `(10,4)`, by contrast, `ℓ = 1` is optimal — which is why the published `(14,10)` numbers look the way they do.)

**Step 3 — the data-unit saving.**

```
repair of a data unit = k + s = 16 + 1 = 17 half-fragments
                      = 8.5 fragments,  against 2k = 32 half-fragments = 16
saving = 1 − 17/32 = 46.9%
helpers contacted    = 17   (15 data + 2 parity, one half-fragment each)
```

**46.9% — above the top of the 25–45% band ADR-026 cites.** The band was measured at high code rates; Vyomanaut's very large `r` is what pushes it out. On data units alone, Hitchhiker is *better* for us than the ADR claims.

**Step 4 — the parity units, which the ADR never accounts for.**

```
parity units = 40 of 56 shards = 71.4%,  repaired at full RS cost = 16 fragments

expected repair cost over a uniformly chosen lost shard
   = (16/56)(8.5) + (40/56)(16.0) = 13.86 fragments
expected saving = 13.4%
```

**Hitchhiker's real expected benefit at Vyomanaut's parameters is 13.4%, not 25–45%.** The ADR's figure is roughly 2–3× optimistic, and the error is not in the survey — it is in applying a data-unit figure to a code where 71% of shards are parity.

### Head-to-head against Paper 62 at `(56,16)`

| Code | Expected repair I/O (fragments) | Saving | Fan-in |
| --- | --- | --- | --- |
| RS (status quo) | 16.00 | — | 16 |
| **Hitchhiker** | **13.86** | **13.4%** | 17 (data) / 16 (parity) |
| LESS, α = 3 | 14.67 | 8.3% | 18 |
| **LESS, α = 4** | **12.41** | **22.4%** | 19 |
| LESS, α = 5 | 10.69 | 33.2% | 20 |

At α ≥ 4, LESS beats Hitchhiker on the metric ADR-026 selected Hitchhiker for — because its reduction is balanced across data and parity and Hitchhiker's is not, and because `r = 40` maximises how much that asymmetry costs. **ADR-026's ranking of the two candidate families inverts at Vyomanaut's parameters.** It does not invert at the parameters the survey figure was measured at, which is why it was not visible before.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Piggybacking on top of RS | A new code family | MDS, arbitrary `(k,r)`, uses the existing RS encoder/decoder as a building block; can fall back to plain RS operation at any time |
| `α = 2` sub-stripes | Higher sub-packetisation | Minimal layout disruption and contiguous reads under hop-and-couple; caps the achievable saving well below the MSR bound |
| Optimising data units only | Balanced optimisation | Simple construction and repair-by-transfer; forfeits `r/(n)` of all shard positions — 7% at `(14,10)`, **71% at `(56,16)`** |
| 72.1% higher encoding time | Faster reconstruction | Correct for a write-once archive; encoding is one-time and offline, reconstruction is repeated and latency-critical |
| Contacting more than `k` machines | Staying at exactly `k` | Measured to *reduce* read latency at Facebook because each machine sends half a block; costs reachability, which is not free under ADR-021's NAT/relay constraints |
| Hop-length = `B/2` | Adjacent-byte coupling | Fully contiguous sub-stripe reads; forces reconstruction of a byte's distant partner even when only part of a block is wanted |

---

## Breaks in Our Case

- **The single-block premise does not hold under ADR-004.** Hitchhiker falls back to plain RS whenever more than one unit of a stripe is missing. ADR-004 defers repair until 32 of 56 fragments are gone. **Under ADR-004 as written, Hitchhiker's saving is 0%.** The paper anticipates and explicitly rejects the accumulate-then-repair strategy — on read-latency grounds that apply to a warm data warehouse and not obviously to a write-once cold archive, but it is a considered rejection, not an oversight. Paper 62 reaches the same conclusion by a different route. This is the finding that should reorder ADR-026's whole discussion, and it is developed there.

- **The 98.08% single-block statistic is a property of HDFS's eager repair.** It describes how often a stripe is observed with exactly one unit missing in a system that repairs promptly. It is not a claim about the failure process, and it cannot be carried into a system that batches by policy. ADR-026's open constraint Q19-3 asks whether the 98% figure holds for P2P consumer networks; the honest answer is that the question does not survive contact with `r0 = 8`.

- **`r = 40` puts us outside the intended design point in a way that helps and hurts.** It helps: 39 available sets for 16 data units means singleton sets and a 46.9% data-unit saving. It hurts, more: 71% of shards are parity and get nothing. Both effects come from the same parameter, and the ADR captures neither.

- **Hop-and-couple assumes a block much larger than the buffer.** The technique is specified for 256 MB HDFS blocks with 1 MB buffers, hopping half a block. Vyomanaut's entire fragment is 256 KB — smaller than the paper's read buffer. At `α = 2` the hop is 128 KB and both sub-stripes are already contiguous within a single small read, so hop-and-couple is trivially satisfied and contributes nothing. No harm, but it should not be listed as a benefit.

- **The audit challenge is affected, and ADR-026 says so without saying how.** ADR-026 records that "audit challenge (ADR-002) must address coupled sub-stripe encoding." Concretely: a parity fragment under Hitchhiker-XOR is not `f_j(b)` but `f_j(b) ⊕ a_i`, so the bytes a provider stores are not the bytes a naive re-derivation would predict. `SHA-256(chunk_data ‖ nonce)` over what the provider actually holds still works unchanged — the challenge is over stored bytes, not over semantically-typed shards. What does change is the *pointer file* (ADR-020, ADR-022), which must record the coupling so a repairer knows which piggyback function sits where. This is smaller than ADR-026 implies but it is not nothing, and it is untested against the F-32 audit redesign that Domain A will produce.

- **Reconstruction requires the second sub-stripe of the first parity unit, always.** Step 1 of decoding RS-decodes `b` from `{b_1..b_k, f_1(b)}` minus the target. Vyomanaut's placement gives no shard a privileged role; under Hitchhiker, specific parity positions become mandatory helpers for specific data positions. That interacts with ADR-014's 20% ASN cap and ADR-005's assignment in ways nobody has looked at: if the holder of parity 1 is unreachable, the optimised path is unavailable and repair falls back to RS.

- **Every measurement is from a datacentre.** 1 Gb/s to the rack, 8 Gb/s above it, 60-machine test cluster, thousands-of-machines production cluster, MapReduce-driven repair. Vyomanaut is 100 Kbps of background budget per peer across residential Indian broadband, with repair performed peer-to-peer by a provider daemon. The *ratios* transfer; none of the latency figures do.

---

## Decisions Influenced

- **[ADR-026](../decisions/ADR-026-repair-bw-optimisation.md) [#17 Repair BW Optimisation]** `REVISED — decision escalated to design council`
  Closes F-44: the primary source is now read, and it does not support the ADR's stated numbers at our parameters. The 25–45% band becomes 46.9% on 16 of 56 shards and 0% on the other 40, for an expected 13.4%. Combined with Paper 62, the candidate ranking inverts, and both candidates go to zero under ADR-004's repair policy. The code-family decision should not be re-made unilaterally on this basis; it should go to council together with the repair-policy question that now sits upstream of it.
  *Because:* the appendix formulas are general in `(k, r)` and the substitution reproduces Paper 62's independently published `(14,10)` figures exactly, so the `(56,16)` numbers are computed rather than extrapolated.

- **[ADR-004](../decisions/ADR-004-repair-protocol.md) [#4 Repair Protocol]** `REVISED`
  Supplies the other half of Paper 62's finding, from the opposite direction: this paper considers batching failures before repair and rejects it, for reasons Vyomanaut should either adopt or explicitly decline. `r0 = 8` and single-block repair optimisation are alternatives, not complements.

---

## Disagreements

- **Against ADR-026's "25–45% BW reduction" for Vyomanaut.** Correct for the surveyed parameters, roughly 2–3× optimistic for ours once parity shards are counted. Corrected in the ADR.

- **Against ADR-026's "sub-packetisation is two sub-stripes — sequential I/O preserved" as a distinguishing advantage.** True, but at `lf = 256 KB` it distinguishes nothing: LESS at α = 4 gives 64 KB sub-blocks that are equally contiguous, and Clay's 164-byte sub-chunks (F-43's corrected figure) are the only entry in the comparison for which contiguity is genuinely at risk. The argument was doing real work when Clay's α was believed to be `40^16`; with the corrected α it separates Hitchhiker from nothing in the candidate set.

- **Against ADR-026's orthogonality claim, inherited from Paper 39.** ADR-026 states that Hitchhiker combined with ADR-004's lazy repair "should achieve additive savings." This paper's own multi-block rule says the opposite: with more than one unit missing, Hitchhiker is RS. The additive result cited from Paper 39 was obtained for Xorbas, an LRC whose local groups compose with accumulation differently. Applied to Hitchhiker at `r0 = 8` the savings are not additive, they are absent.

- **Against the corpus's framing of `n = 56` as the anomaly.** Hitchhiker is defined for arbitrary `(k, r)` and behaves better on data units at our parameters than at Facebook's. What breaks is the data/parity split, which is a function of `r/n`, not of `n`. Consistent with Paper 62's finding that the low code rate, not the stripe width, is what puts Vyomanaut outside the literature's envelope.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q64-1.
