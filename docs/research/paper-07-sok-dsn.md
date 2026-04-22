## Paper 07 — SoK: Decentralized Storage Network

**Authors:** Chuanlei Li, Minghui Xu, Jiahao Zhang, Hechuan Guo, Xiuzhen Cheng — Shandong University
**Venue / Year:** IEEE S&P (workshop) | 2024
**Citations:** —
**Topics:** #1, #2, #3, #5, #6, #8, #13, #19
**ADRs produced:** ADR-001 (confirmed), ADR-002 (confirmed), ADR-005 (confirmed), ADR-014 (confirmed), ADR-015 (confirmed)

---

### Problem Solved

No single paper before this survey classified the full DSN landscape — Sia, Storj, Filecoin, Swarm — against a unified model with three axes: Proof of Storage, consensus algorithm, and incentive mechanism. The result is that each prior system was designed in isolation, without naming the failures created by its design choices. The paper names those failures, classifies them taxonomically, and identifies seven unsolved challenges that remain open across the entire field. For Vyomanaut, the survey provides the authoritative source for the Storj four-subsystem reputation pipeline (ADR-005), the Honest Geppetto attack (ADR-014), the three functions blockchain provides that we must replicate without a blockchain (ADR-015), and the confirmation that Swarm's per-bandwidth payment model failed in production (ADR-011, ADR-012).

---

### Key Findings

**The DSN abstract model is Put / Manage / Get (Section III):**
Every DSN is formally three operations. `Put(D, SM) → CID` uploads data D to a storage miner and returns a content identifier. `Manage(D, SM, ReM)` runs the continuous audit loop — verifying storage proofs, triggering repair, and resisting faults. `Get(CID, ReM) → D` retrieves data from a retrieval miner using the CID. Critically, Manage is the operation that most production systems underspecify — it is where PoS, incentive penalties, and repair all live. Vyomanaut's microservice audit loop is the Manage operation.

**Two classes of Proof of Storage (Section IV):**
Transitory PoS proves data was received at upload time but cannot prove it was retained. Continuous PoS chains proofs over time to prove retention. Every production DSN uses both: transitory on upload, continuous during Manage. The paper establishes that PDP/PoR (Merkle challenge) is the lightweight transitory primitive; PoSt (Proof of Spacetime, iterative chaining of PoR rounds) is the continuous primitive used by Filecoin. Storj and Sia use frequent Merkle challenges as a continuous substitute for PoSt without the ZK overhead. This is the taxonomy that led to Vyomanaut adopting the two-phase design: transitory PoS at upload (signed receipt), continuous PoR via randomised Merkle challenges (ADR-002).

**Filecoin hardware requirement eliminates desktop users (Section IX-A, Table I):**
To qualify as a Lotus miner in Filecoin, a provider must configure 256 GiB RAM and a GPU with at least 11 GB VRAM. This is a direct consequence of Seal-based PoRep (needed to prevent outsourcing and generation attacks). The paper explicitly lists this as a scalability failure — contradicting "universal participation." For Vyomanaut, this is the formal source of the rejection of PoRep in ADR-002 and the rejection of mobile/low-spec providers in ADR-010.

**Storj's four-subsystem reputation pipeline (Section VI):**
The paper distils Storj's reputation model into four stages: (1) proof-of-work identity generation to prevent Sybil floods at onboarding; (2) initial vetting, during which unvetted nodes receive non-critical data under extra erasure redundancy while trust is measured; (3) filtering, which decrements reputation on audit failures and triggers re-vetting if a threshold is crossed; (4) preference, which ranks surviving nodes by throughput and latency, routing new uploads to higher-ranked nodes. This is the system adopted verbatim in ADR-005, with Vyomanaut substituting registration gating (KYC/phone number) for proof-of-work identity generation.

**Honest Geppetto attack (Section VII-B):**
The paper formally names the Honest Geppetto attack from the Storj whitepaper: an adversary operates multiple puppet nodes that accumulate genuine trust over months before coordinating a mass departure or data deletion event. The mitigation requires analysing correlations between storage node operators — specifically ASN and subnet overlap — and preventing any correlated group from holding too high a fraction of any single file's shards. This is the direct source of the 20% ASN cap in ADR-014.

**Swarm SWAP bandwidth incentive failed (Section VI):**
The Swarm Accounting Protocol (SWAP) pays nodes for bandwidth exchange using BZZ tokens. Lakhani et al. (cited) showed that the current parameter settings fail to achieve fair distribution of rewards. The paper categorises this as a known failure mode: bandwidth-based payment creates coupling between the payment layer and the P2P transfer layer that breaks when either is under load. This is the formal justification for Vyomanaut's per-audit-passed payment model in ADR-012 and the explicit rejection of per-GB-transferred payment.

**DHT privacy leakage (Section IX-B):**
IPFS's DHT stores Content Identifiers (CIDs) in public routing tables, exposing query traffic to monitoring. Retrieving a file CID from the DHT reveals that the requester is interested in that content. The paper names this as an unresolved challenge across the field. For Vyomanaut, this is the origin of the HMAC-pseudonymised DHT key requirement in ADR-001 and ADR-021.

**Seven unsolved challenges (Section IX):**
The paper enumerates: (1) file version control, (2) DoS on central coordination entities (Satellite), (3) DHT-level privacy leakage of query traffic, (4) no mechanism to detect or remove illegal content, (5) absence of fine-grained access control beyond uploader-exclusive vs. public, (6) Honest Geppetto attack (correlated node group), (7) bandwidth optimisation. Vyomanaut addresses 3 (HMAC keys), 6 (20% ASN cap), and 7 (lazy repair + future Hitchhiker codes) directly. Challenges 1, 4, and 5 are deliberately out of Vyomanaut V2 scope. Challenge 2 is mitigated by the (3,2,2) microservice cluster with gossip membership (ADR-025).

---

### Trade-offs

| Chosen (by surveyed systems) | Over | Consequence for Vyomanaut |
| --- | --- | --- |
| Filecoin PoRep + PoSt (ZK-SNARK proofs) | Lightweight Merkle challenges | 256 GiB RAM + GPU ≥11 GB required — eliminates desktop providers entirely; ADR-002 rejects PoRep |
| Storj reputation-gated vetting | Cryptographic identity proofs per node | Faster onboarding, lower hardware bar, but creates Honest Geppetto attack window; ADR-005 accepts this trade-off |
| Sia frequent Merkle challenges | Berlekamp-Welch audits | Lightweight for consumer hardware; bypassed if challenge timing is predictable; ADR-002 mitigates via randomised nonce |
| Swarm SWAP bandwidth-as-currency | Escrow-based payment | Clean for symmetric peer exchange; fails structurally for the data-owner-to-provider model; ADR-012 rejects per-bandwidth payment |
| All DSNs — cryptocurrency for payment | Fiat currency | Trustless settlement; price volatility; high onboarding friction; failed in Swarm production; ADR-011 replaces with Razorpay/UPI |
| Smart contracts as trusted third-party for PoS | Off-chain verifier | Public auditability; requires on-chain writes per proof; Vyomanaut replaces with append-only microservice audit log (ADR-015) |

---

### Breaks in Our Case

- **Every surveyed DSN uses blockchain / cryptocurrency as the trust anchor for payment, audit verification, and dispute resolution** ≠ **Vyomanaut uses hardened microservices**
→ Blockchain performs three separable functions: (1) immutable audit log, (2) automatic payment trigger on verified proof, (3) public dispute resolution. Vyomanaut must replicate all three without on-chain writes. Solution: append-only INSERT-only Postgres audit log (function 1), microservice escrow release on audit pass (function 2), V3 Transparent Merkle Log (function 3). Each function is explicitly designed for; none are inherited from blockchain. → ADR-015
- **Filecoin's PoRep Seal prevents Sybil, outsourcing, and generation attacks cryptographically via slow sequential computation** ≠ **Vyomanaut uses registration gating + timing-based deadline + content-hash response**
→ Registration gating (ADR-001) closes Sybil attacks. The response deadline `(chunk_size / upload_speed) × 1.5` (ADR-014 Defence 2) closes outsourcing and just-in-time retrieval. The PoR response `SHA256(chunk_data || challenge_nonce)` closes generation attacks. The protection is weaker than PoRep cryptographically but achievable on commodity desktop hardware. This trade-off is explicit and accepted. → ADR-002, ADR-014
- **Storj's Satellite (coordination entity) is a centralised single point of failure, making DoS a documented vulnerability** ≠ **our availability requirement requires the coordination layer to survive single-node failure**
→ The (3,2,2) quorum cluster (ADR-025) provides N=3 replicas, R=2, W=2. One replica failure does not interrupt service. Gossip membership prevents logical partitions. The DoS threat identified in Section IX-A is structurally mitigated. → ADR-025
- **DHT lookups in IPFS expose Content IDs and query traffic to monitoring nodes** ≠ **our zero-knowledge requirement means file identity must not be revealed during chunk lookup**
→ DHT key = `HMAC-SHA256(chunk_hash, file_owner_key)` where `file_owner_key = HKDF(master_secret, "vyomanaut-dht-v1", file_id)`. The DHT never stores raw CIDs. Only the file owner can reverse-map a DHT key to a chunk. → ADR-001, ADR-021
- **The paper's DSN model assumes blockchain confirmation is required to complete Put and Get operations** ≠ **Vyomanaut completes Put and Get entirely over P2P and microservice without any on-chain transaction**
→ The blockchain's role in DSN.Put (confirming the storage transaction) and DSN.Get (confirming retrieval) is replaced by the microservice's signed receipt exchange (ADR-015). The receipt is the binding proof of both delivery and storage, verifiable by both parties without a blockchain. The P2P transfer layer is fully decoupled from payment (ADR-012).
- **Swarm SWAP treats bandwidth as a currency traded between symmetric peers** ≠ **our model is asymmetric: one data owner uploads, 56 providers store, retrieval is rare and unidirectional**
→ SWAP assumes roughly balanced upload/download relationships between nodes. In Vyomanaut, the data owner uploads once and never stores for others. Providers store but do not act as clients. This asymmetry makes bandwidth-as-currency meaningless — a provider earns zero from SWAP because no one is downloading from them to create a balanced ledger. Per-audit-passed payment (ADR-012) replaces SWAP entirely.

---

### Decisions Influenced

- **ADR-001 [#1 Coordination]** `CONFIRMED`
The hybrid microservice + Kademlia DHT architecture directly responds to the paper's finding that every pure-DHT DSN suffers Satellite-equivalent DoS risks (centralised metadata) and that pure-DHT designs leak CIDs. The microservice handles orchestration; Kademlia handles data-plane lookup with HMAC-pseudonymised keys.
*Because:* Section IX-B identifies DHT privacy leakage as an unresolved field-wide challenge. Our HMAC key approach is the adaptation that closes it within the Kademlia model.
- **ADR-002 [#2 Proof of Storage]** `CONFIRMED`
Two-phase design (transitory PoS at upload + continuous PoR via randomised Merkle challenge) maps directly to the paper's transitory/continuous taxonomy. The rejection of PoRep is formally supported by the hardware cost numbers in Section IX-A.
*Because:* Table I and Section IX-A establish that PoRep + PoSt requires 256 GiB RAM + GPU ≥11 GB — eliminating every desktop provider in Vyomanaut's target demographic.
- **ADR-005 [#5 Peer Selection]** `CONFIRMED`
The four-subsystem pipeline (identity gate → vetting → filtering → preference) is sourced from Section VI's description of Storj's reputation system. Vyomanaut substitutes phone-number registration gating for proof-of-work identity generation, which the paper treats as structurally equivalent (both limit Sybil floods at admission).
*Because:* Section VI names the four subsystems, explains their interaction, and establishes that vetting with extra erasure redundancy is the correct bootstrapping mechanism for untrusted new nodes.
- **ADR-011 [#13 Fiat Escrow]** `CONFIRMED`
The rejection of cryptocurrency payment is directly supported by Section VI's analysis of Swarm SWAP's production failure and the paper's observation that all DSN cryptocurrencies introduce friction. UPI/Razorpay fiat replaces them.
*Because:* The Lakhani et al. citations (Section VI) confirm SWAP's reward distribution fails at current parameter settings — not a temporary miscalibration but a structural failure of the bandwidth-exchange model.
- **ADR-012 [#13 Payment per Audit]** `CONFIRMED`
Per-audit-passed payment is the architectural response to the SWAP failure mode. The paper establishes that any payment model coupling the payment layer to the P2P transfer layer fails for asymmetric storage networks. Audit pass is the only event that is simultaneously verified by the microservice and independent of P2P transfer volume.
*Because:* Section VI's analysis of SWAP shows that bandwidth-as-currency requires symmetric bilateral exchange. Vyomanaut's model is unidirectional — payment for storage presence, not bandwidth consumption.
- **ADR-014 [#19 Adversarial Defences]** `CONFIRMED`
The Honest Geppetto attack is named and sourced from Section VII-B. The 20% ASN cap is the mitigation the paper identifies: analyse correlations between storage nodes, cap the fraction of any file's shards held by any single correlated group.
*Because:* Section VII-B states the mitigation explicitly: "meticulous analysis of storage node correlations and the strategic expansion of the network" to prevent correlated groups from reaching trust threshold simultaneously.
- **ADR-015 [#2 Audit Trail]** `CONFIRMED`
The three functions of blockchain that Vyomanaut must replicate — immutable audit log, automatic payment trigger, public dispute resolution — are directly identified from the paper's analysis of why every DSN uses blockchain. The signed receipt exchange (V2) and Transparent Merkle Log (V3) map to these three functions in order.
*Because:* The paper's comparison in Table I and the discussion in Section VI make explicit that blockchain is used as a trust anchor for all three functions. Without blockchain, each function must be designed explicitly.

---

### Disagreements

- **Lakhani et al. (2022 & 2023, cited in Section VI):** Evaluate Swarm SWAP fairness and find that current parameters do not achieve optimal reward distribution; they propose better parameter settings and novel mechanism instantiations to fix SWAP.
*Implication for us:* Lakhani's work treats SWAP as repairable through parameter tuning. Vyomanaut's analysis concludes SWAP is structurally broken for asymmetric data-owner/provider networks regardless of parameters — not a matter of tuning but of model mismatch. ADR-012's per-audit payment is the correct response, not improved SWAP parameters.
- **Bhagwan et al. (cited as availability baseline):** The paper assumes churn rates from unincentivised P2P systems as representative. Vyomanaut's financially-incentivised providers should exhibit materially higher MTTF.
*Implication for us:* The paper's availability assumptions are worst-case floors, not expected values. Our escrow seizure on departure (ADR-024) and vetting period (ADR-005) are calibrated to move providers well above those floors.

---

### Open Questions

See open-questions.md — questions Q07-1 through Q07-5 (all answered in answered-questions.md).