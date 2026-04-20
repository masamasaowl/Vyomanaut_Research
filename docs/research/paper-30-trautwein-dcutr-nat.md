# Paper 30 — Challenging Tribal Knowledge: Large-Scale Measurement Campaign on Decentralized NAT Traversal

**Authors:** Dennis Trautwein, Cornelius Ihle, Moritz Schubotz, Bela Gipp — University of Göttingen; FIZ Karlsruhe
**Venue / Year:** arXiv:2510.27500 | November 2025
**Topics:** #12, #1
**ADRs produced:** none — ADR-021 confirmed and strengthened; Q13-1 and Q20-1 answered; see Decisions Influenced

---

## Problem Solved

Decentralized P2P systems depend on NAT traversal, yet the performance of fully decentralized NAT traversal had never been measured at scale in a production environment — existing studies used fewer than 2000 peers and data from 2011 or earlier. Conventional wisdom ("tribal knowledge") held that UDP is inherently superior to TCP for hole punching, that relay position and quality meaningfully affect success, and that the 30% failure rate was irreducible. This paper runs the first large-scale longitudinal study of libp2p's DCUtR protocol across 4.4 million traversal attempts from 85,000+ distinct networks in 167 countries, producing a definitive modern baseline and refuting several core assumptions. For Vyomanaut, the paper directly answers Q13-1 (relay overhead for symmetric-NAT providers) and Q20-1 (NAT type distribution in consumer networks), confirming that the three-tier NAT traversal design in ADR-021 is sound and that the 30% relay-fallback scenario is the right worst-case planning assumption.

---

## Key Findings

**Baseline hole-punching success rate: 70% ± 7.1% (Section 5.2.1):**
Across all networks with valid data, the DCUtR protocol achieves a 70% success rate at the hole-punching stage. This is measured on the hole-punching attempt itself, conditional on a successful initial relay connection. The unconditional end-to-end success rate (including relay setup failures) is lower, since approximately 29% of collected data points were excluded due to prerequisite failures — peers that could not even establish a relayed connection in the first place.

**TCP and QUIC achieve statistically indistinguishable success rates (~70% each, Figure 8b):**
When protocol-filtered hole punches are compared, both TCP and QUIC achieve the same ~70% success rate. The reason QUIC appears dominant in unrestricted measurements (Figure 8a, ~80% of successful connections use QUIC) is simply that QUIC's integrated 1-RTT handshake completes faster than TCP's three-way handshake — not because QUIC is more likely to succeed. DCUtR's RTT-based synchronization is precise enough that the historical TCP timing disadvantages (SYN flood detection, stateful firewall RST behavior) are neutralised.

**97.6% of successful hole punches succeed on the first attempt (Figure 9a):**
Only 2.4% of successes required a second or third retry. The built-in retry limit of 3 is therefore consuming resources for diminishing returns — the first attempt is the practically important one.

**Relay position and RTT are nearly independent of success (Section 5.2.2):**
The relay's path location (measured as the fraction of the client-to-remote RTT attributable to the client-to-relay segment) has no discernible impact on hole-punch success rate. A weak negative correlation exists between high relay RTT and success, but the CDFs for success and failure cases overlap almost completely. Any publicly reachable libp2p peer can serve as a relay without degrading outcomes.

**90% of successful connections achieve lower RTT than the relayed path (Figure 6a):**
After a successful hole punch, the median direct RTT is 70% of the relay RTT. 10% of peers have higher direct RTT than relay RTT — possible when the relay sits on a high-speed backbone and the direct path traverses multiple consumer-grade ISP hops.

**Connection Reversal is highly effective when a port mapping exists (Figure 9b):**
Peers with active UPnP/PMP port mappings show a dramatically higher rate of CONNECTION_REVERSED outcomes — the fast path that bypasses hole punching entirely. This confirms that providers who configure port forwarding get direct connections without any relay traffic.

**~30% hard failure rate attributable to symmetric NATs:**
The persistent failure floor is attributable to symmetric NATs (Endpoint-Dependent Mapping) where the port exposed to the relay differs from the port the NAT assigns for the actual peer connection. Birthday-paradox port-scanning could recover ~12.5% of these cases, but with risk of triggering NAT denylist behaviour.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| DCUtR RTT-based synchronisation | ICE/STUN/TURN with centralised signalling | Fully decentralised; any public peer can relay; success rate validated at 70% across 85k networks |
| TCP and QUIC as co-equal transports | UDP-only hole punching | Both achieve ~70%; QUIC establishes connection faster but neither is more likely to succeed |
| Three retries max | Single attempt | Only 2.4% marginal gain from retries 2 and 3; one retry may be worthwhile but three is excessive |
| Relay independence (any public peer may relay) | Privileged high-quality relay infrastructure | Confirmed robust: relay RTT and path location do not meaningfully affect success |
| Connection Reversal as fast path | Always full hole punch | Eliminates hole-punching complexity for UPnP-configured routers; Figure 9b shows meaningful CONNECTION_REVERSED share |

---

## Breaks in Our Case

- **Paper measures libp2p's IPFS network where peers are voluntary with no financial stake and high churn** ≠ **Vyomanaut providers are financially incentivised with MTTF 180–380 days**
→ The 70% success rate is measured on an unincentivised population. Vyomanaut providers with financial incentive to stay connected are more likely to configure port forwarding (UPnP) or use a router with a stable cone NAT, pushing the effective success rate above 70%. The 70% figure is a conservative lower bound for our provider population.
- **Paper's 29% exclusion rate reflects pre-hole-punch failures (relay setup failures, no public IP identified)** ≠ **Vyomanaut's audit deadline requires that the microservice can initiate contact with every provider at any time**
→ The excluded 29% are peers who could not even establish a relayed connection — the worst of the symmetric NAT cases. For Vyomanaut, these providers would fall into the Circuit Relay v2 fallback category and require Vyomanaut-operated relay infrastructure. The 70% direct + 30% relay-fallback planning split in ADR-021 is consistent with this finding.
- **Paper's relay RTT data is global and includes US-centric client deployments** ≠ **Vyomanaut's V2 is India-first with providers behind Indian ISP routers**
→ Indian ISP NAT behaviour is not separately characterised in this dataset. Indian ISPs predominantly deploy carrier-grade NAT (CGNAT) at scale, which behaves as symmetric NAT and is the most relay-dependent case. Q20-1 is partially answered by this paper but the India-specific NAT type distribution is still unknown at V2 launch. The 70% baseline should be treated as an optimistic bound for the Indian residential ISP context until measured empirically.
- **Paper shows relay position independence under random relay selection across the global IPFS network** ≠ **Vyomanaut's relay nodes are Vyomanaut-operated and co-located in Indian cloud regions**
→ Relay independence is good news for Vyomanaut: it means Vyomanaut does not need to maintain geographically optimised relay placement. Any two Vyomanaut-operated relay nodes in any Indian cloud region will perform equivalently for the coordination role. Relay-path RTT adds some latency but does not materially affect hole-punch success.
- **97.6% first-attempt success means the current 3-retry design is nearly wasteful** ≠ **ADR-021 adopted libp2p's default DCUtR retry behaviour without specifying retry count**
→ The default 3-retry cap in DCUtR wastes two relay round-trips on ~97.6% of successful connections. For the audit challenge path, where the response deadline is ~614 ms, this is an unnecessary latency floor for the 30% of connections that must fall back to relay entirely. The relay fallback does not retry hole-punching — it simply uses the relayed path. This paper does not change the relay fallback design but suggests that optimising DCUtR to attempt retry only once (not twice) would have negligible success rate impact.

---

## Decisions Influenced

- **ADR-021 [#12 P2P Transfer Protocol]** `CONFIRMED AND STRENGTHENED`
The three-tier NAT traversal strategy (AutoNAT → DCUtR → Circuit Relay v2) is validated. DCUtR achieves 70% success on the first attempt for cone-NAT providers. The 30% who fall back to Circuit Relay v2 will be served by Vyomanaut-operated relay nodes. The relay independence finding confirms that relay nodes do not need special placement — any relay within India will perform equivalently. The TCP/QUIC transport-agnosticism finding confirms that Vyomanaut's TCP fallback (TCP + Noise + yamux, ADR-021) is not a degraded path for NAT traversal success — its success probability is identical to the QUIC primary path. The 97.6% first-attempt figure validates that DCUtR overhead on the audit challenge path is bounded to two relay round-trips in essentially all cases.
*Because:* The paper provides the production-scale empirical evidence for the exact libp2p stack that ADR-021 adopts. The 70% baseline and relay-independence findings are the core operational parameters the ADR assumed but could not previously quantify.
- **ADR-014 [#19 Adversarial Defences]** `CONFIRMED`
The finding that relay success is independent of relay path position and RTT confirms that a malicious relay cannot selectively degrade hole-punch success by being geographically distant or slow. An adversarial relay can delay the relayed path RTT measurement slightly but the paper shows this has only a minor effect on success probability. The Circuit Relay v2 reservation system (Section 3.1.3) already prevents relay exhaustion attacks by limiting relay resources per connection.
*Because:* Section 5.2.2 directly tests relay position independence across the full network. If relay quality mattered, a targeted attack on relay quality could impair provider connectivity.
- **[Q13-1 — Answered]:**
The paper provides the answer to whether DCUtR relay overhead violates the audit response deadline. The relay RTT adds latency to the hole-punch coordination (two relay round-trips of ~50–200ms each), but once a hole punch succeeds (70% of the time), the direct path has lower RTT than the relay path in 90% of cases. For the 30% who remain relay-dependent, all audit traffic traverses the relay. At 256 KB and 5 Mbps declared upload speed, the deadline is ~614 ms. The paper shows relay RTT rarely exceeds 500 ms in practice (CDF in Figure 6b shows >90% of relay RTTs below 1 second, with most well below that). For Vyomanaut-operated relays co-located in Indian cloud regions, relay RTT to Indian home providers should be consistently under 50 ms. Circuit Relay v2 overhead is acceptable within the 614 ms deadline for V2's India-first deployment. The deadline multiplier of 1.5 provides sufficient margin.
- **[Q20-1 — Partially Answered]:**
The global 70% hole-punch success rate provides a lower bound for V2's India-first provider population. The paper does not characterise Indian ISP NAT behaviour separately. Given India's high CGNAT prevalence among residential ISPs, the Indian-specific success rate may be closer to 60% than 70% — meaning up to 40% of providers may require Circuit Relay v2 as their permanent connection path. This is within the operational planning envelope of ADR-021 but must be measured empirically at launch. The answer to "what fraction of Indian desktop home-router deployments are behind symmetric NAT" remains open for launch telemetry.

---

## Disagreements

- **Conventional wisdom that UDP/QUIC is inherently superior for NAT traversal:** This paper directly refutes this claim with 4.4 million data points. Both TCP and QUIC achieve ~70% success. The libp2p TCP fallback is not a second-class citizen for NAT traversal purposes — only for raw connection establishment speed.
*Implication for us:* ADR-021's TCP + Noise + yamux fallback path is fully viable. Providers behind UDP-blocking middleboxes (Q14-1) lose no hole-punching success probability by using TCP.
- **Guha & Francis (2005) TCP traversal measurement:** Reported 88% success rate for TCP across 93 home NATs. This paper's 70% across 85,000+ networks represents a significantly larger and more recent sample and should be considered the authoritative modern baseline.
*Implication for us:* Use 70% (not 88%) as the hole-punch success planning figure. The 30% relay-fallback case is not rare — it is the expected steady-state for nearly a third of providers.
- **Halkes & Pouwelse (2011):** Found ~64% of peers behind a NAT that should allow hole punching, with >80% of those succeeding. The post-2011 rise of CGNAT has likely eroded this — 30% failure today vs Halkes' implied lower failure rate is consistent with CGNAT becoming ubiquitous.
*Implication for us:* CGNAT is the dominant explanation for Vyomanaut's relay-dependent provider fraction. Providers on CGNAT ISPs will always fall back to Circuit Relay v2. This is an ISP selection filter for the vetting subsystem: providers on known CGNAT ISPs should receive a lower reliability score modifier, not because they are unreliable, but because all their audit traffic carries relay overhead.

---

## Open Questions

See open-questions.md — questions Q30-1 and Q30-2.