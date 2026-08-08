# Paper 61 — POCache: Toward Robust Data Access for Erasure-Coded Storage via Parity-Only Caching

**Authors:** authors affiliated with CUHK (Chinese University of Hong Kong), Patrick P. C. Lee's group
**Venue / Year:** Journal of Parallel and Distributed Computing, 2022 (extended journal version of an MSST 2019 conference paper)
**Citations:** established straggler-tolerance line of work, predecessor cited by LocalityCache (2026)
**Topics:** #12 P2P Transfer Protocol, #2 Proof of Storage (retrieval path)
**ADRs produced:** none yet — informs a future ADR under Topic #12, revisit at Tier 4 of the research plan

---

## Problem Solved

In an erasure-coded system, a read that hits even one slow ("straggler") node stalls the whole reconstruction, because the client must wait for `k` of `n` responses and has no cheap way to substitute the slow one. POCache caches a small number of parity blocks (often just one per stripe) in a fast tier, so a client stuck waiting on a straggler can decode using the cached parity plus the `k-1` blocks that already arrived, instead of blocking. This is a read-path tail-latency problem, not an upload-path one — it does not answer ADR-003's still-open upload-quorum-cancellation gap, but it surfaces a related question about Vyomanaut's own retrieval and audit read path.

---

## Key Findings

- Up to 87.9% P99 latency reduction on HDFS deployments, evaluated with a Redis-based cache tier holding a single extra parity block per stripe.
- Distinguishes "hedged reads" (wait, then retry after a timeout) from "proactive reads" (issue more than `k` requests up front in parallel, decode from whichever `k` arrive first) — proactive reads are strictly faster but cost more network requests per read.
- The caching mechanism is a middle ground: get most of proactive reads' latency benefit without the full request-count overhead, by pre-positioning one extra decodeable block close to the client.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Cache 1 extra parity block per stripe in a fast tier | Full proactive reads (request all n, take fastest k) | Most of the latency win at a fraction of the network overhead; requires a controlled caching tier to exist at all |
| Redis-backed centralized cache | No caching, hedged reads only | Predictable sub-millisecond cache hits; assumes a datacenter-local, operator-controlled cache node |

---

## Breaks in Our Case

- **POCache assumes a dedicated caching tier (Redis) co-located in the same datacenter as the storage nodes, with sub-millisecond access** ≠ **Vyomanaut has no controlled infrastructure between a data owner's client and P2P home providers — there is no natural place to put a cache node**
  → The caching mechanism itself does not deploy as-is. What transfers is the underlying insight, not the mechanism: **Vyomanaut's RS(16,56) already stores 40 more shards than the 16 needed to reconstruct** — the same headroom POCache manufactures with one cached parity block, Vyomanaut may already have for free, 40 times over, if the client retrieval path requests more than 16 shards in parallel.

---

## Decisions Influenced

None yet. This paper does not close a currently-open ADR gap — it raises a new, concrete question about existing retrieval behavior that needs an implementation-level answer before it can become a decision. See Q61-1.

---

## Disagreements

None.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q61-1.
