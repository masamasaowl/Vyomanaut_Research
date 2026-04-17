# ADR-019 — Client-Side Encryption Scheme

**Status:** Proposed — blocked on Szabó et al. 2025
**Topic:** #9 Client-Side Encryption
**Supersedes:** —
**Superseded by:** —
**Research source:** TBD — reading-list Phase 1 #10

---

## Context

All data must be encrypted on the client before leaving the data owner's device. The service never sees plaintext. Two decisions are entangled and must be resolved together: the encryption scheme itself, and the order of encryption and erasure coding (which is tracked separately in [ADR-022](ADR-022-encryption-erasure-order.md)).

## Blocked on

**Szabó et al., "Separation and Optimisation of Encryption and Erasure Coding in Decentralized Storage," FGCS Feb 2025** (reading-list Phase 1 #10).

This paper is MANDATORY before this ADR can be accepted. It directly answers:
- Whether to encrypt-then-code or code-then-encrypt
- The key management cost of each choice
- Whether per-chunk or per-file key derivation is preferable

## Known constraints (from existing papers)

- Encryption must be client-side and mandatory (system design non-functional requirement)
- The service must never hold decryption keys
- Key derivation per chunk must be considered — a single file key means one key compromise exposes all chunks
- Forward secrecy should be considered but its cost is unknown
- Convergent encryption (for deduplication) is not needed — data owners pay for all their own content (Q09-3)
- DHT chunk IDs must not expose file identity — candidate: `HMAC(chunk_hash, file_owner_key)` as the DHT lookup key (Q07-5)

## Decision

**Not yet made.** Fill this ADR after reading Szabó et al. 2025.

## References

- [ADR-022](ADR-022-encryption-erasure-order.md): encrypt-then-code vs code-then-encrypt (same blocker)
- [open-questions.md Q07-5](../research/open-questions.md): DHT pseudonymisation of chunk IDs
