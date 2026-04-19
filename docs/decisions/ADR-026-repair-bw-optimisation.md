# ADR-026 — V3 Repair BW Optimisation: Clay (MSR) and Hitchhiker Candidates

**Status:** Proposed — blocked on Dimakis et al. (Phase 2A #4)
**Topic:** #17 Repair Bandwidth Optimisation
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 19 (EC Survey, Shen et al., ACM ToS 2025), Paper 10 (Giroire)

---

## Context

ADR-003 chose Reed-Solomon RS(s=16, r=40) with lazy repair triggered at r0=8. The Giroire
formulas (Paper 10) show that reconstructing a lost chunk requires contacting k=16 surviving
fragment holders — this is not bandwidth-optimal. For a single-chunk failure, RS must download
k entire chunks to reconstruct one. Information theory guarantees that this can be done with
less data (the regenerating code bound, Dimakis et al. 2010).

At V2 launch scale (hundreds of providers, desktop MTTF 180–380 days), the repair bandwidth
BWavg ≈ 39 Kbps/peer is well within the 100 Kbps background budget. Repair BW optimisation is
therefore a V3 concern, not a V2 blocker. However, the design space must be understood now so
that V3 provider daemon and chunk layout decisions do not foreclose the upgrade path.

Paper 19 (EC Survey) provides the landscape. It identifies two candidate code families:
regenerating codes (specifically Clay codes as the general-parameter MSR implementation) and
piggybacking codes (Hitchhiker), with quantified tradeoffs from Figure 8.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| RS codes only (V2 status quo) | Universally deployed; simple; no sub-packetization | Repair bandwidth is not optimal; must download k full chunks per failure |
| **Clay codes (MSR, general n,k)** | Up to 50% repair BW reduction; MDS (same storage overhead); access-optimal (minimises both BW and I/O); supports our (n=56, k=16) parameters | High sub-packetization: each chunk split into many sub-chunks; non-sequential disk I/O on desktop HDDs; implemented in Ceph/Hadoop but not consumer P2P |
| **Hitchhiker codes (piggybacking)** | 25–45% BW reduction; MDS; limited sub-packetization (two sub-stripes only; sequential I/O preserved); simpler implementation than Clay | Partial BW reduction; not at the Pareto frontier; requires coupling two sub-stripes per encoding |
| LRC (Azure-style) | Local repair reduces I/Os by factor l | Non-MDS (extra storage overhead); local group co-locality required — cannot be guaranteed in consumer P2P; not applicable to V2/V3 |

## Known Constraints (from ADR-003 and Paper 10)

**Current parameters:**

| Parameter | Value |
|---|---|
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

## Blocked On

Dimakis et al. (2010) — reading-list Phase 2A #4. The paper proves the information-theoretic
lower bound on repair bandwidth for single-chunk failures and is the theoretical foundation for
both MSR codes and the Clay code construction. Without it, the quantitative comparison between
Clay and Hitchhiker at our specific (n=56, k=16) parameters cannot be made.

Specific questions that must be answered before this ADR can be accepted:
- What is the sub-packetisation parameter β for Clay codes at (n=56, k=16, d=55)? Does
  this produce unacceptably small sub-chunks for 256 KB fragments on desktop HDDs?
- Does Hitchhiker's 25–45% BW reduction justify its implementation complexity for the gain
  relative to current BWavg (~39 Kbps/peer)?
- At what provider count does BWavg become a meaningful fraction of the background upload
  budget, making the BW reduction economically necessary?

## Decision

**Not yet made.** Candidates are Clay codes and Hitchhiker codes as identified by Paper 19.
Fill this ADR after Phase 2A #4 (Dimakis) is read.

**Update** Clay codes removed as candidate, Hitchhiker retained

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
- Q19-2: What is Clay codes' sub-packetisation level at (n=56, k=16)? Blocked on Dimakis.
- Q19-3: Does the >98% single-chunk failure rate (Facebook warehouse) hold for P2P consumer networks? If correlated failures produce multi-chunk events at meaningful rates, the MSR single-failure optimisation is less well-targeted.

## References

- [Paper 19 — EC Survey](../research/paper-19-ec-survey.md): RS dominance; Clay codes general (n,k); Hitchhiker 25–45% reduction; LRC topology dependency; Figure 8 tradeoff analysis
- [Paper 10 — Giroire](../research/paper-10-giroire-lazy.md): Qpeek formula; BWavg calculation; r0=8 parameter derivation
- [ADR-003](ADR-003-erasure-coding.md): RS parameters (s=16, r=40, r0=8, lf=256 KB) — unchanged by this ADR
- [ADR-004](ADR-004-repair-protocol.md): Lazy repair, 72 h trigger — unchanged by this ADR
- [ADR-014](ADR-014-adversarial-defences.md): 20% ASN cap — must hold under any new code construction
