# ADR-014 — Four Adversarial Provider Defence Classes

**Status:** Accepted
**Topic:** #19 Adversarial Provider Behaviour
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 07

---

## Context

Providers are untrusted parties. They may delete data between audits, fake audit responses, pool with other providers to hold correlated shards, or retrieve data just-in-time for an audit without storing it persistently. Each attack class requires a different mitigation.

## Decision

Four defence classes, each addressing a specific adversarial pattern:

**Defence 1 — Honest Geppetto attack (correlated shard placement):**
A single correlated provider group (same subnet, same ISP AS number) must not hold more than 20% of shards for a single file.
- Placement constraint enforced at write time, not retrospectively
- Implementation: the assignment service tracks ASN/subnet per provider; new assignments skip providers that would push a cluster over 20%
- Activate only after 5 × n shards exist in the network (new networks have too few shards to enforce meaningfully)

**Defence 2 — Outsourcing attack (provider retrieves data on-demand from another node instead of storing it):**
Use Filecoin's Seal concept: require providers to prove data is stored in a way that cannot be reconstructed in real time from a remote source within the challenge response window. The response deadline `(chunk_size / declared_upload_speed) × 1.5` makes just-in-time retrieval from a 5 Mbps connection infeasible for a 256 KB chunk.

**Defence 3 — Just-in-time retrieval (provider pre-caches data only when a challenge is imminent):**
Challenge timing is randomised — providers cannot predict when the next challenge will arrive. The challenge nonce is `HMAC(server_secret, chunk_id + server_timestamp)` where server_timestamp is set by the server, not the provider. Response latency is recorded in the audit receipt and used as a just-in-time detector (`response_latency_ms` field in [ADR-017](ADR-017-audit-receipt-schema.md)).

**Defence 4 — False audit responses (provider sends a plausible-looking response without having the chunk):**
The response must be `SHA256(chunk_data || challenge_nonce)`. Without the chunk, the provider cannot compute a valid response. Verified by the microservice which independently has the expected hash.

## Consequences

**Positive:**
- Each attack class is independently mitigated without dependencies between defences
- JIT detection is built into every audit receipt as a measurable signal

**Negative / trade-offs:**
- ASN diversity cap adds complexity to the assignment service
- Geppetto cap is only enforceable at write time — retrospective enforcement requires rebalancing, which is a repair event

**Open constraints:**
- Distributed correlated failure detection without a central coordinator (Q07-4) is deferred to Phase 2B (EigenTrust research)

## References

- [Paper 07 — SoK](../research/paper-07-sok-dsn.md): Honest Geppetto attack; outsourcing attack; JIT retrieval
- [ADR-002](ADR-002-proof-of-storage.md): PoR challenge design
- [ADR-017](ADR-017-audit-receipt-schema.md): response_latency_ms field
