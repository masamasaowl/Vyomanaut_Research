# Reading List — Updated April 2026

All phases before Phase 2B were completed. The updated list reflects true status and adds the new papers found during the April 2026 review.

---

## Phase 0 — Landscape survey ✅ Complete

| # | Paper | Status |
|---|---|---|
| 1 | BitTorrent — Bram Cohen | ✅ Paper 01 |
| 2 | Kademlia — Maymounkov & Mazieres | ✅ Paper 02 |
| 3 | S/Kademlia — Baumgart & Mies | ✅ Paper 03 |
| 4 | IPFS — Juan Benet | ✅ Paper 04 |
| 5 | Storj V3 Whitepaper | ✅ Paper 05 |

---

## Phase 1 — Coordination, storage, and foundational protocols ✅ Complete

| # | Paper | Status |
|---|---|---|
| 1 | Blake & Rodrigues (HotOS 2003) | ✅ Paper 06 |
| 2 | SoK: DSN — Li et al. | ✅ Paper 07 |
| 3 | Bhagwan et al. — Availability (IPTPS 2003) | ✅ Paper 08 |
| 4 | Bolosky et al. — Feasibility (SIGMETRICS 2000) | ✅ Paper 09 |
| 5 | Giroire et al. — Lazy Repair (GlobeCom 2010) | ✅ Paper 10 |
| 6 | Bailis et al. — Coordination Avoidance (VLDB 2015) | ✅ Paper 11 |
| 7 | Dynamo — DeCandia et al. (SOSP 2007) | ✅ Paper 12 |
| 8 | libp2p Specification | ✅ Paper 13 |
| 9 | RFC 9000 — QUIC | ✅ Paper 14 |
| 10 | Szabó et al. — Encrypt-Erasure Separation (FGCS 2025) | ✅ Paper 15 |
| 11 | AONT-RS — Resch & Plank (FAST 2011) | ✅ Paper 16 |
| 12 | RFC 8439 — ChaCha20-Poly1305 | ✅ Paper 17 |
| 13 | Tahoe-LAFS — Wilcox-O'Hearn et al. (USENIX Security 2008) | ✅ Paper 18 |
| 14 | Filecoin Whitepaper — Protocol Labs (2017) | ⬜ READ NEXT — Economic mechanism prerequisite for ADR-024. Skip all blockchain/token/PoRep sections. Read only §3 (economic model) and §4 (SLA enforcement). |

---

## Phase 2A — Erasure coding, peer measurement, geographic routing ✅ Complete

| # | Paper | Status |
|---|---|---|
| 1 | EC Survey — Shen et al. (ACM ToS 2025) | ✅ Paper 19 |
| 2 | IPFS Measurement — Trautwein et al. (SIGCOMM 2022) | ✅ Paper 20 |
| 3 | Gnutella/Napster Measurement — Saroiu et al. (MMCN 2002) | ✅ Paper 21 |
| 4 | MSR Codes for All Parameters — Goparaju et al. (arXiv 2016) | ✅ Paper 22 |
| 5 | Cold Data Erasure Codes — Yin et al. (Applied Sciences 2023) | ✅ Paper 23 |
| 6 | Coral DSHT — Freedman & Mazières (NSDI 2004) | DEFERRED V3 — geographic routing, not V2 blocking |

---

## Phase 2B — Reputation, adversarial behaviour, DHT churn ⚠️ Partial

| # | Paper | Status | Priority |
|---|---|---|---|
| 1 | EigenTrust — Kamvar et al. (WWW 2003) | ✅ Paper 24 | — |
| 2 | Handling Churn in a DHT — Rhea et al. (USENIX ATC 2004) | ⬜ READ NEXT | HIGH — validates ADR-006 polling interval; confirms Kademlia preference for long-lived contacts aligns with incentivized provider behaviour |
| 3 | IRON File Systems — Prabhakaran et al. (SOSP 2005) | ✅ Paper 25 | — |
| 4 | TrustDSN | ❌ DOES NOT EXIST — see replacement below | — |

**Phase 2B #4 replacement:** "PeerTrust: Supporting Reputation-Based Trust for Peer-to-Peer Electronic Communities" — Xiong & Liu, IEEE Transactions on Knowledge and Data Engineering, Vol. 16, No. 7, 2004. This is the foundational non-blockchain reputation paper that pre-dates EigenTrust's production deployments. Key extraction target: How reputation is aggregated from local transaction feedback without a central authority; how new node bootstrapping is handled; how collusion attacks are mitigated. Feeds ADR-024 (economic mechanism: penalty structures, vetting period lengths, held-earnings rationale). Source: IEEE Xplore, doi:10.1109/TKDE.2004.1318566

---

## Phase 2C — Provider-side storage engine ✅ Complete

| # | Paper | Status |
|---|---|---|
| 1 | LSM-Tree — O'Neil et al. (Acta Informatica 1996) | ✅ Paper 26 |
| 2 | WiscKey — Lu et al. (FAST 2016) | ✅ Paper 27 |

---

## Phase 3 — Key management ✅ Complete

| # | Paper | Status |
|---|---|---|
| 1 | Tahoe-LAFS — Wilcox-O'Hearn et al. | ✅ Paper 18 (read in Phase 1) |

---

## Phase 4 — NEW: Production NAT traversal and disk failure data

These two papers were not in the original plan but directly answer blocking open questions.

| # | Paper | Priority | Why |
|---|---|---|---|
| 4-1 | "Challenging Tribal Knowledge — Large-Scale Measurement Campaign on Decentralized NAT Traversal" — Balduf et al. (arXiv 2025, arxiv.org/abs/2510.27500) | HIGH | First large-scale DCUtR study: 4.4M traversal attempts, 70% hole-punch success, TCP=UDP under DCUtR. Answers Q13-1 (relay overhead for symmetric-NAT providers) and Q20-1 (Indian router NAT type distribution). Must be read before the libp2p integration module is built. |
| 4-2 | "Flash Reliability in Production: The Expected and the Unexpected" — Schroeder, Lagisetty, Merchant (FAST 2016, usenix.org/conference/fast16/...) | MEDIUM | Empirical latent sector error and spatial failure correlation rates in consumer SSDs and HDDs. Answers Q25-1 (proactive scrubbing) and Q25-2 (sparse vLog placement necessity). Must be read before the WiscKey storage engine ships to production. |

---

## Phase 5 — Economic mechanism ❌ Blocked on Phase 1 #14 (Filecoin) + Phase 2B #4 replacement (PeerTrust)

| # | Paper | Status | Why |
|---|---|---|---|
| 5-1 | Filecoin Whitepaper §3–4 | ⬜ Prerequisite from Phase 1 | Pricing model and SLA enforcement |
| 5-2 | PeerTrust — Xiong & Liu (IEEE TKDE 2004) | ⬜ Phase 2B #4 replacement | Reputation aggregation, penalty structure design |
| 5-3 | "Incentive Mechanisms in Peer-to-Peer Networks — A Systematic Literature Review" — Ihle et al. (ACM Computing Surveys 2023, doi:10.1145/3578581) | ⬜ NEW | Comprehensive survey of monetary, reputation, and service incentives across P2P network types. Directly feeds ADR-024 parameter space: minimum vetting period, held-earnings percentage, penalty structure design. More directly applicable to Vyomanaut than TrustDSN would have been. |

After reading Phase 5, ADR-024 can be written and the pricing model finalised.

---