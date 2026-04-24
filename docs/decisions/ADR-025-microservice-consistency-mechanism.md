# ADR-025 — Microservice Cluster: (3,2,2) Quorum + Gossip

**Status:** Accepted
**Topic:** #1 Coordination Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 12 — Dynamo

---

## Context

[ADR-001](./ADR-001-coordination-architecture.md) adopted a hybrid microservice + Kademlia DHT architecture and specified that the coordination microservice uses a "quorum read mechanism for consistency." That ADR left the quorum parameters and the membership/failure detection mechanism unspecified. Without concrete parameters, engineers cannot implement the microservice cluster. The cluster must tolerate the failure of one of its replicas without losing availability, and must detect failed replicas without a centralised registry.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Managed consensus service (etcd / Consul) | Battle-tested; strong consistency; automatic leader election | Introduces an external operational dependency; adds latency for every coordinated write; overkill for a cluster of 3–5 nodes |
| Single-node microservice | Simple to implement | Single point of failure; violates the availability requirement stated in ADR-001 |
| **N=3 quorum + gossip membership (Dynamo model, storng + eventual consistency)** | Proven at Amazon scale; no external dependency; tolerates one replica failure; gossip is self-healing | Eventual membership consistency (up to ~10 s stale); requires seed node configuration at deploy time |

## Decision

The coordination microservice is deployed as a cluster of **N=3 replicas**.

**Quorum parameters:**

| Parameter | Value | Meaning |
|---|---|---|
| N | 3 | Total replicas storing each metadata record |
| R | 2 | Minimum replicas that must respond to a read |
| W | 2 | Minimum replicas that must acknowledge a write |
| R + W | 4 > N | Quorum overlap guaranteed; stale reads impossible when all nodes are healthy |

R=2, W=2 means the cluster tolerates the failure of one replica for both reads and writes. This matches the constraint from [Paper 11](../research/paper-11-bailis-coordination.md) (Bailis): the 6 non-I-confluent operations (escrow debit, score floor, chunk placement, token validation, physical delete prohibition, escrow seizure) are handled by the single authoritative payment/assignment service, not by quorum vote. The quorum applies to metadata reads and the I-confluent append operations.

**Membership and failure detection:**

Each microservice replica contacts one randomly chosen peer every second to reconcile membership change histories (gossip). Membership changes are persisted on disk before being gossiped. Logical ring partitions are prevented by two designated seed nodes whose addresses are provided via static configuration at deploy time. Seeds are fully functional replicas, not dedicated nodes.

A replica considers a peer failed if that peer does not respond to its messages (local failure detection — no global consensus on failure state required). On detecting a failed peer, the healthy replicas continue serving requests using the remaining R/W replicas. The failed replica is retried periodically.

**Coordination routing:**

For latency-sensitive paths (audit challenge dispatch, chunk assignment decisions, capability token validation), use **client-driven coordination**: the service client caches the cluster membership and routes requests directly to the responsible replica, bypassing a load balancer. The client refreshes its membership cache from a random replica every 10 seconds, or immediately on detecting a stale view.

For internal tooling and administrative paths, load-balancer routing is acceptable.

**Background task throttling:**

Background tasks (Merkle log compaction, materialised view refresh, repair job queuing) monitor the 99th percentile of foreground DB read latency over the last 60 seconds. If that latency approaches a preset threshold (starting value: 50 ms), background task slice allocation is reduced. This prevents background work from impacting the audit challenge SLA.

**What this ADR does NOT cover:**

- Horizontal database sharding (not needed in V2 — peak ~3 payment releases/sec, confirmed by [ADR-016](./ADR-016-payment-db-schema.md))
- Provider-facing gossip (providers are managed by polling, [ADR-006](./ADR-006-polling-interval.md))
- P2P data-plane routing (Kademlia, [ADR-001](./ADR-001-coordination-architecture.md))

## Consequences

**Positive:**
- One replica failure does not interrupt service — availability is maintained with N=3, R=2, W=2
- Gossip membership is self-healing: replicas added or removed without manual intervention beyond seed configuration
- Client-driven coordination removes load-balancer as a latency source for hot paths (30+ ms improvement at 99.9th percentile, Dynamo Section 6.4)
- No external dependency (no etcd/Consul to operate)

**Negative / trade-offs:**
- Membership view may be up to ~10 s stale at the client; a newly joined replica may not receive requests immediately
- Seed node addresses must be stable — if both seeds become unavailable simultaneously, new replicas cannot discover the cluster
- Gossip at 1 peer/s adds ~N messages/s of intra-cluster traffic; acceptable for a 3-node cluster

**Open constraints:**
- Background task threshold of 50 ms is a starting value; must be tuned empirically after launch
- If the cluster grows beyond 5 replicas, evaluate whether managed consensus (etcd) becomes simpler to operate than hand-rolled gossip

## References

- [Paper 12 — Dynamo](../research/paper-12-dynamo.md): N/R/W quorum; gossip membership; client-driven coordination; background admission control
- [ADR-001](ADR-001-coordination-architecture.md): hybrid microservice + Kademlia architecture; quorum read mechanism (now parameterised by this ADR)
- [ADR-013](ADR-013-consistency-model.md): I-confluence map; the 6 coordinated operations that the microservice cluster must handle
- [ADR-016](ADR-016-payment-db-schema.md): peak throughput confirms single-server Postgres is not a bottleneck; horizontal sharding not needed
