# ADR-005 — Storj 4-Subsystem Reputation Pipeline for Peer Selection

**Status:** Accepted
**Topic:** #5 Peer Selection Algorithm
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 01, 07

---

## Context

New providers must be admitted, monitored, and ranked without opening the network to Sybil attacks or concentrating data in unreliable nodes. The initial assignment of chunks to providers must be random enough to avoid correlated failures while still filtering for reliability.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Pure random assignment | Simple; no bias | Assigns data to untested nodes; no quality signal |
| EigenTrust global reputation | Distributed; converges to honest consensus | Requires gossip infrastructure; bootstrapping problem for new nodes |
| **Storj 4-subsystem pipeline** | Combines identity gating + vetting + filtering + preference; proven in production | Centralised scoring creates a choke point; vetting period creates onboarding friction |

## Decision

Adopt Storj's four-subsystem reputation pipeline:

**Subsystem 1 — Identity gate:**
Registration with KYC/phone number. Limits Sybil node flood without cryptographic cost. Acts as the admission control that S/Kademlia's cryptographic node IDs would otherwise provide.

**Subsystem 2 — Vetting:**
New nodes receive non-critical chunks under high erasure redundancy. The system monitors their audit pass rate and uptime over a 4–6 month period. During vetting, the data on these nodes has additional redundancy to compensate for unknown reliability.

**Subsystem 3 — Filtering:**
Nodes failing audits or retrieval challenges are downgraded in the scoring system, not immediately ejected. Ejection requires sustained failure to prevent false positives from transient outages.

**Subsystem 4 — Preference:**
After vetting, nodes are ranked by measured throughput and latency from audit responses. High-ranked nodes receive more new chunk assignments.

**Initial peer selection (before score history exists):**
Based on BitTorrent's optimistic unchoking principle — random assignment from the vetted pool. Independence of randomly selected hosts from a filtered pool is confirmed (Bhagwan, Paper 08).

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

## References

- [Paper 01 — BitTorrent](../research/paper-01-bittorrent.md): random peer set robustness; optimistic unchoking
- [Paper 07 — SoK](../research/paper-07-sok-dsn.md): Storj 4-subsystem pipeline; Honest Geppetto attack
- [Paper 08 — Bhagwan](../research/paper-08-bhagwan-availability.md): independence of randomly selected hosts confirmed
