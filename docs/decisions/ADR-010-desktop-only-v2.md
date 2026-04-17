# ADR-010 — No Mobile Providers in V2; Deferred to V3

**Status:** Accepted
**Topic:** #5 Peer Selection (scope decision)
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 06, 08

---

## Context

The original system design included mobile phones as potential storage providers. Three papers independently show that mobile inclusion in V2 would harm the existing provider network rather than help it.

## Evidence for deferral

**From Paper 05 (Storj):** MTTF assumed for NAS operators is 6–12 months. Inclusion of mobile operators would bring the network MTTF to ~1 month — requiring significantly higher erasure n and dramatically increasing repair bandwidth.

**From Paper 06 (Blake & Rodrigues):** Every new flaky node burdens the existing reliable nodes' bandwidth. Repair bandwidth cost per low-MTTF node is borne by all other nodes in the network.

**From Paper 08 (Bhagwan):** 6.4 joins/leaves per host per day in a DHT with no financial friction. Financial friction reduces this — but mobile OS background execution limits mean a phone may appear offline not because the owner left but because the OS killed the daemon.

**Provider tier collapse:** With mobile excluded, our remaining tiers (NAS, corporate desktop, home desktop) have MTTFs in the 180–380 day range — close enough that a single parameter set applies. Tiering adds complexity with no benefit in V2.

## Decision

No mobile providers in V2. The provider tier model collapses to a single tier with:
- MTTF range: 180–380 days
- Single erasure parameter set ([ADR-003](ADR-003-erasure-coding.md))
- Single repair threshold ([ADR-004](ADR-004-repair-protocol.md))
- 4–6 month vetting period instead of tier-based differentiation

Mobile providers may be introduced in V3 after studying V2 network performance. Before that, the following must be researched:
- iOS/Android background execution limits (reading-list Phase 1 #11)
- Mobile MTTF in a financially-incentivised context (Q05-3)
- Mobile-specific erasure parameters (Q09-6)
- Mobile departure thresholds (Q09-5)

## Consequences

**Positive:**
- Simpler parameter space — single MTTF range, single erasure set
- Repair bandwidth budget is manageable at desktop MTTF
- No OS background execution complexity in V2

**Negative / trade-offs:**
- Excludes a significant population of potential providers at launch
- Limits network density in the short term

**Open constraints:**
- This decision is explicitly temporary and must be revisited for V3
