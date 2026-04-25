# ADR-014 — Four Adversarial Provider Defence Classes

**Status:** Accepted
**Topic:** #19 Adversarial Provider Behaviour
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 07, 20, 21, 37, 38, 41

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
Use Filecoin's Seal concept: require providers to prove data is stored in a way that cannot be reconstructed in real time from a remote source within the challenge response window. The response deadline is `(chunk_size / p95_measured_upload_throughput) × 1.5`, where
`p95_measured_upload_throughput` is the 95th percentile of upload throughput observed
during the provider's vetting period audit responses. Measured throughput is derived from audit response times during vetting:
  observed_throughput = chunk_size / response_latency_ms  (bytes/ms → KB/s)

The p95 value (the 95th-percentile slowest observed response, excluding TIMEOUT events)
is stored as `p95_throughput_kbps` per provider in the reliability scoring DB. For a provider with p95_throughput = 500 KB/s:
  deadline = (256 KB / 500 KB/s) × 1.5 = 0.512 s × 1.5 = 768 ms

This is derived from measured behaviour, not self-declaration. A provider cannot extend
their deadline by declaring a low upload speed because the deadline is computed from
what the vetting period observed them actually doing.
For new providers (no vetting history): use the pool-median p95 throughput as the initial
value. This is updated after every audit response using an exponentially weighted moving
average (same EWMA used for per-provider RTO in [ADR-006](./ADR-006-polling-interval.md)). Latency floor — anomalous fast response detection:
If response_latency_ms < (chunk_size / p95_throughput_kbps) × 0.3, the response is
flagged as anomalously fast (consistent with JIT retrieval from a co-located node).

Anomaly handling:
- Flag the receipt: add boolean field `jit_flag BOOLEAN DEFAULT false` to audit_receipts
- 1 jit_flag in a 7-day window: log only
- 3+ jit_flags from the same provider in a 7-day window: apply a 0.5× weight to that
  provider's audit passes in the 24h scoring window ([ADR-008](./ADR-008-reliability-scoring.md)) for the next 30 days
- 3+ jit_flags AND identical response_latency_ms within ±5ms across multiple chunks:
  escalate to manual review queue (collusion signal)


**Defence 3 — Just-in-time retrieval (provider pre-caches data only when a challenge is imminent):**
Challenge timing is randomised — providers cannot predict when the next challenge will arrive. The challenge nonce is `HMAC(server_secret, chunk_id + server_timestamp)` where server_timestamp is set by the server, not the provider. Response latency is recorded in the audit receipt and used as a just-in-time detector (`response_latency_ms` field in [ADR-017](ADR-017-audit-receipt-schema.md)).

**Defence 4 — False audit responses (provider sends a plausible-looking response without having the chunk):**
The response must be `SHA256(chunk_data || challenge_nonce)`. Without the chunk, the provider cannot compute a valid response. Verified by the microservice which independently has the expected hash.

**Defence 5 — Service-denial attack (provider stores data, passes audits, refuses retrieval):**
A provider can submit valid SHA256(chunk_data || challenge_nonce) responses indefinitely — proving data is stored — while refusing libp2p retrieval connections from the data owner. PoR audits cannot detect this.

- Structural mitigation: RS(16,56) requires > 40 of 56 providers to refuse simultaneously before retrieval is impossible. The 20% ASN cap limits any single correlated group to ≤ 11 providers — well below the 40-provider denial threshold.
- Monitoring path: retrieval failures should be surfaced to the reliability scorer (Q41-1). A provider with repeated retrieval failures against multiple data owners should be downgraded and their chunks re-assigned.
- Source: [Paper 41](../research/paper-41-vakilinia-incentive-compatible-dsn.md) (Vakilinia et al., arXiv 2022)

## Consequences

**Positive:**
- Each attack class is independently mitigated without dependencies between defences
- JIT detection is built into every audit receipt as a measurable signal
- The 20% ASN cap serves dual purpose: adversarial (Honest Geppetto attack) and reliability (Paper 38 proves RS(16,56) requires the cap to maintain its durability guarantee under real-world correlated failures). The cap is required for both the security and availability properties of the system.
- Field-wide challenge coverage: [Paper 07](../research/paper-07-sok-dsn.md) (SoK DSN) enumerates seven unsolved challenges across the entire DSN field. Vyomanaut's four defence classes directly close three of them: Challenge 3 (DHT privacy leakage — closed by HMAC-pseudonymised keys, [ADR-001](./ADR-001-coordination-architecture.md)); Challenge 6 (Honest Geppetto — closed by Defence 1, 20% ASN cap); Challenge 7 (bandwidth optimisation — addressed by lazy repair in [ADR-004](./ADR-004-repair-protocol.md) and V3 Hitchhiker in [ADR-026](./ADR-026-repair-bw-optimisation.md)). Challenge 2 (DoS on coordination entity) is mitigated by the (3,2,2) quorum in [ADR-025](./ADR-025-microservice-quorum-gossip_1.md).

**Negative / trade-offs:**
- ASN diversity cap adds complexity to the assignment service
- Geppetto cap is only enforceable at write time — retrospective enforcement requires rebalancing, which is a repair event

**Open constraints:**
- Distributed correlated failure detection without a central coordinator (Q07-4) is deferred to Phase 2B (EigenTrust research)

## References

- [Paper 07 — SoK DSN](../research/paper-07-sok-dsn.md): Honest Geppetto attack named and sourced; seven unsolved DSN challenges enumerated; Vyomanaut closes challenges 3, 6, and 7

- [Paper 41 — Vakilinia et al.](../research/paper-41-vakilinia-incentive-compatible-dsn.md): service-denial identified as a fifth adversarial class; RS(16,56) structural mitigation and ASN cap bound the attack
- [Paper 20 — IPFS Measurement](../research/paper-20-trautwein-ipfs.md): Table 2 measures AS concentration in production; top 10 ASes = 64.9% of all IPs; validates the threat the 20% cap addresses
- [Paper 37 — SHELBY](../research/paper-37-shelby-incentive-compatibility.md): Observation 1 formally proves outsourcing attack is deterred when audit frequency × reconstruction cost > storage cost; Vyomanaut's deadline is the timing-domain equivalent
- [Paper 38 — Nath et al.](../research/paper-38-nath-correlated-failures.md): elevates 20% ASN cap from adversarial mitigation to reliability requirement; without cap, RS(16,56) can be worse than simpler schemes under correlated failures
- [Paper 21 — Saroiu et al.](../research/paper-21-saroiu-p2p-measurement.md): 30% of peers misreport bandwidth; confirms declared speed is not a trustworthy input for security-critical parameters
- [ADR-005](ADR-005-peer-selection.md): vetting period measures throughput directly; these measurements feed the p95_throughput_kbps value
- [ADR-002](ADR-002-proof-of-storage.md): PoR challenge design
- [ADR-017](ADR-017-audit-receipt-schema.md): response_latency_ms field