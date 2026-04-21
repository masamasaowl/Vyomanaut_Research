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
Answer: Yes. Full behaviour documented in open-questions.md. Group A (P2P only, always works): chunk retrieval via pointer file, direct provider-to-provider transfer. Group B (degrades gracefully): audit challenges queue, reliability score updates batch on recovery, DHT records begin expiring after 24h. Group C (blocks until recovery): new uploads, provider registration, payment release, repair job queuing. No data is ever lost during a microservice outage because all 56 RS shards remain on provider disks and the 40-fragment redundancy buffer absorbs the gap.

**Q06-1** At what MTTF does repair bandwidth exceed 30% of total system bandwidth?
Answer: MTTF=1mo → 310 KB/s; MTTF=3mo → 103 KB/s; MTTF=6mo → 52 KB/s. Formula from Blake & Rodrigues: BW/node = 2 × (space per node) / MTTF.

**Q06-2** What is the minimum MTTF the escrow model must enforce?
Answer: Repair cost per churn event must not exceed the marginal revenue from that provider. Low-MTTF entry is unprofitable for existing nodes, creating a self-enforcing floor. → [ADR-005](../decisions/ADR-005-peer-selection.md)

**Q07-1** Lightest PoS that resists just-in-time retrieval on a 5 Mbps provider?
Answer: Response deadline = `(chunk_size / declared_upload_speed) × 1.5`. Challenge timing is randomised (nonce = HMAC(server_secret, chunk_id + server_ts)). → [ADR-002](../decisions/ADR-002-proof-of-storage.md), [ADR-014](../decisions/ADR-014-adversarial-defences.md)

**Q07-2** Does the Honest Geppetto attack have a practical mitigation?
Answer: Placement constraint at write time: no correlated group (same ASN/subnet) holds more than 20% of shards for one file. Enforced at assignment time, not retrospectively. → [ADR-014](../decisions/ADR-014-adversarial-defences.md)

**Q07-3** What replaces blockchain as the neutral audit trail?
Answer: Write-once append-only audit log. Provider receipt signed with Ed25519, microservice countersignature over provider_sig + timestamp. Stored in append-only Postgres (INSERT only). V3 upgrade path: Transparent Merkle Log. → [ADR-015](../decisions/ADR-015-audit-trail.md)

**Q07-5** How to pseudonymise chunk IDs in the DHT without breaking FIND_VALUE?
Answer: DHT lookup key = `HMAC-SHA256(chunk_hash, file_owner_key)` where `file_owner_key = HKDF(master_secret, "vyomanaut-dht-v1", file_id)`. The DHT never sees chunk_hash or file_id. The availability service holds dht_keys as opaque bytes and republishes without the file_owner_key. Full four-phase flow (upload, retrieval, republication, audit) documented in open-questions.md.

**Q08-2** What polling interval t separates a sleeping provider from a dead one?
Answer: Sweet spot 12–24 h. ADR-006 starts at t=24h. Minimum safe value ≥ 8h (diurnal absence window per Bhagwan). → [ADR-006](../decisions/ADR-006-polling-interval.md)

**Q09-1** Correct repair trigger threshold for desktop providers?
Answer: 72 hours. Safely exceeds Bolosky's weekend absence peak (99.7% of weekend absences resolve within 70h). → [ADR-006](../decisions/ADR-006-polling-interval.md), [ADR-007](../decisions/ADR-007-provider-exit-states.md)

**Q09-2** How does the provider tier model change erasure coding parameters?
Answer: Desktop-only V2 collapses the tier model. Uniform parameter set applies to all providers. → [ADR-010](../decisions/ADR-010-desktop-only-v2.md), [ADR-003](../decisions/ADR-003-erasure-coding.md)

**Q09-3** Can convergent encryption be safely used for deduplication?
Answer: No need to explore this. Deduplication is not a requirement. Data owners pay for all their content. Each AONT key K is fresh random per segment. → [ADR-022](../decisions/ADR-022-encryption-erasure-order.md)

**Q10-1** What is the repair bandwidth spike Qpeek when a provider fails?
Answer: For N=1000, s=16, r=40, r0=8: Qpeek ≈ 793 GB per failure event. At 100 Kbps per peer, repair completes in ~8 hours — within the 12-hour safety window.

**Q10-2** What is the optimal lazy repair strategy for a mixed-tier network?
Answer: Single r0=8 for all providers (desktop-only V2 collapses the tier model). Per-tier r0 is a V3 consideration. → [ADR-004](../decisions/ADR-004-repair-protocol.md)

**Q10-3** Desktop-only BWavg at current parameters?
Answer: BWavg ≈ 13.7 KB/s per peer at MTTF=300 days (Giroire Eq. 2, α=1/7200 per hour). Well within the 100 KB/s background budget.

**Q10-4** How does splitting alpha into temporary + permanent departures change optimal r?
Answer: Conservative assumption (treating all departures as permanent) safely overestimates repair bandwidth. The resulting r=40 is robust to any alpha split. No change to ADR-003 is warranted. → [ADR-003](../decisions/ADR-003-erasure-coding.md)

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
Answer: No. Token expiry is a time-based recency guarantee (Bailis Section 5.3). Removing expiry defeats the security purpose. Accept the coordination cost.

---

## NAT Traversal, Transport, Encryption (Papers 13–18)

**Q13-1** Does Circuit Relay v2 violate the audit response deadline for symmetric-NAT providers?
Answer: No. Paper 30 (Trautwein et al., 4.4M traversal attempts) shows relay RTT in production is well below 500 ms in 90%+ of cases. Vyomanaut-operated relay nodes in Indian cloud regions will have consistently < 50 ms relay RTT to Indian home providers. The 1.5× deadline multiplier provides sufficient margin. → [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md), [ADR-002](../decisions/ADR-002-proof-of-storage.md)

**Q15-2** Does code-then-encrypt with per-chunk keys offer meaningful additional security over AONT-RS?
Answer: No. Code-then-encrypt with 56 keys per file offers no meaningful security improvement over AONT-RS. AONT-RS achieves computational security 2^256 with zero external key management at storage overhead ≈ Rabin. → [ADR-022](../decisions/ADR-022-encryption-erasure-order.md)

**Q16-1** What is AONT encoding throughput on minimum-spec hardware without AES-NI?
Answer: Benchmarking protocol specified in `docs/research/benchmarking-protocol.md`. ADR-019 targets ≤200 ms per 14 MB segment on OMAP-class hardware. ChaCha20 achieves ~186 ms at 75 MB/s without AES-NI. → [ADR-019](../decisions/ADR-019-client-side-encryption.md)

**Q17-1** What fraction of V2 target providers lack AES-NI?
Answer: Estimated 85–90% have AES-NI (Intel Core i3+ post-2011, AMD Ryzen all generations). Remaining 10–15% are very old or very budget machines. Both cipher paths must be production-quality. CPUID detection at daemon startup determines which path is used. Benchmarking protocol in `docs/research/benchmarking-protocol.md`. → [ADR-019](../decisions/ADR-019-client-side-encryption.md)

**Q18-1** What Argon2id parameters are correct for minimum-spec Indian desktop hardware?
Answer: Starting values t=3, m=64MB, p=4. Benchmarking protocol specified in `docs/research/benchmarking-protocol.md`. Fallback: t=2, m=32MB if median > 500 ms on target hardware. → [ADR-020](../decisions/ADR-020-key-management.md)

**Q18-2** Should the Ed25519 signing key be derived from master_secret or stored separately?
Answer: Store separately, encrypted under a key derived from master_secret. Reasoning: a compromised signing key can be rotated without rotating master_secret and re-encrypting all data. Implementation spec in `docs/research/benchmarking-protocol.md`. → [ADR-020](../decisions/ADR-020-key-management.md)

---

## Erasure Code Selection, Storage Engine, Disk Reliability (Papers 19, 25–27)

**Q19-2** What is the sub-packetisation level for Clay/MSR codes at (n=56, k=16)?
Answer: Paper 22 (Goparaju): α = ρ^(k·C(r,ρ)). For our parameters at d=n-1: ρ=40, C(40,40)=1, α = 40^16 ≈ 10^25. Computationally intractable. MSR and Clay codes are not feasible at (n=56, k=16) for any d. Hitchhiker codes (α=2, two sub-stripes) are the only viable V3 candidate. → [ADR-026](../decisions/ADR-026-repair-bw-optimisation.md)

**Q20-2** Should DHT record republication interval be reduced to 12h, or should record expiry extend to 48h?
Answer: Reduce republication to 12h. Keep record expiry at 24h. This provides the same 12h buffer as the IPFS production design (confirmed by Paper 20). → [ADR-006](../decisions/ADR-006-polling-interval.md)

**Q25-1** Is proactive continuous disk scrubbing justified?
Answer: No. Paper 32 (Schroeder et al., Google fleet study) shows the base UE rate is 2–6 per 1,000 drive days — low enough that 24h PoR audit challenges are sufficient detection. Scrubbing should be reactive: triggered by the first audit FAIL (which indicates a 30× elevated risk of further failures), not run continuously on all providers at all times. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q25-2** Is sparse non-contiguous vLog placement required to reduce within-provider burst correlation?
Answer: Not for V2. Paper 32 shows bad block clustering operates at chip granularity (hundreds of contiguous blocks). Since Vyomanaut holds only 1 of 56 shards per file per provider, a chip failure corrupting a contiguous vLog region damages shards from many different files, each of which retains 55 surviving shards across independent providers. Sequential vLog layout is acceptable. Revisit in V3 if burst failures exceed the RS redundancy budget. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q26-1** Is the chunk index in the hot, warm, or cold data regime per the Five Minute Rule?
Answer: The question is moot at 256 KB values. WiscKey eliminates value movement from compaction entirely — write amplification ≈ 1.0 regardless of index access regime. The 44-byte index entries have negligible amplification. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q26-2** What fraction of Indian desktop providers use SSDs vs HDDs, and does it change the storage engine design?
Answer: WiscKey is correct for both. vLog appends are sequential (HDD-optimal). Single-thread random vLog reads at 256 KB complete in ~1 ms on SSD and ~12–15 ms on HDD — both well within the 614 ms audit deadline. SSD/HDD mix does not change the design. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)

**Q27-1** Should the vLog use fixed or variable entry size?
Answer: Fixed entry size (262,212 bytes = 256 KB chunk + headers). Makes GC deterministic — tail advancement requires only arithmetic, no entry parsing. See benchmarking-protocol.md for rate limiter calibration protocol. → [ADR-023](../decisions/ADR-023-provider-storage-engine.md)