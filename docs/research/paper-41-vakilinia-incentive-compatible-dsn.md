## Paper 41 — An Incentive-Compatible Mechanism for Decentralized Storage Network

**Authors:** Iman Vakilinia, Weihong Wang, Jiajun Xin — University of North Florida; Hong Kong University of Science and Technology
**Venue / Year:** arXiv:2208.09937 | August 2022
**Topics:** #2, #18, #19
**ADRs produced:** none — ADR-002 and ADR-014 confirmed; ADR-024 gains one new gap identification; see Decisions Influenced

---

### Problem Solved

Existing DSNs (Filecoin, Sia, Storj) require continuous microservice- or network-issued proof-of-storage verification at every epoch regardless of whether the client reports any problem. This imposes energy and cost overhead proportional to the number of providers times audit frequency, and it is still vulnerable to a service-denial attack where a provider passes the periodic network proof while refusing to actually return data to the client when requested. This paper models the storage contract as a repeated dynamic game between a client and a provider, proves that the subgame-perfect equilibrium is {Share, No Challenge} when payoffs are set correctly, and demonstrates that challenge-on-demand (client-triggered, not continuously network-triggered) is sufficient to deter dishonest storage while eliminating the service-denial attack. For Vyomanaut, the paper's primary value is identifying the service-denial attack class — a provider that stores data and passes audits but refuses retrieval to clients — as a gap not explicitly addressed in ADR-014, and confirming that Vyomanaut's microservice-issued audit design already escapes the continuous-verification overhead the paper criticises.

---

### Key Findings

**The repeated dynamic game has {Share, No Challenge} as the unique subgame-perfect equilibrium when S3 >> S1 and C3 > C1 (Section 4.1):**
The game tree has six leaf outcomes. Backward induction shows the provider always prefers Proof over No Proof when challenged (No Proof triggers full compensation penalty). Given that, the provider prefers Sharing over Not Sharing when the cost of being successfully challenged (forced to provide proof, lose the "shortcut" benefit) exceeds the benefit of withholding. The client's dominant strategy is No Challenge when the provider shares, since challenging is costly. When payoffs are set to satisfy the four inequalities (equations 1–4 in the paper), the mechanism is incentive-compatible without continuous verification.

**The service-denial attack is a distinct threat class not prevented by PoS alone (Section 1, Section 4.1):**
A provider can submit a valid Merkle-path proof to the network while refusing to send the actual data to the client. The client gets no data; the provider gets paid; the network records a clean audit. This attack is structurally absent from PoS-only mechanisms. The paper closes it by routing the proof response through the oracle/TTP back to the client — the proof and the data delivery are coupled: a provider cannot pass a challenge without simultaneously delivering the data to the client.

**Merkle path verification is O(log n) in tree height and < 0.2 ms for any file size up to 1 GB (Tables 2–6):**
Verification time is independent of file size and approaches 0.02–0.17 ms across all segment sizes (128B to 128 KB) and file sizes (10 MB to 1 GB). Merkle root calculation and path generation do scale with file size and segment count, but for Vyomanaut's 256 KB fragment size, the numbers are negligible: a 256 KB segment generates a tree of height 1 with a single leaf — Merkle root is just SHA256(chunk_data), matching ADR-017's `response_hash = SHA256(chunk_data || challenge_nonce)` exactly.

**Challenge cost X deters malicious clients from issuing spurious challenges (Section 4.1, equation 4):**
The mechanism requires that challenging is costly for the client (C3 > C1), deterring denial-of-service attacks by clients who would repeatedly challenge a behaving provider. The cost of challenge X must be set high enough to deter abuse but low enough to ensure C_c > C4 when the provider is genuinely not sharing — meaning the client still challenges when they have a legitimate grievance.

---

### Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Client-triggered challenge only (no continuous network PoS) | Periodic network-issued PoS at every epoch | Eliminates continuous verification overhead; service-denial attack is closed by coupling proof to data delivery; requires client to be online to detect non-sharing |
| Oracle network as the verifying intermediary | Fully on-chain verification | Off-chain Merkle root calculation avoids prohibitive on-chain compute costs; oracle introduces a trust assumption (Chainlink is centralised in practice) |
| Merkle tree as the PoS construction | zk-SNARKs, RSA accumulators, bilinear VC | No trusted setup; digest is a single hash (minimal on-chain storage); verification is sub-millisecond |
| Smart contract as TTP | Central server TTP or purely peer-based | Tamper-resistant enforcement without a central party; requires blockchain (not Vyomanaut's chosen architecture) |
| Subgame-perfect equilibrium analysis | Nash equilibrium only | Stronger solution concept — rules out non-credible threats; the backward-induction result is stronger than Paper 40's Nash result |

---

### Breaks in Our Case

- **Paper requires a blockchain smart contract to enforce the payoff structure as a trusted TTP** ≠ **Vyomanaut uses a hardened microservice as the coordination and verification layer; no blockchain is used (ADR-001, ADR-015)**
→ The microservice plays the role of the TTP/oracle that the paper assigns to the smart contract + Chainlink oracle combination. The four payoff inequalities are enforced by the microservice's audit logic and escrow engine (ADR-024) rather than by smart contract code. The game-theoretic result is identical — the subgame-perfect equilibrium {Share, No Challenge} is achieved when the microservice correctly implements the penalty structure. The specific implementation substrate (Ethereum vs. Postgres + Razorpay) is irrelevant to the equilibrium analysis.
- **Paper's challenge is client-triggered only — the provider is audited only when the client complains** ≠ **Vyomanaut issues microservice-driven audit challenges at a uniform 24-hour frequency regardless of client complaints (ADR-002, ADR-006)**
→ Vyomanaut's design is strictly stronger than the paper's mechanism. The paper proves that client-triggered-only auditing is sufficient for incentive compatibility. Vyomanaut additionally audits proactively, which raises the effective inspection probability pau from near-zero (client-only) toward 1 (daily audit on every chunk). As Paper 37 (SHELBY Theorem 1) establishes, pau ≈ 1 makes the Nash conditions trivially satisfied for any positive penalty tau. Vyomanaut's continuous audit model is not required by game theory but adds robustness against providers who rely on clients failing to notice non-sharing.
- **Paper's service-denial attack assumes the network PoS (proof submitted to DSN nodes) is decoupled from client data delivery** ≠ **Vyomanaut's audit challenge response is a hash computed directly from chunk_data, and delivery to the client is a separate P2P retrieval step**
→ This is a genuine gap. The paper's mechanism closes the service-denial attack by having the oracle forward the data to the client as part of the proof response — the provider cannot pass the proof without simultaneously delivering the data. Vyomanaut's PoR audit (ADR-002) verifies storage via SHA256(chunk_data || challenge_nonce) but does not couple this to a data retrieval event. A Vyomanaut provider could pass all audits (data is stored, hash response is valid) but selectively refuse to respond to client retrieval requests over the libp2p P2P layer (ADR-021). ADR-014 does not enumerate this attack class explicitly. The mitigation in Vyomanaut is indirect: the reliability score (ADR-008) would fall if retrieval failures were monitored, but the current system audits storage presence, not retrieval willingness. This is identified as a new gap — see Q41-1.
- **Paper assumes a single client and single provider per contract** ≠ **Vyomanaut distributes one file across 56 providers, each holding one RS shard, with 16 required for reconstruction (ADR-003)**
→ The paper's game tree is a bilateral contract. In Vyomanaut the data owner retrieves from any k=16 of 56 providers independently. A service-denial attack by one provider out of 56 does not prevent reconstruction — the data owner has 55 other providers to contact. The attack only becomes effective if > 40 providers (all 56 minus the k=16 reconstruction threshold) simultaneously refuse retrieval. This is bounded by the 20% ASN cap (ADR-014 Defence 1), which limits correlated groups to ~11 providers — well below the 40-provider threshold needed for denial to be effective.
- **Paper implements the mechanism on Ethereum with Chainlink oracle, running gas costs of ~192k units per challenge** ≠ **Vyomanaut has no blockchain layer; challenge cost is a microservice API call**
→ The gas cost numbers in Table 1 are specific to the Ethereum implementation and have no bearing on Vyomanaut's architecture. The microservice challenge dispatch is effectively zero-cost compared to on-chain execution.

---

### Decisions Influenced

- **ADR-002 [#2 Proof of Storage]** `CONFIRMED WITH ATTACK GAP IDENTIFIED`
The paper confirms that Merkle-path-based PoS with challenge-response is the correct design for lightweight auditing in a DSN. The formal game-theoretic proof that client-triggered challenge-only is sufficient for incentive compatibility validates that Vyomanaut's microservice-driven challenge approach (which is strictly more frequent) is more than sufficient. The paper also confirms that Merkle path verification at 256 KB segment size completes in under 0.03 ms — well within the 614 ms audit deadline. However, the paper identifies the service-denial attack class: a provider passing PoS challenges while refusing retrieval. This attack class is not enumerated in ADR-002 or ADR-014. Vyomanaut's structural mitigation (RS(16,56) means > 40 providers must refuse simultaneously for an effective denial) bounds the risk, but the attack is not explicitly addressed at the protocol level. Q41-1 tracks this.
*Because:* Section 1 and Section 4.1 formally prove the service-denial attack is possible in any PoS-only system where proof submission is decoupled from data delivery to the client. The paper's coupling mechanism (oracle forwards data as part of proof response) is the specific fix Vyomanaut does not implement.
- **ADR-014 [#19 Adversarial Defences]** `GAP IDENTIFIED — SERVICE-DENIAL NOT ENUMERATED`
ADR-014 enumerates four adversarial provider behaviour classes: Honest Geppetto (correlated placement), outsourcing (reconstruct on demand), just-in-time retrieval, and false audit responses. The paper identifies a fifth class: service-denial (store data and pass audits, but refuse retrieval to the data owner). Vyomanaut's RS(16,56) redundancy provides a structural bound — an effective service-denial attack requires > 40 providers to refuse simultaneously, which the 20% ASN cap bounds at ~11 correlated providers. The attack is therefore infeasible at the network level even if individual providers attempt it. This finding does not require an ADR change, but the gap should be documented as a known-accepted risk in ADR-014's open constraints. Q41-1 captures the open question of whether client-retrieval monitoring should be added to the reliability scorer.
*Because:* Section 1's explicit identification of service-denial as a distinct attack class separate from data deletion or false PoS is new information relative to what ADR-014 was written against.
- **ADR-024 [#18 Economic Mechanism]** `CONFIRMED — PAYOFF CONDITIONS ALREADY SATISFIED`
The paper's four payoff inequalities (equations 1–4) map cleanly to Vyomanaut's escrow design. Equation 1 (provider prefers any outcome over No Proof, i.e. escrow seizure is the worst outcome) is satisfied: seizure of the full 30-day rolling escrow window is the worst provider outcome (ADR-024). Equation 2 (Proof dominates No Proof when challenged) is satisfied by the same seizure mechanism. Equation 3 (cost of Proof when challenged exceeds benefit of sharing honestly) maps to Vyomanaut's audit-pass payout being lower than the escrow balance at risk. Equation 4 (challenging is costly for the client) is not a Vyomanaut constraint — the data owner does not submit challenges; the microservice does. The bilateral game structure does not fully apply to Vyomanaut's architecture because the challenger and the payee are different entities (microservice vs data owner). The result is that the paper's design conditions reduce to the participation and honesty constraints already confirmed by Paper 37 (SHELBY Theorem 1).
*Because:* Section 4.1's backward induction proof using the four payoff inequalities is the same game-theoretic argument as Paper 37's Theorem 1 expressed in different notation. Both reach the same conclusion: given a trusted verifier (microservice or oracle) with a credible penalty mechanism (escrow seizure or collateral slash), honest storage is the subgame-perfect equilibrium.

---

### Disagreements

- **Paper advocates for client-triggered-only challenges as sufficient, avoiding continuous network verification:** This is a valid finding for a system where the client is always online and capable of detecting non-sharing. For Vyomanaut's cold archive use case, data owners may not access their data for months — relying on the data owner to trigger challenges would leave stored data unverified for the entire absence period.
*Implication for us:* Vyomanaut's microservice-driven continuous audit is the correct design for cold storage specifically because the data owner cannot be assumed to be monitoring. The paper's client-triggered model is sufficient for interactive storage but not for archival.
- **Paper routes proof response through the oracle/TTP back to the client to close the service-denial attack:** This design couples every proof event to a data delivery event, which is expensive — every audit requires the provider to retransmit the challenged segment to the oracle.
*Implication for us:* Vyomanaut's audit challenge uses a hash response (SHA256(chunk_data || nonce)) rather than data retransmission. This is more efficient but does not close the service-denial attack at the audit level. The RS(16,56) redundancy closes it structurally instead — a more elegant solution that scales better.

---

### Open Questions
See open-questions.md — questions Q41-1 and Q41-2.