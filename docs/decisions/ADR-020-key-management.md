# ADR-020 — Key Management Strategy

**Status:** Proposed — deferred to Phase 3
**Topic:** #10 Key Management Strategy
**Supersedes:** —
**Superseded by:** —
**Research source:** TBD

---

## Context

If the data owner loses their key, their data is unrecoverable unless a key management strategy exists. Key rotation and delegation to the service for re-encryption/republication are also open. This ADR cannot be resolved until [ADR-019](ADR-019-client-side-encryption.md) (encryption scheme) and [ADR-022](ADR-022-encryption-erasure-order.md) (encrypt-then-code order) are accepted.

## Blocked on

- ADR-019 (encryption scheme must be decided first)
- ADR-022 (encrypt-then-code vs code-then-encrypt affects the key management cost)
- Phase 3 research into key management systems

## Known constraints (from existing papers)

- If encrypt-then-code is chosen: each chunk is encrypted with the same key → one key per file, simpler but more exposure if key is leaked
- If code-then-encrypt is chosen: each chunk can have an independent key → key management cost multiplied by n=56 per file
- Key loss is a catastrophic failure mode — the service must not hold keys (zero-knowledge requirement) but some recovery mechanism is needed
- IPFS IPNS proxy signature approach (Q04-3): if the data owner agrees, a microservice can hold a delegated signing key for DHT republication purposes only — this is not a decryption key

## Decision

**Not yet made.** Fill this ADR after Phase 3 research.

## References

- [ADR-019](ADR-019-client-side-encryption.md): encryption scheme (must be accepted first)
- [ADR-022](ADR-022-encryption-erasure-order.md): order decision (must be accepted first)
- [open-questions.md Q04-3](../research/open-questions.md): IPNS proxy signature as partial solution
