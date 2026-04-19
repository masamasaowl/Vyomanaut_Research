# Paper 06 — High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two

**Authors:** Charles Blake & Rodrigo Rodrigues, MIT Laboratory for Computer Science
**Venue / Year:** HotOS IX (USENIX), May 2003 | 6 pages
**Topics:** #3, #4, #5, #6

---

## Problem Solved

The paper extends the CAP theorem to P2P storage: three properties cannot coexist simultaneously:
- High peer churn (Partition tolerance)
- Data always accessible (Availability)
- Store large amounts of data at scale (Consistency/Scalability)

In the case of partition, one must choose between A or C.

Additional findings:
1. Idle upstream bandwidth is the limiting resource volunteers contribute — not idle disk space
2. Average bandwidth consumed per node = `2 × (space per node) / uptime`
3. Quantifies the relation between node lifetime vs number of nodes given k and bandwidth
4. Provides availability data in 6 nines (gold standard: 2.6 seconds downtime per month)
5. Presents math of consumed bandwidth after applying erasure, delayed redundancy calls, and admission control

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| 6-nines availability assumption | Realistic consumer availability | Upper bound model for adverse situations |
| Low bandwidth + high replication factor | Realistic modern values | Conservative; useful for designing for worst case |

---

## Breaks in Our Case

- **Assumes a million cable modem users at 200 Kbps upload to hold 1000 TB** ≠ **modern ISP speeds (100 Mbps symmetrical for ₹600)**
  → The paper's bandwidth numbers are conservative; modern ISP trends significantly favour the architecture. Research needed to apply formulas to our specific model.

---

## Decisions Influenced

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]:** Calculating the total bandwidth of the network is more integral than the combined storage it offers. Volatility of users directly decreases storage capacity as bandwidth is wasted in replication.

- **[ADR-007](../decisions/ADR-007-provider-exit-states.md) [#7 Provider Exit]:** Promised exit — a user announces the time they will be unavailable — saves bandwidth by triggering replication only for true departures. Failing a promise leads to direct collection of fines from the linked payments account. (`Decision #4` in original README)

- **[ADR-006](../decisions/ADR-006-polling-interval.md) [#6 Polling]:** Polling interval described as t. A 24-hour timeout reduces maintenance bandwidth by a factor of 30× compared to instant timeouts. (`Decision #6` in original README)

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]:** Erasure coding allows t=25 for k=15, proving 8× bandwidth savings compared to replication — finalising erasure coding as the redundancy protocol.

- **[ADR-011](../decisions/ADR-011-desktop-only.md) [#5 Peer Selection]:** The architecture must favour stable providers over maximum node count. Every new flaky node burdens existing reliable nodes' bandwidth.

---

## Disagreements

- **Storj and Filecoin in production:** The paper clearly states that common home storage suppliers cannot run the architecture — but Storj and Filecoin demonstrate it works in production with the right coding, membership delay, and admission control.

---

## Quantitative Results (from engineering questions)

**At what MTTF does repair bandwidth become unsustainable?**
Assuming each provider serves 50 GB:
- MTTF = 1 month → BW/node = 2 × 50 GB / 30 days ≈ 3.3 GB/day = ~310 KB/s continuous
- MTTF = 3 months → BW/node = 2 × 50 GB / 90 days ≈ 1.1 GB/day = ~103 KB/s continuous
- MTTF = 6 months → BW/node = 2 × 50 GB / 180 days ≈ 556 MB/day = ~52 KB/s continuous

If background OS provides 100 KB/s, then MTTF must exceed 3 months for an average user.

**Economic viability floor:**
Entry of low-MTTF providers is unprofitable for existing nodes. If `repair cost due to peer churn > revenue per provider`, good providers subsidise bad ones — the network is at a loss.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q06-1 through Q06-3.
