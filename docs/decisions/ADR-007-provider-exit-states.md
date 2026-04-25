# ADR-007 — Four Provider Exit States

**Status:** Accepted
**Topic:** #7 Provider Exit State Mechanism
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 06, 08, 09, 12, 29

---

## Context

Providers will leave the network for many reasons — some planned, some not. Each departure type has a different bandwidth cost and financial consequence. The system must distinguish between them to avoid triggering unnecessary repair and to enforce financial penalties only where appropriate.

## Decision

Four exit states, with distinct repair triggers and escrow consequences:

| State | Definition | Repair trigger | Escrow consequence |
|---|---|---|---|
| **Accidental / temporary absence** | Provider goes offline unexpectedly for < 72 h | None — wait for timeout | Reliability score decremented per polling cycle |
| **Promised downtime** | Provider declares a planned absence in advance (0–72 h window) | None — wait for promised period p | Penalise directly if promise is broken; fine deducted from linked payments account |
| **Permanent silent departure** | Provider absent for > 72 h with no announcement | Trigger after 72 h | Seize escrow earnings; sign provider out of network |
| **Announced departure** | Provider explicitly notifies the system of permanent exit | Trigger immediately on announcement | Release pending escrow; sign provider out of network |

### Returning provider after silent departure

When a provider is declared "Permanently silent departed" (absence > 72h):

1. The microservice sets `providers.status = DEPARTED` (soft delete — physical row removal is
   prohibited per [ADR-013](./ADR-013-consistency-model.md) consistency model).
2. All of the provider's chunk assignments are hard-removed from the `chunk_assignments` table.
   The chunks are already in the repair queue; the assignments exist only as a routing record
   for challenge dispatch. Removing them stops all further challenge issuance to this provider.
3. The provider's escrow is seized ([ADR-024](./ADR-024-economic-mechanism.md)).
4. The provider's Peer ID is removed from the DHT routing tables on the next republication
   cycle ([ADR-001](./ADR-001-coordination-architecture.md): availability service manages DHT records).

If the provider comes back online after step 2:

- They still hold their copy of the chunk data on disk.
- The microservice will no longer issue challenges to them (their assignments are removed).
- If the provider daemon connects to the microservice (heartbeat or direct connection):
  - The microservice identifies them by Peer ID and finds `status = DEPARTED`.
  - The microservice returns HTTP 403 ("Provider account inactive").
  - The provider daemon logs this and halts operation.
- If the provider wants to rejoin: they must go through the full registration flow.
  Their existing chunk data on disk is incidental (not tracked, not audited, not paid).
  Rejoining creates a new provider_id; there is no "rejoin with existing data" protocol.

This is the correct design: the departed provider's chunk has already been replaced by repair.
Two providers holding the same shard is not a durability problem (more replicas is fine) but
it is a payment problem. Hard-removing assignments on departure ensures the payment system
never issues credits for untracked storage.

The vetting period begins from scratch on rejoin. No prior score history is carried over.

**Promise enforcement:**
A provider who promises downtime but overruns the promised period is automatically reclassified to Permanent silent departure. Fines are deducted from the linked payment account (UPI/Razorpay). The promised exit mechanism saves repair bandwidth by triggering replication only for true departures.

**Emergency lower bound:**
Regardless of which state applies, if available fragments for any chunk drop to s=16 (reconstruction floor), repair is triggered immediately.

## Consequences

**Positive:**
- Promised downtime and announced departure give the repair system advance notice, allowing it to pre-select replacement providers before the gap materialises
- Escrow seizure on silent departure creates a financial deterrent against vanishing
- Four-state model matches Bhagwan's ([paper-08](../research/paper-08-bhagwan-availability.md)) two-component availability model (daily churn + permanent departure)

**Negative / trade-offs:**
- Promise enforcement requires the payment microservice to be operational at all times (coordination dependency)
- Providers must have a payment account linked at registration to enable fine collection

**Open constraints:**
- The 72 h threshold is calibrated for desktop. Mobile absence patterns are different (Q09-5) — revisit in V3.

## References

- [Paper 06 — Blake & Rodrigues](../research/paper-06-blake-rodrigues.md): promised exit saves bandwidth
- [Paper 08 — Bhagwan](../research/paper-08-bhagwan-availability.md): two-component model motivates the state machine
- [Paper 09 — Bolosky](../research/paper-09-bolosky-feasibility.md): 72 h threshold from bimodal distribution
- [Paper 12 — Dynamo](../research/paper-12-dynamo_1.md): Section 4.8.1 independently confirms that transient outages must not trigger rebalancing; permanent departure requires an explicit announcement mechanism
- [Paper 29 — Filecoin](../research/paper-29-filecoin-whitepaper.md): Manage.RepairOrders three-case escalation structure maps to the four exit states; independent validation from the only deployed graduated-penalty DSN
