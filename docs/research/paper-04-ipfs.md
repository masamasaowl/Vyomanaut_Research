# Paper 04 — IPFS: Content Addressed, Versioned, P2P File System

**Authors:** Juan Benet, 2015
**Topics:** #1, #2

---

## Problem Solved

IPFS is the closest existing system to our architecture. Knowing its strengths and failures avoids rediscovering solved problems.

1. It moves more towards Web3 (data not saved on a single server) whereas we try to incentivise the data each node holds
2. Files are addressable by their content, not location — they survive as providers change
3. We address a chunk by the hash of its content, not by the provider who stores it → ensures files survive peer churn

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Immutable Merkle DAG | Mutable file structure | Future updates requiring PATCH of a file while it is with the publisher become difficult |
| DHT lookup before retrieval | Direct HTTP | First retrieval is slower than an HTTP request |

---

## Breaks in Our Case

- **No persistent storage guarantee** ≠ **our core promise to data owners**
  → IPFS is a general-purpose content-addressable distributed file system, not a contracted storage network

- **No incentive for storage** ≠ **our escrow model**
  → BitSwap's tit-for-tat block exchange is replaced by escrow-motivated direct transfers. The transfer mechanism (direct P2P, chunked, pipelined) is retained; the incentive mechanism is replaced.

- **No access control — any node can retrieve** ≠ **our data owner privilege model**
  → An additional control layer is needed; a retrieve call must be authenticated

---

## Decisions Influenced

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** Use Coral with small store values for info about other nodes and large store values for metadata of actual files

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]:** Verification of a chunk can be done by the content addressing mechanism of Merkle DAGs — if the hash doesn't match, the chunk is corrupted. A separate proof of storage mechanism must exist for challenge-response audits.

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** To store a chunk, pin it to the node; to remove or delete, unpin the object.

- **[ADR-009](../decisions/ADR-009-reliability-scoring.md) [#8 Reliability Scoring]:** The reliability score can be based on storage uptime and response latency

- **[ADR-017](../decisions/ADR-017-payment-db-schema.md) [#13 Escrow]:** The provider earnings ledger uses these mathematical relations from BitSwap:
  - Debt ratio: `r = bytes_sent / (bytes_recv + 1)`
  - Probability of sending to a debtor: `P(send|r) = 1 - (1 / (1 + exp(6 - 3r)))`

---

## Disagreements

Disagreements in literature criticise the Web3 aspect and not the properties useful for this project.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q04-1 through Q04-3.
