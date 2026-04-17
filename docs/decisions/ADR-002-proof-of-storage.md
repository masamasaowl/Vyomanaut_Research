# ADR-002 — Use PoR Merkle Challenge for Proof of Storage

**Status:** Accepted
**Topic:** #2 Proof of Storage
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 07

---

## Context

The system must periodically verify that providers are actually storing the chunks they have been assigned. Three mechanisms were evaluated: PoRep+PoSt (Filecoin), Merkle challenges (Sia/PoR), and Berlekamp-Welch random stripe audits (Storj).

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| PoRep + PoSt (Filecoin) | Cryptographically robust; anti-Sybil | 256 GB RAM + GPU ≥11 GB VRAM required — eliminates all desktop users |
| Berlekamp-Welch random stripe audits | Avoids audit false positives | Stripe-level, not chunk-level; disputed by Szabó et al. |
| **PoR Merkle challenge** | Lightweight; works on consumer hardware; computationally inexpensive | Can be bypassed if challenge timing is predictable (mitigated by ADR-014) |

## Decision

Two-phase proof of storage:

1. **Transitory PoS** — at upload time, the provider signs a receipt confirming the chunks are stored. This is immediate and synchronous.
2. **Continuous PoR** — random Merkle challenges issued on an ongoing schedule. Challenge timing is randomised to prevent just-in-time retrieval attacks. Response deadline enforced: `(chunk_size / declared_upload_speed) × 1.5`.

Both phases produce signed receipts recorded in the audit trail (see ADR-015).

## Consequences

**Positive:**
- Works on all target hardware (desktop, NAS)
- Computationally inexpensive — audit overhead stays within the 5% CPU budget

**Negative / trade-offs:**
- Weaker than PoRep — does not prove data was physically encoded on the device
- Requires randomised challenge timing to prevent bypass (operational complexity)

**Open constraints:**
- Transparent Merkle Log (ADR-015) must be operational for proofs to be independently verifiable
