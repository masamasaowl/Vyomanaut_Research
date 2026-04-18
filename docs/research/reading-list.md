# Reading List

---

## Phase 0 — Landscape survey

**Motive:** Get a fair idea about developments in P2P file transfer and distributed storage networks.

| # | Title | Type | Priority | Topics | Key extraction target |
|---|---|---|---|---|---|
| 1 | Incentives Build Robustness in BitTorrent — Bram Cohen, 2003 | Paper | MANDATORY | #1 | See [paper-01](paper-01-bittorrent.md) ✅ |
| 2 | Kademlia: A P2P information system based on XOR metric — Maymounkov & Mazieres | Paper | MANDATORY | #1 | See [paper-02](paper-02-kademlia.md) ✅ |
| 3 | S/Kademlia: A Practicable Approach Towards Secure Key-Based Routing — Baumgart & Mies | Paper | MANDATORY | #1 | See [paper-03](paper-03-skademlia.md) ✅ |
| 4 | IPFS: Content Addressed, Versioned, P2P File System — Juan Benet, 2015 | Paper | MANDATORY | #1, #2 | See [paper-04](paper-04-ipfs.md) ✅ |
| 5 | Storj V3 whitepaper — Storj Labs, 2018/2024 | Whitepaper | MANDATORY | #1, #2, #13 | See [paper-05](paper-05-storj.md) ✅ |

---

## Phase 1 — Coordination, storage systems, and foundational protocols ✅ (mostly complete)

**Goal:** Decide #1 Coordination Architecture, #2 Proof of Storage, #12 P2P Transfer Protocol, #14 Consistency Model, and inform #15 Encryption-Erasure Interaction.

**Remaining:** Items 8, 9, 11 below. Items 10 and 7 are done.

| # | Title | Type | Priority | Status | Topics | Key extraction target | Why this position |
|---|---|---|---|---|---|---|---|
| 1 | High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two — Blake & Rodrigues, HotOS 2003 | Paper | MANDATORY | ✅ [paper-06](paper-06-blake-rodrigues.md) | #1 | | |
| 2 | SoK: Decentralised Storage Network — Li et al., IEEE S&P | Paper | MANDATORY | ✅ [paper-07](paper-07-sok-dsn.md) | #1–3, #5, #13 | | |
| 3 | Understanding Availability in Peer-to-Peer Applications — Bhagwan, Savage, Voelker, IPTPS 2003 | Paper | MANDATORY | ✅ [paper-08](paper-08-bhagwan-availability.md) | #5, #6, #8 | | |
| 4 | Feasibility of a Serverless Distributed File System — Bolosky et al., ACM SIGMETRICS 2000 | Paper | MANDATORY | ✅ [paper-09](paper-09-bolosky-feasibility.md) | #1, #6, #11 | | |
| 5 | P2P Storage Systems: A Practical Guideline to Be Lazy — Giroire et al., GlobeCom 2010 | Paper | MANDATORY | ✅ [paper-10](paper-10-giroire-lazy.md) | #3, #4, #6, #17 | | |
| 6 | Coordination Avoidance in Database Systems — Bailis et al., VLDB 2015 | Paper | MANDATORY | ✅ [paper-11](paper-11-bailis-coordination.md) | #14 | | |
| 7 | Dynamo: Amazon's Highly Available Key-Value Store — SOSP 2007 | Paper | MANDATORY | ✅ [paper-12](paper-12-dynamo.md) | #1, #4, #6 | **Done.** Old list did not mark this complete. ADR-025 produced. | Corrected status |
| 8 | libp2p Specification & Architecture | Docs | MANDATORY | ⬜ **READ NEXT** | #1, #12 | Kademlia implementation + NAT traversal + hole-punching strategies. Closes ADR-021 jointly with RFC 9000. | ADR-021 is the most consequential unresolved architectural decision for the data plane. Without it, the provider daemon cannot be designed. |
| 9 | QUIC: A UDP-Based Multiplexed and Secure Transport — RFC 9000 | RFC | MANDATORY | ⬜ **READ NEXT** | #12 | Connection migration (connection ID not IP/port); multiplexed streams; 0-RTT handshake for reconnecting providers. Closes ADR-021 jointly with libp2p spec. | Same urgency as libp2p. Read together. |
| 10 | Separation and Optimization of Encryption and Erasure Coding — Szabó et al., FGCS Feb 2025 | Paper | MANDATORY | ✅ [paper-15](paper-15-szabo-encrypt-erasure-separation.md) | #3, #9, #15 | **Done as Paper 15.** Confirmed encrypt-then-code preserves zero-knowledge. Did not close ADR-022. | Corrected status |
| 11 | AONT-RS: Blending Security and Performance in Dispersed Storage Systems — Resch & Plank, FAST 2011 | Paper | MANDATORY | ⬜ | #9, #15 | The All-or-Nothing Transform combined with erasure coding makes individual chunks unintelligible without a threshold of them — this is the strongest argument for code-then-encrypt. The paper gives a direct performance and security comparison with encrypt-then-code. Closes Q15-2 and unblocks ADR-022. | **New addition.** Szabó 2025 is done but ADR-022 is not closed. This is the paper that answers the remaining half of Q15-2: does code-then-encrypt offer meaningful additional security? Must be read before ADR-022 can be accepted. Placed immediately after Szabó in the sequence. |
| 12 | RFC 8439 — ChaCha20 and Poly1305 for IETF Protocols | RFC | MANDATORY | ⬜ | #9 | AEAD construction spec: nonce size, key size, authentication tag, streaming API. Confirms whether ChaCha20-Poly1305 is suitable for 256 KB chunk encryption on provider hardware without AES-NI. Closes ADR-019 primitive selection. | **New addition.** ADR-019 is blocked on choosing an encryption primitive. No paper in the prior list specified a cipher. RFC 8439 is the canonical spec for ChaCha20-Poly1305, which is the correct choice for provider hardware that may lack AES-NI acceleration (NAS devices, older desktops). Only needs Section 2 (cipher) and Section 4 (AEAD construction). |
| 13 | Filecoin Whitepaper — Protocol Labs, 2017 | Whitepaper | MANDATORY | ⬜ | #1, #2, #13 | Sections 3.1–3.2 only: PoRep and PoSt mechanisms. Skip all token/blockchain sections. Feeds Q05-8 (per-segment pricing) and ADR-024 (economic mechanism). | Moved to end of Phase 1 because ADR-021 (data plane) is more urgent than ADR-024 (economics). |

---

## Phase 2A — Erasure coding, peer measurement, and geographic routing

**Goal:** Finalise #3 Erasure Coding, validate ADR-006 parameters with real churn data, and inform #5 Peer Selection (geographic component).

**Read in this order.** Reordered from original: EC survey first (landscape before diving into Dimakis), Trautwein second (real churn data feeds everything), Saroiu third (session length feeds α calibration), Dimakis fourth (foundational repair bandwidth theory), cold data fifth, Coral last (geographic proximity is enhancement, not blocker).

| # | Title | Type | Priority | Topics | Key extraction target | Why this position |
|---|---|---|---|---|---|---|
| 1 | Survey of the Past/Present/Future of Erasure Coding — Shen, Lee et al., ACM ToS Dec 2024 | Paper | MANDATORY | #3 | RS vs LRC vs MSR vs MLEC comparison. Maps the landscape before Dimakis so the foundational paper lands in context. Section on cold storage trade-offs feeds ADR-018. | Promoted to first. Reading Dimakis without the landscape survey produces a narrow view. The survey is the correct entry point to the EC phase, analogous to how SoK was the correct entry to Phase 1. |
| 2 | Design & Evaluation of IPFS — Trautwein et al., SIGCOMM 2022 | Paper | MANDATORY | #1, #5, #6 | Real production measurements: lookup latency, content decay, churn rates. Section 5 (availability decay) feeds Q08-3. Also feeds Q07-4 (correlated failure modelling at scale). | Promoted from Phase 2A #3 (was last). Real churn numbers are needed to calibrate Q10-4 (split α) and validate that Giroire's Markov chain model holds in practice. Must be read before Dimakis derives repair parameters from it. |
| 3 | Measurement Study of P2P File Sharing — Saroiu et al., 2002 | Paper | MANDATORY | #3, #5, #6 | Median node session < 1 hour. Table 2–4: actual uplink/downlink distributions. Feeds α_temporary in Q10-4. | Kept in position. Session length data is required input to the split-α question before Dimakis. |
| 4 | Minimum Storage Regenerating Codes — Dimakis et al., 2010 | Paper | MANDATORY | #3, #17 | **FOUNDATIONAL:** proves the information-theoretic lower bound on repair bandwidth. All subsequent repair papers build on this. Required before ADR-017 (repair BW optimisation) can be revisited. | No change in relative position. Now fourth rather than fifth because churn data (items 2 and 3) must precede it. |
| 5 | Erasure Codes for Cold Data in Distributed Storage Systems — Chao Yin et al. | Paper | MANDATORY | #3 | Consumer device storage = cold data model. Storage overhead vs reconstruction speed trade-off. Feeds ADR-018 (hot/cold bands) with parameter guidance for the cold band. | No change. |
| 6 | Coral DSHT — Freedman & Mazières, NSDI 2004 | Paper | RECOMMENDED | #1, #5 | Sloppy DHT: store pointers not values. Geographic clustering for provider selection. Section 3 is the key target. | **Deprioritised from MANDATORY to RECOMMENDED.** Geographic proximity (Q01-4) is not blocking any accepted ADR for V2 launch. Peer selection works without it. This is a V2 enhancement, not a V2 requirement. Read only after items 1–5 are done. |

---

## Phase 2B — Reputation, adversarial behaviour, and DHT churn

**Goal:** Validate #8 Reliability Scoring, close Q07-4 (correlated failure), Q08-4 (adaptive polling), and confirm ADR-014 (adversarial defences) is complete.

**Reordered from original:** EigenTrust first (most open questions resolved), Handling Churn in DHT second (challenges Storj's MTTF assumption — directly feeds Q10-4 and ADR parameter confidence), IRON third (adversarial simulation), TrustDSN fourth (economic mechanism prereq).

| # | Title | Type | Priority | Topics | Key extraction target | Why this position |
|---|---|---|---|---|---|---|
| 1 | EigenTrust: Managing Trust in P2P Networks — Kamvar et al., WWW 2003 | Paper | MANDATORY | #8 | Global reputation from local pairwise interactions. Key question: can it propagate correlated failure signals without a central coordinator? Closes Q07-4. Informs Q08-4 (adaptive polling interval as a function of trust score). | **Promoted from #17 to first in Phase 2B.** Two open questions (Q07-4 and Q08-4) are directly blocked on this paper. The reliability scorer microservice cannot be finalised until correlated failure detection is resolved. |
| 2 | Handling Churn in a DHT — Rhea et al., NSDI 2004 | Paper | MANDATORY | #1, #6 | DHT churn handling strategies. MTTF measured at hours-to-days under no financial friction — directly challenges Storj's 6–12 month assumption cited in Paper 05's Disagreements. Feeds Q10-4 (whether split α changes optimal r). | **Promoted from #19 to second.** This paper was flagged in Paper 05's Disagreements as challenging our MTTF assumption. Its churn measurements are an input to verifying whether our repair parameters are safe. Must be read before parameter finalisation. |
| 3 | IRON File Systems — SOSP 2005 | Paper | MANDATORY | #19 | Adversarial provider simulation: partial writes, silent corruption, Byzantine storage nodes. Confirms or extends ADR-014. | No change in relative order. ADR-014 is accepted; this is confirmatory. |
| 4 | TrustDSN: Reputation Without Blockchain for Distributed Storage | Paper | MANDATORY | #8, #19 | Reputation system without blockchain as trust anchor. Required reading before ADR-024 (economic mechanism) can be attempted. | No change. Kept last because it depends on understanding EigenTrust first. |

---

## Phase 2C — Provider-side storage engine (new phase)

**Goal:** Decide #16 Provider-Side Storage Engine (ADR-023). No papers were previously assigned for Phase 2C.

**Why this phase exists:** ADR-023 is blocked on research but had no assigned reading material. The decision is between an LSM-tree (write-optimised, low read amplification with a bloom filter layer) and an object store or flat-file model (simpler, appropriate for read-heavy cold storage). Two papers close it.

| # | Title | Type | Priority | Topics | Key extraction target | Why added |
|---|---|---|---|---|---|---|
| 1 | The Log-Structured Merge-Tree — O'Neil, Cheng, Gawlick, O'Neil, Acta Informatica 1996 | Paper | MANDATORY | #16 | Foundational LSM design: tiered compaction, bloom filters for negative-lookup elimination, read/write amplification analysis. Gives the theoretical basis for evaluating whether an LSM is appropriate for 256 KB chunks on provider desktop disks. | **New addition.** ADR-023 had no reading material. The LSM-tree is the primary alternative to a flat object store for chunk indexing. This paper provides the baseline theory. Focus: Section 3 (cost analysis) and Section 5 (bloom filter integration). |
| 2 | WiscKey: Separating Keys from Values in SSD-Conscious Storage — Lu, Pilkandla, Arpaci-Dusseau, FAST 2016 | Paper | MANDATORY | #16 | LSM adapted for large values: separates key index (LSM) from value log (append-only), reducing write amplification for value sizes ≥ 1 KB. At 256 KB chunks this is the correct architecture. Gives measured throughput for random reads and writes at our value size range. | **New addition.** WiscKey directly parameterises ADR-023. Our chunk size (256 KB) is exactly in the range where WiscKey's separation strategy outperforms standard RocksDB. Section 3 (design) and Section 5 (evaluation with value sizes 1 KB–256 KB) are the key targets. |

---

## Phase 3 — Key management (new phase)

**Goal:** Decide #10 Key Management Strategy (ADR-020). Cannot begin until ADR-019 and ADR-022 are both Accepted.

**Why this phase exists:** ADR-020 was marked "Phase 3" with no assigned reading. The central problem — zero-knowledge storage where the service never holds keys but key loss is a catastrophic failure — has been solved in production exactly once, in Tahoe-LAFS.

| # | Title | Type | Priority | Topics | Key extraction target | Why added |
|---|---|---|---|---|---|---|
| 1 | Tahoe: The Least-Authority Filesystem — Wilcox-O'Hearn et al., USENIX Security 2008 | Paper | MANDATORY | #9, #10 | Capability-based key management: read capabilities, write capabilities, and verify capabilities are separately delegable without the storage server ever seeing plaintext. Key recovery via distributed erasure of the master secret (m-of-n threshold). Addresses Q07-5 (access control for selective sharing) simultaneously. | **New addition.** Tahoe-LAFS is the only deployed system that solved the exact problem: zero-knowledge distributed storage with provider-side key isolation, key delegation, and recovery. ADR-020 cannot be responsibly decided without reading it. Section 3 (capability model) and Section 4 (implementation) are the primary targets. Section 5 (security analysis) is mandatory for the zero-knowledge argument. |

---