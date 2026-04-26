# Vyomanaut V2 — Accepted Trade-offs

**Status:** Reference document — do not edit to re-litigate closed decisions.  
**Purpose:** Each trade-off here is settled. The entries below explain what was chosen, what was
rejected, what was gained, and what was consciously accepted as a cost. If you want to revisit
one, open a new ADR and cite this document. Do not quietly build around a trade-off you dislike.

---

## How to read this document

Every entry has the form:

> **We chose X over Y.** We gain A. We accept B.

The gain is the reason the decision exists. The acceptance is the cost that must be managed,
not eliminated. A system that tries to avoid the acceptance without reversing the decision is
fighting itself. Know which you are doing before you start.

---

## Part I — Storage and Durability

### 1. Erasure coding over replication

**We chose Reed-Solomon RS(16, 56) over three-way replication.**  
We gain 2.5× storage overhead instead of 3×, and eight times less repair bandwidth per failure event under steady-state conditions (Blake & Rodrigues, Paper 06).  
We accept that reconstruction requires contacting 16 surviving fragment holders — repair bandwidth is not minimised per-failure; it is minimised in aggregate through lazy deferral. A provider replacing a crashed node must download k=16 chunks, not a fraction of one.

### 2. Wide-stripe RS over standard-stripe RS

**We chose n=56 shards over the production standard of n ≤ 20.**  
We gain 40 parity fragments of redundancy — the network survives the simultaneous loss of any 40 of 56 providers for any file, dramatically exceeding the fault tolerance of any comparable deployed system.  
We accept that we are operating in wide-stripe territory where MSR regenerating codes (Clay codes) are computationally intractable, leaving RS as the only feasible code family. We also accept the strongest form of the diminishing-return effect under correlated failures identified by Nath et al. (Paper 38) — the ASN cap is not optional; it is a co-requisite for the durability guarantee.

### 3. Lazy repair over eager repair

**We chose lazy repair (trigger at r0=8 above the reconstruction floor) over triggering repair immediately on every fragment loss.**  
We gain approximately 38× lower repair bandwidth compared to an eager policy at our parameters. The difference between earning periods and steady-state bandwidth costs is the economic foundation of the provider model.  
We accept that a file's redundancy sits below full for the duration of the lazy window. A provider that departs during this window finds their slot has been pre-allocated to a replacement, not that repair is waiting.

### 4. r=40 as the fixed redundancy level

**We chose r=40 (analytically optimal per Giroire Formula 4) over a lower r such as r=16.**  
We gain the unique r that minimises repair bandwidth at s=16, r0=8. Any other r increases bandwidth. The optimality is not intuitive — it is a mathematical result, not an engineering preference.  
We accept 3.5× storage overhead (56/16). Choosing r=20 would save storage but increase repair costs. Choosing r=60 would waste storage without reducing costs. r=40 is the saddle point.

### 5. The 20% ASN cap as both a security and a durability constraint

**We chose to bound any single correlated provider group (by ASN) to 20% of any file's shards, over allowing free assignment.**  
We gain two independent guarantees from one mechanism: the Honest Geppetto adversarial attack (colluding providers mass-departing) is bounded, and the correlated failure reversal identified by Nath et al. (where a large-m erasure scheme can be worse than a simpler scheme under real-world correlated failures) is structurally prevented. The 20% cap keeps the maximum correlated event at ~11 shards, leaving 45 survivors — 29 above the reconstruction floor.  
We accept that the assignment service must track ASN per provider and enforce the cap at write time. Assignments that would violate the cap are declined. In a small provider pool, this can reduce placement flexibility.

### 6. WiscKey key-value separation over standard RocksDB

**We chose WiscKey-style separation (RocksDB for the 44-byte chunk index, append-only vLog for 256 KB chunk values) over storing keys and values together in RocksDB.**  
We gain write amplification of approximately 1.0 at 256 KB values. Standard RocksDB produces 10–14× write amplification at this value size. On a provider storing 50 GB of chunks, this is the difference between writing 50 GB and writing 500–700 GB to their storage device.  
We accept the need to manage vLog GC on chunk deletion, implement a single-writer goroutine for concurrent write safety, and write a crash recovery tail-scan. A flat object store would have lower code complexity but no built-in index, no Bloom filters, and no compaction behaviour to tune.

### 7. Sequential vLog layout over per-chunk scattered placement

**We chose to append chunks sequentially to the vLog by upload time, over placing chunks at randomised or non-adjacent disk offsets.**  
We gain HDD-optimised write performance (sequential appends have no seek overhead) and simpler crash recovery (scan from last known tail).  
We accept that a spatial disk failure (surface scratch across a contiguous region) can corrupt shards from many different files simultaneously. This is bounded at the network level by RS(16,56): each affected file retains 55 surviving shards on independent providers. Sparse placement is deferred to V3 pending empirical failure data.

---

## Part II — Network and Peer Discovery

### 8. Hybrid microservice + Kademlia over pure DHT

**We chose a hybrid architecture (microservice for orchestration, Kademlia DHT for chunk-address lookup) over full decentralisation.**  
We gain admission control (registration gating deters Sybil attacks without cryptographic ID generation costs), reliable audit scheduling, and a stable authority for payment computation. Pure Kademlia DHT discovery fails operationally below ~100 peers and creates Eclipse attack surfaces without an admission gate.  
We accept that the microservice is a dependency. If it goes down, new uploads halt, audit scheduling pauses, and repair jobs queue. The data plane (P2P chunk transfer) continues uninterrupted. Storj v3.1 made the same trade-off and fully removed their DHT — we retain Kademlia for data-plane lookup only and are not making the same mistake of using it for provider discovery.

### 9. Centralised audit scoring over distributed reputation

**We chose a single authoritative microservice scorer over distributed peer-to-peer reputation systems (EigenTrust, PeerTrust).**  
We gain correctness: the Shelby proof (Paper 37, Proposition 1) formally demonstrates that peer-to-peer audit reports without a trusted backstop collapse to universal dishonesty as the unique Nash equilibrium. Centralising the scorer is not a convenience — it is the only architecture that produces honest outcomes.  
We accept that the microservice is the single source of truth for reliability scores and that the three-window rolling score (ADR-008) must be computed and stored there, not distributed. This is the non-I-confluent operation that cannot be relaxed.

### 10. (3, 2, 2) microservice quorum over a single-node microservice

**We chose N=3 replicas with R=2, W=2 over a simpler single-instance deployment.**  
We gain single-replica fault tolerance: one replica can fail without interrupting audit scheduling, challenge dispatch, or payment processing.  
We accept gossip membership management overhead, the need for seed node configuration, and the operational complexity of a three-node cluster. The alternative — a single microservice instance — makes a DoS event or server failure a complete service outage.

### 11. libp2p + QUIC over raw QUIC or gRPC

**We chose the libp2p framework with QUIC v1 as the primary transport over building NAT traversal, peer identity, and DHT integration from scratch, or using gRPC over HTTP/2 over TCP.**  
We gain a production-proven P2P stack with three-tier NAT traversal (AutoNAT → DCUtR → Circuit Relay v2), connection migration, independent stream delivery, and a Kademlia DHT implementation — all tested at IPFS and Filecoin scale.  
We accept a significant external dependency on the libp2p release cycle and the need to maintain a custom DHT key validator (for HMAC-pseudonymised chunk keys) across library upgrades. gRPC would have been simpler for a server-to-server model but does not solve connection migration or NAT traversal.

### 12. Circuit Relay v2 fallback for symmetric NAT over excluding symmetric-NAT providers

**We chose to support symmetric-NAT providers via Circuit Relay v2 over requiring all providers to be publicly reachable.**  
We gain access to approximately 30% of Indian home desktop operators who are behind CGNAT or symmetric NAT routers — a significant fraction of the target provider population.  
We accept that all audit traffic for relay-dependent providers is proxied through Vyomanaut-operated relay nodes, adding ~50 ms of relay overhead per challenge interaction. We also accept the operational cost of maintaining relay infrastructure (at minimum 3 nodes at launch).

### 13. HMAC-pseudonymised DHT keys over plaintext content IDs

**We chose DHT lookup keys of the form `HMAC(chunk_hash, file_owner_key)` over storing real chunk hashes or file IDs in the DHT.**  
We gain DHT privacy: a monitoring node observing DHT traffic cannot correlate lookup requests with file identity. Only the file owner, who holds the master secret, can reverse-map a DHT key to its chunk. This closes Challenge 3 of the seven unsolved DSN challenges identified by the SoK survey (Paper 07).  
We accept that the DHT key validator is a custom extension to libp2p's Kademlia implementation. It must be explicitly pinned and preserved across library upgrades — a standard libp2p upgrade that resets the namespace configuration to defaults would silently break all chunk lookups.

### 14. Provider heartbeat every 4 hours over relying on DHT republication

**We chose a signed 4-hour heartbeat from each provider to the microservice over depending on DHT republication (12-hour interval) for fresh provider addresses.**  
We gain reliable audit challenge delivery despite DHCP lease rotations — Indian residential ISPs commonly rotate addresses on 24-hour cycles, meaning the DHT record can be stale by up to 12 hours at challenge time. The heartbeat guarantees the microservice has an address no older than 4 hours.  
We accept that the provider daemon must maintain a persistent outbound HTTPS connection schedule and that a provider who disables their machine without the daemon running accumulates stale-address TIMEOUT audit results before the 72-hour departure threshold fires.

---

## Part III — Encryption and Key Management

### 15. AONT-RS over encrypt-then-code or code-then-encrypt

**We chose the All-or-Nothing Transform applied before Reed-Solomon coding (AONT-RS) over encrypting with a separate key then coding, or coding then encrypting per-shard.**  
We gain the elimination of external key management: the AONT key K is embedded in the erasure-coded data and recoverable only when k=16 shards are assembled. No key server, no per-file AES key to store or rotate, and no per-chunk key infrastructure (which would require 56 distinct keys per file).  
We accept that the pointer file becomes the sole retrieval credential. Losing the pointer file and having no backup means permanent, unrecoverable data loss. The master secret hierarchy (ADR-020) manages this risk, but the risk cannot be eliminated — it is the price of zero-knowledge storage.

### 16. ChaCha20-256 as the default AONT cipher over AES-256

**We chose ChaCha20-256 as the default AONT cipher path (with AES-256-CTR as a fast path on AES-NI hardware) over AES-only.**  
We gain a constant-time cipher that requires no hardware acceleration — on Indian home desktops without AES-NI (estimated 10–15% of the target population), ChaCha20 is 3× faster than software AES and free of cache-timing vulnerabilities.  
We accept two code paths in the daemon (ChaCha20 and AES-CTR) and the need for CPUID detection at startup. On AES-NI hardware, ChaCha20 is slightly slower than AES-GCM — the fast path is not the universal path.

### 17. Master-secret-derived key hierarchy over per-file random keys

**We chose a three-level key hierarchy derived from a single Argon2id master secret (master_secret → file_key → pointer file encryption key) over generating independent random keys per file.**  
We gain a single backup surface: one passphrase (and optionally a BIP-39 mnemonic) recovers all files on any device. No per-file key management burden on the data owner.  
We accept that the master secret is a single point of failure — loss of both the passphrase and the mnemonic means permanent loss of all files, with no support path. This must be disclosed clearly at onboarding.

### 18. 0-RTT disabled for audit interactions over enabling it for all reconnects

**We chose to disable QUIC 0-RTT session resumption for connections carrying audit challenges or signed receipts, while permitting it for pure chunk data transfers.**  
We gain protection against 0-RTT replay attacks — an attacker replaying a valid audit response could falsely credit a provider with a pass for data they no longer hold.  
We accept one additional round trip per audit reconnect after a provider's nightly absence. At the audit deadline of ~614 ms, this is manageable but not free.

---

## Part IV — Payment and Incentives

### 19. Held-earnings escrow over pre-committed collateral

**We chose to hold a portion of already-earned payments (30-day rolling window) over requiring providers to stake capital before earning, as Filecoin does.**  
We gain zero entry barrier: any Indian home desktop owner with a UPI-linked bank account can join without pre-committing money they do not yet have. The model is calibrated for the Indian V2 provider demographic, where pre-commitment would eliminate most of the target market.  
We accept that new providers have little at stake early — a provider with only a few days of earnings has limited escrow at risk. The 4–6 month vetting period with a 60-day hold and 50% release cap partially compensates, but a provider who leaves on day 2 faces a smaller financial deterrent than one leaving on day 180.

### 20. Payment per audit passed over payment per GB stored or per GB transferred

**We chose per-audit-passed payment over per-GB-stored monthly payment or per-GB-transferred bandwidth payment.**  
We gain decoupling of the payment layer from the P2P transfer layer — a microservice outage cannot accumulate credit liability, and a transfer failure does not interrupt audit scoring. We also eliminate the delete-and-restore attack (paying for ingress incentivises providers to delete and re-store data) and the bandwidth-as-currency failure mode that destroyed Swarm's SWAP protocol.  
We accept that providers earn equally for storing popular data and data that is never retrieved. The escrow model pays for availability, not utility. A data owner pays the same whether or not they retrieve their files.

### 21. Fiat escrow via Razorpay over cryptocurrency

**We chose Razorpay/UPI fiat escrow over a token-based payment mechanism as used by Filecoin, Storj, and Swarm.**  
We gain zero crypto-wallet friction for Indian providers and data owners, instant UPI settlement, zero per-transaction merchant fee, and compatibility with the Indian regulatory environment.  
We accept India-only operation at launch (Razorpay and UPI require Indian bank accounts), dependence on Razorpay's API stability, and the need to implement all escrow hold-and-release logic in the microservice rather than delegating it to a smart contract. The `PaymentProvider` interface is designed to add international gateways without rewriting payment logic.

### 22. Route's `on_hold` API over Razorpay Escrow+

**We chose Razorpay Route with programmable settlement hold (`on_hold_until`) as our escrow primitive over Razorpay's native Escrow+ product.**  
We gain full programmatic control over hold/release/seizure via standard REST APIs, with no NBFC registration or trustee approval required.  
We accept that Route is not a "real" escrow from a legal standpoint — it is a settlement hold on a payment transfer, not a regulated tri-party account. All escrow logic lives in the microservice's internal ledger. The legal exposure of this distinction is an open question managed at the product level, not the engineering level.

### 23. Deterministic pricing over auction pricing

**We chose a fixed storage rate per GB per month (set at contract creation, not adjusted by network demand) over an auction mechanism that discovers price dynamically.**  
We gain predictability for both providers (they know what they will earn) and data owners (they know what they will pay). Contracted cold storage with month-long SLAs is incompatible with fluctuating pricing.  
We accept that the storage rate may diverge from market rate over time. If the rate is set too high, data owners seek alternatives. If too low, providers churn. The rate is a product decision calibrated at launch and revisited empirically — the architecture does not adjust it automatically.

### 24. Non-transferable escrow over transferable earnings

**We chose to make the escrow balance identity-bound (non-transferable to another provider_id) over allowing providers to assign their balance to other accounts.**  
We gain deterrence against identity-cycling attacks — a provider cannot earn under one identity, transfer the balance, and depart without seizure consequences.  
We accept that this must be enforced at the payment service level as a hard rule, not merely a policy. The microservice refuses transfer requests unconditionally.

---

## Part V — Proof of Storage and Auditing

### 25. PoR Merkle challenges over PoRep + PoSt

**We chose periodic randomised Merkle challenges (`SHA256(chunk_data || nonce)`) over Proof of Replication and Proof of Spacetime as used by Filecoin.**  
We gain compatibility with commodity desktop hardware — Filecoin's PoRep requires 256 GB RAM and a GPU with at least 11 GB VRAM, which eliminates every provider in Vyomanaut's target demographic.  
We accept a weaker cryptographic guarantee. PoRep proves data was physically encoded on-device. Our PoR only proves the provider can produce a correct hash response within a deadline. Three compensating mitigations close the gaps: registration gating (Sybil), response deadline (outsourcing), and randomised challenge timing (JIT retrieval). These mitigations are co-requisites — the PoR design does not hold in isolation without them.

### 26. Timing-based outsourcing prevention over cryptographic sealing

**We chose the response deadline `(chunk_size / p95_measured_upload_throughput) × 1.5` over Filecoin's cryptographic Seal operation as the outsourcing deterrent.**  
We gain a mechanism that works on any hardware without GPU or extended computation requirements.  
We accept a weaker guarantee — the deadline is timing-based, not cryptographically enforced. A provider co-located with another provider on the same network segment might be able to fetch the chunk fast enough to pass. The 20% ASN cap limits this attack surface by preventing co-location of correlated providers.

### 27. INSERT-only audit receipts over mutable audit records

**We chose an append-only INSERT-only audit receipt table (enforced at the Postgres row security policy level, not only in application code) over a mutable audit log.**  
We gain tamper-evidence: no receipt can be altered or deleted after creation. Both provider and microservice sign every receipt, creating non-repudiation for both parties.  
We accept that the audit table grows monotonically and requires periodic archival. At the V2 launch scale of hundreds of providers and daily audits, this is manageable. At V3 scale (tens of thousands of providers × thousands of chunks), the INSERT rate approaches Postgres throughput limits and probabilistic sampling may be necessary — but that requires re-validating incentive compatibility per Theorem 1 of the Shelby proof.

### 28. V2 signed receipt exchange over immediate public audit verifiability

**We chose V2 signed receipts (both provider and microservice sign; stored in append-only Postgres) over a publicly verifiable audit trail from day one.**  
We gain operational simplicity — no distributed Merkle log infrastructure to maintain at launch.  
We accept that in V2, a data owner must trust the microservice's countersignature. Disputes cannot be resolved independently of Vyomanaut. The V3 Transparent Merkle Log (daily published Merkle root over all receipts) closes this gap — it is the explicit upgrade path, not an afterthought.

---

## Part VI — Operational Constraints

### 29. 72-hour departure threshold over a shorter or longer window

**We chose 72 hours of absence as the threshold before declaring a provider permanently departed and triggering repair, over a shorter window (e.g., 8 hours) or longer window (e.g., 1 week).**  
We gain safety against false-positive repair triggers from normal nightly absences (peak µ=14h) and weekend absences (peak µ=64h). Bolosky's bimodal distribution (Paper 09) shows 99.7% of weekend absences resolve within 70 hours — 72 hours is the first safe value above this.  
We accept up to 72 hours of reduced redundancy before a truly departed provider's chunks begin repair. During this window, the file is below full redundancy but above the reconstruction floor (r0=8 parity fragments above s=16). A second departure during this window is survivable; a third concurrent departure for the same file is not guaranteed to be.

### 30. 24-hour polling interval over shorter or adaptive intervals

**We chose t=24 hours as the audit challenge period over a shorter interval (e.g., 1 hour) or an adaptive interval.**  
We gain approximately 30× bandwidth savings over an instant timeout (Blake & Rodrigues, Paper 06). Daily challenges are sufficient for a write-once cold storage workload — the data does not change between audits.  
We accept that a provider can delete their chunks and pass no audits for up to 24 hours before the first FAIL is recorded. The exponential LossRate formula makes this period safe at r0=8. Adaptive polling (shorter intervals for lower-scored providers) is a V3 enhancement.

### 31. Desktop-only providers in V2 over including mobile

**We chose to exclude mobile providers from V2 entirely, deferring to V3.**  
We gain a single parameter set that works for all V2 providers. Including mobile at MTTF ~30–90 days would push per-provider repair bandwidth to ~130 Kbps — above the 100 Kbps background budget — burdening every desktop provider in the network for every mobile provider that churns.  
We accept a smaller launch-day provider pool and deferral of a large potential market. The India-first residential desktop and NAS population is large enough to establish the network without mobile.

### 32. Minimum viable network gate over allowing early uploads

**We chose to block all data owner uploads until seven network conditions are simultaneously satisfied (≥56 providers, ≥5 ASNs, ≥3 metro regions, full microservice quorum, ≥56 Razorpay accounts active, ≥3 relay nodes, audit secret loaded) over permitting partial uploads in a degraded state.**  
We gain the guarantee that no file is ever stored with insufficient redundancy from the moment the network opens. A file stored with only 30 providers would fail the RS(16,56) encoding — an impossible state to recover from cleanly.  
We accept that the launch day is gated on provider recruitment. Vyomanaut cannot open to data owners until the provider pool is deep enough. This is a deliberate sequencing of the two-sided market problem.

### 33. India-only payment processing at launch over global from day one

**We chose Razorpay/UPI India-only at launch over building a multi-gateway payment system immediately.**  
We gain faster time to launch, zero per-transaction fee, and instant settlement — properties unavailable in the global payment landscape without significant added complexity.  
We accept that international providers and data owners cannot participate at launch. The `PaymentProvider` interface is designed specifically so that Stripe Connect (or another international gateway) can be added as an implementation without rewriting payment logic. India-only is a launch constraint, not a permanent architectural limit.

---

## Part VII — What This Architecture Is Not Trying to Do

The following properties were considered and explicitly rejected. They are recorded here to
prevent them from re-entering the design in disguise.

**It is not trying to eliminate server trust entirely in V2.** Public verifiability — the
ability for any third party to verify audit receipts without trusting Vyomanaut — is a V3
goal. In V2, data owners must trust the microservice's countersignature. The V3 Transparent
Merkle Log is the upgrade path. Building it in V2 would delay launch without changing the
core product experience for the initial user base.

**It is not trying to use blockchain.** Not now, not in V3. The three functions blockchain
provides (immutable log, payment trigger, dispute resolution) have been replicated by
specific non-blockchain mechanisms. The reasons are product-level: NBFC registration for
Razorpay Escrow+, token price volatility, and Indian regulatory uncertainty around
cryptocurrency all make blockchain a liability for the India-first market. The system is
designed to never need it.

**It is not trying to make storage cheap by accepting lower durability.** The LossRate < 10⁻¹⁵
per year target is not negotiable. The erasure parameters were derived to satisfy it, not
to minimise storage overhead. r=40 is the analytically optimal value, not an overprovisioned
one. Reducing r to save storage cost would increase repair bandwidth and move the loss rate
into observable territory.

**It is not trying to pay providers for what they do not prove.** Payment per audit passed
means providers who are offline, who have deleted their data, or who return wrong hashes earn
nothing. The system does not credit hoped-for storage — only verified storage. This is not
a payment philosophy; it is the mechanism by which the Nash equilibrium is maintained.

**It is not trying to abstract away correlated failure.** The 20% ASN cap is not a policy
that can be relaxed for a well-known provider group or a trusted operator. Nath et al.
(Paper 38) proved that large-m erasure schemes can be strictly worse than simpler schemes
under correlated failures when the correlation is not bounded structurally. The cap is the
structural bound. There are no exceptions.

**It is not trying to serve hot data in V2.** The Cold Storage Band is the only band
implemented at launch. The Hot Storage Band (low-latency, higher-redundancy, per-retrieval
payment) is explicitly deferred to V3. Building for hot data access patterns now would
change the erasure parameters, the payment model, and the provider hardware requirements —
in ways that have not been researched. It is not ready.

---

## Appendix — Trade-off decision registry

| Trade-off | ADR | Research source |
|-----------|-----|-----------------|
| RS over replication | ADR-003 | Papers 06, 10 |
| Wide stripe (n=56) | ADR-003 | Papers 19, 22, 38 |
| Lazy repair over eager | ADR-004 | Papers 06, 10, 39 |
| 20% ASN cap (dual purpose) | ADR-014 | Papers 07, 20, 38 |
| WiscKey over standard RocksDB | ADR-023 | Papers 25, 26, 27, 32 |
| Sequential vLog layout | ADR-023 | Papers 25, 32 |
| Hybrid microservice + DHT | ADR-001 | Papers 02, 03, 05, 07 |
| Centralised audit scoring | ADR-008 | Papers 24, 31, 37, 40 |
| (3,2,2) quorum | ADR-025 | Paper 12 |
| libp2p + QUIC | ADR-021 | Papers 13, 14, 30 |
| Circuit Relay v2 fallback | ADR-021 | Paper 30 |
| HMAC-pseudonymised DHT keys | ADR-001 | Papers 04, 07 |
| 4-hour heartbeat | ADR-028 | Papers 20, 28, 30 |
| AONT-RS encoding | ADR-022 | Papers 15, 16 |
| ChaCha20 as default cipher | ADR-019 | Paper 17 |
| Master-secret key hierarchy | ADR-020 | Papers 17, 18 |
| 0-RTT disabled for audit | ADR-021 | Paper 14 |
| Held-earnings escrow | ADR-024 | Papers 05, 29, 33 |
| Payment per audit passed | ADR-012 | Papers 05, 07, 33 |
| Fiat over cryptocurrency | ADR-011 | Papers 07, 33, 35 |
| Route over Escrow+ | ADR-011 | Paper 35 |
| Deterministic pricing | ADR-024 | Paper 33 |
| Non-transferable escrow | ADR-024 | Papers 33, 37 |
| PoR over PoRep | ADR-002 | Papers 05, 07, 29 |
| Timing-based outsourcing prevention | ADR-014 | Papers 29, 37 |
| INSERT-only audit receipts | ADR-015 | Papers 07, 11 |
| V2 signed receipts over public verifiability | ADR-015 | Paper 07 |
| 72-hour departure threshold | ADR-006, ADR-007 | Papers 08, 09, 12 |
| 24-hour polling interval | ADR-006 | Papers 06, 08 |
| Desktop-only V2 | ADR-010 | Papers 05, 06, 08, 10 |
| Minimum viable network gate | ADR-029 | Papers 10, 36, 38 |
| India-only at launch | ADR-011 | Papers 33, 35 |