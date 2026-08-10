# ADR-069 — Three-tier scale simulation topology

**Status:** Accepted
**Track:** LTS
**Topic:** #16 Simulation & Scale *(new topic)*
**Supersedes:** — *(extends `MVP §8.3`'s `--sim-count`)*
**Research source:** `architecture.md §27.4/§27.5/§27.7`; NFR-043; NFR-045; this section's arithmetic

## Context

`--sim-count` spawns N full provider daemons as goroutines, each with its own storage engine. At
~150 MB per daemon, 10,000 nodes needs 1.5 TB. The flag was designed for the five-node demo and does
not generalise — but the questions that need 10,000 nodes are coordination-plane questions
(NFR-043's unmeasured INSERT ceiling, audit dispatch throughput, scoring convergence, repair-queue
drain rate, relay slot demand) that do not require a storage engine on the provider side at all.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Scale `--sim-count` as-is** | No new concepts | 1.5 TB for 10,000. Physically impossible on any single machine |
| **Containers, one per node** | Realistic isolation | Container overhead dominates; ~200–500 per desktop before the kernel objects, and each still carries a storage engine |
| **Pure discrete-event model** | Millions of nodes, trivially | Tests a model, not the implementation. Cannot surface a Postgres ceiling or a query plan regression — which is the entire point |
| **Three-tier population — chosen** | The storage path is genuinely exercised by the real tier; the coordination plane is exercised at full population; fits one desktop | Synthetic nodes must produce cryptographically valid audit responses without storing bytes, which is the design's one non-trivial requirement |

## Decision

1. **Three tiers** — Real (full daemon), Lightweight (transport + protocol, in-memory chunk store),
   Synthetic (protocol surface only). Selected by `--sim-tier={real,light,synthetic}` alongside
   `--sim-count`.
2. **Synthetic providers derive chunk bytes deterministically**: `chunk_bytes = PRF(node_seed ‖
   chunk_id)`, so audit responses are correct and indistinguishable from a real provider's. A
   synthetic node stores one 32-byte seed, not chunks. The microservice is not told which tier a
   provider is — that is what makes it a valid test of the microservice.
3. **`127.0.0.0/8` per-node addressing** on Linux, removing the per-source-IP ephemeral port ceiling.
4. **PgBouncer in transaction-pooling mode** is a required component of the harness, not an optional
   optimisation.
5. **Simulation runs are never a substitute for physical-fleet runs.** Both are required before any
   scale claim is published; the harness records tier composition in every result so no number is
   ever quoted without it.
6. **`--sim-tier` is refused unless the build has real `go-libp2p`** — a compile-time guard, so
   nobody accidentally scale-tests the substituted stack.

## Consequences

Ten thousand coordination-plane nodes on a 32 GB desktop, for free. NFR-043's ceiling becomes
measurable, which unblocks the write-back that `architecture.md §27.4` has been waiting on. The
tier composition is recorded with every result, so a published figure can never be mistaken for
10,000 full providers.

**Open constraints:**

- The synthetic tier cannot exercise repair, because repair requires a provider to actually hold
  shards. Repair-queue behaviour must be measured on the lightweight tier and extrapolated, with the
  extrapolation stated. Recorded, not hidden.
- Whether the PRF should be the same construction as the AONT key stream or an independent one is a
  design detail for the implementing session.

---
