# Paper 02 — Kademlia: A P2P Information System Based on XOR Metric

**Authors:** Peter Maymounkov & David Mazieres, New York University
**Topics:** #1

---

## Problem Solved

How, in an infinitely large library, can each visitor find any book individually?

1. Each book and visitor is given an ID
2. Closeness between two IDs is calculated using XOR
3. Each visitor holds a small handbook of other visitors at specific distances (k-buckets)
4. Each node learns about other nodes as a side effect of a search — self-configuration ends the peer churn problem
5. When a resource is needed: call the visitor nearest to the resource ID → they point to someone even closer → converge to the closest node in logarithmic steps
6. A 10-million-node network takes roughly 20 hops to locate a resource
7. In high peer flooding and churn, k-buckets keep long-lived nodes alive

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| XOR metric throughout | Pastry / Tapestry routing | Symmetric, simpler routing guarantees |
| Keep older live node over new node | Replace on arrival | Improves long-term stability; Gnutella graph property |
| alpha=3 parallel lookups | Sequential O(log n) lookups | O(alpha × log n) messages but better fault tolerance |
| 24-hour key-value refresh | Permanent storage | Values require active maintenance to survive |
| Random node IDs | Role/geography-based IDs | Personalised ID distribution not possible |

---

## Breaks in Our Case

- **Kademlia only stores IDs as keys, not large files** ≠ **we need actual file storage**
  → File storage must be handled by a completely separate protocol; Kademlia strictly handles provider lookup for a given fileID

- **24-hour key refresh is implicit** ≠ **our availability service must actively maintain it**
  → The availability service must republish key-value pairs every 24 hours; only one node republishes per window

---

## Decisions Influenced

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Provider discovery, metadata routing, and peer lookup all happen via Kademlia. To find a chunk: use FIND_VALUE. For replication candidate selection: use FIND_NODE to find the k best candidates.

- **[ADR-008](../decisions/ADR-008-reliability-scoring.md) [#8 Reliability Scoring]:** Providers with high uptime continue to stay connected (Gnutella graph property) — they must be proportionally rewarded

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Caching can be handled autonomously by the nodes

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection]:** New storage provider join procedure — if the head of the k-bucket responds, reject the new value and shift the table up; if it doesn't respond, remove the oldest node

- **[ADR-006](../decisions/ADR-006-polling-interval.md) [#6 Polling]:** The availability checker must update the key-value pair every 24 hours; Kademlia does not keep stale values

---

## Disagreements

- **S/Kademlia (Paper 03):** BitTorrent contains malicious nodes at any given time — base Kademlia has no defence against this.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q02-1 through Q02-5.
