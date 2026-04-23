# ADR-005 — Storj 4-Subsystem Reputation Pipeline for Peer Selection

**Status:** Accepted
**Topic:** #5 Peer Selection Algorithm
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 01, 05, 07, 08, 20, 21, 40

---

## Context

New providers must be admitted, monitored, and ranked without opening the network to Sybil attacks or concentrating data in unreliable nodes. The initial assignment of chunks to providers must be random enough to avoid correlated failures while still filtering for reliability.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Pure random assignment | Simple; no bias | Assigns data to untested nodes; no quality signal |
| EigenTrust global reputation ([paper-24](../research/paper-24-eigentrust.md))| Distributed; converges to honest consensus | Requires gossip infrastructure; bootstrapping problem for new nodes |
| **Storj 4-subsystem pipeline** | Combines identity gating + vetting + filtering + preference; proven in production | Centralised scoring creates a choke point; vetting period creates onboarding friction |

## Decision

Adopt Storj's four-subsystem reputation pipeline:

**Subsystem 1 — Identity gate:**
Registration with KYC/phone number. Limits Sybil node flood without cryptographic cost. Acts as the admission control that S/Kademlia's cryptographic node IDs would otherwise provide.

**Subsystem 2 — Vetting:**
New nodes receive non-critical chunks under high erasure redundancy. The system monitors their audit pass rate and uptime over a 4–6 month period. During vetting, the data on these nodes has additional redundancy to compensate for unknown reliability.

Vetting period quantification: a storage node's estimated audit success probability exceeds 99% after 80 consecutive successful audits ([Paper 05](../research/paper-05-storj.md), Section 7.2 — Jeffrey's prior β(0.5,0.5)). The 4–6 month vetting period is calibrated to accumulate at least 80 clean audit events at the 24-hour polling interval.

**Subsystem 3 — Filtering:**
Nodes failing audits or retrieval challenges are downgraded in the scoring system, not immediately ejected. Ejection requires sustained failure to prevent false positives from transient outages.

**Subsystem 4 — Preference:**
After vetting, nodes are ranked by measured throughput and latency from audit responses. High-ranked nodes receive more new chunk assignments.

Concentration prevention: the Preference subsystem uses Power of Two Choices — two candidates are drawn at random from the vetted pool and the higher-scored one receives the assignment. Any vetted provider has non-zero selection probability. This prevents runaway Matthew-effect concentration where a single provider accumulates all new chunk assignments. ([Paper 05](../research/paper-05-storj.md), Section 4.15)

**Initial peer selection (before score history exists):**
Based on BitTorrent's optimistic unchoking principle — random assignment from the vetted pool. Independence of randomly selected hosts from a filtered pool is confirmed (Bhagwan, [Paper 08](../research/paper-08-bhagwan-availability.md)).

**Correlated failure prevention (Honest Geppetto):**
No cluster sharing the same ASN or subnet may hold more than 20% of shards for a single file. This is a placement constraint enforced at write time. Implement only after 5 × n shards exist in the network.

## Consequences

**Positive:**
- Registration gating prevents Sybil attacks without hardware overhead
- 4–6 month vetting period naturally selects for stable desktop providers (MTTF >180 days)
- Random initial assignment gives near-independent failure probabilities

**Negative / trade-offs:**
- Vetting period creates onboarding friction — providers must wait before receiving full assignments
- Scoring is centralised in the microservice — becomes a hardened dependency
- ASN diversity cap is a placement constraint that adds complexity to the assignment service

**Open constraints:**
- This ADR assumes desktop-only V2 (see [ADR-010](ADR-010-desktop-only-v2.md))
- Geographic proximity as an additional selection criterion is deferred pending Coral DSHT research (Q01-4)
- Kademlia fragility lesson: Storj v3.1 removed Kademlia from provider discovery after finding operational fragility without a fallback. Vyomanaut retains Kademlia for chunk-address lookup only (not provider discovery, which goes through the microservice). This distinction must be preserved — Kademlia is not a provider directory. ([paper-05](../research/paper-05-storj.md), Section 4.6 v3.1 changelog)

## References

- [Paper 01 — BitTorrent](../research/paper-01-bittorrent.md): random peer set robustness; optimistic unchoking
- [Paper 05 — Storj](../research/paper-05-storj.md): four-subsystem pipeline sourced verbatim; Power of Two Choices for concentration prevention; 80-audit Jeffrey's prior threshold informing vetting period length; v3.1 Kademlia removal lesson
- [Paper 07 — SoK DSN](../research/paper-07-sok-dsn.md): four-subsystem pipeline confirmed as the industry standard; Honest Geppetto attack is the adversarial driver for the correlated-failure placement constraint in the Preference subsystem
- [Paper 08 — Bhagwan et al.](../research/paper-08-bhagwan-availability.md): independence of randomly selected hosts confirmed; two-component churn model motivates vetting subsystem
- [Paper 20 — IPFS Measurement](../research/paper-20-trautwein-ipfs.md): confirms < 2.3% of P2P nodes are cloud-hosted; desktop-first provider model is operationally viable at 300 k+ node scale
- [Paper 21 — Saroiu et al.](../research/paper-21-saroiu-p2p-measurement.md): 30% of peers misreport bandwidth; confirms vetting must measure directly, not trust self-reports
- [Paper 40 — Buragohain et al.](../research/paper-40-buragohain-p2p-incentives.md): low-contribution peers pull equilibrium downward; Preference subsystem (weighted assignment) is game-theoretically necessary to maintain the stable high-contribution equilibrium
