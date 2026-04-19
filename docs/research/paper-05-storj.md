# Paper 05 — Storj: A Decentralised Cloud Storage Network Framework

**Authors:** Storj Labs, Inc.
**Venue / Year:** v3.0 Oct 2018, v3.1 Mar 2024
**Topics:** #1, #2, #3, #4, #5

---

## Problem Solved

*(Listed at start of each topic in the detailed notes of the paper)*

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Central coordination server (Satellite) | Kademlia DHT | Dropped decentralisation for operational reliability — see Appendix A |
| Reed-Solomon over simple replication | Replication | Optimised redundancy at the cost of repair complexity |
| Explicit node selection | Dynamo-style assignment | Geographic, performance, and reputation filtering becomes possible |
| Stripe-level erasure coding | Segment-level | Disputed — see Szabó et al. (reading-list Phase 1 #10) |
| Random stripe audits (Berlekamp-Welch) | Pre-generated Merkle proofs | Avoids audit false positive risk |
| No payments for first 6 months | Immediate payment | Incentivises long-lived nodes during vetting period |

---

## Breaks in Our Case

- **Assumes MTTF of 6–12 months (NAS operators)** ≠ **inclusion of mobile operators would bring MTTF to ~1 month**
  → Changes erasure parameters and increases repair bandwidth. (Note: mobile deferred to V3 by [ADR-011](../decisions/ADR-011-desktop-only.md), but the parameter sensitivity must be understood)

- **Targets desktop users with symmetric bandwidth** ≠ **mobile operators have 1–5 Mbps upload**
  → Piece size needs to be reduced for mobile (V3 concern)

- **Depends on Satellite for repairs** ≠ **our need to be robust during mobile user volatility**
  → Repair must be designed to operate without a central dependency

- **6-month waiting period for payment** ≠ **average user expectation**
  → Taxing for an average provider; must be redesigned

- **No background execution — dedicated daemon on desktop** ≠ **our requirement for background-capable operation**
  → Background execution strategy required — addressed by [ADR-009](../decisions/ADR-009-background-execution.md)

---

## Decisions Influenced

- **[ADR-0015](../decisions/ADR-0015-audit-trail.md) [#2 Proof of Storage]:** Repair bandwidth consumption is the biggest challenge of the entire project

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]:** Use erasure coding instead of simple replication; look for more optimised erasure algorithms. Use four erasure parameters (k, m, o, n) for long-tail transfers. Starting values: k=29, n=80, m=35, o=52 — but must be optimised for our MTTF, as much higher n may be needed.

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]:** Use Berlekamp-Welch error detection instead of Merkle challenges for audits

- **[ADR-009](../decisions/ADR-009-reliability-scoring.md) [#8 Reliability Scoring]:** For the vetting period, use Bayesian audit scoring with Jeffery prior β(0.5, 0.5) → 80 audits give 99% trust in provider

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]:** A pointer must be created and maintained at the data owner end storing: node IDs, erasure parameters, piece IDs, encryption info, repair threshold, piece hashes, and a signature

- **[ADR-011](../decisions/ADR-011-escrow-payments.md) [#13 Escrow]:** Adopt Neuman's proxy protocol, which reads used bandwidth agnostically of payments; incremental accounting prevents fraud

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Utilise bloom filters for garbage collection to check if the marked-out segment has been deleted

---

## Disagreements (on Storj's own weaknesses)

- **"Design & Evaluation of IPFS" (Trautwein, 2022):** DHT is the optimised method for node discovery — contradicts Storj's decision to drop Kademlia

- **"Handling Churn in a DHT" (Rhea et al., 2004):** Shows MTTF of a few hours or days — challenges Storj's 6–12 month MTTF assumption

- **"High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two" (Blake & Rodrigues, 2003):** Storj chooses availability and scalability, accepting high repair bandwidth; we can disagree with this trade-off

- **Satellite coordination** dropped the trustlessness of the network from data owners' perspective

- **Repair consumed significant bandwidth** → Walrus and EC survey tackled this with regenerating codes and 2D coding schemes

- **Provider payments are volatile with the Storj token** — providers earn for storing, not for continuous availability

- **Encrypt-then-erasure opposed** by: Szabó et al., *Separation and Optimisation of Encryption and Erasure Coding in Decentralized Storage* (reading-list Phase 1 #10)

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q05-1 through Q05-2.
