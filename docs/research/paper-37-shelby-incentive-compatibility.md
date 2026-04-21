## Paper 37 — Rationally Analyzing SHELBY: Proving Incentive Compatibility in a Decentralized Storage Network

**Authors:** Michael Crystal (Stanford University), Guy Goren (Aptos Labs), Scott Duke Kominers (Harvard University and a16z crypto)
**Venue / Year:** arXiv:2510.11866 | October 2025
**Topics:** #2, #18, #19
**ADRs produced:** none — ADR-002 and ADR-024 confirmed; see Decisions Influenced

---

### Problem Solved

Decentralized storage protocols consistently make claims of incentive compatibility but almost none prove them formally. Without proof, a system that distributes storage across many nodes may still fail to decentralize it in any meaningful sense, because self-interested providers have a dominant strategy to shirk. This paper provides the first formal game-theoretic proof that a deployed DSN protocol — SHELBY, from Aptos Labs and Jump Crypto — achieves full incentive compatibility under natural parameter conditions. The core mechanism: infrequent but perfectly verifiable on-chain audits of auditors discipline high-frequency cheap off-chain peer audits. For Vyomanaut, the paper's primary value is establishing a formal proof that the equilibrium Vyomanaut's economic design relies on actually exists, and providing the exact parameter conditions (Theorem 1) that ADR-024's escrow calibration must satisfy to be incentive-compatible.

---

### Key Findings

**Proposition 1 — Off-chain audits alone collapse to universal shirking:**
If the audit mechanism relies solely on off-chain peer audits with no trusted backstop, the unique pure-strategy Nash equilibrium is full dishonesty: every storage provider stores nothing, performs no audits, and reports all other providers as having passed. The proof is tight — reporting 1 for others is strictly dominant (free reward, no cost), so all providers do it; given that all pass regardless, there is no incentive to store data or audit. This is the formal statement of the "who audits the auditor" problem. Any P2P storage system that relies only on peer-issued audit reports without a trusted verifier operates in this equilibrium.

**Observation 1 — Storing dominates on-the-fly reconstruction when fst × k × cr > cst:**
A provider who skips storage might attempt to reconstruct a chunk on demand during a challenge by fetching k chunks from k other providers. This is only viable if the reconstruction cost is less than the storage cost. With k=10, fst > 3 audits/chunk/month, and cst/cr ≈ 2.5 (SHELBY's real-world parameters), storing always dominates reconstruction. The condition fst × k × cr > cst is the formal analogue of Vyomanaut's response deadline constraint.

**Theorem 1 — Three conditions for honest equilibrium uniqueness:**
With a backstop of occasional perfectly verifiable on-chain audit-of-auditor inspection (probability pau per reported success), the honest strategy is the unique Nash equilibrium if and only if:
(i) slashing discourages false reporting: tau ≥ ((1 − pau)/pau) × rau + (1/(ε × pau)) × cau
(ii) audit rewards outweigh audit costs: rau ≥ (1/(1 − ε)) × cau
(iii) storage rewards exceed storage costs: rst ≥ cst

Condition (iii) is the participation constraint. Conditions (i) and (ii) are the honesty constraints on the auditing layer. All three must hold jointly for the system to be incentive-compatible.

**Theorems 2 and 3 — Coalition resistance up to N/2:**
Without commitment power among coalition members, the honest strategy is robust against any coalition of fewer than N/2 providers — no joint deviation can improve all members' payoffs. With commitment power, a coalition can avoid internal auditing costs but cannot suppress data storage, because non-coalition members (the majority) will audit truthfully and honest audits cannot be faked. The maximum collective gain from a coalition of size |J| with commitment power is bounded by |J|² × cau — a function of wasted audit costs, not stolen storage rewards.

**Proposition 3 — Vector commitments close the commitment-power gap:**
Adding a vector commitment over successful audit responses to the on-chain audit submission eliminates the residual coalition gain even when coalition members have full commitment power. A false 1 report that cannot be backed by a stored inclusion proof is slashed on inspection. This modification adds minimal overhead (one Merkle root per audit report) and converts the strong equilibrium from "near honest" to "exactly honest."

**Back-of-envelope calibration:**
Using SHELBY's real-world parameters (rst ≈ $30/TB/month, cst ≈ $5/TB/month, cr ≈ $2/TB, pau ≈ 2×10⁻⁴, tau ≳ $1000), the conditions in Theorem 1 are comfortably satisfied with large margins. The expected penalty for a false 1 report (pau × tau ≈ $0.2) exceeds the reward (rau ≈ $5×10⁻⁷) by five orders of magnitude. This is not a tight boundary — the design operates deep in the safe regime.

---

### Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Occasional on-chain audit-of-auditor verification | Fully on-chain auditing of every storage event | Drastically lower verification cost; incentive compatibility achieved at reasonable cost; requires calibrating tau, pau correctly |
| Peer-to-peer off-chain audits backstopped by on-chain spot checks | Trusted central auditor | Removes need for trusted auditing oracle; replaces it with a need for a trusted on-chain verification mechanism |
| Majority voting on peer audit reports | Single auditor authority | Sybil-resistant report aggregation; collusion is bounded but not eliminated below N/2 threshold |
| Nash equilibrium as solution concept | Dominant-strategy incentive compatibility | Achievable at reasonable cost; dominant strategy remains an open goal noted by the authors |
| Static model | Dynamic reputation model | Within-period guarantees without relying on reputation; dynamic strategy space not captured |

---

### Breaks in Our Case

- **SHELBY uses a blockchain as the trusted on-chain verifier — the verification is fully public and trustless** ≠ **Vyomanaut's "on-chain" equivalent is the hardened microservice, which is trusted but not publicly verifiable in V2**
→ The microservice plays the role of the on-chain verifier in Theorem 1. For the incentive-compatibility conditions to hold, the microservice must function as the incorruptible backstop — it must be unable to be bribed or coopted by a coalition of providers. The (3,2,2) quorum (ADR-025) and append-only audit log (ADR-015) are the structural guarantees that make the microservice function as a credible verifier. The V3 Transparent Merkle Log upgrade (ADR-015) would bring Vyomanaut closer to SHELBY's public verifiability guarantee.
- **SHELBY's providers audit each other directly — every provider acts as both auditee and auditor** ≠ **Vyomanaut's audit challenges are issued exclusively by the microservice; providers never rate each other**
→ Proposition 1's collapse applies to systems where peer-issued reports are the only mechanism. Vyomanaut avoids this entirely: the microservice is the sole auditor. This is a structural escape from Proposition 1, not a parameter calibration — the auditor's honesty is guaranteed by the microservice's architecture (single authoritative scorer, ADR-008), not by slashing conditions on peer auditors. The downside is that the microservice itself must be trusted; Proposition 1 applies if the microservice is compromised.
- **SHELBY's reconstruction cost argument (Observation 1) uses k=10 chunks from k different providers** ≠ **our response deadline enforces the same constraint through timing rather than cost**
→ Both mechanisms achieve the same goal: making on-the-fly reconstruction during an audit challenge economically or temporally infeasible. SHELBY's approach is cost-based (fst × k × cr > cst). Vyomanaut's approach is deadline-based: `(chunk_size / declared_upload_speed) × 1.5`. The timing constraint is weaker than a cryptographic constraint (Seal) but stronger than a pure cost argument on high-bandwidth connections.
- **SHELBY's slashing is enforced on-chain automatically and transparently** ≠ **Vyomanaut's escrow seizure is enforced by the payment microservice at the 72h departure threshold**
→ The slashing penalty tau in Theorem 1 maps to Vyomanaut's held-earnings seizure. The key difference is timing: SHELBY slashes per failed audit proof during the epoch; Vyomanaut seizes all held earnings at the 72h departure threshold. Whether Vyomanaut's threshold-based binary seizure satisfies the condition tau ≥ ((1 − pau)/pau) × rau requires explicit numerical verification (Q37-2).
- **SHELBY's model has N providers each independently auditing and storing, all in a single epoch** ≠ **Vyomanaut has a staged model: vetting (4–6 months), then full participation, with a graduated reliability score**
→ The paper's static model applies cleanly to Vyomanaut's post-vetting steady state. The vetting period is an additional entry cost not modelled in the paper, which further raises the cost of Sybil-style identity attacks beyond what Theorem 1's conditions require.

---

### Decisions Influenced

- **ADR-002 [#2 Proof of Storage]** `CONFIRMED`
Proposition 1 provides the formal justification for why PoR challenges backed by a trusted verifier are necessary and not merely sufficient. Without the microservice as a trusted backstop verifying challenge responses, the unique equilibrium is every provider faking every audit — a provable collapse, not an edge case. The current design (randomised Merkle challenge → response must equal SHA256(chunk_data || nonce) → microservice verifies) is the minimum architecture that escapes this collapse. The response deadline (ADR-014 Defence 2) implements Observation 1's cost argument in the timing domain.
*Because:* Proposition 1's proof shows that ρ = 1 for all j is strictly dominant regardless of actual auditing — the collapse is inevitable unless a trusted verification backstop exists. The microservice's challenge-response verification is that backstop.
- **ADR-014 [#19 Adversarial Defences]** `CONFIRMED`
Observation 1 formally proves that the outsourcing attack (a provider reconstructs data on the fly during an audit rather than storing it) is deterred when fst × k × cr > cst. Vyomanaut's response deadline `(chunk_size / declared_upload_speed) × 1.5` enforces the timing equivalent of this condition. The paper shows that storing dominates reconstruction by a factor of ~16× at SHELBY's parameters; Vyomanaut's audit frequency (one per 24h) and response deadline provide an analogous structural deterrent.
*Because:* Observation 1 quantifies exactly why timing-based outsourcing prevention works — the cost of reconstruction exceeds the cost of storage when the audit frequency is high enough. Our 24h audit cycle satisfies the spirit of this condition for desktop providers at our declared upload speeds.
- **ADR-024 [#18 Economic Mechanism]** `PARAMETER CONDITIONS ADDED`
Theorem 1's three conditions are directly applicable to Vyomanaut's escrow calibration. Mapping the paper's parameters to Vyomanaut:
    - tau = escrow seized at 72h departure (held-earnings balance at time of seizure)
    - pau = probability of microservice challenge on any given chunk per polling cycle ≈ 1 (microservice challenges every chunk every 24h, not probabilistically in V2)
    - rau = per-audit-pass earning (ADR-012 payout per audit passed)
    - rau ≥ cau is satisfied trivially since audit response cost (disk read + hash computation) is negligible compared to the per-audit payout
    - rst ≥ cst is condition (iii): the monthly storage earnings rate must exceed the provider's marginal storage cost per chunk, which ADR-024's storage rate is designed to satisfy
    - Condition (i) at pau ≈ 1 simplifies to: tau ≥ rau × (1 − pau)/pau ≈ 0, meaning the seizure amount does not need to be large relative to per-audit earnings when pau is near 1. This is a significantly weaker requirement than SHELBY faces at pau = 2×10⁻⁴.
    *Because:* When audit verification is deterministic (every challenge verified, not probabilistic), the conditions for incentive compatibility collapse to the participation constraint alone: rst ≥ cst. Vyomanaut's design with daily audit challenges on every chunk approaches this deterministic limit. The 30-day rolling escrow window provides a large tau buffer well above any minimum required by Theorem 1 at pau ≈ 1.
- **ADR-008 [#8 Reliability Scoring]** `CONFIRMED`
The paper's Proposition 1 confirms that the three-window rolling score (ADR-008) in a single authoritative microservice is the only viable architecture: distributed peer scores collapse to the fully dishonest equilibrium. The centralisation of the scorer in the microservice is not a compromise — it is the structural requirement for escaping Proposition 1's Nash equilibrium. The paper independently confirms what ADR-008's design implicitly assumed.
*Because:* Proposition 1's proof demonstrates that any mechanism where providers self-report on each other produces the collapse equilibrium. ADR-008's design where the microservice is the sole issuer and verifier of audit challenges eliminates this failure mode entirely.

---

### Disagreements

- **Filecoin whitepaper (Paper 29) and most other DSN whitepapers:** The paper notes that essentially all existing DSN protocols assert incentive compatibility without formal proof. Filecoin states a desired incentive-compatibility property but does not prove it (Section 1.2). Storj discusses "incentive alignment" informally.
*Implication for us:* Vyomanaut's economic design should be verifiable against Theorem 1's conditions before V2 launch. The conditions are numerically checkable once the storage rate (paise/GB/month) is set as a product decision (ADR-024 open constraint).
- **EigenTrust (Paper 24):** EigenTrust assumes peer-to-peer interaction data exists and that the pre-trusted set anchors convergence. Proposition 1 shows this assumption is insufficient — without a trusted verifier backstop, peer trust scores collapse to all-1 regardless of honesty.
*Implication for us:* The earlier decision to defer EigenTrust-based distributed reputation to V3 (Q07-4, deferred-v3) is further validated. Distributed reputation without a trusted verifier is formally a Nash equilibrium failure.

---

### Open Questions

See open-questions.md — questions Q37-1 and Q37-2.