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

## Phase 1 — Coordination and storage systems

**Goal:** Decide #1 Coordination Architecture and #2 Proof of Storage.

| # | Title | Type | Priority | Topics | Key extraction target |
|---|---|---|---|---|---|
| 1 | High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two — Blake & Rodrigues, HotOS 2003 | Paper | MANDATORY | #1 | See [paper-06](paper-06-blake-rodrigues.md) ✅ |
| 2 | SoK: Decentralised Storage Network — Li et al., IEEE S&P | Paper | MANDATORY | #1–3, #5, #13 | See [paper-07](paper-07-sok-dsn.md) ✅ |
| 3 | Understanding Availability in Peer-to-Peer Applications — Bhagwan, Savage, Voelker, IPTPS 2003 | Paper | MANDATORY | #5, #6, #8 | See [paper-08](paper-08-bhagwan-availability.md) ✅ |
| 4 | Feasibility of a Serverless Distributed File System Deployed on an Existing Set of Desktop PCs — Bolosky et al., ACM SIGMETRICS 2000 | Paper | RECOMMENDED | #1, #6, #11 | See [paper-09](paper-09-bolosky-feasibility.md) ✅ |
| 5 | P2P Storage Systems: A Practical Guideline to Be Lazy — Giroire et al., GlobeCom 2010 | Paper | MANDATORY | #3, #4, #6, #17 | See [paper-10](paper-10-giroire-lazy.md) ✅ |
| 6 | Coordination Avoidance in Database Systems — Bailis et al., VLDB 2015 | Paper | MANDATORY | #14 | See [paper-11](paper-11-bailis-coordination.md) ✅ |
| 7 | Dynamo: Amazon's Highly Available Key-Value Store — SOSP 2007 | Paper | MANDATORY | #4, #6 | N/R/W quorum model; gossip-based failure detection |
| 8 | libp2p Specification & Architecture | Docs | MANDATORY | #1, #12 | Kademlia implementation + NAT traversal |
| 9 | QUIC: A UDP-Based Multiplexed and Secure Transport — RFC 9000 | RFC | MANDATORY | #12 | Connection migration and multiplexed streams |
| 10 | Separation and Optimization of Encryption and Erasure Coding — Szabó et al., FGCS Feb 2025 | Paper | MANDATORY | #3, #9, #15 | **BLOCKING:** must decide encrypt-then-code vs code-then-encrypt before chunk design. Directly targets ADR-009 and ADR-015. |
| 11 | Filecoin Whitepaper — Protocol Labs, 2017 | Whitepaper | MANDATORY | #1, #2, #13 | Sections 3.1–3.2 only: PoRep and PoSt mechanisms. Skip all token/blockchain sections. |

---

## Phase 2A — Erasure coding mechanism

**Goal:** Decide #3 Erasure Coding and #15 Encryption-Erasure Interaction (if possible).

| # | Title | Type | Priority | Topics | Key extraction target |
|---|---|---|---|---|---|
| 11 | Coral DSHT — Freedman & Mazières, NSDI 2004 | Paper | MANDATORY | #1, #5 | Sloppy DHT: store pointers not values. Geographic clustering for provider selection. Section 3 is the key target. |
| 12 | Design & Evaluation of IPFS — Trautwein et al., SIGCOMM 2022 | Paper | MANDATORY | #1, #5, #6 | Real production measurements: lookup latency, content decay, churn rates. Section 5 (availability decay). |
| 13 | Measurement Study of P2P File Sharing — Saroiu et al., 2002 | Paper | MANDATORY | #3, #5, #6 | Median node session < 1 hour. Table 2–4: actual uplink/downlink distributions. |
| 14 | Survey of the Past/Present/Future of Erasure Coding — Shen, Lee et al., ACM ToS Dec 2024 | Paper | MANDATORY | #3 | Read this first — landscape map for all EC schemes. RS vs LRC vs MSR vs MLEC comparison. |
| 15 | Minimum Storage Regenerating Codes — Dimakis et al., 2010 | Paper | MANDATORY | #3, #17 | **FOUNDATIONAL:** proves the information-theoretic lower bound on repair bandwidth. All subsequent repair papers build on this. |
| 16 | Erasure Codes for Cold Data in Distributed Storage Systems — Chao Yin et al. | Paper | MANDATORY | #3 | Consumer device storage = cold data model. Storage overhead vs reconstruction speed tradeoff. |

---

## Phase 2B — Repair bandwidth optimisation

**Goal:** Decide #4 Replication / Repair Protocol and #17 Repair Bandwidth Optimisation.

| # | Title | Type | Priority | Topics | Key extraction target |
|---|---|---|---|---|---|
| 17 | EigenTrust: Managing Trust in P2P Networks | Paper | MANDATORY | #8 | Global reputation from local interactions |
| 18 | TrustDSN: Reputation Without Blockchain for Distributed Storage | Paper | MANDATORY | #8, #19 | Reputation system without blockchain |
| 19 | Handling Churn in a DHT | Paper | MANDATORY | #1, #6 | DHT churn handling strategies |
| 20 | IRON File Systems — SOSP 2005 | Paper | MANDATORY | #19 | Adversarial provider simulation: partial writes, silent corruption |
