# ADR-023 — Provider-Side Storage Engine

**Status:** Proposed — deferred to Phase 2C
**Topic:** #16 Provider-Side Storage Engine
**Supersedes:** —
**Superseded by:** —
**Research source:** TBD

---

## Context

Providers store chunks on their local disk. The storage engine determines how chunks are indexed, retrieved, and garbage-collected on the provider's machine. The choice between an LSM-tree (write-optimised) and an object store (read-optimised) depends on the access pattern for cold vs hot storage.

## Blocked on

Phase 2C research. No papers on this topic have been read yet.

## Known constraints (from existing papers)

- IPFS uses a pinning model: to store a chunk, pin it; to delete, unpin it (Paper 04). This is simple but may not scale to thousands of chunks per provider.
- Bloom filters are used for garbage collection to check if a marked segment has been deleted (Paper 05).
- CPU budget: ≤5% — the storage engine must not be write-heavy under normal conditions ([ADR-009](ADR-009-background-execution.md))
- Cold storage: access is infrequent; durability is paramount — LSM-tree's write amplification is acceptable
- Hot storage: access is frequent; retrieval latency is important — object store may be preferable

## Decision

**Not yet made.** Fill this ADR after Phase 2C research.
