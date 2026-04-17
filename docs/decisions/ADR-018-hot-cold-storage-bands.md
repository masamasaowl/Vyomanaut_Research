# ADR-018 — Explicit Hot and Cold Provider Storage Bands

**Status:** Accepted (Cold band only in V2; Hot band deferred to V3)
**Topic:** #5 Peer Selection (storage type)
**Supersedes:** —
**Superseded by:** —
**Research source:** Vyomanaut design decision (no external paper)

---

## Context

Providers have different hardware profiles. Some have large disks and stable connections; others have fast connections but limited storage. Data owners also have different needs: some data is accessed frequently (hot), some is archived (cold). Matching provider profile to data access pattern improves both economics and performance.

## Decision

Two explicit bands, with provider consent given at join time:

**Cold Storage Band:**
- High storage capacity
- High MTTF (desktop, NAS)
- Low bandwidth acceptable
- High space overhead in erasure (big chunks for durability)
- Payment basis: per GB stored per month
- Target: archive data, backups, infrequently accessed files

**Hot Storage Band:**
- Low storage, high bandwidth, low latency
- Lower space overhead in erasure (smaller chunks for retrieval speed)
- Payment basis: per GB stored + per retrieval
- Target: frequently accessed data, application assets

**V2 scope:** Only Cold Storage band is implemented. Hot Storage is a V3 addition.

**Provider enrolment flow:**
1. Provider declares their band preference at join time (Cold or Hot)
2. System validates the claim during vetting (bandwidth and latency tests for Hot; storage capacity test for Cold)
3. Data owners specify their requirement (Hot or Cold) at upload time and are charged accordingly

## Consequences

**Positive:**
- Explicit consent avoids mismatched expectations between provider hardware and data requirements
- Charging data owners differently for Hot vs Cold creates a sustainable pricing model
- Cold-only V2 is simpler to implement and matches the desktop-first decision ([ADR-010](ADR-010-desktop-only-v2.md))

**Negative / trade-offs:**
- Hot band requires latency-optimised provider selection — adds complexity to the assignment service
- Two payment bases (Cold: per GB/month; Hot: per GB + per retrieval) require separate accounting paths

**Open constraints:**
- Hot band erasure parameters differ from Cold band — must be researched separately in Phase 2A before Hot band launch
