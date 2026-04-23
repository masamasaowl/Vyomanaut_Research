# ADR-008 — Three-Window Rolling Reliability Score

**Status:** Accepted
**Topic:** #8 Reliability Scoring Model
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 01, 08, 11, 24, 31, 32, 37, 40

---

## Context

The system needs a reliability score per provider that drives chunk assignment, payment, and exit decisions. The score must weight recent behaviour heavily (to detect deterioration quickly) without discarding long-term history (to avoid penalising providers for isolated incidents).

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| EigenTrust ([Paper-24](../research/paper-24-eigentrust.md)) global score | Distributed; resistant to collusion | Requires gossip infrastructure; bootstrapping problem |
| Single rolling window | Simple | Either too reactive (short) or too slow to respond (long) |
| **Three rolling windows (24 h / 7 d / 30 d)** | Captures short, medium, and long-term patterns | Slightly more complex to maintain |

## Decision

Maintain three rolling audit windows per provider:

| Window | Duration | Weight | Captures |
|---|---|---|---|
| Short | 24 h | Highest | Recent behaviour; quick deterioration detection |
| Medium | 7 d | Medium | Weekly patterns (weekend behaviour) |
| Long | 30 d | Lowest | Long-term reliability history |

Final score = weighted combination of the three windows. Exact weights are a product decision and should be tuned empirically after launch.

**Bootstrapping new providers:**
New providers have no history. Initial score is set by the optimistic random assignment principle (BitTorrent [Paper-01](../research/paper-01-bittorrent.md)): assign a random non-critical chunk and observe the first audit response. Score grows from this baseline.

**Score floor enforcement:**
Score decrement on audit fail is a non-I-confluent operation (Bailis [Paper 11](../research/paper-11-bailis-coordination.md)) — it requires a single authoritative scorer to enforce the floor. This is handled by the microservice, not distributed.

**Score and assignment:**
After vetting, nodes are ranked by their weighted score. High-scored nodes receive more new chunk assignments (Preference subsystem — [ADR-005](ADR-005-peer-selection.md)). Runaway concentration (Matthew effect) is mitigated by the 20% shard cap per ASN/subnet ([ADR-014](ADR-014-adversarial-defences.md)).

## Consequences

**Positive:**
- Three windows give a nuanced signal that neither overreacts to single failures nor ignores gradual degradation
- Aligns with Bhagwan's finding ([paper-08](../research/paper-08-bhagwan-availability.md)) that availability is time-varying and must be modelled continuously
 - The centralised microservice scorer is not a compromise. [Paper 24](../research/paper-24-eigentrust.md) (EigenTrust) proves that distributed reputation requires pairwise interaction data between evaluating nodes — data that does not exist when the sole auditor is the microservice. Centralisation is architecturally necessary, not merely convenient.

**Negative / trade-offs:**
- Score decrement requires coordination (single scorer) — this is a known non-I-confluent operation
- Window weights must be tuned empirically; wrong weights create perverse incentives

**Open constraints:**
- Adaptive polling interval based on score history (Q08-4) is a V3 enhancement
- EigenTrust distribution of correlation signals (Q07-4) is deferred to Phase 2B research
- Failure clustering signal: [Paper 32](../research/paper-32-schroeder-flash-reliability.md) (Schroeder et al.) shows a provider with a recent uncorrectable error has 30× elevated probability of further failures. A provider showing > 1 audit FAIL within a rolling 7-day window should be treated as in an elevated-failure regime — triggering an accelerated re-audit of all their chunks — not merely logged as a score decrement. The current three-window model does not implement this. Add as a V2 implementation task.

## References

- [Paper 08 — Bhagwan](../research/paper-08-bhagwan-availability.md): time-varying availability; weight recent behaviour
- [Paper 01 — BitTorrent](../research/paper-01-bittorrent.md): optimistic unchoking as bootstrapping mechanism
- [Paper 11 — Bailis](../research/paper-11-bailis-coordination.md): score decrement is non-I-confluent; requires single scorer
- [Paper 24 — EigenTrust](../research/paper-24-eigentrust.md): distributed scoring requires peer-to-peer interaction history that does not exist in Vyomanaut's audit architecture; confirms centralised scorer is the only viable design
- [Paper 31 — PeerTrust](../research/paper-31-peertrust.md): adaptive dual-window scoring — lower of full-window vs recent-window; structural basis for the partial hold trigger used in ADR-024; confirms centralised auditor-only model
- [Paper 32 — Schroeder et al.](../research/paper-32-schroeder-flash-reliability.md): prior errors are 30× predictive of future uncorrectable errors; failure-clustering signal not yet in ADR-008 scoring model
 [Paper 37 — SHELBY](../research/paper-37-shelby-incentive-compatibility.md): Proposition 1 formally confirms centralised auditor is necessary; peer self-reports collapse to universal dishonesty
 - [Paper 40 — Buragohain et al.](../research/paper-40-buragohain-p2p-incentives.md): low-contribution peers pull system equilibrium downward; scorer-driven assignment reduction is the correct enforcement mechanism
