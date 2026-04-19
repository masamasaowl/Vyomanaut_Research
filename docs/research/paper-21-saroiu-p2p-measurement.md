# Paper 21 — A Measurement Study of Peer-to-Peer File Sharing Systems

**Authors:** Stefan Saroiu, P. Krishna Gummadi, Steven D. Gribble — University of Washington
**Venue / Year:** Multimedia Computing and Networking (MMCN) 2002 | Published 2001
**Topics:** #5, #6, #8
**ADRs produced:** none

---

## Problem Solved

Early P2P system designs assumed peers were homogeneous and cooperative. This paper refutes
both assumptions with measurement data from Gnutella and Napster — the two largest deployed
P2P systems of the era. It characterises actual peer uptime, bandwidth, latency, and free-riding
behaviour from traces of 509 k Napster peers and 1.2 M Gnutella peers. For Vyomanaut the
paper's primary value is the session-duration distribution, which is the unincentivized α_temporary
input that Q10-4 (whether splitting the failure rate into temporary and permanent components
changes the optimal erasure parameter r) requires. The paper also validates two structural
decisions already made: the vetting subsystem (ADR-005) is exactly the response the paper calls
for, and direct measurement of peer characteristics rather than trusting self-reports (ADR-005
Identify protocol, ADR-002 PoR challenge) is independently recommended.

---

## Key Findings

**Session duration (Figure 6, right):**
The median session duration for both Napster and Gnutella peers is approximately 60 minutes.
The graph is limited to sessions under 12 hours using the create-based method; longer sessions
cannot be characterised from the trace length. The important implication: the vast majority
of unincentivized P2P peers treat a session as a transient, task-duration event — connect to
download, then disconnect. This is the α_temporary baseline.

**IP-level uptime (Figure 6, left):**
Only 20% of peers in either system had IP-level uptime of 93% or more. The Gnutella and
Napster populations are nearly identical at the IP level, confirming the finding is independent
of application-level incentives. The best 20% of Napster peers have application-level uptime
of 83% or more; the best 20% of Gnutella peers only 45% or more.

**Bandwidth distribution (Figure 3):**
92% of Gnutella peers have downstream bottleneck bandwidth of at least 100 Kbps. Upstream
is the binding constraint: 22% of peers have upstream bottleneck bandwidth of 100 Kbps or
less. The most common connections are cable modem and DSL (1–3.5 Mbps downstream).
Only 8% of peers have upstream bandwidth of at least 10 Mbps.

**Free-riding (Section 3.3):**
25% of Gnutella peers share no files. In Napster, 60–80% of users share 80–100% of all files,
meaning 20–40% of users contribute little or nothing. The top 7% of Gnutella peers by file
count collectively offer more files than all other peers combined.

**Self-reported bandwidth misreporting (Figure 11):**
Up to 30% of Napster peers reporting a bandwidth of 64 Kbps or less actually had significantly
higher measured bandwidth. Peers have an incentive to underreport bandwidth to avoid being
used as upload sources. The paper concludes that robust systems must directly measure peer
characteristics rather than trust self-reports.

**Overlay resilience (Section 3.6):**
Gnutella's connectivity follows a power-law distribution (index α=2.3). Random failures
require removing more than 60% of nodes to fragment the overlay. However, targeted removal
of the top 4% highest-degree nodes shatters the overlay entirely. Power-law topologies are
robust against random churn but fragile against targeted attack.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Direct bandwidth measurement (SProbe) | Trusting self-reported bandwidth | Reveals systematic misreporting; adds measurement overhead to onboarding |
| Separate tracking of IP-level and application-level uptime | Single uptime metric | Shows that host availability ≠ application availability; gap explains why PoR challenges fail even when host is online |
| Create-based method for session duration | Naive session length average | Avoids truncation bias from long sessions spanning the trace window; limits characterisation to sessions < 6 hours (half the 12-hour trace window) |

---

## Breaks in Our Case

- **Median session duration ~60 minutes in unincentivized file-sharing systems** ≠ **Vyomanaut
  targets MTTF 180–380 days under financial incentives**
  → The 60-minute median is α_temporary for Q10-4: the rate of temporary departures that return
    within one polling cycle. For Vyomanaut, this rate is dramatically lower due to escrow
    incentives, but the unincentivized floor is now confirmed from two independent datasets
    (Saroiu 2002, Trautwein 2022/Paper 20). Giroire's single α lumps temporary and permanent
    departures together; splitting them requires knowing α_temporary and α_permanent separately
    (Q10-4). This paper provides the worst-case value of α_temporary.

- **Free-riding rate of 25% in unincentivized Gnutella** ≠ **Vyomanaut's escrow model requires
  providers to actively store data to pass audits and earn**
  → Free-riding is structurally impossible in Vyomanaut: a provider who stores nothing cannot
    compute a valid PoR response (`SHA256(chunk_data || challenge_nonce)`) and earns nothing.
    The escrow model eliminates the free-rider problem entirely. This paper motivates why
    escrow-per-audit-passed (ADR-012) was the correct payment design, not per-session or per-GB.

- **Bandwidth misreporting rate of ~30% in Napster** ≠ **Vyomanaut measures bandwidth directly
  during vetting (ADR-005, Identify protocol)**
  → The paper's recommendation — measure characteristics directly rather than trust self-reports
    — is already implemented. The vetting subsystem's latency and throughput measurements
    during the 4–6 month vetting period (ADR-005) serve exactly this purpose. Self-reported
    storage capacity is not trusted; only audit pass rates and measured response latencies count.

- **Gnutella overlay fragile against targeted removal of top 4% highest-degree nodes** ≠ **our
  microservice + Kademlia hybrid (ADR-001) does not depend on any single high-degree peer**
  → Our Kademlia DHT does not concentrate routing through high-degree peers the way Gnutella's
    unstructured overlay does. Provider lookup and DHT routing are spread across all providers
    in logarithmic hop counts. The microservice handles coordination; the DHT handles data-
    plane routing. Neither has a single highest-degree node to attack.

- **Data is from 2001 — modem and early broadband era** ≠ **our 2024 Indian ISP environment
  with 100 Mbps symmetrical for ₹600/month**
  → The bandwidth distributions (22% of peers with upstream ≤ 100 Kbps) are entirely obsolete
    for the Indian ISP context of 2024. Only the session-duration and uptime findings transfer,
    as those reflect human behaviour patterns rather than network infrastructure.

---

## Decisions Influenced

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection]** `CONFIRMED`
  The paper's central recommendation — measure peer characteristics directly rather than trust
  self-reports — is the design principle behind the vetting subsystem. Bandwidth and latency
  are measured during audit interactions (response_latency_ms in ADR-017), not taken from
  provider declarations. The paper provides historical evidence for why this design choice is
  necessary: up to 30% of peers in a peer-to-peer system will misreport bandwidth.
  *Because:* Section 3.5 quantifies the misreporting rate directly and recommends measurement.

- **[ADR-012](../decisions/ADR-012-payment-basis.md) [#13 Payment per Audit Passed]** `CONFIRMED`
  The free-rider problem in unincentivized P2P systems (25% of Gnutella peers share nothing)
  is eliminated by Vyomanaut's design: payment per audit passed (ADR-012) requires providers
  to actually store data. Providers who store nothing cannot compute valid PoR responses and
  earn nothing. The paper motivates why any passive or time-based payment model would fail;
  the audit-pass model is the only one that ties earnings to actual storage.
  *Because:* Section 3.3 shows that passive participation (storing no files) is the dominant
  failure mode in systems without storage-contingent rewards.

- **Q10-4 [Open — partially informed]:**
  This paper provides the worst-case α_temporary for Q10-4 (Giroire's single failure rate vs
  a split α_temporary / α_permanent model). In an unincentivized system, α_temporary dominates
  — most departures are temporary (median session 60 min, most peers return). In Vyomanaut's
  incentivized system, α_temporary is dramatically lower (providers don't want to lose escrow
  earnings). The practical implication for Q10-4: if α_temporary is small relative to
  α_permanent in Vyomanaut's actual deployment, then treating all departures as permanent in
  Giroire's formula (the conservative assumption underlying ADR-003) is safe — it
  overestimates repair bandwidth, which is a safe direction to err. The question of whether
  splitting α changes the optimal r therefore resolves conservatively: keep r=40.

---

## Disagreements

- **Bhagwan et al. (Paper 08):** Used IP-address probing and found average availability of
  ~30% per host. Saroiu's data shows 20% of peers have IP-level uptime ≥93%. These are
  not contradictory — Bhagwan reports availability averaged across all peers including highly
  churning ones; Saroiu's 20%/93% threshold is a different slice of the same distribution.
  *Implication for us:* Both papers are consistent. The 30% overall availability from Bhagwan
  is the mean; Saroiu shows the distribution is heavily right-skewed — a few highly stable
  hosts contribute most availability. Our vetting subsystem selects from the right tail.

- **Reading-list Phase 2A #3 positioning (Saroiu placed before Dimakis):**
  The reading list originally placed this paper to feed Q10-4 before Dimakis derives the
  optimal r formula. Now that Q10-4 partially resolves conservatively (keep r=40), the urgency
  of reading Dimakis remains high but for a different reason: sub-packetisation at (n=56, k=16)
  for Clay codes (Q19-2) requires Dimakis's formula directly.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q21-1.
