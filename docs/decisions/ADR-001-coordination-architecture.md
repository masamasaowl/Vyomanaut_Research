# ADR-001 — Use Microservices + Kademlia DHT for Coordination

**Status:** Accepted
**Topic:** #1 Coordination Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 01, 02, 03, 04, 05, 07, 20, 28

---

## Context

The system needs a coordination layer to handle provider discovery, chunk lookup, metadata routing, and repair orchestration. The original V1 failed partly due to inefficient peer discovery and transfer. The choice is between a fully decentralised DHT, a fully centralised server, or a hybrid.

Pure decentralisation (DHT for everything) does not work below ~100 peers and creates Sybil/Eclipse attack surfaces without registration gating. Pure centralisation creates a single point of failure and a DoS target.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Pure Kademlia DHT | No central entity; self-healing | Requires cryptographic node IDs; unsafe below ~100 peers; no admission control |
| Centralised satellite (Storj model) | Simple; performant | Single point of failure; DoS target; reduces provider trust |
| **Hybrid: hardened microservices + Kademlia** | Admission control via registration; DHT for chunk lookup; separation of concerns | Not fully decentralised; microservice becomes a hardened dependency |

## Decision

Hybrid architecture: 
1. Hardened microservices handle orchestration:
		   - registration audit
		   - scheduling, payment
		   - repair triggering
		   - reputation scoring
2. Kademlia DHT handles all data-plane lookups ([paper-02](../research/paper-02-kademlia.md)):
			 - provider discovery 
			 - chunk location (FIND_VALUE) and 
			 - replication candidate selection     (FIND_NODE)

Implementation details:

From [paper-03](../research/paper-03-skademlia.md)
- Node IDs: derived from registered account UUID (registration acts as certificate authority, replacing cryptographic ID generation from S/Kademlia)
- Disjoint lookup paths: d=4,8 — maintains 99% efficiency at 30% adversarial node share 
- Replication parameter: k=8,16 (2×d)
- Sibling list: s=20, c=2.5 — failure probability ~5×10⁻⁷

From [paper-07](../research/paper-07-sok-dsn.md)
- To prevent DoS on the central microservice: quorum read mechanism for consistency (latency trade-off accepted). 
- During DHT lookups, content addressing must not reveal file identity (zero-knowledge requirement).

From [paper-04](../research/paper-04-ipfs.md)
- To store a chunk, pin it to the node; to remove or delete, unpin the object
- Use Coral (deferred to V3)

- Caching handled autonomously by nodes ([paper-02](../research/paper-02-kademlia.md))
- Bloom filters used for garbage collection ([paper-05](../research/paper-05-storj.md))
- Key-value pairs refreshed every 12 h by the availability service (Update after reading [Paper 20](../research/paper-20-trautwein-ipfs.md))


## Consequences

**Positive:**
- Admission control via registration limits Sybil nodes without cryptographic overhead
- DHT provides self-healing peer discovery; logarithmic lookup in O(log n) hops
- Separation between orchestration and data plane is clean

**Negative / trade-offs:**
- Not fully decentralised — microservice is a dependency and must be hardened against DoS
- Pure decentralisation not achievable with this model
- Adds quorum-read latency on microservice reads

**Open constraints:**
- Microservice must be operated by Vyomanaut; if it goes down, orchestration stops (data plane P2P still functions)
- This decision holds only while registration gating remains effective
- DHT privacy: HMAC(chunk_hash, file_owner_key) as the DHT lookup key closes the DHT privacy leakage challenge identified as unsolved across the entire DSN field ([Paper 07](../research/paper-07-sok-dsn.md), Section IX-B). Only the file owner can reverse-map a DHT key to its chunk. This is an active design requirement, not just a recommendation.
- DHT proximity optimisation (global sampling): Periodic global sampling — DHT
lookups for random target prefixes, selecting the closest responding node — provides
a 24% lookup latency reduction in wide-area networks (Rhea et al., [Paper 28](../research/paper-28-rhea-dht-churn.md)). This
is not enabled in V2. V2's India-only provider network has sufficiently homogeneous
inter-node latency (5–40 ms between Indian cities) that XOR routing naturally
approximates geographic proximity, and the relay overhead for NAT-traversal-dependent
providers (50 ms) dominates any DHT-hop savings. 

## References

- [Paper 20 — IPFS Measurement](../research/paper-20-trautwein-ipfs.md): production confirmation of k=16 and alpha=3; DHT server mode requirement for all providers; AS concentration empirically validates 20% ASN cap; IPFS post-v0.5 uses it.
- [Paper 28 — Handling Churn in a DHT](../research/paper-28-rhea-dht-churn.md): validates preference of long-lived-nodes; periodic k-bucket refresh confirmed as the correct churn-handling strategy;
- [Paper 07 — SoK DSN](../research/paper-07-sok-dsn.md): hybrid microservice + DHT architecture; HMAC-pseudonymised DHT keys close the field-wide DHT privacy challenge; the three blockchain functions Vyomanaut must replicate without on-chain writes

