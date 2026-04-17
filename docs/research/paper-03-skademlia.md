# Paper 03 — S/Kademlia: A Practicable Approach Towards Secure Key-Based Routing

**Authors:** Ingmar Baumgart and Sebastian Mies, Institute of Telematics, Universität Karlsruhe, Germany
**Topics:** #1

---

## Problem Solved

Kademlia by itself is not secure. This paper shows how to avoid Eclipse and Sybil attacks by adversarial nodes. The core requirement: attackers must not be free to choose an ID, and must not be able to easily generate multiple node IDs.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Public-key-hashed node IDs | Instant random ID generation | Node ID generation is no longer instantaneous |
| Supervised signatures | Pure decentralisation | Compromises full decentralisation |
| Sibling list (s=20, c=2.5) | No sibling list | Adds network overhead; failure probability ~5×10⁻⁷ |
| Disjoint paths (d=4,8) | Single lookup path | Network traffic multiplied by d; guarantees survival against adversarial nodes |

---

## Breaks in Our Case

- **Cryptographic node ID generation assumed** ≠ **our registration-gated model**
  → Creating public-key-hashed node IDs may not be necessary if registration (KYC/phone number) is proven effective as the certificate authority. This is the supervised signature approach from the paper — pure decentralisation is not achievable with this model.

---

## Decisions Influenced

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Node IDs derived from a public key make it computationally expensive for an attacker to flood the network. Alternative: if registration gating is proven effective, cryptographic IDs are not needed.

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Maintain disjoint paths d=4,8 and replication parameter k=8,16 (i.e. 2×d). System maintains high efficiency (99%) even at 30% share of adversarial nodes.

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Every peer maintains a sibling list with s=20, c=2.5 — gives failure probability of ~5×10⁻⁷. Guarantees survival of each replica. Makes k-bucket splitting algorithm unnecessary.

- **[ADR-012](../decisions/ADR-012-p2p-transfer-protocol.md) [#12 P2P Transfer]:** Nodes communicate via gRPC or QUIC (self-decision based on this paper's recommendations).

---

## Disagreements

None noted yet.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q03-1 through Q03-3.
