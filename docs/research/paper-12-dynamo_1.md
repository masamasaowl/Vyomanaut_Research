# Paper 12 — Dynamo: Amazon's Highly Available Key-value Store

**Authors:** Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall, Werner Vogels — Amazon.com
**Venue / Year:** SOSP 2007
**Citations:** ~8,000
**Topics:** #1, #4, #6
**ADRs produced:** [ADR-025 — Microservice Cluster: (3,2,2) Quorum + Gossip Membership](../decisions/ADR-025-microservice-quorum-gossip.md)

---

## Problem Solved

Amazon's internal services need a key-value store that is always writeable — no request may be rejected due to node failure or network partition. Traditional RDBMS systems sacrifice availability for consistency, which is unacceptable when a shopping-cart write must succeed even during a datacenter outage. The paper's core contribution is demonstrating that eventual consistency, object versioning, and application-assisted conflict resolution can be combined into a production system that sustains 99.9995% availability at tens of millions of requests per day. For Vyomanaut V2, the paper closes the open question of how to configure the coordination microservice cluster for quorum reads and provides a concrete gossip-based membership model to replace the unspecified "quorum read mechanism" mentioned in ADR-001.

---

## Key Findings

**SLA target:** 99.9th percentile of read and write operations within 300 ms. Average latencies (1.55 ms read, 1.9 ms write for client-driven coordination) are an order of magnitude better than 99.9th percentile — the tail dominates, not the mean.

**Quorum configuration:** The common production configuration is (N=3, R=2, W=2). Setting R + W > N gives quorum-like consistency. Divergent versions occurred in only 0.06% of requests in production — almost entirely from concurrent writers, not failures.

**Client-driven coordination:** Routing requests through a load balancer adds ~30 ms at the 99.9th percentile vs a client that caches ring membership and routes directly. The client polls a random node every 10 seconds for membership updates.

**Gossip frequency:** Each node contacts one random peer per second to reconcile membership change histories. Seed nodes (discovered via external config or static list) prevent logical ring partitions during bootstrap.

**Background task admission control:** Background tasks (anti-entropy, hinted handoff delivery) are throttled by monitoring the 99th percentile of foreground DB read latency over the last 60 seconds against a preset threshold (50 ms). If foreground latency approaches the threshold, background slice count is reduced.

**Load balancing:** Strategy 3 (Q/S tokens per node, equal-sized fixed partitions) reduces per-node membership metadata by three orders of magnitude vs Strategy 1 (random tokens), and achieves the best load-balancing efficiency. Partition files can be relocated as atomic units, simplifying bootstrapping and archival.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Eventual consistency ("always writeable") | Strong consistency | Writes never rejected; conflict resolution pushed to reads; application must handle divergent versions |
| Sloppy quorum + hinted handoff | Strict quorum membership | Survives temporary node unavailability; hinted replicas must be delivered on recovery |
| Vector clocks with semantic reconciliation | Last-write-wins | Preserves all concurrent updates; application must implement merge logic |
| Client-driven coordination | Load-balancer routing | 30 ms reduction at 99.9th percentile; requires clients to cache stale membership for up to 10 s |
| Gossip (1 peer/s) for membership | Centralized registry | No single point of failure; eventual consistency of membership view; logical partitions prevented by seeds |
| Anti-entropy via Merkle trees | Full re-sync | Only divergent key ranges transferred; Merkle tree must be recalculated on node join/leave (mitigated by fixed partitions in Strategy 3) |
| Explicit join/leave for permanent departures | Timeout-only departure detection | Transient outages never trigger rebalancing; permanent departures require administrator action |

---

## Breaks in Our Case

- **Dynamo operates in a trusted, non-hostile Amazon internal environment** ≠ **our adversarial public provider network**
  → Dynamo's design patterns apply only to our microservice cluster (operator-controlled, trusted). Provider-facing interactions require the full adversarial defence stack (ADR-014). Dynamo's lack of authentication cannot be replicated on the provider side.

- **Dynamo replicates N full object copies** ≠ **our erasure-coded chunk model (s=16, r=40)**
  → Dynamo's replication and repair concepts are not applicable to the data storage plane. They apply only to the microservice metadata store (provider registry, chunk location records, audit receipts).

- **Dynamo uses consistent hashing ring for data partitioning** ≠ **our Kademlia DHT for chunk lookup**
  → Dynamo's ring model is not adopted for the data plane. Consistent hashing is only relevant if we ever shard the microservice database horizontally; Kademlia remains the routing mechanism for chunk lookups.

- **Dynamo's gossip contacts 1 random peer per second in a datacenter** ≠ **our consumer desktop providers behind NAT with variable connectivity**
  → 1-second gossip is feasible between microservice replicas in a datacenter. Provider-to-provider gossip at this frequency would be excessive given dynamic IPs and NAT traversal overhead. Gossip applies only to our microservice cluster; providers are handled by our polling model (ADR-006).

- **Dynamo targets objects typically < 1 MB** ≠ **our 256 KB fragments × 56 shards = up to 14 MB per file segment**
  → Dynamo's latency targets (300 ms at 99.9th percentile) are for metadata-scale objects. Our microservice stores only metadata (chunk IDs, provider IDs, audit receipts) — all well under 1 KB — so Dynamo's latency profile applies cleanly. Chunk transfer latency is a separate concern handled by the P2P layer.

- **Dynamo relies on application-assisted semantic reconciliation for conflicting versions** ≠ **our write-once, append-only audit log**
  → Our audit receipts are immutable once written (INSERT only, no UPDATE — ADR-015, ADR-017). No version conflict is possible. Vector clocks are not needed for our append-only operations.

---

## Decisions Influenced

- **[ADR-025](../decisions/ADR-025-microservice-quorum-gossip.md) [#1 Coordination — NEW]** `ACCEPTED`
  Microservice cluster configuration: N=3 replicas, R=2 reads, W=2 writes. Gossip-based membership: each replica contacts one random peer per second; seed nodes prevent logical partitions. Client-driven coordination (direct routing to the responsible replica) is preferred over load-balancer indirection for latency-sensitive paths (audit challenge dispatch, chunk assignment). Background tasks throttled by foreground latency monitoring.
  *Because:* Dynamo's (3,2,2) configuration is a well-tuned production baseline proven at extreme load. Client-driven routing reduces 99.9th-percentile latency by 30+ ms over load-balancer routing.

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]** `STRENGTHENED`
  The "quorum read mechanism" stated in ADR-001 is now concrete: (N=3, R=2, W=2) with client-driven routing. The existing decision to tolerate latency in exchange for consistency is validated. ADR-001 does not change; ADR-025 fills in the implementation parameters.

- **[ADR-007](../decisions/ADR-007-provider-exit-states.md) [#7 Provider Exit]** `CONFIRMED`
  Dynamo Section 4.8.1 establishes the same principle independently: node outages "rarely signify a permanent departure and therefore should not result in rebalancing." Permanent departure requires an explicit mechanism (administrator-initiated in Dynamo; announced-departure state in our model). Our four-state exit machine is consistent with Dynamo's production findings.

- **[ADR-006](../decisions/ADR-006-polling-interval.md) [#6 Availability & Polling]** `CONFIRMED`
  Dynamo's failure detection relies on a purely local notion: node A considers node B failed if B does not respond to A's messages. No global failure state view is needed. This confirms that our polling service (which independently checks each provider and does not need a global consensus on who is down) is correctly scoped.

---

## Disagreements

- **Bailis et al. (Paper 11, VLDB 2015):** Dynamo's vector clocks and semantic reconciliation are a heavyweight solution to a problem our architecture avoids entirely by making audit receipts and chunk records immutable. I-confluence analysis (Paper 11) gives a more precise tool for deciding where coordination is needed than Dynamo's blanket eventual-consistency model.
  *Implication for us:* Do not adopt vector clocks. Our append-only log design already prevents the divergence scenarios Dynamo was built to handle.

- **Dynamo Section 6.6 (scalability limit):** Full membership gossip (each node aware of all others) works for a few hundred nodes but does not scale to tens of thousands. The paper acknowledges this and points to O(1) DHT systems as the solution.
  *Implication for us:* Our microservice cluster will never exceed a handful of replicas (3–5). Full-membership gossip is therefore appropriate. For the provider-facing DHT, Kademlia already solves the scalability problem Dynamo acknowledges.

- **Trautwein et al. (SIGCOMM 2022, reading-list Phase 2A #12):** IPFS production measurements show that gossip protocols at consumer-device scale produce very different churn profiles than datacenter gossip. Dynamo's gossip latency and convergence assumptions are datacenter-specific.
  *Implication for us:* Confirmed — gossip is scoped to the microservice cluster only; provider churn is handled separately (ADR-006, ADR-007).

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q12-1 through Q12-2.
