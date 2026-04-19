# Open Questions

Live tracker. Every question lives here once. Add new questions here; reference this file from paper summaries.

Status values: `open` · `answered` · `deferred-v3` · `rejected`

> **How to use this file:**
> Before adding a question from a new paper, search here first. If it already exists, update the blocked-on reference. If a paper answers a question, update the status and add the answer.

---

## From Paper 01 — BitTorrent

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q01-1 | How to avoid the tracker — it is efficient but vulnerable? | answered | Kademlia (Paper 02) |
| Q01-2 | Should requests be kept lined up so TCP is always loaded? | answered | Yes — pipelining improves throughput; confirmed in #12 research |
| Q01-3a | What criteria decide the reliability of a peer? | answered | Storj 4-subsystem pipeline (Paper 07) — [ADR-005](../decisions/ADR-005-peer-selection.md) |
| Q01-3b | How to decide the initial reliability of a new peer? | answered | Random optimistic assignment — confirmed Paper 07 |
| Q01-3c | How to re-rank peers based on reliability over time? | answered | 3-window rolling score (Paper 08) — [ADR-008](../decisions/ADR-008-reliability-scoring.md) |
| Q01-4 | Is geographic proximity a viable assignment parameter? | open | Phase 2A — Coral DSHT (reading-list Phase 2A #11) |
| Q01-5 | Does reliability-proportional storage assignment create Matthew effect (runaway concentration)? | open | Prototype telemetry after V2 launch |
| Q01-6 | Should providers with higher speed, uptime, and storage be proportionally rewarded by the escrow model? | open | Phase 5 — Economic mechanism design (#18) |

---

## From Paper 02 — Kademlia

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q02-1 | Edge cases where lookup doesn't converge — how to handle? | answered | S/Kademlia sibling nodes fix it (Paper 03) |
| Q02-2 | How to decide replication parameter k? | answered | S/Kademlia sets k=8,16 (Paper 03) |
| Q02-3 | How to set branching factor b (accelerated lookups)? | answered | libp2p Kademlia: k=20 bucket size (configure to k=16 per ADR-001), alpha=3 parallel queries. Binary XOR tree has branching factor 2; alpha=3 concurrency gives O(log n / 3) round trips in practice. |

| Q02-4 | Should the data owner also be part of the DHT as a node with no stored value? | answered | No — tracker handles data owner; DHT handles provider swarm only |
| Q02-5 | How to implement republication of key-value pairs in k-buckets efficiently? | answered | One node republishes per hour; availability service uses same logic |

---

## From Paper 03 — S/Kademlia

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q03-1 | Should node IDs be hashed from a public key, or is registration gating sufficient? | answered | Registration gating chosen — [ADR-001](../decisions/ADR-001-coordination-architecture.md) |
| Q03-2 | Is pure P2P sibling broadcast feasible at low peer counts? | answered | No — doesn't work below ~100 peers; hybrid required |
| Q03-3 | Has a better P2P coordination method emerged since Kademlia (2003) / S/Kademlia (2007)? | answered | libp2p is the current standard in large P2P networks |

---

## From Paper 04 — IPFS

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q04-1 | Which P2P replication protocol: BitSwap, libp2p, QUIC, or alternative? | answered | Framework = libp2p. Transport = QUIC v1 (RFC 9000) primary; TCP + Noise + yamux fallback. BitSwap is not used — direct escrow-motivated transfer replaces it. NAT traversal: AutoNAT → DCUtR → Circuit Relay v2. ADR-021 accepted. |
| Q04-2 | Can the Merkle DAG links array store chunk-to-provider mappings for redundant siblings? | answered | No — Kademlia handles provider lookup; mixing use-cases creates confusion |
| Q04-3 | How to use IPNS for file routing if the data owner is not part of the network? | answered | Microservice proxy signatures can resolve DHT republication (needs implementation design) |

---

## From Paper 05 — Storj

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q05-1 | What challenges or drawbacks does a central Satellite pose? Can all its tasks be decentralised? | open | SoK (Paper 07) partially answers; full answer requires prototype |
| Q05-2 | At what MTTF does the model become viable, given repair bandwidth as the biggest constraint? | answered | MTTF ≥ 3 months for 100 Kbps background budget — Blake & Rodrigues formula (Paper 06) |
| Q05-3 | At what mobile MTTF does the business model break (repair bandwidth costs exceed revenue)? | deferred-v3 | Mobile deferred to V3 by [ADR-010](../decisions/ADR-010-desktop-only-v2.md). The crossover calculation uses Blake & Rodrigues formula. Revisit when mobile is introduced. |
| Q05-4 | What is the minimum vetting period and held-earnings percentage that deters provider exit while keeping providers economically engaged? | open | Blocked on: escrow research papers + economic modelling. Not answerable without a payment model paper. |
| Q05-5 | What is the minimum response quorum for a PoR batch audit to be statistically valid? (Originally: minimum k' for Berlekamp-Welch — reframed for PoR since we chose Merkle challenges.) | open | Blocked on: reading-list Phase 1 #10 (Szabó) and Phase 2A #15 (Dimakis). |
| Q05-6 | Storj v3.1 dropped Kademlia entirely. ADR-001 keeps the hybrid DHT. Can providers still serve data to each other if our coordination microservice has downtime? | open | Critical resilience test of [ADR-001](../decisions/ADR-001-coordination-architecture.md). The P2P data plane must serve stored data via DHT without the microservice. Needs explicit design: which operations require the microservice and which fall back to DHT-only. |
| Q05-7 | With n=80 node IDs at 64 bytes each, the pointer per segment is 5+ KB of metadata. At what file size does per-segment metadata overhead become unacceptable, and should small files be inlined? | open | Blocked on: own file size distribution telemetry at launch. A pilot survey of upload patterns is needed before this can be decided. |
| Q05-8 | Should we adopt per-segment pricing (not just per-byte) to cover metadata overhead, and how does this interact with our fiat escrow model? | open | Blocked on: reading-list Phase 1 Filecoin whitepaper (#11) + economic mechanism research (#18) |

---

## From Paper 06 — Blake & Rodrigues

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q06-1 | At what provider MTTF does repair bandwidth exceed 30% of total system bandwidth? | answered | MTTF=1mo → 310 KB/s; MTTF=3mo → 103 KB/s; MTTF=6mo → 52 KB/s (see paper-06 for formula) |
| Q06-2 | What is the minimum provider MTTF the escrow model must enforce to remain economically viable? | answered | Repair cost per churn event must not exceed provider revenue; low-MTTF provider entry is unprofitable for existing nodes |
| Q06-3 | What value of polling timeout t should the system use? | open | 8–24 h range established; finalise after prototype — [ADR-006](../decisions/ADR-006-polling-interval.md) |

---

## From Paper 07 — SoK: DSN

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q07-1 | What is the lightest proof-of-storage that resists just-in-time retrieval on a 5 Mbps mobile provider? | answered | Response deadline = (chunk_size / declared_upload_speed) × 1.5; randomise challenge timing — [ADR-002](../decisions/ADR-002-proof-of-storage.md) |
| Q07-2 | Does the Honest Geppetto attack have a practical mitigation without centrally watching all nodes? | answered | Placement constraint at write time: no cluster holds >20% of shards for one file — [ADR-014](../decisions/ADR-014-adversarial-defences.md) |
| Q07-3 | What replaces blockchain as the neutral audit trail both parties trust? | answered | Write-once audit log + signed receipts — [ADR-015](../decisions/ADR-015-audit-trail.md) |
| Q07-4 | Can correlated failure detection be built into the reliability scorer without a central coordinator seeing all audit results simultaneously? | open | Blocked on: reading-list Phase 2B #17 (EigenTrust paper). EigenTrust distributes reputation computation — check if it can propagate correlation signals without centralisation. |
| Q07-5 | How do we pseudonymise chunk IDs in the DHT to prevent a monitoring node from reconstructing which files a client is accessing, while still allowing Kademlia FIND_VALUE to function? | open | Candidate: derive DHT key as `HMAC(chunk_hash, file_owner_key)` so only the file owner can map DHT key → chunk. Blocked on: reading-list Phase 1 #10 (encryption and key management papers). Lookup overhead unknown. |

---

## From Paper 08 — Bhagwan (Availability)

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q08-1 | What is the actual MTTF of a financially-incentivised desktop storage provider? | open | Own provider telemetry — must be measured empirically at launch. No paper answers this. |
| Q08-2 | What polling interval t separates a sleeping provider from a dead one? | answered | t sweet spot: 12–24 h; start at 24 h — confirmed by Blake & Rodrigues |
| Q08-3 | At what provider count does a ~20%/day turnover rate produce more repair bandwidth than the network's idle upload capacity can absorb? | open | Partially computable with Blake & Rodrigues formula: inputs are turnover rate × repair BW per event (chunk_size × parity_count). Blocked on: stable ADR-003 parameters and first network size estimates. |
| Q08-4 | Should polling interval t be adaptive — shorter for providers with a history of short absences, longer for reliably long sessions? | open | Blocked on: reading-list Phase 2B #17 (Reliability Scoring). Adaptive t would reduce repair bandwidth for predictably-absent providers without compromising departure detection for true exits. |
| Q08-5 | Can the reliability scorer distinguish a diurnally-absent provider from a permanently-departed one before t expires, without polling more frequently? | open | Core tension between bandwidth savings (long t) and early departure detection. Blocked on: #8 reliability scoring model research. Bhagwan's two-component model (daily churn + permanent departure) is the theoretical basis. |

---

## From Paper 09 — Bolosky (Feasibility)

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q09-1 | What is the correct repair trigger threshold for desktop vs mobile providers? | answered | Desktop: 72 h (Bolosky bimodal); mobile deferred to V3 — [ADR-006](../decisions/ADR-006-polling-interval.md) |
| Q09-2 | How does the provider tier model change the erasure coding parameters inherited from Storj? | answered | Desktop-only V2 collapses the tier model; uniform parameters apply — [ADR-010](../decisions/ADR-010-desktop-only-v2.md) |
| Q09-3 | Can convergent encryption be safely used for deduplication in a zero-knowledge system? | answered | Not needed — deduplication is not a requirement; data owner pays for all their content |
| Q09-4 | What is the actual free-space fraction on modern Indian smartphones in 2024, given photo/video storage dominance? | deferred-v3 | Mobile deferred to V3. Bolosky's 50% figure is from year-2000 corporate desktops. A pilot survey of 100 target-market phones is needed before mobile launch. |
| Q09-5 | Should the 72-hour departure threshold be mobile-tier-specific, given that mobile absence patterns don't include 64-hour weekend stretches? | deferred-v3 | Mobile deferred to V3. Mobile absence is driven more by OS background execution limits than by human behaviour. Revisit when reading-list Phase 1 #11 (Background OS Execution) is studied. |
| Q09-6 | What is the maximum safe lazy-update window for a mobile tier at 1-month MTTF? | deferred-v3 | Mobile deferred to V3. Window must be shorter than the expected time to propagate k copies across k independent mobile providers. Blocked on: #3 and #4 research at mobile-tier parameters. |

---

## From Paper 10 — Giroire (Lazy Repair)

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q10-1 | What is repair bandwidth spike Qpeek when a provider fails? | answered | For N=1000, s=16, r=40, r0=8: Qpeek=793 GB/peer; at 100 Kbps takes 8 h < 12 h window |
| Q10-2 | What is the optimal lazy repair strategy for a mixed-tier network — single or per-tier r0? | answered | V2 uses single r0=8 (desktop-only, uniform tier); per-tier approach deferred to V3 |
| Q10-3 | Desktop-only BWavg at current parameters? | answered | BWavg ≈ 13.7 KB/s per peer at MTTF=300 days (α=1/7200 per hour, Giroire Eq. 2) — well within 100 KB/s background budget |
| Q10-4 | How does the optimal r change when provider temporary absence (diurnal churn) is separated from permanent departure? ... | open → **partially resolved** | Saroiu (Paper 21) provides the unincentivized α_temporary floor (median session ~60 min in zero-incentive systems). In Vyomanaut's incentivized model, α_temporary is orders of magnitude lower, meaning the conservative assumption in Giroire (treating all departures as permanent) safely overestimates repair bandwidth. The optimal r=40 is therefore robust to the α split. Formally proving the split-α formula still requires Dimakis (Phase 2A #4). |
| Q10-5 | At launch (D << N² × lf), repair load is unevenly distributed. Which providers bear the excess repair burden, and how does this affect their BWavg? | open | Blocked on: [ADR-005](../decisions/ADR-005-peer-selection.md) and the repair assignment algorithm. Uneven block distribution at launch concentrates repair on a small set of high-reliability providers. |

---

## From Paper 11 — Bailis (Coordination Avoidance)

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q11-1 | How to implement the PN-counter CRDT for escrow balance without losing auditability? | answered | Append-only escrow_events table with idempotency key — [ADR-016](../decisions/ADR-016-payment-db-schema.md) |
| Q11-2 | What is the exact DB schema for the audit receipt table satisfying all I-confluence requirements? | answered | 12-field schema with Ed25519 dual signatures — [ADR-017](../decisions/ADR-017-audit-receipt-schema.md) |
| Q11-3 | How to apply I-confluence analysis to a new operation before adding it to the codebase? | answered | 5-step process: state invariant → ask merge question → check Table 2 → scope coordination → document in PR |
| Q11-4 | Under concurrent payment releases, does Postgres serialisable isolation on the payment service become a throughput bottleneck? | answered | No. Peak throughput at 10,000 providers ≈ 3 releases/sec. Paper 11 shows single-server coordination cost is 0.19% of cycles. Not a bottleneck in V2. |
| Q11-5 | At what provider count does the chunk assignment service (the one non-I-confluent write) become a bottleneck? | answered | At 1,000 uploads/day the assignment service handles ~0.01 ops/sec. Not a bottleneck until V3+ scale. |
| Q11-6 | Can capability token validation be made I-confluent by pre-computing all valid tokens at upload time? | answered | No. Token expiry is a time-based recency guarantee (Sec 5.3). Removing expiry defeats the security purpose. Accept the coordination cost. |

---

## From Paper 12 — Dynamo

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q12-1 | At what microservice cluster size does hand-rolled gossip membership (ADR-025) become harder to operate than managed consensus (etcd/Consul)? | open | Dynamo Section 6.6 notes that full-membership gossip works for hundreds of nodes but does not scale to tens of thousands. Our cluster is 3–5 replicas so gossip is appropriate in V2. The threshold to revisit is: if the cluster ever grows beyond ~10 replicas, evaluate etcd. Blocked on: production telemetry post-launch. |
| Q12-2 | Should hinted handoff be implemented for the coordination microservice metadata store when one replica is temporarily unavailable during a write? | open | Dynamo uses hinted handoff to preserve write availability during single-node outages — the write is sent to a substitute replica with a hint, then replayed to the intended replica on recovery. Our (3,2,2) quorum survives one failure without hinted handoff, so it is not strictly necessary in V2. Whether the added complexity is worth the durability improvement during rolling restarts is an operational decision. Blocked on: implementation phase and observed failure patterns in staging. |

---

## From Paper 13 — libp2p

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q13-1 | Does libp2p Circuit Relay v2 introduce enough additional round-trip latency to violate the audit response deadline for providers behind symmetric NAT? | open | The audit response deadline is (chunk_size / declared_upload_speed) × 1.5. At 256 KB and 5 Mbps, this is ~614 ms. A relay adds 50–200 ms depending on relay geographic distance. This is likely acceptable for India-first V2 deployment (Vyomanaut-operated relay nodes would be co-located in Indian cloud regions), but it must be measured. Blocked on: network measurement with relay nodes during pre-launch testing. If relay overhead consistently exceeds 300 ms, the deadline multiplier must be increased to 2.0 for relay-only providers, or such providers must be excluded from auditing. |
| Q13-2 | Should libp2p GossipSub be used for repair event propagation to surviving chunk holders, or should repair job orchestration remain exclusively in the microservice? | open | GossipSub would allow the surviving fragment holders to self-organise a repair without a central repair scheduler. This reduces microservice load during a burst failure event but introduces a new failure mode: if GossipSub mesh formation fails during the same network partition that caused the chunk loss, repair is silently delayed. The microservice-driven repair model (ADR-004) is deterministic and auditable; GossipSub adds complexity with unclear benefit for V2 scale. Blocked on: repair scheduler implementation in V2; revisit if microservice repair throughput becomes a bottleneck at V3 scale (>10,000 providers). |

---

## From Paper 14 — RFC 9000 (QUIC)

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q14-1 | What is the fallback when UDP is blocked at a provider's ISP or home router? Does libp2p handle TCP/WebTransport fallback transparently, and what fraction of Indian home ISPs block UDP in practice? | answered | libp2p automatically falls back to TCP + Noise XX + yamux when UDP is unreachable (Paper 13). The fallback is transparent — multiaddr dialling selects the best reachable transport. The UDP-block rate at Indian ISPs is unknown and must be measured at launch, but the fallback path is architecturally in place. |
| Q14-2 | Should 0-RTT be disabled for all chunk transfer connections to eliminate replay risk, and what is the p99 latency penalty for forcing 1-RTT on every reconnect from a returning provider? | open | Blocked on: prototype measurement. RFC 9000 Section 8.1 allows servers to reject 0-RTT data selectively. The latency cost depends on round-trip time to the microservice and the frequency of provider reconnects relative to audit interval. |
| Q14-3 | During a QUIC connection migration mid-audit-challenge, response_latency_ms in the audit receipt will spike due to path validation (PATH_CHALLENGE / PATH_RESPONSE). How do we distinguish a migration event from a JIT retrieval attempt? Should the audit receipt schema (ADR-017) include an optional migration_event flag, and if so, what proves the flag is honest? | open | Blocked on: libp2p spec (migration signalling) and empirical data from provider telemetry at launch. Migration events from stable desktop providers with static IPs should be rare enough that false-positive JIT detections are negligible in V2. Revisit if migration-flag abuse emerges. |

---

## From Paper 15 — Szabó et al. (Encrypt-Erasure Separation)

| ID | Question | Status | Answer / Blocked on |
|---|---|---|---|
| Q15-1 | If mobile providers are introduced in V3 and operator partnerships are unavailable, can client-side RS erasure coding on a mobile device fit within battery and CPU budgets at our parameters (s=16, r=40, lf=256 KB)? | open | Blocked on: mobile MTTF research (V3) and Phase 1 #11 (Background OS Execution). The paper shows proxy-based offloading reduces this cost by ~20% but requires ISP cooperation. |
| Q15-2 | The paper confirms encrypt-then-code preserves zero-knowledge. Does code-then-encrypt offer any meaningful additional security guarantee that would justify its higher key management cost (56 keys per file vs 1)? | answered | Code-then-encrypt with per-chunk keys offers no meaningful additional security over AONT-RS for our threat model. AONT-RS achieves computational security of 2^256 with zero external key management, at storage overhead ≈ Rabin. The threshold alone is sufficient. |

---

## From Paper 16 — AONT-RS

| ID    | Question | Status | Answer / Blocked on |
|-------|----------|--------|---------------------|
| Q16-1 | What is AONT encoding throughput on minimum-spec provider hardware without AES-NI (e.g., dual-core 1.8 GHz)? Does the ≤5% CPU budget (ADR-009) hold for 14 MB segments? | open | Blocked on: benchmark on target hardware before V2 launch. If AES-NI is absent and throughput drops below 10 MB/s, switch to ChaCha20-Poly1305 (RFC 8439). |
| Q16-2 | If the pointer file is the sole retrieval credential, should the pointer file itself be stored redundantly via a separate AONT-RS encoding (recursive), or via a threshold backup with trusted parties? What is the recovery path if the data owner loses their pointer file? | open | Blocked on: ADR-020 (pointer file management, deferred to Phase 3 after Tahoe-LAFS paper). |

---

## From Paper 17 — RFC 8439 (ChaCha20-Poly1305)

| ID    | Question | Status | Blocked on |
|-------|----------|--------|------------|
| Q17-1 | What fraction of V2 target providers (Indian home desktop, NAS) lack AES-NI? Is the 3× performance gap real for our specific hardware distribution, or is AES-NI ubiquitous enough that a single AES path is sufficient? | open | Provider telemetry at launch. If >20% of providers lack AES-NI, both paths must be maintained. If <5%, ChaCha20-only simplifies the codebase. |
| Q17-2 | Should the pointer file nonce counter be stored in the daemon's local database or in the pointer file itself (as a version number)? If stored externally, what happens when a provider device is restored from backup with a stale counter? | open | Blocked on: ADR-020 (pointer file management, Phase 3). A stale counter after device restore could cause nonce reuse — the recovery procedure must address this. |

---

## From Paper 18 — Tahoe-LAFS

| ID    | Question | Status | Blocked on |
|-------|----------|--------|------------|
| Q18-1 | What Argon2id parameters (t, m, p) are correct for minimum-spec Indian entry-level hardware (~2 GB RAM, dual-core)? The ADR uses t=3, m=64MB as a starting point — is this achievable within the session-start latency budget? | open | Blocked on: benchmark on target hardware before V2 launch. Target: ≤500ms on minimum-spec device. |
| Q18-2 | Should the owner's Ed25519 signing key (used to sign pointer files) also be derived from the master secret via HKDF, or generated separately and stored encrypted in the local key store? Deriving it makes recovery simpler; storing it separately allows rotation without re-uploading all pointer files. | open | Blocked on: V2 implementation design. If the signing key is derived, rotating it (after a compromise) requires re-signing all pointer files. If stored separately, the key store becomes a second backup target. |

---

## From Paper 19 — EC Survey (Shen et al., ACM ToS 2025)

| ID | Question | Status | Blocked on |
|---|---|---|---|
| Q19-1 | Our n=56 puts us in wide-stripe territory (Section 3.3). ECWide's combined locality assumes rack-based topology for local repair groups. Is there any equivalent locality structure in a P2P consumer network that could give a local repair benefit, or does wide-stripe RS without locality remain the only viable option? | open | Phase 2A #5 (Erasure Codes for Cold Data) and network topology telemetry at V2 launch. |
| Q19-2 | Clay codes support general (n, k) and are access-optimal MSR codes. What is the sub-packetisation level β for our (n=56, k=16, d=55) configuration... | **answered** | Goparaju et al. (Paper 22) Construction 2 formula: α = ρ^(k·C(r,ρ)). For our parameters at d=n-1: ρ=40, C(40,40)=1, α = 40^16 ≈ 10^25 sub-chunks per 256 KB fragment — computationally intractable. MSR/Clay codes are not feasible at (n=56, k=16) for any d. Only Hitchhiker codes (α=2, two sub-stripes) remain viable for V3. |
| Q19-3 | The survey cites a Facebook warehouse cluster measurement showing >98% of per-stripe failure events are single-chunk failures — the case MSR codes optimise for. Does this hold in a P2P consumer desktop network with correlated failures (power outages, ISP events, OS updates)? If multi-chunk events are common, MSR's single-failure optimisation is less well-targeted. | open | Blocked on: Phase 2A #2 (Trautwein et al., IPFS production measurements) and own provider telemetry at launch. |

---

## From Paper 20 — Trautwein et al. (IPFS, SIGCOMM 2022)

| ID | Question | Status | Blocked on |
|---|---|---|---|
| Q20-1 | IPFS shows 45.5% of discovered peers are always unreachable without hole-punching (measured before DCUtR was production-deployed). What fraction of Indian desktop home-router deployments are behind symmetric NAT (Circuit Relay v2 required vs cone NAT / DCUtR)? This determines the relay infrastructure load Vyomanaut must operate. Linked to Q13-1. | open | Network measurement during pre-launch provider onboarding testing. |
| Q20-2 | IPFS uses a 12h republication interval with a 24h expiry, providing a 12-hour buffer against delayed republication. Our availability service currently republishes every 24h (ADR-001, ADR-006) — zero buffer against delay. Should the republication interval be reduced to 12h, or should record expiry be extended to 48h to restore the buffer? | open | Implementation decision — must be resolved before the availability service is built. Both options are operationally equivalent; 12h republication doubles availability-service DHT write traffic. |
| Q20-3 | IPFS demonstrates an unincentivized churn floor of 87.6% of sessions under 8h. Vyomanaut's escrow model targets MTTF 180–380 days. What held-earnings percentage and minimum vetting period are required to empirically achieve the target MTTF from an operator population that, without incentives, would behave like IPFS peers? Linked to Q05-4. | open | Blocked on: Phase 5 economic mechanism research (ADR-024). The IPFS data provides the unincentivized baseline; the escrow design determines how far above it Vyomanaut can push its providers. |

---

## From Paper 21 — Saroiu et al. (Gnutella/Napster Measurement, MMCN 2002)

| ID | Question | Status | Blocked on |
|---|---|---|---|
| Q21-1 | The paper shows that only the top 20% of unincentivized P2P peers have IP-level uptime ≥93%. In Vyomanaut's vetting subsystem (ADR-005), the 4–6 month vetting period is designed to select from this reliable tail. What fraction of registered providers in V2 will be rejected or downgraded during vetting, and does the vetting period length need to be calibrated to the actual uptime distribution of Indian desktop users rather than the Storj NAS assumption? | open | Own provider telemetry at V2 launch. The paper's 20%/93% figure is from unincentivized 2001 broadband users; the Indian desktop provider population under financial incentives in 2025 will differ. |

---

## From Paper 23 — Yin et al. (Cold Data Erasure Codes, Applied Sciences 2023)

| ID | Question | Status | Blocked on |
|---|---|---|---|
| Q23-1 | The paper's write throughput crossover (EC = replication) is at ~512 KB block sizes. Vyomanaut's fragment size is 256 KB, just below the crossover. At upload time, the provider encodes and stores 56 fragments of 256 KB each. Is the ~10–15% write throughput penalty at 256 KB vs 512 KB observable in practice on Indian desktop hardware, and does it warrant increasing lf to 512 KB? The metadata overhead per segment scales with n=56 fragments regardless of lf; only the upload I/O pattern changes. | open | Benchmark on target hardware at V2 launch. If write time per segment at lf=256 KB vs lf=512 KB shows >20% difference, revisit lf. This requires changing ADR-003, which has downstream effects on ADR-004 (Qpeek formula) and ADR-006 (departure threshold). |