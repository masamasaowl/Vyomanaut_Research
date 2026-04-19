# Paper 20 — Design and Evaluation of IPFS: A Storage Layer for the Decentralized Web

**Authors:** Dennis Trautwein, Aravindh Raman, Gareth Tyson, Ignacio Castro, Will Scott, Moritz Schubotz, Bela Gipp, Yiannis Psaras — Protocol Labs, Telefonica Research, HKUST, QMUL, University of Göttingen, FIZ Karlsruhe
**Venue / Year:** ACM SIGCOMM 2022 | Amsterdam
**Topics:** #1, #5, #6, #8
**ADRs produced:** none

---

## Problem Solved

No prior study had measured a large-scale, deployed Kademlia DHT from multiple vantage points
at once and quantified the actual churn profile, AS distribution, NAT reachability, and lookup
latency that operators experience in production. IPFS is the largest deployed libp2p + Kademlia
system (300 k DHT-serving nodes, 3 M+ weekly content retrievals) and is the reference
implementation for the very stack Vyomanaut V2 depends on (ADR-021). The paper gives
ground-truth performance and churn numbers that no theoretical model or synthetic benchmark
can provide. For Vyomanaut specifically it answers two questions directly: what fraction of
desktop P2P peers are permanently unreachable due to NAT (and therefore cannot serve as
DHT servers or receive direct audit challenges), and what the real operational parameter for
DHT record republication should be to avoid silent record expiry.

---

## Key Findings

**Churn without financial friction (Figure 8):**
87.6% of IPFS sessions last less than 8 hours. Only 2.5% of sessions exceed 24 hours. This is the
single most important number in the paper for Vyomanaut: it represents the floor — the worst
case churn rate achievable without any financial incentive. Vyomanaut's escrow model is
designed to push providers toward the MTTF 180–380 day range. Any real deployment churn
numbers will lie between the IPFS baseline (session median ~1–2 hours) and the Bolosky
corporate desktop baseline (session median ~14 hours nightly, ~64 hours weekends). Financial
friction quantifiably reduces churn; this paper establishes the unincentivized lower bound.

**Reliable peer concentration (Figure 7a):**
Only 1.4% of 198,964 discovered peers maintained >90% uptime over the measurement window.
The geographic distribution of these reliable peers is remarkably even — the largest single
country share (US) is just 0.3% of all reliable peers.

**Reachability / NAT (Section 5.1):**
45.5% of all discovered IP addresses were always unreachable. Of the remaining 54.5%, a
substantial fraction required relay to connect. The paper cites DCUtR hole-punching as
"currently under test" (this was 2022; it is now production-deployed in libp2p, as captured
in ADR-021). This 45.5% hard-unreachable figure is the operational NAT ceiling that the
three-tier NAT traversal in ADR-021 is designed to address.

**AS concentration (Table 2 and Section 5.2):**
The top 10 ASes contain 64.9% of all IPFS IP addresses. More than 50% of all IPs are in
just 5 ASes, with two Chinese ASes (CHINANET-BACKBONE and CHINA169-BACKBONE) alone
accounting for more than 30% of all observed IP addresses. Geographic spread is wide (152
countries, 2715 ASes) but correlated concentration within ASes is severe.

**Cloud hosting (Table 3):**
Only 2.3% of IPFS nodes run on major cloud platforms. The overwhelming majority run on
personal or on-premises hardware. Contabo (0.44%), Amazon (0.39%), and Azure (0.33%) are
the top three cloud providers by IP share. This validates the assumption that a distributed
storage network populated by commodity hardware operators is a realistic deployment model.

**DHT performance (Sections 6.1, 6.2):**
Single DHT walk median: 622 ms for content retrieval (terminates on first record found).
Publication DHT walk median: 33.8 s for full publication (must find k=20 closest peers).
Both DHT walks for retrieval (provider record + peer record) complete in under 2 s for 50%
of operations. This is dramatically better than BitTorrent Mainline DHT (median over 1 minute)
and is attributed to the DHT server/client distinction: peers that cannot receive incoming
connections do not participate in routing tables, eliminating dead-end hops.

**DHT server/client distinction (Section 2.3):**
New peers join as DHT clients by default. They use AutoNAT to test reachability: if more than
three peers can initiate connections to the new peer, it upgrades to DHT server mode. DHT
servers store routing table entries and provider records; DHT clients only query. This
single design decision is credited with the dramatic improvement in lookup performance between
early IPFS and the post-v0.5 deployment measured in this paper.

**Provider record parameters (Section 3.1):**
Provider records are replicated on k=20 closest peers. Republication interval: 12 hours.
Expiry interval: 24 hours. This means a provider that fails to republish within 24 hours
loses its DHT presence entirely. The gap between republication interval (12h) and expiry
(24h) provides a 12-hour buffer. An availability service running at a 24-hour republication
interval has no buffer against delays or failures.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| DHT server/client distinction (AutoNAT) | All peers as full DHT participants | Dramatically reduces dead-end hops; NAT-unreachable peers are excluded from routing tables; 45.5% of discovered peers are unreachable |
| k=20 for provider record replication | Lower k (faster publication) | Protects records against churn; publication requires finding 20 closest peers — this dominates publication latency (87.9% of publication delay is the DHT walk) |
| 12h republish / 24h expiry | Longer intervals | Keeps DHT records fresh despite churn; creates operational coupling between availability service uptime and provider record persistence |
| Opportunistic Bitswap before DHT lookup | DHT lookup immediately | Saves a full DHT walk for hot content; adds 1s timeout overhead when Bitswap fails — a constant floor on cold-content retrieval |
| Fire-and-forget RPC for provider record storage | Wait-for-acknowledgment | Lower publication latency; some records may not land due to timeout spikes (5s TCP dial timeout, 45s WebSocket handshake timeout visible as spikes in Figure 9c) |

---

## Breaks in Our Case

- **IPFS has no financial incentive for uptime — 87.6% of sessions are under 8 hours**
  ≠ **Vyomanaut's escrow model targets MTTF 180–380 days for V2 desktop providers**
  → The IPFS churn figures are the unincentivized floor. Vyomanaut's providers must sustain
    sessions orders of magnitude longer. The escrow seizure on silent departure (ADR-007) is
    the enforcement mechanism. Empirical validation of actual MTTF under financial incentives
    requires launch telemetry — no prior paper models this exact regime.

- **IPFS uses k=20 for both bucket size and provider record replication** ≠ **our k=16
  (S/Kademlia, ADR-001)**
  → k=20 is IPFS's production-validated parameter, chosen by practical experience. Our k=16
    comes from S/Kademlia's disjoint-path design (d=8, k=2×d=16). The paper validates that
    k in the range 16–20 is operationally sound; our lower value reduces record redundancy
    slightly but is within the validated range.

- **IPFS provider record republication interval is 12 hours, expiry is 24 hours** ≠ **our
  availability service republishes every 24 hours (ADR-001, ADR-006)**
  → With a 24-hour republication interval and a 24-hour expiry, a single delayed republication
    cycle causes record expiry. Providers disappear from the DHT silently with no departure
    signal. The IPFS design provides a 12-hour buffer (republish at 12h, expire at 24h).
    Our availability service must either republish every 12 hours or the record expiry must
    be extended to 48 hours to maintain the same buffer.

- **IPFS's DHT client mode allows NAT'd peers to query but not be queried** ≠ **all Vyomanaut
  providers must be DHT servers (publicly reachable) to receive audit challenges and chunk
  requests**
  → A provider behind symmetric NAT that cannot be reached directly cannot receive audit
    challenges unless via Circuit Relay v2. Our three-tier NAT traversal (ADR-021) handles
    this, but relay overhead adds to audit response latency (Q13-1). The 45.5% always-
    unreachable figure in IPFS is a worst-case indicator for an unincentivized network; the
    fraction for financially-incentivized desktop providers (who have reason to configure
    port forwarding) should be lower but is unknown at launch.

- **Top 10 ASes contain 64.9% of IPFS IP addresses — AS concentration is severe in practice**
  ≠ **our 20% ASN cap (ADR-014) targets this exact failure mode**
  → The IPFS measurement directly validates the threat that motivated ADR-014. An uncapped P2P
    network naturally concentrates in a small number of ASes. The 20% cap is the correct
    mitigation, and this paper provides empirical evidence that without such a cap, correlated
    failures by AS are a structural risk.

---

## Decisions Influenced

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]** `CONFIRMED`
  k=20 is the IPFS production-validated bucket size; k=16 per S/Kademlia is within the
  operational range. Alpha=3 for parallel lookups is confirmed as producing sub-second median
  DHT walk times (622 ms for retrieval). The DHT server/client distinction is confirmed as
  the critical design decision that separates low-latency from high-latency DHT performance.
  All Vyomanaut desktop providers must be DHT servers (reachable via libp2p NAT traversal).
  *Because:* IPFS post-v0.5 achieves sub-second lookup medians using exactly this architecture.

- **[ADR-006](../decisions/ADR-006-polling-interval.md) [#6 Polling and Departure Threshold]** `STRENGTHENED — REPUBLICATION GAP IDENTIFIED`
  The paper reveals that a 24-hour republication interval against a 24-hour expiry leaves zero
  buffer. Our current availability service design (republish every 24h) must be revised: either
  reduce republication to every 12 hours, or extend record expiry to 48 hours. The 12h/24h
  IPFS parameters provide the production-validated reference. This does not change the polling
  interval (t=24h for provider health checks) — it only affects DHT record republication.
  *Because:* Figure 9c shows fire-and-forget RPCs have non-trivial failure rates at timeouts;
  a record republication that fails silently causes the provider to disappear from the DHT.

- **[ADR-014](../decisions/ADR-014-adversarial-defences.md) [#19 Adversarial Defences]** `CONFIRMED`
  The AS concentration finding (top 10 ASes = 64.9% of IPs) provides empirical evidence that
  an unconstrained P2P deployment naturally concentrates in a small number of ASes. The 20%
  ASN cap is the correct structural mitigation. The paper also shows that geographic spread
  and AS diversity are not the same: IPFS spans 152 countries but is dominated by 5 ASes.
  Country-level diversity does not imply AS-level resilience.
  *Because:* Table 2 and Figure 7d directly measure the degree of AS concentration in a
  deployed P2P network of comparable scale.

- **[ADR-010](../decisions/ADR-010-desktop-only-v2.md) [#5 Peer Selection]** `CONFIRMED`
  The finding that fewer than 2.3% of IPFS nodes run on cloud infrastructure confirms that
  the target population for Vyomanaut (desktop, NAS, personal hardware) is the natural habitat
  of P2P storage nodes. The paper also confirms that this population can sustain a network
  of 300 k+ nodes without cloud dependency — the desktop-first model is operationally viable.

---

## Disagreements

- **BitTorrent Kademlia (Crosby & Wallach 2007, Wolchok & Halderman 2010):** Show median
  DHT lookup latency over 1 minute in BitTorrent Mainline DHT due to dead nodes and slow links.
  *Implication for us:* The DHT client/server distinction (AutoNAT-based) is the critical
  change that separates IPFS's sub-second lookup median from BitTorrent's minutes. Our
  Kademlia deployment must enforce DHT server eligibility for all providers.

- **Bhagwan et al. (Paper 08):** Measured 6.4 joins/leaves per host per day in Overnet (DHT
  without financial friction). Trautwein's IPFS data shows 87.6% of sessions under 8 hours —
  consistent with Bhagwan's finding. Both papers agree: unincentivized P2P has extreme churn.
  *Implication for us:* The Bhagwan-derived MTTF assumption (0.3 median availability) is an
  upper bound on un-incentivized churn severity. Our financially-incentivized providers should
  do materially better, but the unincentivized floor is confirmed by two independent datasets.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q20-1 through Q20-3.
