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
| Q02-3 | How to set branching factor b (accelerated lookups)? | open | libp2p spec — reading-list Phase 1 #8 |
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
| Q04-1 | Which P2P replication protocol: BitSwap, libp2p, QUIC, or alternative? | open | Phase 1 — libp2p spec + RFC 9000 (reading-list Phase 1 #8, #9) |
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
| Q10-4 | How does the optimal r change when provider temporary absence (diurnal churn) is separated from permanent departure? Giroire uses a single α — does splitting into α_temporary and α_permanent change the optimal r derived from Equation (3)? | open | Blocked on: Phase 2A (Erasure Coding papers). Only α_permanent should enter the LossRate formula; α_temporary is handled by timeout t. Whether this changes the optimal r requires rederiving Eq. (3) with a split α. |
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
