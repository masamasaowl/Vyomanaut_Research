# Paper 28 — Handling Churn in a DHT

**Authors:** Sean Rhea, Dennis Geels, Timothy Roscoe, John Kubiatowicz — University of California, Berkeley and Intel Research, Berkeley
**Venue / Year:** USENIX ATC 2004
**Citations:** ~900
**Topics:** #1, #6
**ADRs produced:** none — ADR-001 confirmed; ADR-006 gains concrete timeout formula; see Decisions Influenced

---

## Problem Solved

DHTs were designed with clean mathematical properties (O(log N) lookup in N nodes, O(log N) routing state) but existing mature implementations — FreePastry and MIT Chord — fail badly when subjected to the churn rates actually observed in deployed P2P networks like Kazaa (median session time: 2.4 minutes). FreePastry drops the majority of lookup requests; Chord completes them but lookup latency grows to multiple seconds. The paper identifies three root causes — reactive failure recovery triggering positive feedback cycles, poorly-chosen message timeouts, and the absence of proximity-aware neighbor selection — and resolves all three through a modified DHT called Bamboo, tested on an emulated 1000-node wide-area network. For Vyomanaut, the paper directly validates Kademlia's core design preference (keep long-lived nodes over new ones) and provides the concrete timeout calculation formula that the availability service should use when probing provider liveness.

---

## Key Findings

**Churn rates in deployed P2P systems (Table 1):**
Median session times range from 60 minutes (Gnutella, Napster, Overnet) to under 1 minute (Kazaa). All six surveyed studies find median availability around 30%. The paper explicitly targets resilience down to 1-minute median session times — the worst real-world case.

**Reactive recovery creates positive feedback cycles (Section 4.1):**
When a node's access link is congested, timeouts cause it to incorrectly conclude that neighbours have failed. Reactive recovery sends repair messages immediately, adding more load to the already-congested link. This causes more timeouts, which cause more recovery messages — a congestion collapse spiral. FreePastry's poor performance in Figure 6 is directly attributable to this mechanism.

**Periodic recovery breaks the feedback loop (Section 4.1.2):**
Instead of reacting to detected failures, each node shares its leaf set with one random member every fixed period, regardless of observed failures. Changes propagate in O(log k) phases. Under high churn, periodic recovery uses less bandwidth than reactive recovery and produces dramatically lower lookup latency (Figure 8). Under low churn it uses slightly more bandwidth — this is the only trade-off.

**TCP-style timeout calculation is the best approach for recursive routing (Section 4.2.1):**
Each node tracks per-neighbour round-trip time with an exponentially-weighted mean (AVG) and variance (VAR):

`RTO = AVG + 4 × VAR`

This is the original TCP algorithm (Jacobson & Karels 1988). Under all churn rates, it outperforms both fixed 5-second timeouts and Vivaldi virtual-coordinate-based timeouts. Fixed 5-second timeouts more than double mean lookup latency even at light churn (Figure 9).

**Vivaldi-based timeouts are adequate for moderate churn:** Virtual coordinate timeouts stay within 2× of TCP-style until median session times drop below 23 minutes. Below that, divergence is rapid. For iterative routing (where direct per-neighbour RTT history is unavailable), virtual coordinates are the practical fallback.

**Proximity neighbor selection (PNS) gives 24–42% latency reduction (Section 4.3.3):**
Global sampling — performing DHT lookups for random identifiers with the desired routing-table prefix, then selecting the closest responding node — reduces mean lookup latency by 24% for virtually no bandwidth increase. Adding a second technique (sampling neighbours' inverse neighbours, or recursive Tapestry-style sampling) brings the reduction to ~42% at 40% more bandwidth. Simply sampling neighbours' neighbours (without the recursive step) is surprisingly ineffective.

**Kademlia's preference for long-lived nodes (Section 5, future work):**
The paper explicitly names Kademlia's design of keeping existing long-lived neighbours over newly-discovered ones (only replacing on confirmed failure) as an alternative approach to churn resilience, noting this targets the "infant mortality" observed in P2P networks — many new joiners leave quickly. This design is distinct from Bamboo's approach but addresses the same problem.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Periodic leaf-set maintenance (one random peer per period) | Reactive leaf-set propagation (broadcast on every change) | Avoids positive feedback under congestion; uses slightly more bandwidth at low churn; converges in O(log k) phases |
| TCP-style per-neighbour RTO (EWMA + 4×VAR) | Fixed 5-second timeout | Accurate failure detection under churn; requires per-neighbour RTT history; only applicable for recursive routing |
| Vivaldi virtual coordinate timeouts | TCP-style (for iterative routing) | Works without direct RTT history; within 2× of TCP-style until heavy churn; degrades rapidly below 23-min median session |
| Global sampling + one additional technique | Pure global sampling | 42% latency reduction vs 24%; 40% more bandwidth; best practical balance |
| Recursive routing | Iterative routing | Lower latency; enables TCP-style timeouts directly; congestion control is simpler |
| Leaf set for correctness, routing table for performance | Single structure for both | Correct lookups guaranteed even with incomplete routing table; proximity optimisation cannot break correctness |

---

## Breaks in Our Case

- **Paper models zero-incentive P2P systems with median session times of minutes** ≠ **Vyomanaut providers are financially incentivised with MTTF 180–380 days**
→ The churn rates analysed (1–128 minute median sessions) are the unincentivised floor. Vyomanaut providers churn orders of magnitude less frequently. The paper's lessons for DHT maintenance design still apply — the periodic-vs-reactive choice matters at any churn rate — but the parameter tuning (period length, leaf-set size) must be calibrated for our dramatically lower churn rate.
- **Bamboo/Pastry uses a leaf set and routing table geometry** ≠ **our Kademlia-based DHT (ADR-001) uses k-buckets without an explicit leaf set**
→ The periodic-vs-reactive recovery distinction transfers directly: Kademlia's k-bucket maintenance already uses a periodic gossip model (contact random peer every second — ADR-025) which is Bamboo's periodic recovery principle. The leaf-set concept maps to Kademlia's sibling list. The lesson is architectural, not geometry-specific.
- **Paper's TCP-style RTO is computed per-DHT-neighbour, measured over lookup round-trips** ≠ **Vyomanaut's availability service polls providers with audit challenges, not DHT lookups**
→ The RTO formula (`AVG + 4 × VAR`) applies to the audit challenge polling timeout, not to Kademlia routing. The availability service maintains a per-provider RTT history from audit response times — the same inputs as TCP-style timeout calculation. This is the direct mapping for ADR-006.
- **Paper's positive feedback cycle occurs when a node's access link is congested** ≠ **our availability service is a centralised microservice, not a P2P node**
→ The microservice cannot create a feedback cycle by sending repair messages on its own congested link — it is the initiator of challenges, not a routing participant. The feedback-cycle risk applies to the provider daemons' Kademlia routing traffic, not to audit polling.
- **Paper tests a homogeneous Poisson churn model** ≠ **our bimodal absence distribution (Bolosky: nightly µ=14h, weekend µ=64h)**
→ Poisson churn produces uniformly distributed session lengths. Bolosky's bimodal model produces concentrated absence periods with predictable return times. The periodic recovery advantage is stronger in a bimodal model: during the nightly absence peak, many providers go offline simultaneously — reactive recovery would send a burst of leaf-set propagation messages, whereas periodic recovery absorbs this as a normal period boundary.

---

## Decisions Influenced

- **ADR-001 [#1 Coordination]** `CONFIRMED`
Kademlia's design of preferring long-lived contacts (retain existing k-bucket entries over new arrivals; replace only on confirmed failure) is independently validated. The paper names this explicitly as the approach targeting "infant mortality" in P2P networks. For Vyomanaut, this means a newly-joined provider does not displace a long-tenure provider from the DHT routing tables until the long-tenure provider is confirmed unavailable — the correct behaviour for our financially-incentivised provider population where long-tenure correlates with reliability (ADR-008 scoring confirms this empirically). The periodic gossip-based leaf-set maintenance in Bamboo maps to Kademlia's periodic k-bucket refresh, already specified in ADR-001 (refresh every 12h) and ADR-025 (microservice gossip at 1 peer/s).
*Because:* Section 5 names Kademlia's long-lived preference as a complementary churn-handling mechanism, and Section 4.1.2 proves that periodic (not reactive) recovery is the correct architecture under any realistic churn rate.
- **ADR-006 [#6 Polling]** `STRENGTHENED — TIMEOUT FORMULA SPECIFIED`
The availability service currently specifies a 24-hour polling interval (t=24h) and 72-hour departure threshold. The paper adds a concrete timeout formula for within-session failure detection. When the availability service sends an audit challenge and awaits a response, the response deadline should not be a fixed value — it should be computed per provider as `RTO = AVG + 4 × VAR`, where AVG and VAR are the exponentially-weighted mean and mean variance of that provider's recent audit response times. This distinguishes a slow-to-respond provider (high variance, should wait longer) from one that has genuinely departed. The 72-hour departure threshold (macro-level) is not changed; the RTO formula governs the micro-level per-challenge timeout.
*Because:* Figure 9 shows that even a modest fixed timeout (5 seconds) more than doubles mean detection latency vs per-peer TCP-style RTO calculation. The formula is directly applicable to audit challenge response detection.

---

## Disagreements

- **FreePastry design (reactive recovery):** The paper's data shows FreePastry fails to complete a majority of lookups at moderate churn (23-minute median session) due to reactive recovery feedback cycles.
*Implication for us:* Our availability microservice must not reactively broadcast membership changes to all providers on detecting a single failure — it must process failures at the periodic audit schedule rate. This is already the design (audit challenges fire on a fixed schedule, not on failure events), so no change is needed.
- **Fixed timeout proponents (NFS-style RPC):** Traditional client-server systems use fixed or exponentially-backed-off timeouts because the server rarely fails.
*Implication for us:* P2P providers fail regularly (diurnal and weekend absences). Fixed audit challenge timeouts will cause either false-positive departure declarations (too short) or delayed repair triggering (too long). Per-provider TCP-style RTO is the correct implementation.
- **Vivaldi virtual coordinate advocates:** Virtual coordinates allow timeout estimation without direct RTT history, useful for iterative routing.
*Implication for us:* Our availability service polls each provider directly (recursive contact, not iterative DHT lookup), so per-provider RTT history is always available. Vivaldi is not needed for audit polling; it remains a valid consideration for Kademlia routing table timeout estimation in the libp2p DHT layer (Q28-2).

---

## Open Questions

See open-questions.md — questions Q28-1 and Q28-2.