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
| --- | --- | --- |
| Managed consensus service (etcd / Consul) | Battle-tested; strong consistency; automatic leader election | Introduces an external operational dependency; adds latency for every coordinated write; overkill for a cluster of 3–5 nodes |
| Single-node microservice | Simple to implement | Single point of failure; violates the availability requirement stated in ADR-001 |
| **N=3 quorum + gossip membership (Dynamo model, storng + eventual consistency)** | Proven at Amazon scale; no external dependency; tolerates one replica failure; gossip is self-healing | Eventual membership consistency (up to ~10 s stale); requires seed node configuration at deploy time |

## Decision

The coordination microservice is deployed as a cluster of **N=3 replicas**.

**Quorum parameters:**

| Parameter | Value | Meaning |
| --- | --- | --- |
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

- The audit verifier under ADR-059 is stateless — any replica holding a file's authenticator keys can issue and verify a challenge independently, with no counter or spent-challenge state shared across the cluster. The audit path adds no operation to ADR-013's six coordinated operations and stays outside F-35's and F-76's blast radius. Had the Juels–Kaliski sentinel family been adopted, a spent-sentinel counter would have been a seventh.
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

## Addendum — Windowed p99 Control Signal for the Background-Task Throttle

*Appended after the M-OBS/13/14 milestone review. Originally proposed there as a standalone "ADR-033," before the real ADR-033 (`audit-receipts-partitioning`) came to exist. Filed here instead of as a new ADR because it corrects this ADR's own "Background task throttling" decision (above) rather than making a new one — the 60-second-window requirement was already decided here; what follows fixes an implementation that drifted from it.*

**Context.** The "Background task throttling" paragraph above already specifies the control signal precisely: p99 of foreground DB read latency *"over the last 60 seconds."* NFR-028 restates the same window. The M-OBS implementation (Session OBS.1.1) instead wired the throttle to `histogram_quantile` over the `vyomanaut_db_read_latency_seconds` Prometheus histogram — which is **cumulative since process start**, not a 60-second window. After hours of healthy operation, a genuine latency burst gets diluted by accumulated history, so the throttle fires late or never — precisely when the audit challenge SLA this mechanism exists to protect most needs it. This is a scalability-critical control loop; the mismatch matters at load, not at demo scale.

**Decision.** Separate the **control signal** from the **exported metric**:

- `internal/metrics` maintains a dedicated **60-second sliding-window p99 estimator** for foreground DB reads (a rotating per-second bucketed latency counter, or a bounded ring buffer). Expose `metrics.ForegroundReadP99Window() time.Duration`. The background-task throttle reads only this — never the cumulative histogram.
- The cumulative `vyomanaut_db_read_latency_seconds` histogram remains, unchanged, for dashboards and alerts (architecture.md §23).
- A single `metrics.ForegroundReadLatency.Observe(d)` call updates **both** the histogram and the window, so instrumentation call sites don't need to know about the split.
- Add a `0.04 s` bucket to the histogram so it can visualize "approaching 50 ms" (NFR-028's own phrasing). Final bucket boundaries: `{0.01, 0.025, 0.04, 0.05, 0.1}`.

**Consequences.**

- *Positive:* the throttle reacts to *recent* latency, as originally intended above; the exported histogram stays a correct cumulative series for dashboards; no PromQL recording-rule gymnastics needed for an in-process control loop.
- *Negative / trade-offs:* a small amount of additional in-process state (≤ 60 buckets); `Observe()` performs two updates instead of one.
- *Affected:* architecture.md §23 (bucket boundaries note), NFR-028's implementation note; Sessions OBS.1.1, OBS.1.2, and the M12 Session 12.1.1 throttle read site.

**Status:** Proposed (this addendum only — the rest of ADR-025 is unaffected and remains Accepted).

## References

- [Paper 12 — Dynamo](../research/paper-12-dynamo.md): N/R/W quorum; gossip membership; client-driven coordination; background admission control
- [ADR-001](ADR-001-coordination-architecture.md): hybrid microservice + Kademlia architecture; quorum read mechanism (now parameterised by this ADR)
- [ADR-013](ADR-013-consistency-model.md): I-confluence map; the 6 coordinated operations that the microservice cluster must handle
- [ADR-016](ADR-016-payment-db-schema.md): peak throughput confirms single-server Postgres is not a bottleneck; horizontal sharding not needed
