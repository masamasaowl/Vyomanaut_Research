# Paper 07 — SoK: Decentralized Storage Network

**Authors:** Chuanlei Li, Minghui Xu, Jiahao Zhang, Hechuan Guo, Xiuzhen Cheng, Shandong University
**Venue / Year:** IEEE S&P
**Topics:** #1, #2, #5, #8, #13, #19

---

## Problem Solved

1. No single paper before it summarised the entire DSN landscape
2. Provides classification of major systems: Sia, Storj, Filecoin, Swarm
3. Names the failures each design choice creates
4. Points out how data updation is very complex in DSNs
5. Defines Proof of Storage and consensus mechanisms
6. Identifies the biggest unsolved challenges:
   - File version control
   - Network hijacking giving DoS (central entity risk)
   - Privacy leaks through DHTs
   - Illegal content cannot be traced in the network
   - No access control — data owner cannot share selective access
   - Honest Geppetto attack
   - Bandwidth optimisations

---

## Trade-offs (from surveyed systems)

| System | Chosen | Over | Consequence |
|---|---|---|---|
| Filecoin | PoRep + PoSt | Lightweight Merkle challenges | 256 GB RAM + GPU with ≥11 GB VRAM required — eliminates mobile and desktop users; anti-Sybil only for NAS operators |
| Storj | Reputation-gated vetting | Cryptographic identity proofs | Faster onboarding, lower hardware requirements, but introduces Honest Geppetto attack |
| Sia | Frequent Merkle challenges | Berlekamp-Welch PoS | Lightweight for mobile but bypassable if challenge timing is predictable |
| Swarm | Bandwidth-as-currency (SWAP) | Escrow-based payment | Clean for symmetric peer set, breaks for data owner & provider structure, failed in production |
| All DSNs | Cryptocurrency for payments | Fiat | — |

---

## Breaks in Our Case

- **Every surveyed DSN uses blockchain/cryptocurrency as trust anchor** ≠ **we use hardened microservices**
  → We must create an audit trail that replaces blockchain's three functions: immutable log, automatic payment trigger, public dispute resolution

- **Filecoin's heavy crypto challenges prevent Sybil attacks** ≠ **we use registration gating**
  → Weaker but sufficient if registration is enforced effectively

- **Satellite architecture of Storj is centralised and prone to DoS** ≠ **our availability requirement**
  → Hardened microservice with quorum read mechanism for consistency (introduces latency)

- **DHT lookups expose Content IDs** ≠ **our zero-knowledge requirement**
  → Content addressing must not reveal file identity during DHT lookups via Kademlia

---

## Decisions Influenced

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Reconsider the coordination microservice as trust anchor vs a public ledger. Blockchain implements automatic verifiability. We would have to create an audit trail. A hardened service prevents compromise.

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** To prevent DoS in central microservice, implement quorum read mechanism for consistency — introduces latency.

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage — DECIDED]:**
  - Continuous Proof of Storage: PoR (Merkle challenges) — PoRep and PoSt are computationally too heavy
  - Transitory PoS: at upload time, node signs a receipt of stored chunks

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection — DECIDED]:** Storj's four-subsystem reputation pipeline adopted:
  1. **Identity gate:** registration with KYC/phone number limits Sybil node flood without cryptographic cost
  2. **Vetting:** new nodes receive non-critical chunks under high erasure redundancy to monitor and build trust
  3. **Filtering:** nodes failing audits or retrieval challenges are downgraded, not immediately ejected
  4. **Preference:** after vetting, rank surviving nodes by throughput and latency measured during audits; high-ranked nodes receive more new assignments

- **[ADR-019](../decisions/ADR-019-adversarial-defences.md) [#19 Adversarial Behaviour — DECIDED]:** Four defences adopted:
  1. **Honest Geppetto attack:** a single correlated provider group (same subnet, same ISP AS number) must not hold multiple shards of the same file
  2. **Outsourcing attack:** use Filecoin's Seal idea
  3. **Just-in-time retrieval:** unpredictable challenge times; response deadline = (chunk_size / declared_upload_speed) × 1.5

- **[ADR-004](../decisions/ADR-004-audit-trail.md) [Audit Trail]:** Blockchain does three jobs we must replicate:
  1. Immutable audit log — provider submitted proof X at time T
  2. Automatic payment trigger — proof verified → escrow released
  3. Public dispute resolution — anyone can inspect the proof chain
  Solution: maintain a write-once audit log; all receipts are verifiable. Gives an auditable paper trail without blockchain.

---

## Disagreements

- **Lakhani et al. (2022 & 2023):** Do not model payment based on bandwidth exchange — it failed for Swarm.
  *Implication for us:* Confirmed by ADR-014 (payment per audit passed, not per GB transferred)

- **"Understanding Availability" (Bhagwan, Savage, Voelker, IPTPS 2003):** Individual MTTF < nodes going offline together — correlated failures must be modelled separately.

---

## Mitigation designs (from engineering questions)

**Honest Geppetto mitigation (Storj approach):**
1. Analyse correlations between shards
2. If cluster C holds shards {i, j, k} and `|{i,j,k}| / n > ceiling (e.g. 20%)`, new assignments pause until major peer churn happens
3. Placement constraint applied at write time
4. Implement only after 5 × n shards exist in the network (new networks have too few shards)

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q07-1 through Q07-3.
