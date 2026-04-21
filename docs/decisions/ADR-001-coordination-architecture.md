# ADR-001 — Use Microservices + Kademlia DHT for Coordination

**Status:** Accepted
**Topic:** #1 Coordination Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 01, 02, 03, 04, 05, 07

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

Hybrid architecture: hardened microservices handle orchestration (registration, audit scheduling, payment, repair triggering, reputation scoring). Kademlia DHT handles all data-plane lookups: provider discovery, chunk location (FIND_VALUE), and replication candidate selection (FIND_NODE).

Implementation details:
- Node IDs: derived from registered account UUID (registration acts as certificate authority, replacing cryptographic ID generation from S/Kademlia)
- Disjoint lookup paths: d=4,8 — maintains 99% efficiency at 30% adversarial node share
- Replication parameter: k=8,16 (2×d)
- Sibling list: s=20, c=2.5 — failure probability ~5×10⁻⁷
- Key-value pairs refreshed every 12 h by the availability service (Update after reading [Paper 20](../research/paper-20-trautwein-ipfs.md))
- Caching handled autonomously by nodes
- Bloom filters used for garbage collection

To prevent DoS on the central microservice: quorum read mechanism for consistency (latency trade-off accepted).

During DHT lookups, content addressing must not reveal file identity (zero-knowledge requirement).

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
