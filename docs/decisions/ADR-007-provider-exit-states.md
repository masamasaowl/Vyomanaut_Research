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
