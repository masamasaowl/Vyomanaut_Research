# Answered Questions

Every question here is closed. The answer is stated, the ADR that locks it in is referenced.
These questions should not be reopened unless the underlying ADR is superseded.

---

## Coordination and DHT (Papers 01–04)

**Q01-1** How to avoid the tracker as a single point of failure?
Answer: Kademlia DHT replaces the tracker for all peer and chunk discovery. → [ADR-001](../decisions/ADR-001-coordination-architecture.md)

**Q01-2** Should requests be pipelined so the connection is always loaded?
Answer: Yes. Multiple chunk transfer streams run in parallel over one QUIC connection (stream independence, no HOL blocking). → [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md)

**Q01-3a** What criteria decide a peer's reliability?
Answer: Storj's four-subsystem pipeline: identity gate, vetting period, audit-pass filtering, throughput preference. → [ADR-005](../decisions/ADR-005-peer-selection.md)

**Q01-3b** How to set initial reliability for a new peer?
Answer: Random optimistic assignment from the vetted pool (equivalent to BitTorrent's optimistic unchoking). → [ADR-005](../decisions/ADR-005-peer-selection.md)

**Q01-3c** How to re-rank peers over time?
Answer: Three rolling windows (24h / 7d / 30d), weighted combination. → [ADR-008](../decisions/ADR-008-reliability-scoring.md)

**Q02-1** Edge cases where Kademlia lookup doesn't converge?
Answer: S/Kademlia sibling list (s=20, c=2.5) guarantees replica survival. Failure probability ~5×10⁻⁷. → [ADR-001](../decisions/ADR-001-coordination-architecture.md)

**Q02-2** How to set replication parameter k?
Answer: k=16 (S/Kademlia: k = 2×d where d=8 disjoint paths). → [ADR-001](../decisions/ADR-001-coordination-architecture.md)

**Q02-3** How to set DHT branching factor and concurrency?
Answer: k-bucket size k=16 (configured from libp2p default of 20), alpha=3 parallel lookups, binary XOR tree branching factor 2. O(log n / 3) round trips in practice. → [ADR-001](../decisions/ADR-001-coordination-architecture.md), [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md)

**Q02-4** Should the data owner participate in the DHT as a node?
Answer: No. Data owners interact only with the microservice. DHT is for the provider swarm.

**Q02-5** How to handle efficient DHT key-value republication?
Answer: Availability service republishes every 12 hours per provider record. Record expiry is 24 hours, providing a 12-hour buffer against delayed republication. → [ADR-001](../decisions/ADR-001-coordination-architecture.md), [ADR-006](../decisions/ADR-006-polling-interval.md)

**Q03-1** Should node IDs be hashed from a public key, or is registration gating sufficient?
Answer: Registration gating (KYC/phone number) is sufficient for V2. Cryptographic node IDs are not required when an admission gate exists. → [ADR-001](../decisions/ADR-001-coordination-architecture.md)

**Q03-2** Is pure P2P sibling broadcast feasible at low peer counts?
Answer: No. Does not work reliably below ~100 peers. Hybrid microservice model is required at launch. → [ADR-001](../decisions/ADR-001-coordination-architecture.md)

**Q03-3** Has a better P2P coordination method emerged since Kademlia/S/Kademlia?
Answer: libp2p (with QUIC v1) is the current production standard across IPFS, Filecoin, and Ethereum devp2p. → [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md)

**Q04-1** Which P2P transfer protocol — BitSwap, libp2p, QUIC, or other?
Answer: libp2p framework with QUIC v1 (RFC 9000) as primary transport. TCP + Noise XX + yamux as fallback. BitSwap is not used — direct escrow-motivated transfer replaces it. Three-tier NAT traversal: AutoNAT → DCUtR → Circuit Relay v2. → [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md)

**Q04-2** Can the Merkle DAG links array store chunk-to-provider mappings?
Answer: No. Mixing DHT lookup metadata with Merkle tree structure creates confusion. Provider lookup belongs in Kademlia.

---

## Erasure Coding and Repair (Papers 05, 06, 10)

**Q05-2** At what MTTF does the model become viable given repair bandwidth?
Answer: MTTF ≥ 3 months for a 100 Kbps background budget. Below 3 months, repair costs exceed the budget. → [ADR-003](../decisions/ADR-003-erasure-coding.md)

**Q05-6** Can providers still serve data if the coordination microservice has downtime?
Answer: Yes. Group A (P2P only, always works): chunk retrieval via pointer file, direct provider-to-provider transfer. Group B (degrades gracefully): audit challenges queue, reliability score updates batch on recovery, DHT records begin expiring after 24h. Group C (blocks until recovery): new uploads, provider registration, payment release, repair job queuing. No data is ever lost during a microservice outage.

**Q06-1** At what MTTF does repair bandwidth exceed 30% of total system bandwidth?
Answer: MTTF=1mo → 310 KB/s; MTTF=3mo → 103 KB/s; MTTF=6mo → 52 KB/s. Formula: BW/node = 2 × (space per node) / MTTF (Blake & Rodrigues).

**Q06-2** What is the minimum MTTF the escrow model must enforce?
Answer: Repair cost per churn event must not exceed the marginal revenue from that provider. Low-MTTF entry is unprofitable for existing nodes, creating a self-enforcing floor. → [ADR-005](../decisions/ADR-005-peer-selection.md)

**Q07-1** Lightest PoS that resists just-in-time retrieval on a 5 Mbps provider?
Answer: Response deadline = `(chunk_size / declared_upload_speed) × 1.5`. Challenge timing randomised via `HMAC(server_secret, chunk_id + server_ts)`. → [ADR-002](../decisions/ADR-002-proof-of-storage.md), [ADR-014](../decisions/ADR-014-adversarial-defences.md)

**Q07-2** Does the Honest Geppetto attack have a practical mitigation?
Answer: Placement constraint at write time: no correlated group (same ASN/subnet) holds more than 20% of shards for one file. Enforced at assignment time, not retrospectively. → [ADR-014](../decisions/ADR-014-adversarial-defences.md)

**Q07-3** What replaces blockchain as the neutral audit trail?
Answer: Write-once append-only audit log. Provider receipt signed with Ed25519, microservice countersignature over provider_sig + timestamp. Stored in append-only Postgres (INSERT only). V3 upgrade: Transparent Merkle Log. → [ADR-015](../decisions/ADR-015-audit-trail.md)

**Q07-5** How to pseudonymise chunk IDs in the DHT without breaking FIND_VALUE?
Answer: DHT lookup key = `HMAC-SHA256(chunk_hash, file_owner_key)` where `file_owner_key = HKDF(master_secret, "vyomanaut-dht-v1", file_id)`. The DHT never sees chunk_hash or file_id. → [ADR-001](../decisions/ADR-001-coordination-architecture.md)

**Q08-2** What polling interval t separates a sleeping provider from a dead one?
Answer: Sweet spot 12–24h. ADR-006 starts at t=24h. Minimum safe value ≥ 8h (diurnal absence window). → [ADR-006](../decisions/ADR-006-polling-interval.md)

**Q09-1** Correct repair trigger threshold for desktop providers?
Answer: 72 hours. Safely exceeds Bolosky's weekend absence peak (99.7% of absences resolve within 70h). → [ADR-006](../decisions/ADR-006-polling-interval.md), [ADR-007](../decisions/ADR-007-provider-exit-states.md)

**Q09-2** How does the provider tier model change erasure coding parameters?
Answer: Desktop-only V2 collapses the tier model. Uniform parameter set applies to all providers. → [ADR-010](../decisions/ADR-010-desktop-only-v2.md), [ADR-003](../decisions/ADR-003-erasure-coding.md)

**Q09-3** Can convergent encryption be safely used for deduplication?
Answer: Deduplication is not a requirement. Each AONT key K is fresh random per segment. → [ADR-022](../decisions/ADR-022-encryption-erasure-order.md)

**Q10-1** What is the repair bandwidth spike Qpeek when a provider fails?
Answer: For N=1000, s=16, r=40, r0=8: Qpeek ≈ 793 GB per failure event. At 100 Kbps per peer, repair completes in ~8 hours — within the 12-hour safety window.

**Q10-2** What is the optimal lazy repair strategy for a mixed-tier network?
Answer: Single r0=8 for all providers (desktop-only V2 collapses the tier model). → [ADR-004](../decisions/ADR-004-repair-protocol.md)

**Q10-3** Desktop-only BWavg at current parameters?
Answer: BWavg ≈ 13.7 KB/s per peer at MTTF=300 days (Giroire Formula 1). Well within the 100 KB/s background budget.

**Q10-4** How does splitting alpha into temporary + permanent components change optimal r?
Answer: Conservative assumption (treating all departures as permanent) safely overestimates repair bandwidth. r=40 is robust to any alpha split. No change to ADR-003 warranted. → [ADR-003](../decisions/ADR-003-erasure-coding.md)

**Q10-5** At small network size, which providers absorb uneven repair load?
Answer: Repair assignment uses the identical Power of Two Choices algorithm as normal chunk assignment (ADR-005), weighted by provider reliability score and free capacity. No special-case repair logic is required. Giroire BWavg = 39 Kbps/peer is far enough below the 100 Kbps budget that even 3× variance at launch is safe. → [ADR-004](../decisions/ADR-004-repair-protocol.md), [ADR-005](../decisions/ADR-005-peer-selection.md)

**Q11-1** How to implement the PN-counter CRDT for escrow balance without losing auditability?
Answer: Append-only `escrow_events` table with idempotency key. Balance = SUM(DEPOSIT) − SUM(RELEASE, SEIZURE) per provider_id. No UPDATE ever. → [ADR-016](../decisions/ADR-016-payment-db-schema.md)

**Q11-2** What is the exact DB schema for the audit receipt table?
Answer: 12-field schema with Ed25519 dual signatures. INSERT-only enforced via Postgres row security. → [ADR-017](../decisions/ADR-017-audit-receipt-schema.md)

**Q11-3** How to apply I-confluence analysis to a new operation?
Answer: Five steps: (1) state invariant explicitly, (2) ask merge question, (3) check Bailis Table 2, (4) scope coordination to minimum, (5) document in PR.

**Q11-4** Does Postgres serialisable isolation on the payment service become a bottleneck?
Answer: No. Peak throughput at 10,000 providers ≈ 3 releases/sec. Single-server coordination cost is 0.19% of cycles per Bailis.

**Q11-5** At what provider count does the chunk assignment service become a bottleneck?
Answer: At 1,000 uploads/day the assignment service handles ~0.01 ops/sec. Not a bottleneck until V3+ scale.

**Q11-6** Can capability token validation be made I-confluent?
Answer: No. Token expiry is a time-based recency guarantee. Removing expiry defeats the security purpose. Accept the coordination cost.

---

## NAT Traversal, Transport, Encryption (Papers 13–18)

**Q12-2** Should hinted handoff be implemented for the (3,2,2) microservice quorum?
Answer: No, for V2. The (3,2,2) quorum absorbs single-replica failures without hinted handoff — if replica A is down during a write, replicas B and C both ACK (W=2 satisfied). Anti-entropy gossip reconciles A's state within seconds of its return. Hinted handoff only adds value if two replicas fail simultaneously, which breaks quorum regardless and requires operator intervention. → [ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md)

**Q13-1** Does Circuit Relay v2 violate the audit response deadline for symmetric-NAT providers?
Answer: No. Paper 30 (Trautwein et al., 4.4M traversal attempts) shows relay RTT is well below 500 ms in 90%+ of cases. Vyomanaut-operated relay nodes in Indian cloud regions will have consistently < 50 ms relay RTT to Indian home providers. The 1.5× deadline multiplier provides sufficient margin. → [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md)

**Q14-2** What is the actual p99 latency cost of forcing 1-RTT for audit reconnects?
Answer: 1-RTT reconnect adds 5–90 ms depending on geography and NAT type (5ms same-city, up to 40ms cross-city, +50ms Circuit Relay overhead for symmetric-NAT providers). The worst case is 90ms, well within the 614ms audit deadline for a 5 Mbps provider. 0-RTT remains disabled for audit interactions. → [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md)

**Q15-2** Does code-then-encrypt with per-chunk keys offer meaningful additional security over AONT-RS?
Answer: No. Code-then-encrypt with 56 keys per file offers no meaningful security improvement over AONT-RS. AONT-RS achieves computational security 2^256 with zero external key management at storage overhead ≈ Rabin. → [ADR-022](../decisions/ADR-022-encryption-erasure-order.md)

**Q16-1** What is AONT encoding throughput on minimum-spec hardware without AES-NI?
Answer: Benchmarking protocol specified in `docs/research/benchmarking-protocol.md`. ChaCha20 achieves ~186 ms per 14 MB segment at 75 MB/s without AES-NI — within the ≤5% CPU budget. → [ADR-019](../decisions/ADR-019-client-side-encryption.md)

**Q17-1** What fraction of V2 target providers lack AES-NI?
Answer: Estimated 85–90% have AES-NI. Both cipher paths must be production-quality. CPUID detection at daemon startup determines which path is used. Benchmarking protocol in `docs/research/benchmarking-protocol.md`. → [ADR-019](../decisions/ADR-019-client-side-encryption.md)

**Q18-1** What Argon2id parameters are correct for minimum-spec Indian desktop hardware?
Answer: Starting values t=3, m=64MB, p=4. Fallback: t=2, m=32MB if median > 500ms. Benchmarking protocol in `docs/research/benchmarking-protocol.md`. → [ADR-020](../decisions/ADR-020-key-management.md)

**Q18-2** Should the Ed25519 signing key be derived from master_secret or stored separately?
Answer: Store separately, encrypted under a key derived from master_secret. A compromised signing key can then be rotated without rotating master_secret and re-encrypting all data. Implementation spec in `docs/research/benchmarking-protocol.md`. → [ADR-020](../decisions/ADR-020-key-management.md)

**Q28-1** What initial RTO value should be used for a new provider with no response-time history?
Answer: RTO_init = 2000 ms for brand-new providers. After 5 audit responses, switch to AVG + 4×VAR (TCP-style, Rhea et al.). Store `(avg_rtt_ms FLOAT, var_rtt_ms FLOAT, sample_count INT)` per provider in the reliability scoring DB. Initial pool-derived default: avg=500ms, var=100ms (RTO = 900ms once the first 5 samples exist). → [ADR-006](../decisions/ADR-006-polling-interval.md)

**Q28-2** Should libp2p Kademlia be configured for periodic global sampling for proximity optimisation?
Answer: No, for V2. V2 is India-only; all providers are within ~3,000 km. Inter-city latency variance (5–40ms) is dominated by NAT traversal overhead (50ms for relay-dependent providers). A 24% reduction on 20ms DHT hops saves ~5ms — invisible against relay overhead. Enable if V2 expands internationally or if measured median DHT hop latency exceeds 80ms at V2 launch. → [ADR-001](../decisions/ADR-001-coordination-architecture.md)

**Q30-1** How many relay nodes are required at V2 launch?
Answer: 3 relay nodes, one per Indian cloud availability zone (Mumbai AZ1, Mumbai AZ2, Chennai/Hyderabad). Sizing: 30% of providers require relay permanently (Paper 30). At 300 initial providers, that is 90 relay-dependent peers. Three nodes at 128 concurrent reservations each gives 384 slots — 4.3× headroom. Scale to 6 nodes when provider count exceeds 1,500 (relay utilisation exceeds 60%). → [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md)

---

## Erasure Code Selection, Storage Engine, Disk Reliability (Papers 19, 25–27)

**Q19-2** What is the sub-packetisation level for Clay/MSR codes at (n=56, k=16)?
Answer: Paper 22 (Goparaju): α = ρ^(k·C(r,ρ)). For our parameters at d=n-1: α = 40^16 ≈ 10^25. Computationally intractable. MSR and Clay codes are not feasible at (n=56, k=16). Hitchhiker codes (α=2, two sub-stripes) are the only viable V3 candidate. → [ADR-026](../decisions/ADR-026-repair-bw-optimisation.md)

**Q20-2** Should DHT record republication interval be reduced to 12h, or should record expiry extend to 48h?
Answer: Reduce republication to 12h. Keep record expiry at 24h. This provides the same 12h buffer as the IPFS production design. → [ADR-006](../decisions/ADR-006-polling-interval.md)

**Q25-1** Is proactive continuous disk scrubbing justified?
Answer: No. Paper 32 (Schroeder et al.) shows the base UE rate is 2–6 per 1,000 drive days. Scrubbing should be reactive: triggered by the first audit FAIL. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q25-2** Is sparse non-contiguous vLog placement required to reduce within-provider burst correlation?
Answer: Not for V2. Sequential vLog layout is acceptable — a chip failure corrupting a contiguous vLog region damages shards from many different files, each retaining 55 surviving shards. Revisit in V3. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q26-1** Is the chunk index in the hot, warm, or cold data regime per the Five Minute Rule?
Answer: The question is moot at 256 KB values. WiscKey eliminates value movement from compaction — write amplification ≈ 1.0 regardless of index access regime. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q26-2** What fraction of Indian desktop providers use SSDs vs HDDs, and does it change the storage engine design?
Answer: WiscKey is correct for both. vLog appends are sequential (HDD-optimal). Single-thread random vLog reads at 256 KB complete in ~1 ms SSD / ~12–15 ms HDD — both within the 614ms audit deadline. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q27-1** Should the vLog use fixed or variable entry size?
Answer: Fixed entry size (262,212 bytes = 256 KB chunk + headers). GC tail advancement requires only arithmetic, no entry parsing. Rate limiter calibration protocol in `benchmarking-protocol.md`. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q35-1** How should the Razorpay Route on_hold_until timestamp be set to target "first 3 business days" release?
Answer: Embed a static `rbi_bank_holidays_YYYY` lookup table in the microservice, updated each December. The monthly release job runs on the 23rd of each month: (1) find the last calendar day of the current month that is not Saturday, Sunday, or in the RBI holiday set, (2) set `on_hold_until` = that date at 23:59:59 IST, (3) Route releases on the next business day, landing within the first 1–3 days of the following month. Update the holiday table as part of the December release deployment each year. → [ADR-024](../decisions/ADR-024-economic-mechanism.md)

**Q38-2** At what correlated failure rate does RS(16,56) become worse than a simpler scheme?
Answer: Never, under the 20% ASN cap. Paper 38's reversal condition requires the correlated failure size to approach the parity count (r=40). With the 20% ASN cap, the maximum correlated failure is bounded at 12 shards (ceil(0.20 × 56)). Surviving shards after the worst-case event: 44 — which is 28 above the s=16 reconstruction floor. This margin makes the reversal categorically impossible at our parameters. → [ADR-003](../decisions/ADR-003-erasure-coding.md), [ADR-014](../decisions/ADR-014-adversarial-defences.md)

**Q41-1** How should the service-denial attack be monitored and scored?
Answer: Three layers. (1) Structural mitigation already in place: RS(16,56) requires > 40 simultaneous refusals for effective denial; the 20% ASN cap limits any correlated group to ~11 — categorically insufficient for a network-level attack. (2) Scoring signal: data owners may submit a `retrieval_failure_report` containing `{chunk_id, provider_id, attempt_ts, libp2p_error_code, data_owner_sig}`. Reports lacking a libp2p error code are rejected. Score impact applies only when 3 independent data owners report failure from the same provider within a rolling 72h window. Scoring weight: 0.3× of an audit FAIL in the 24h window. (3) V3 upgrade: repair-event interactions provide microservice-visible retrieval evidence, eliminating the need for data owner reports. → [ADR-014](../decisions/ADR-014-adversarial-defences.md), [ADR-017](../decisions/ADR-017-audit-receipt-schema.md), [ADR-008](../decisions/ADR-008-reliability-scoring.md)

**Q41-2** At what provider count does daily full-audit become infeasible?
Answer: Full daily audit scales to approximately 100,000 providers × 10,000 chunks before Postgres INSERT throughput (≈5,000–10,000 rows/sec on an optimised single instance) becomes the bottleneck. At N=10,000 providers × 10,000 chunks, the challenge rate is 1,157/sec — well within capacity. At 100,000 × 10,000 = 1B challenges/day = 11,574/sec, sharding or probabilistic sampling becomes necessary. SHELBY Theorem 1 Condition (i) must be re-verified against the new pau value if probabilistic sampling is introduced. → [ADR-002](../decisions/ADR-002-proof-of-storage.md)

---

## Economic Mechanism (Papers 29, 31, 33, 35, 37, 40, 41)

**Q39-1 (method resolved; value gate set)** — Combined Hitchhiker + lazy repair adoption decision
Answer: The analytical framework is established. Combined BWavg = Giroire BWavg × (1 − Hitchhiker_reduction) ≈ 39 × 0.65 ≈ 25 Kbps/peer. The decision gate for Hitchhiker adoption in V3 is: if observed V2 BWavg exceeds 60 Kbps/peer (54% above Giroire prediction) over the first 6 months, implement Hitchhiker. If observed ≤ 60 Kbps/peer, implementation complexity is not justified. → [ADR-026](../decisions/ADR-026-repair-bw-optimisation.md)