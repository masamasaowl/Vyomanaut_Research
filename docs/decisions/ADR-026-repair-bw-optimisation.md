# ADR-026 — V3 Repair BW Optimisation: Clay (MSR) and Hitchhiker Candidates

**Status:** Accepted
**Topic:** #17 Repair Bandwidth Optimisation
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 10, 19, 22, 36, 39, 55

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

[Paper 19](../research/paper-19-ec-survey.md) (EC Survey) provides the landscape. It identifies two candidate code families:
regenerating codes (specifically Clay codes as the general-parameter MSR implementation) and
piggybacking codes (Hitchhiker), with quantified tradeoffs from Figure 8.

An additional consideration at V3 scale: [Paper 36](../research/paper-36-dalle-failure-correlation.md) (Dalle et al.) proves the real repair bandwidth standard deviation is 22× higher than the independent model predicts at any practical deployment size. The BWavg ≈ 39 Kbps/peer that makes Hitchhiker's 25–45% reduction appear marginal is a mean. At V3 scale, the burst demand during an ASN-level outage is the binding constraint, not the mean. Hitchhiker's reduction may be more economically necessary than the mean calculation suggests.

Quantifying the V3 benefit using [Paper 10's](../research/paper-10-giroire-lazy.md) formulas: Hitchhiker's 25–45% bandwidth reduction directly reduces both BWavg (Formula 1) and Qpeek (Formula 2) proportionally. At V3 scale (N=10,000), BWavg per peer decreases with N but the failure arrival rate increases. The exact crossover at which Hitchhiker becomes economically necessary can be computed by substituting the Hitchhiker-adjusted β into Formula 1 and comparing to the background bandwidth budget.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| RS codes only (V2 status quo) | Universally deployed; simple; no sub-packetization | Repair bandwidth is not optimal; must download k full chunks per failure |
| **Clay codes (MSR, general n,k)** | Up to 50% repair BW reduction; MDS (same storage overhead); access-optimal (minimises both BW and I/O); supports our (n=56, k=16) parameters | High sub-packetization: each chunk split into many sub-chunks; non-sequential disk I/O on desktop HDDs; implemented in Ceph/Hadoop but not consumer P2P |
| **Hitchhiker codes (piggybacking)** | 25–45% BW reduction; MDS; limited sub-packetization (two sub-stripes only; sequential I/O preserved); simpler implementation than Clay | Partial BW reduction; not at the Pareto frontier; requires coupling two sub-stripes per encoding |
| LRC (Azure-style) | Local repair reduces I/Os by factor l | Non-MDS (extra storage overhead); local group co-locality required — cannot be guaranteed in consumer P2P; not applicable to V2/V3 |

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
If Clay codes are adopted in V3, the chunk layout changes: each chunk is sub-packetised into
β sub-chunks. The provider daemon must store chunks in sub-chunk granularity. The pointer file
schema (ADR-020, ADR-022) must include sub-packetisation parameters. The audit receipt
(ADR-017) challenges a sub-chunk, not a full chunk. These are structural changes that cannot
be retrofitted without a network-wide migration. The decision must be made before V3 provider
daemon architecture is finalised.

**Why LRCs are ruled out now:**
LRC local repair requires that all nodes in a local group (k/l nodes) are co-located and
simultaneously reachable. In a P2P consumer network with providers behind NAT on different ISPs
across India, group co-locality cannot be guaranteed. The repair benefit collapses to RS-level
repair in the worst case. Non-MDS storage overhead is an additional penalty. LRC is rejected
for V2 and V3 absent a fundamental change in provider network topology.

## Resolution

This ADR previously listed itself as blocked on "Dimakis et al. (2010) — reading-list Phase 2A
# 4." That citation was mislabeled: reading-list Phase 2A #4 is Goparaju et al. (Paper 22), a
related but distinct paper on MSR sub-packetisation — not Dimakis. Dimakis et al. 2010 had never
actually been assigned a paper number or read directly; it's now documented as **Paper 55**.
Paper 22 was correctly used already (see References) and is what actually answered the
sub-packetisation question below — Paper 55's role is to ground the cut-set-bound citation this
ADR always rested on, not to change Paper 22's numeric conclusion.

The three questions originally gating acceptance:

- **What is the sub-packetisation parameter β for Clay codes at (n=56, k=16, d=55)?** Answered
  by Paper 22: α ≥ 40^16 — computationally intractable, and far too fine-grained for 256 KB
  fragments on desktop HDDs regardless. Clay is ruled out.
- **Does Hitchhiker's 25–45% BW reduction justify its implementation complexity relative to
  current BWavg (~39 Kbps/peer)?** Not at V2 scale — 39 Kbps is already within the 100 Kbps
  budget. See Decision below for the V3 case.
- **At what provider count does BWavg become a meaningful fraction of the background upload
  budget, making the BW reduction economically necessary?** Not yet answered precisely — this
  is an operational-scale question, not a code-family one, and is tracked as Q26-1 rather than
  blocking acceptance of the structural decision below.

## Decision

**Hitchhiker codes are the V3 repair-bandwidth candidate. Clay codes are ruled out.** Paper 55
(Dimakis et al. 2010) supplies the cut-set bound both code families are measured against;
applied to Vyomanaut's (n=56, k=16) parameters via Paper 22's sub-packetisation formula, Clay's
minimum sub-packetisation level is astronomically large (α ≥ 40^16) — computationally
intractable, and far too fine-grained for 256 KB fragments on desktop HDDs regardless.
Hitchhiker's structural requirement is far lighter: two coupled sub-stripes, sequential I/O
preserved, a 25–45% repair-bandwidth reduction. This is the structural decision V3 provider
daemon architecture needs now — chunk layout, the pointer file schema (ADR-020, ADR-022), and
audit challenge design (ADR-002) only need to accommodate a two-sub-stripe coupling, not
arbitrary sub-packetisation — even though the operational question of exactly when to switch is
not yet decided (below).

Combination with lazy repair ([ADR-004](./ADR-004-repair-protocol.md)): [Paper 39](../research/paper-39-silberstein-lazy-recovery.md) (Silberstein) confirms lazy recovery and bandwidth-efficient codes are orthogonal. Figure 7 shows Xorbas+LAZY outperforms either alone by more than 2×. Hitchhiker combined with [ADR-004's](./ADR-004-repair-protocol.md) lazy repair should achieve additive savings — the total V3 bandwidth reduction could significantly exceed Hitchhiker's standalone 25–45%.

Quantitative savings at V2 parameters: applying 25–45% bandwidth reduction to V2's BWavg ≈ 39 Kbps/peer gives an effective BWavg of 21–29 Kbps/peer. At V2 scale this saving is not economically necessary — 39 Kbps is already within the 100 Kbps budget.

**What's not yet decided:** the specific provider count N at which the saving becomes
economically necessary rather than optional, particularly once Paper 36's (Dalle et al.)
finding — real repair-bandwidth variance runs 22× higher than the mean-based estimate — is
accounted for. Burst demand during a correlated ASN-level outage may cross that threshold well
before the mean-based BWavg calculation would suggest. That crossover is an operational-scale
question, not a code-family question, and is left to a dedicated V3-scoping pass once real
provider-count growth data exists to compute it against — tracked as Q26-1, not blocking this
ADR's acceptance, since the code-family decision above is what V3 chunk-layout work actually
depends on.

## Consequences

**If Clay codes are adopted (V3):**

- Repair bandwidth reduced by up to 50% for single-chunk failures
- Sub-packetisation changes chunk storage layout at the provider daemon level
- Pointer file schema must be versioned to include sub-packetisation parameters
- Audit challenge (ADR-002) must target sub-chunks, not full chunks

**If Hitchhiker codes are adopted (V3):**

- Repair bandwidth reduced by 25–45% with minimal implementation change
- Sub-packetisation is two sub-stripes — sequential I/O preserved on desktop HDDs
- Audit challenge (ADR-002) must address coupled sub-stripe encoding

**If neither is adopted (V3+):**

- RS codes remain the codec; BWavg scales linearly with provider count
- At 10,000 providers with MTTF=300 days, BWavg remains ~39 Kbps/peer — still within budget
- No structural change required

**Open constraints:**

- ~~Q19-2: What is Clay codes' sub-packetisation level at (n=56, k=16)?~~ Resolved by Paper 55 + Paper 22: α ≥ 40^16, computationally intractable. Clay is ruled out; this constraint no longer applies.
- Q19-3: Does the >98% single-chunk failure rate (Facebook warehouse) hold for P2P consumer networks? If correlated failures produce multi-chunk events at meaningful rates, the MSR single-failure optimisation is less well-targeted. Still open — requires production failure-event data, not literature; tracked in open-questions.md.
- Q26-1 (new): At what provider count N does Hitchhiker's 25–45% repair-bandwidth reduction become economically necessary rather than optional, once Paper 36's 22× burst-variance multiplier is accounted for? Not blocking — the code-family decision above doesn't depend on pinning down N precisely, only on knowing which family to design chunk layout around.

## References

- [Paper 55 — Dimakis et al.](../research/paper-55-dimakis-network-coding-distributed-storage.md): derives the cut-set bound — the fundamental storage-vs-repair-bandwidth tradeoff curve — that both Clay (the MSR point) and Hitchhiker's comparison baseline are measured against; the theoretical foundation this ADR's application of Papers 19/22/36/39 already assumed but never directly cited until now
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): BWavg and Qpeek formulas quantify Hitchhiker's 25–45% reduction in concrete per-peer Kbps savings; Formula 1 determines the exact N at which Hitchhiker becomes economically necessary
- [Paper 19 — EC Survey](../research/paper-19-ec-survey.md): RS dominance; Clay codes general (n,k); Hitchhiker 25–45% reduction; LRC topology dependency; Figure 8 tradeoff analysis
- [Paper 22 — Goparaju et al.](../research/paper-22-goparaju-msr-codes.md): sub-packetisation formula applied to (n=56, k=16) yields α ≥ 40^16; Clay codes computationally intractable; Q19-2 definitively answered
- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): BWavg is mean only; real σ is 22× higher at any practical scale; burst demand may make Hitchhiker's 25–45% reduction more necessary at V3 than mean calculation implies
- [Paper 39 — Silberstein et al.](../research/paper-39-silberstein-lazy-recovery.md): lazy recovery and bandwidth-efficient codes are orthogonal; combining Hitchhiker with ADR-004 lazy repair yields additive savings greater than either alone
- [ADR-003](ADR-003-erasure-coding.md): RS parameters (s=16, r=40, r0=8, lf=256 KB) — unchanged by this ADR
- [ADR-004](ADR-004-repair-protocol.md): Lazy repair, 72 h trigger — unchanged by this ADR
- [ADR-014](ADR-014-adversarial-defences.md): 20% ASN cap — must hold under any new code construction
