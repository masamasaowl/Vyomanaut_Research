# Paper 29 — Filecoin: A Decentralized Storage Network

**Authors:** Protocol Labs (Juan Benet, Nicola Greco, et al.)
**Venue / Year:** Protocol Labs Whitepaper | July 2017
**Citations:** ~3,500
**Topics:** #2, #13, #18, #19
**ADRs produced:** none — ADR-024 gains concrete SLA enforcement constraints; decision remains Proposed pending Phase 5

---

## Problem Solved

Filecoin addresses the incentive alignment problem in decentralised storage: how do you force a storage provider to actually keep data, not just claim they do, and how do you tie payment to proof rather than to trust? The paper introduces a formal DSN scheme with a fault tolerance model, two novel proof primitives (PoRep and PoSt), a two-sided market (Storage Market + Retrieval Market), and an SLA enforcement mechanism built around graduated collateral penalties. The Manage.RepairOrders protocol — check proofs every Δproof epochs, penalise on each miss, cancel after Δfault consecutive misses — is the direct production-system equivalent of Vyomanaut's audit-fail → score-decrement → escrow-seizure chain, and it provides the specific structural template ADR-024 needs for penalty graduation.

---

## Key Findings

**The (f, m)-tolerant fault model (Section 2.1.2):**
A Put execution is (f, m)-tolerant if data is stored across m independent providers and the system survives f byzantine failures. With erasure coding, f = m − x where x is the reconstruction threshold. Applied to Vyomanaut: m = 56, x = s = 16, f = 40. The paper formalises what ADR-003 implements numerically.

**The Seal operation (Section 3.4.2):**
Seal is a slow, sequential computation applied to provider data during PoRep.Setup. Its design requirement is that running Sealτ must take 10–100× longer than the honest challenge-prove-verify sequence. This time gap makes outsourcing attacks (fetching data on-demand to pass a challenge) computationally infeasible — the provider cannot retrieve and re-seal in time. Vyomanaut does not use Seal (hardware cost eliminates desktop providers), but the functional goal — make just-in-time retrieval infeasible — is implemented by our response deadline of `(chunk_size / upload_speed) × 1.5` (ADR-014 Defence 2).

**PoSt as time-ordered sequential PoRep (Section 3.3–3.4.4):**
Proof-of-Spacetime chains PoRep iterations: the output proof of round i becomes the input challenge of round i+1. This sequentiality prevents parallelisation and proves data was held continuously for t rounds. Vyomanaut rejects PoSt because its hardware requirement (GPU + 256 GB RAM for Seal) eliminates desktop providers. The functional equivalent — randomised periodic Merkle challenges issued by the microservice — achieves continuous-presence verification without sequential ZK proofs.

**Manage.RepairOrders — the SLA enforcement mechanism (Section 4.3.4):**
At every block, the network checks each storage assignment in the AllocationTable:

- Missing or invalid proof → penalise collateral proportionally
- Missing proofs for more than Δfault consecutive epochs → cancel order, re-introduce piece to the Storage Market
- Every storage miner for a piece faulty → piece lost, client refunded

Three parameters govern SLA enforcement: Δproof (audit frequency), Δfault (fault tolerance window), and collateral (amount at risk). These three map directly to Vyomanaut's audit interval (24h), 72h departure threshold, and held-earnings escrow.

**Collateral proportional to storage commitment (Section 4.3.2):**
Miners pledge collateral proportional to sector size before receiving orders. Collateral is lost on missed proofs and returned if all proofs are generated correctly through the contract duration. This is the pre-committed penalty model. Vyomanaut's escrow holds earned-but-not-yet-released payments rather than pre-committed collateral — a behavioural difference with economic consequences: Filecoin's model deters entry of low-MTTF providers (must stake before earning); Vyomanaut's model deters exit of established providers (has something to lose).

**Storage Market settlement phase (Section 5.2.3):**
Payment is released only after the network verifies proof of storage, not after the provider claims delivery. The settlement structure is: order matching (bid/ask) → deal order signed by both parties → proof generation by miner → network verification → payment release. This is the verifiable market design our escrow implements: audit pass = proof, microservice = verifier, escrow release = payment.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| PoRep (physically unique replica per provider) | Simple Merkle PoR | Proves physical independence; eliminates Sybil, outsourcing, and generation attacks; requires 256 GB RAM + GPU — eliminates all desktop users |
| On-chain orderbook | Off-chain matching | Every bid is public and binding; prevents information asymmetry; blockchain becomes the bottleneck for upload frequency |
| Graduated collateral penalty (partial slash per miss, full cancel at Δfault) | Binary seizure at first miss | Proportional deterrence; providers survive transient outages; strategic providers can predict the fault window and exploit it |
| Collateral pledged pre-service (proportional to storage promised) | Collateral held from earnings | Deters low-MTTF entry; creates capital barrier for new providers; effectively limits participation to operators with capital to stake |
| Retrieval Market with per-chunk micropayments | Payment per retrieval completion | Makes partial delivery fair; requires payment channel infrastructure; not feasible without off-chain payment support |

---

## Breaks in Our Case

- **Filecoin uses blockchain as the trust anchor for proof verification and payment release** ≠ **Vyomanaut uses a hardened microservice**
→ The microservice plays the role of the Network in Filecoin's RepairOrders loop. Audit receipts (ADR-015, ADR-017) replace blockchain-stored proofs. The append-only Postgres audit log is the AllocationTable equivalent. No structural change required — the role mapping is exact.
- **Filecoin's Seal requires 10–100× longer computation than honest challenge-prove-verify** ≠ **our desktop providers cannot run GPU-based Seal**
→ Seal is the mechanism that makes outsourcing attacks infeasible in Filecoin. Vyomanaut's functional substitute is the response deadline `(chunk_size / upload_speed) × 1.5` (ADR-014). The guarantee is weaker (timing-based, not cryptographic) but implementable on commodity hardware.
- **Filecoin's Δfault is a count of consecutive missed proof epochs** ≠ **our departure threshold is absolute time (72 h)**
→ Filecoin's Δfault is relative to Δproof: if Δproof = 1 hour and Δfault = 3, the effective window is 3 hours. Our 72-hour threshold is calibrated to Bolosky's bimodal absence distribution (Paper 09), not derived from audit frequency. The timeout-based model is more appropriate for a system where absence is expected and predicted.
- **Filecoin's collateral is pledged proportional to storage before any earning begins** ≠ **our escrow holds a portion of already-earned payments**
→ Pre-commitment (Filecoin) creates a capital barrier that deters low-capital entrants but also legitimate home desktop providers. Held-earnings (Vyomanaut) creates a deterrent to exit once established, without a capital entry barrier. This is the correct model for the India-first target population where providers will not pre-commit capital.
- **Filecoin penalises proportionally per missed proof, then cancels at Δfault** ≠ **Vyomanaut's current model seizes all pending earnings at the 72 h threshold (binary)**
→ Filecoin's graduated approach avoids punishing transient outages severely while still escalating to full cancellation for persistent failures. Whether a graduated intermediate penalty between score decrement and full seizure improves provider retention is open (Q29-1).
- **Filecoin's Retrieval Market pays per data chunk received** ≠ **ADR-012 pays per audit passed, decoupled from retrieval**
→ Filecoin's per-retrieval micropayment ties payment to actual data delivery, not continuous availability. Vyomanaut chose audit-pass payment specifically to decouple the payment path from the P2P transfer path (ADR-012). The Retrieval Market is not adopted.

---

## Decisions Influenced

- **ADR-024 [#18 Economic Mechanism]** — `PARTIALLY INFORMED, REMAINS PROPOSED`
Three concrete constraints from Filecoin's SLA enforcement model now apply to ADR-024:
(1) **Penalty graduation:** Filecoin penalises proportionally per missed proof before cancellation. ADR-024 must specify whether Vyomanaut uses binary seizure at 72h or a graduated model (warn at 48h, partial penalty at 60h, full seizure at 72h). The choice affects provider retention for providers with legitimate transient outages.
(2) **Collateral/escrow sizing:** Filecoin's collateral is proportional to storage committed. For Vyomanaut's held-earnings model, this maps to: the fraction of earnings held in escrow should scale with the number of chunks assigned, not be a flat percentage. A provider holding 10,000 chunks has more at stake than one holding 100.
(3) **Repair re-introduction:** Filecoin re-introduces a cancelled order at the current market price, not the original contract price. Vyomanaut's repair scheduler (ADR-004) assigns replacement chunks to new providers from the vetted pool — structurally equivalent. The repair cost (bandwidth + new provider onboarding) is borne by the escrow seized from the departed provider.
ADR-024 still needs: PeerTrust (Phase 2B #4 replacement) for reputation-weighted pricing; Ihle et al. (Phase 5 #3) for general incentive mechanism design.
*Because:* Filecoin is the only deployed system that has formalized incentive-compatible DSN contracts with a gradated penalty structure at scale.
- **ADR-002 [#2 Proof of Storage]** `CONFIRMED`
Section 3.1 provides Filecoin's own motivation: PoR-class proofs (the mechanism we use) are insufficient to prevent Sybil, outsourcing, and generation attacks in an open network. Filecoin designed PoRep to close these gaps. We accept the weaker PoR guarantee because we close the same attack classes through other means: Sybil via registration gating (ADR-001), outsourcing via response deadline (ADR-014), generation via content-address verification (SHA256(chunk || nonce)). The design is explicitly weaker than PoRep and that is the accepted trade-off.
*Because:* The paper names the three attack classes PoRep was built to prevent, confirming that our non-PoRep mitigations must address all three explicitly.
- **ADR-014 [#19 Adversarial Defences]** `CONFIRMED`
The Seal operation (Section 3.4.2) is Filecoin's primary defence against outsourcing attacks. The design requirement — Seal takes 10–100× longer than honest challenge-prove-verify — is functionally equivalent to our response deadline constraint: `(chunk_size / declared_upload_speed) × 1.5`. Both mechanisms make JIT retrieval infeasible by making the time budget for retrieval shorter than the retrieval round trip. The paper validates the design goal; our mechanism achieves it without hardware dependence.
*Because:* Section 3.4.2 explicitly states the timing constraint requirement for outsourcing attack prevention, which is the same constraint ADR-014 Defence 2 implements via the deadline formula.
- **ADR-007 [#7 Provider Exit States]** `CONFIRMED`
Filecoin's Manage.RepairOrders three-case logic — (1) miss → penalise, (2) Δfault misses → cancel and repair, (3) all miners faulty → client refunded — maps exactly to Vyomanaut's four exit states. Accidental absence = case 1. Permanent silent departure at 72h = case 2. The "all miners faulty" case maps to what our r0=8 redundancy threshold prevents: repair is triggered at 24 available fragments (s+r0), not at s=16 (reconstruction floor), so the "piece lost" scenario is structurally avoided before it can occur.
*Because:* The RepairOrders escalation structure independently validates the three-tier response: monitor → penalise → repair, which is the design principle behind our exit state machine.

---

## Disagreements

- **Filecoin's blockchain-based public verifiability:** Any full node can verify any storage proof at any time without trusting the network operator. Vyomanaut's V2 signed receipt system (ADR-015) requires trusting the microservice countersignature in V2; the V3 Transparent Merkle Log upgrade is needed to achieve public verifiability.
*Implication for us:* The V3 Merkle Log (ADR-015) is the correct path toward Filecoin-equivalent public dispute resolution. This is a known gap in V2, not an architectural flaw.
- **Filecoin's per-retrieval micropayment model:** Retrieval Miners are paid per data block received, not per continuous availability. This directly ties earnings to actual utility delivered to the client.
*Implication for us:* ADR-012 chose per-audit-passed payment specifically because per-retrieval payment requires coupling the payment path to the P2P transfer layer — the exact coupling Swarm's SWAP failed to sustain. The Filecoin model works when the retrieval market is off-chain gossip-based; ours requires the microservice to remain decoupled from transfers.

---

## Open Questions

See open-questions.md — questions Q29-1 and Q29-2.