# Paper 24 — The EigenTrust Algorithm for Reputation Management in P2P Networks

**Authors:** Sepandar D. Kamvar, Mario T. Schlosser, Hector Garcia-Molina — Stanford University
**Venue / Year:** WWW 2003 | Budapest, Hungary
**Topics:** #8, #19
**ADRs produced:** none — ADR-008 confirmed; Q07-4 answered (deferred-v3); see Decisions Influenced

---

## Problem Solved

P2P file-sharing networks have no accountability for what a peer uploads: malicious peers flood the network with inauthentic files and the network has no way to distinguish them from honest ones. Prior reputation systems either aggregated ratings from only a few neighbours (narrow view) or flooded the network with queries for every peer's opinion (congestion). EigenTrust solves both problems: it proves that a peer's global reputation is the left principal eigenvector of a matrix of normalised local trust values, and that this eigenvector can be computed in a distributed, gossip-like manner in fewer than 10 iterations on a 1000-peer network. For Vyomanaut, the paper closes Q07-4 — it confirms that distributed reputation propagation without a central coordinator is theoretically achievable, and simultaneously reveals why the specific mechanism cannot be applied to V2's centralised-audit architecture.

---

## Key Findings

**The core algorithm (Section 4):**
Each peer i scores its interactions with peer j as `sij = sat(i,j) − unsat(i,j)`. Normalised: `cij = max(sij,0) / Σ max(sij,0)`. Global trust vector `t` is the left principal eigenvector of matrix C: `t = lim(n→∞) (C^T)^n p`, where `p` is a distribution over pre-trusted peers. In practice this converges in fewer than 10 iterations.

**Pre-trusted peers are mandatory (Section 4.5):**
Without a pre-trusted set `p`, a malicious collective forms a closed loop in the trust graph. A random surfer entering the collective cannot escape. Adding `p` as a floor — `t^(k+1) = (1−a)C^T t^(k) + ap` — breaks the loop at every step. Pre-trusted peers also guarantee that C is irreducible and aperiodic, which guarantees convergence. No pre-trusted peer may belong to a malicious collective; this is the single point of fragility in the algorithm.

**Distributed score management (Section 5):**
Each peer's trust score is computed not by the peer itself but by M score managers, located via a DHT (CAN or Chord). A peer's score manager is found by hashing the peer's unique ID into the DHT space. Multiple hash functions create M independent score managers, providing redundancy. Majority vote settles conflicts when some managers are malicious.

**Probabilistic selection over deterministic (Section 7.2, Figure 3 vs Figure 4):**
Always downloading from the highest-trust peer concentrates load on a single node — Matthew effect. Selecting peer j with probability proportional to `tj` gives a load distribution nearly identical to a random system, while still reducing inauthentic downloads to ~10% even at 70% malicious peer share (Figure 5).

**Convergence speed (Figure 1):**
Residual `||t^(k+1) − t^(k)||` drops below measurement noise in under 10 iterations regardless of network size. This means a full global trust recomputation takes fewer rounds than one audit cycle (~24h).

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Left eigenvector of C (global consensus) | Local pairwise ratings only | Full network view; malicious peers cannot inflate their own scores by rating each other |
| Normalised cij (fraction of positive interactions) | Raw sij counts | Prevents arbitrarily large inflation by malicious peers; loses the distinction between zero-interaction and bad interaction |
| Pre-trusted peer anchor `p` | Pure decentralisation | Guarantees convergence and breaks collectives; requires at least one seed that cannot be malicious |
| Probabilistic download selection | Deterministic top-scorer selection | Prevents runaway concentration; allows new peers to build reputation |
| DHT-based score managers | Self-reported scores | Prevents a peer from computing its own trust value; DHT randomises assignment to prevent targeted manipulation |

---

## Breaks in Our Case

- **EigenTrust's trust signal is pairwise: peer i rates peer j after downloading from j** ≠ **Vyomanaut's audit is microservice-driven: only the coordination microservice issues challenges and records pass/fail**
→ Providers in Vyomanaut never directly rate each other. There is no "peer i downloaded from peer j and found the file authentic" event. The audit pass/fail for provider j is known only to the microservice. Applying EigenTrust requires either (a) providers to directly retrieve from each other and rate the interaction — not currently part of the architecture — or (b) the microservice to act as every peer's rating proxy, collapsing back to a centralised model.
- **EigenTrust requires a gossip/DHT infrastructure for score managers** ≠ **our V2 architecture keeps reputation scoring in a single authoritative microservice (ADR-008, non-I-confluent floor enforcement)**
→ Adding score manager DHT infrastructure would require providers to store and propagate trust values for other providers. This is the "gossip infrastructure" identified as a blocker in ADR-008. The bootstrapping problem for new providers (no interaction history) is partially solved by EigenTrust's pre-trusted peers, but a new provider in Vyomanaut already has a bootstrapping mechanism (random optimistic assignment, ADR-005).
- **EigenTrust detects malicious content providers (peers who send inauthentic files)** ≠ **Vyomanaut detects storage reliability (providers who delete data or fail audits over time)**
→ The nature of the trust signal is different. A file-authenticity rating is binary per transaction. A storage reliability signal is a rate measured over thousands of audit events per provider per month. EigenTrust's normalised cij values assume enough interactions to produce a meaningful ratio — a new provider with 10 audits has the same cij structure as one with 10,000. Our 3-window rolling score (ADR-008) weights recency explicitly, which EigenTrust does not.
- **EigenTrust's pre-trusted peers are a small fixed set (e.g., network founders)** ≠ **our trust anchor is the registration microservice, which gates all admission**
→ The registration-gated model (ADR-001) plays the same role as EigenTrust's pre-trusted peers — it ensures that only known entities participate. This is a structural equivalent, not a deficiency. The difference is that EigenTrust distributes computation away from its anchor; our architecture keeps the anchor as the authoritative scorer.
- **EigenTrust was simulated on 102 peers (good + malicious), converging in <10 iterations** ≠ **V2 launch target is hundreds to thousands of providers each with thousands of audit receipts**
→ Convergence speed is not the bottleneck. The input data (pairwise interaction counts) is the bottleneck: Vyomanaut providers do not interact with each other in a way that generates these counts.

---

## Decisions Influenced

- **ADR-008 [#8 Reliability Scoring]** `CONFIRMED`
The three-window rolling score (24h / 7d / 30d) in a single authoritative microservice remains the correct V2 design. EigenTrust is the correct alternative if providers had pairwise interactions and if gossip infrastructure were available. Neither condition holds in V2. The specific rejection reason in ADR-008 — "requires gossip infrastructure; bootstrapping problem for new nodes" — is confirmed by this paper: the score manager DHT requires operational investment that does not pay off when the underlying signal (pairwise provider ratings) does not exist in the architecture.
*Because:* Section 4.6 distributed algorithm requires each peer to query Ai (peers who downloaded from them) and Bi (peers they downloaded from) — these sets are empty in a storage network without direct provider-to-provider data exchange.
- **ADR-005 [#5 Peer Selection — Preference Subsystem]** `CONFIRMED`
The probabilistic selection principle (choose provider j with probability proportional to score tj, not always the top scorer) is exactly Vyomanaut's preference subsystem. ADR-005 assigns more new chunks to higher-scored providers but not all chunks. Figure 3 in the paper shows why deterministic selection is wrong: one provider eventually accumulates all reputation and serves all traffic. The 20% ASN cap (ADR-014) is the structural equivalent of EigenTrust's collision damping — it prevents any single cluster from dominating.
*Because:* Figure 4 shows probabilistic selection produces near-uniform load distribution while still reducing bad actors to <10% impact.
- **Q07-4 [Answered — deferred-v3]:**
EigenTrust confirms that distributed reputation propagation without a central coordinator seeing all audit results simultaneously is achievable — when pairwise interactions exist between nodes. In Vyomanaut V2, they do not: the microservice is the sole auditor and providers do not rate each other. Correlated failure detection therefore remains centralised at the microservice via the ASN cap (ADR-014). In V3, if repair events create direct provider-to-provider chunk transfers, those interactions can serve as EigenTrust-style rating events — provider i fetching a chunk from provider j during repair could produce a reliability rating for j. This is the V3 research question. Q07-4 status: deferred-v3.

---

## Disagreements

- **Aberer & Despotovic (ACM CIKM 2001), Cornelli et al. (WWW 2002):** Prior P2P reputation systems aggregated local ratings from only a few neighbours. EigenTrust generalises this to the whole network.
*Implication for us:* Our 3-window score is a local-only signal (Vyomanaut's microservice is the sole auditor) — but unlike Gnutella-era systems, our audit data is complete and objective. The limitation of prior systems (narrow view) does not apply when the auditor is centralised and sees all events.
- **Sybil attack vulnerability (Section 7.3.1):** EigenTrust's probabilistic newcomer slot (10% fixed chance for unknown peers) allows a Sybil attacker to dominate the newcomer pool with ghost identities.
*Implication for us:* Vyomanaut's registration gating (KYC/phone number, ADR-001) closes this attack vector before it reaches the scoring layer. The Sybil concern that EigenTrust must work around does not exist in our architecture.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q24-1 and Q24-2.