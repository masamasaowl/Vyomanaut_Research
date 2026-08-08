# ADR-026 — V3 Repair BW Optimisation: Clay (MSR) and Hitchhiker Candidates

**Status:** Accepted — **Decision under design-council review (opened by Papers 62 and 64)**
**Topic:** #17 Repair Bandwidth Optimisation
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 10, 19, 22, 36, 39, 55, **62, 64**

> **Revision note (Papers 62, 64).** Three corrections and one escalation, recorded in full below and
> summarised here so nothing downstream is read from the stale version:
>
> 1. **Clay's sub-packetisation is `α = 1,600`, not `α ≥ 40^16`.** F-43 confirmed by a second,
>    independent primary source. Clay stays ruled out, but on I/O grounds, not tractability.
> 2. **Hitchhiker's expected benefit at `(56,16)` is 13.4%, not 25–45%.** Hitchhiker optimises data
>    units only; 40 of our 56 shards are parity and get nothing. F-44 closed — the primary source is
>    read and it does not support the number this ADR carried.
> 3. **LESS (Paper 62) is a third candidate and outranks Hitchhiker at our parameters** — 22.4%
>    balanced across all 56 shards at `α = 4`, against Hitchhiker's 13.4%.
> 4. **Escalation.** Under ADR-004's `r0 = 8`, *every* candidate in this ADR — Clay, Hitchhiker and
>    LESS alike — delivers **zero** repair-bandwidth saving, because all of them optimise
>    single-block repair and lazy repair by construction never performs one. The code-family
>    question is downstream of a repair-policy question that has never been asked. **No new
>    code-family decision is made here.** See *Escalation to design council*.

---

## Context

[ADR-003](./ADR-003-erasure-coding.md) chose Reed-Solomon RS(s=16, r=40) with lazy repair triggered at r0=8. The Giroire
formulas ([Paper 10](../research/paper-10-giroire-lazy.md)) show that reconstructing a lost chunk requires contacting k=16 surviving
fragment holders — this is not bandwidth-optimal. For a single-chunk failure, RS must download
k entire chunks to reconstruct one. Information theory guarantees that this can be done with
less data (the regenerating code bound, Dimakis et al. 2010).

At V2 launch scale (hundreds of providers, desktop MTTF 180–380 days), the repair bandwidth
BWavg ≈ 39 Kbps/peer is well within the 100 Kbps background budget. Repair BW optimisation is
therefore a V3 concern, not a V2 blocker. However, the design space must be understood now so
that V3 provider daemon and chunk layout decisions do not foreclose the upgrade path.

[Paper 19](../research/paper-19-ec-survey.md) (EC Survey) provided the original landscape. It identifies two candidate code families:
regenerating codes (specifically Clay codes as the general-parameter MSR implementation) and
piggybacking codes (Hitchhiker), with quantified tradeoffs from Figure 8. **This ADR previously
rested on that one figure.** [Paper 64](../research/paper-64-hitchhiker-piggybacking.md) (the
Hitchhiker primary source) and [Paper 62](../research/paper-62-less-io-efficient-repair.md) (LESS,
FAST 2026) have now been read, and both change the picture materially.

An additional consideration at V3 scale: [Paper 36](../research/paper-36-dalle-failure-correlation.md) (Dalle et al.) proves the real repair bandwidth standard deviation is 22× higher than the independent model predicts at any practical deployment size. The BWavg ≈ 39 Kbps/peer that makes a 25–45% reduction appear marginal is a mean. At V3 scale, the burst demand during an ASN-level outage is the binding constraint, not the mean.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| RS codes only (V2 status quo) | Universally deployed; simple; no sub-packetization; **the only option whose saving does not depend on repair granularity** | Repair bandwidth is not optimal; must download k full chunks per failure |
| **Clay codes (MSR, general n,k)** | Up to 50% repair BW reduction; MDS; access-optimal; supports our (n=56, k=16) parameters; sub-packetisation is **1,600, not intractable** (corrected) | 164-byte sub-chunks at `lf`=256 KB; Paper 62 measures 286 average I/O seeks at (14,10) against RS's 10 — pathological on desktop HDDs |
| **Hitchhiker codes (piggybacking)** | 46.9% reduction **on data units** at our parameters — above the surveyed band; two sub-stripes only; sequential I/O preserved | **Optimises data units only.** At (56,16) that is 16 of 56 shards; expected saving over a uniformly-chosen lost shard is **13.4%** |
| **LESS (Paper 62, FAST 2026)** — *new candidate* | 22.4% at `α=4`, **balanced across all 56 shards**; MDS, systematic, general `(n,k)`; layers over RS rather than replacing it | Fan-in rises 16 → 19; needs GF(2¹⁶); MDS coefficient search never run at `n−k`=40; **net loss at `α`=2** for our code rate |
| LRC (Azure-style) | Local repair reduces I/Os by factor l | Non-MDS; local group co-locality cannot be guaranteed in consumer P2P; unchanged, still ruled out |

## Known Constraints (from ADR-003 and Paper 10)

**Current parameters:**

| Parameter | Value |
| --- | --- |
| s (reconstruction threshold) | 16 |
| r (redundancy fragments) | 40 |
| n (total shards) | 56 |
| r0 (lazy repair trigger) | 8 |
| lf (fragment size) | 256 KB |
| BWavg at MTTF=300 days | ~39 Kbps/peer |

**Why this ADR matters for V3:**
If a sub-packetised code is adopted in V3, the chunk layout changes: each chunk is split into
`α` sub-chunks. The provider daemon must store chunks at sub-chunk granularity. The pointer file
schema (ADR-020, ADR-022) must record the sub-packetisation parameters and — under Hitchhiker
specifically — the piggyback coupling. The audit receipt (ADR-017) challenge may need to address
a sub-chunk rather than a full chunk. These are structural changes that cannot be retrofitted
without a network-wide migration.

**Why LRCs are ruled out now:** unchanged. LRC local repair requires that all nodes in a local group
are co-located and simultaneously reachable. In a P2P consumer network with providers behind NAT on
different ISPs across India, group co-locality cannot be guaranteed, and non-MDS storage overhead is
an additional penalty. Rejected for V2 and V3 absent a fundamental change in provider topology.

---

## Correction 1 — Clay sub-packetisation (F-43, now confirmed)

This ADR previously stated that Clay's minimum sub-packetisation is `α ≥ 40^16` and rejected Clay as
"computationally intractable." **That figure is wrong by a factor of 2.7 × 10²².**

Paper 62 gives the MSR sub-packetisation bound as `α = (n−k)^⌈n/(n−k)⌉`. Substituting, with the
working shown per the standing rule:

```
n = 56,  k = 16,  n − k = 40
⌈n/(n−k)⌉  = ⌈56/40⌉ = 2
α          = 40² = 1,600
sub-chunk  = 256 KB / 1,600 = 163.8 bytes
```

Cross-check against Paper 62's own published table: at `(14,10)` the same formula gives `4⁴ = 256`,
which is exactly the `α = 256` it reports for Clay. The interrogation (F-43) reached 1,600 through
Goparaju's `d = n−1` form; the two independent routes agree.

**Clay remains ruled out, on the second argument, not the first.** 164-byte random reads are
pathological on desktop HDDs, and Paper 62 measures the consequence directly: 286 average I/O seeks
per repair at `(14,10)` against RS's 10.

This matters beyond tidiness, because the two arguments have different shelf lives. Intractability
never improves. A seek-cost argument weakens as providers move to SSDs — and Paper 62's own
sensitivity sweeps show exactly that direction, with Clay's disadvantage narrowing as bandwidth rises
and packet sizes grow. **Clay's rejection should be re-examined, not assumed permanent, whenever the
provider hardware profile is revised.**

## Correction 2 — Hitchhiker's benefit at our parameters (F-44, closed)

This ADR previously carried "25–45% BW reduction" from Paper 19's Figure 8. The primary source
(Paper 64) is now read, and its appendix gives the saving as a function of `(k, r)` rather than as a
band. Substituting:

```
sets available      = r − 1 = 39,  data units = k = 16
                    → every set is a singleton, s = 1
XOR+ tail parameter = ℓ; last ℓ units cost k + r + ℓ − 2 = 54 + ℓ  vs  2k = 32
                    → ℓ = 0 is forced; XOR+ degenerates to plain Hitchhiker-XOR

data unit repair    = k + s = 17 half-fragments  vs  2k = 32   →  46.9% saving, 17 helpers
parity unit repair  = unoptimised, full RS = 16 fragments      →   0% saving
```

**Hitchhiker optimises reconstruction of data units only.** Paper 64 states this in §3; Paper 62's
Table 1 confirms it independently, classifying Hitchhiker's reduced repair I/O for data-and-parity as
*No*. At Facebook's `(10,4)` that forfeits 4 of 14 shard positions. At `(56,16)` it forfeits **40 of
56 — 71.4% of every stripe.**

```
expected repair I/O over a uniformly chosen lost shard
   = (16/56)(8.5) + (40/56)(16.0)
   = 13.86 fragments  vs  16.00 for plain RS
   = 13.4% expected saving
```

The 25–45% band was not misreported by the survey; it was measured at high code rates and applied
here to a code where `r/n = 0.714`. **The number this ADR should have carried is 13.4%.**

Both substitutions were validated by reproducing Paper 62's independently published `(14,10)` figures
for Hitchhiker (avg 7.50, min 6.50, max 10.00) exactly.

## Correction 3 — LESS enters, and outranks Hitchhiker

Paper 62's Equation (7) at `(n=56, k=16)`, validated by first reproducing all twelve of its own
published `(14,10)` entries:

| Candidate | Expected repair I/O (fragments) | Saving | Fan-in | Balanced across shards? |
| --- | --- | --- | --- | --- |
| RS (status quo) | 16.00 | — | 16 | n/a |
| Hitchhiker | 13.86 | 13.4% | 17 / 16 | **No** — data units only |
| LESS, α = 2 | 17.34 | **−8.4%** | 17 | Yes |
| LESS, α = 3 | 14.67 | 8.3% | 18 | Yes |
| **LESS, α = 4** | **12.41** | **22.4%** | 19 | Yes |
| LESS, α = 5 | 10.69 | 33.2% | 20 | Yes |

**At `α ≥ 4`, LESS beats Hitchhiker on the exact metric this ADR selected Hitchhiker for** — because
its reduction is balanced and Hitchhiker's is not, and because `r = 40` maximises what that asymmetry
costs. The ranking does not invert at the survey's parameters, which is why it was invisible until
the primaries were read.

Two constraints on LESS that Hitchhiker does not have:

- **`α = 2` is a net loss for us.** LESS's sufficiency condition is `k > ⌈n/(α+1)⌉`; at `k=16, n=56`
  that needs `α ≥ 3`, with usable margin only from `α = 4`. Vyomanaut's code rate `k/n = 0.286` sits
  outside every configuration LESS evaluates (`k/n ≥ 0.71`).
- **Galois field.** LESS's MDS sufficient condition is `2^w ≥ nα + (n−k−1)·C(n−1,k)`; at our
  parameters `C(55,16) ≈ 2.97 × 10¹³`, so the condition demands `w ≥ 50`. It is only sufficient, but
  the paper's brute-force search for feasible coefficients covers only `n−k ≤ 4`, and at `α = 4` we
  need 224 distinct non-zero coefficients against GF(2⁸)'s 255. **GF(2¹⁶) is the realistic floor —
  a codec change, not a parameter change.**

---

## Escalation to design council — the repair-policy question sits upstream

Every candidate in this ADR optimises **single-block repair**. ADR-004 never performs one.

Lazy repair defers until a chunk drops to `s + r0 = 24` available fragments — **32 fragments missing,
repaired in a single batched event.** Both primaries state their multi-block behaviour explicitly:

- **Hitchhiker** (Paper 64, §6.6): with more than one unit missing, reconstruction is performed
  exactly as under RS — reading `k` entire units. Saving = **0%**.
- **LESS** (Paper 62, §3.4): multi-block repair is optimised only when all failed blocks share one
  block group; otherwise it falls back to conventional repair. At `r0 = 8` the 32 losses are spread
  across all `α+1 = 5` groups. Saving = **0%**. Repairing per-group instead costs `5 × 12.4 ≈ 62`
  fragments against RS's 16 — **3.9× worse** — because RS's conventional repair amortises one
  `k`-fragment read across every missing shard and LESS's does not.
- **Clay** is an MSR construction with the same single-failure target.

**Under ADR-004 as written, the V3 repair-bandwidth saving from any candidate in this ADR is zero.**

Two further consequences:

1. **Q19-3 does not survive contact with `r0 = 8`.** The open constraint asks whether Facebook's
   98.08% single-block failure rate holds for P2P consumer networks. The 98.08% figure describes how
   often a stripe is *observed* with one unit missing in a system that repairs promptly. It is a
   property of eager repair, not of the failure process. With `r0 = 8`, single-fragment repair events
   are ~0% of repair events **by policy**, whatever the failure process does. Q19-3 is closed as
   mis-posed and replaced by Q26-2.

2. **The orthogonality claim inherited from Paper 39 does not transfer.** This ADR states that
   Hitchhiker combined with ADR-004's lazy repair "should achieve additive savings," citing
   Silberstein's Xorbas+LAZY result. Xorbas is an LRC whose local groups repair small numbers of
   losses independently and therefore compose with accumulation differently from a
   single-block-optimised MDS code. Paper 64's own multi-block rule says the opposite for
   Hitchhiker. **That sentence is withdrawn** pending a check of whether Paper 39's degradation
   regime is comparable to `r0 = 8` (Q26-3).

**What the council must rule on, in order:**

| # | Question | Why it must come first |
| --- | --- | --- |
| **1** | Does Vyomanaut want a repair policy under which single-block repair ever happens? | Determines whether *any* code-family decision has value. Options: keep `r0 = 8` and abandon repair-BW optimisation entirely; add an eager path for permanent-departure-triggered losses alongside the lazy path; or lower `r0`. Giroire's 38× eager-vs-lazy saving is far larger than the 13–33% on offer, so the likely answer is *keep lazy repair and drop the V3 optimisation* — but that is a ruling, not an assumption. |
| **2** | *If and only if* (1) produces single-block repair events: which family? | On current evidence LESS at `α = 4` (22.4%, balanced, RS-compatible, GF(2¹⁶)) over Hitchhiker (13.4%, data-only, GF(2⁸)). Clay stays out on I/O. |
| **3** | Does ADR-004's trigger flow already contain an eager path? | ADR-004 §Decision step 4 gates repair on `count ≤ 24`, but its scheduler-priority paragraph refers to "jobs triggered by confirmed permanent departures (72h threshold crossed)" as a distinct class. If permanent departure triggers immediate per-chunk repair, single-block repair events *do* occur and question (2) is live. **This is a genuine specification ambiguity in ADR-004 and is flagged, not resolved here.** |

**No code-family decision is made in this revision.** The prior decision text is retained below,
struck through where it is now known to be wrong, so that anything built against it can be traced.

---

## Decision (prior — retained, superseded pending council)

> ~~**Hitchhiker codes are the V3 repair-bandwidth candidate. Clay codes are ruled out.**~~
> ~~Clay's minimum sub-packetisation level is astronomically large (α ≥ 40^16) — computationally intractable.~~
> ~~Hitchhiker's structural requirement is far lighter: two coupled sub-stripes, sequential I/O preserved, a 25–45% repair-bandwidth reduction.~~
> ~~Hitchhiker combined with ADR-004's lazy repair should achieve additive savings.~~

**What survives from the prior decision:**

- Clay is ruled out. (Reason changed: I/O seeks at 164-byte sub-chunks, not tractability.)
- LRC is ruled out. (Unchanged: topology.)
- V3 chunk layout must accommodate *some* sub-packetisation, and the pointer file schema (ADR-020,
  ADR-022) must be versioned to carry `α`. This is true for LESS and Hitchhiker alike and is the
  forward-compatibility requirement that motivated this ADR. **It remains actionable now.**

**What is withdrawn:**

- The choice of Hitchhiker as *the* V3 candidate.
- The 25–45% figure.
- The additive-savings claim.
- The `α ≥ 40^16` figure.

**Provisional recommendation to council, not a decision:** if single-block repair events exist at
all, LESS at `α = 4`. If they do not, close this ADR as *no V3 repair-BW optimisation* and reinvest
the design budget in Domain P (confidentiality-preserving repair, F-69), where the repair path has an
unresolved correctness problem rather than an efficiency one.

## Consequences

**Structural forward-compatibility (unchanged, still required):**

- The pointer file schema (ADR-020, ADR-022) must be versioned to carry a sub-packetisation parameter
  `α`, even if `α = 1` for all of V2.
- The provider daemon's chunk store must not assume a fragment is an opaque indivisible blob at the
  API boundary, so that sub-chunk addressing remains possible without a wire-format break.

**If LESS is adopted (V3):**

- 22.4% repair-bandwidth reduction at `α = 4`, applying uniformly to all 56 shard positions.
- Fragment splits into 4 sub-blocks of 64 KB. Repair fan-in rises 16 → 19 — an ADR-021 reachability
  cost, not just a bandwidth one.
- **Codec change to GF(2¹⁶)** and a one-time brute-force search for MDS-feasible coding coefficients
  at `n−k = 40`, which no published work has performed.
- Encode throughput falls ~43% (Paper 62's `(14,10)` measurement) — feeds ADR-009 and F-27.
- Audit challenge may address a fragment or a sub-block; must be settled *together with* the Domain A
  audit redesign (F-32), not before it.

**If Hitchhiker is adopted (V3):**

- 46.9% reduction on the 16 data shards, 0% on the 40 parity shards; 13.4% expected.
- Two sub-stripes; contiguity is trivially satisfied at `lf = 256 KB` (the fragment is smaller than
  Paper 64's read buffer, so hop-and-couple contributes nothing and should not be claimed as a
  benefit).
- Specific parity positions become mandatory helpers for specific data positions — an interaction
  with ADR-005 assignment and ADR-014's ASN cap that has not been analysed.

**If neither is adopted (V3+):**

- RS remains the codec; no structural change required. **On present evidence this is the most likely
  outcome**, because `r0 = 8` removes the operation both codes optimise.

**Open constraints:**

- ~~Q19-2: Clay's sub-packetisation at (n=56, k=16)?~~ Resolved twice: `α = 1,600`. Clay ruled out on I/O.
- ~~Q19-3: Does the >98% single-chunk failure rate hold for P2P consumer networks?~~ **Closed as
  mis-posed** — the statistic is a property of eager repair, not of the failure process. Replaced by Q26-2.
- Q26-1: At what provider count `N` does a repair-bandwidth reduction become economically necessary
  rather than optional, once Paper 36's 22× burst-variance multiplier is accounted for? Still open;
  now subordinate to Q26-2.
- **Q26-2 (new):** Does Vyomanaut want a repair policy under which single-block repair events occur?
  Blocks the entire code-family question. → design council.
- **Q26-3 (new):** Is Paper 39's (Silberstein) lazy-recovery degradation regime comparable to
  `r0 = 8`, or was the Xorbas+LAZY additivity result obtained at a much shallower degradation depth?
  Blocks reinstating the orthogonality claim.
- **Q26-4 (new):** Does ADR-004's permanent-departure trigger produce immediate per-chunk repair
  (single-block) or does everything gate on `count ≤ 24`? Specification ambiguity in ADR-004,
  flagged not resolved. → ADR-004.

## References

- [Paper 62 — LESS](../research/paper-62-less-io-efficient-repair.md): third candidate family, balanced across data and parity; 22.4% at `α=4` for `(56,16)`; second independent confirmation of Clay `α = 1,600`; multi-block fallback rule that zeroes the saving at `r0 = 8`
- [Paper 64 — Hitchhiker (Rashmi et al.)](../research/paper-64-hitchhiker-piggybacking.md): the primary source F-44 required; general-`(k,r)` appendix formulas giving 46.9% on data units and 0% on parity at our parameters; explicit multi-block fallback to RS; explicit rejection of accumulate-then-repair
- [Paper 55 — Dimakis et al.](../research/paper-55-dimakis-network-coding-distributed-storage.md): the cut-set bound both families are measured against
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): BWavg and Qpeek formulas; the 38× eager-vs-lazy saving that dominates every figure in this ADR
- [Paper 19 — EC Survey](../research/paper-19-ec-survey.md): Figure 8, the single source this ADR previously rested on; superseded for both candidate families by Papers 62 and 64
- [Paper 22 — Goparaju et al.](../research/paper-22-goparaju-msr-codes.md): sub-packetisation formula. **The `α ≥ 40^16` substitution previously attributed here was a substitution error, not a source error.**
- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): BWavg is a mean; real σ is 22× higher; burst demand is the binding constraint
- [Paper 39 — Silberstein et al.](../research/paper-39-silberstein-lazy-recovery.md): the additivity claim borrowed from here is **withdrawn pending Q26-3**
- [ADR-003](ADR-003-erasure-coding.md): RS parameters — unchanged by this ADR
- [ADR-004](ADR-004-repair-protocol.md): lazy repair at `r0=8` — **now known to be upstream of this ADR, not orthogonal to it**
- [ADR-014](ADR-014-adversarial-defences.md): 20% ASN cap — must hold under any new code construction
