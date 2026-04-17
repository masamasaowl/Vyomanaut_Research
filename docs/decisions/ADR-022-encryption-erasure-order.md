# ADR-022 — Encrypt-Then-Code vs Code-Then-Encrypt

**Status:** Proposed — blocked on Szabó et al. 2025
**Topic:** #15 Encryption-Erasure Interaction
**Supersedes:** —
**Superseded by:** —
**Research source:** TBD — reading-list Phase 1 #10

---

## Context

This is a binary architectural decision with significant downstream consequences for key management and chunk design. It must be resolved before the chunk encoding pipeline is designed. Storj used encrypt-then-code; Szabó et al. (2025) directly challenges this choice.

## Blocked on

**Szabó et al., "Separation and Optimisation of Encryption and Erasure Coding in Decentralized Storage," FGCS Feb 2025** (reading-list Phase 1 #10).

This paper is the primary source for this decision. It directly answers the key management cost of each approach.

## Known tradeoffs (from existing papers)

**Encrypt-then-code (Storj's choice):**
- One key per file — simpler key management
- Erasure coding operates on ciphertext — no interaction between encryption and coding layers
- All n=56 chunks carry the same key material implicitly (losing one chunk key = losing file access)

**Code-then-encrypt:**
- Each of the n=56 chunks can have an independent key
- Key management cost: n=56 keys per file, vs 1 key per file for encrypt-then-code
- Exposed by Szabó et al. as potentially more secure at the chunk level

**What Storj's choice costs us:**
If encrypt-then-code is chosen, a provider who has access to all s=16 fragments and the key can reconstruct the file. The key must never reach the provider — the zero-knowledge guarantee depends on it.

## Decision

**Not yet made.** Fill this ADR after reading Szabó et al. 2025. This ADR is a prerequisite for [ADR-019](ADR-019-client-side-encryption.md) and [ADR-020](ADR-020-key-management.md).

## References

- [Paper 05 — Storj](../research/paper-05-storj.md): stripe-level erasure coding (encrypt-then-code)
- [ADR-003](ADR-003-erasure-coding.md): erasure coding parameters (must be stable before this is decided)
- [ADR-019](ADR-019-client-side-encryption.md): encryption scheme (co-dependent with this decision)
