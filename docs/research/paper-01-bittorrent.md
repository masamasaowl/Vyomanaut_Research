# Paper 01 — Incentives Build Robustness in BitTorrent

**Authors:** Bram Cohen
**Venue / Year:** May 22, 2003
**Topics:** #1

---

## Problem Solved

1. Infinite scaling as incoming downloaders arrive
2. Track pieces
3. Handle peer churn
4. Maintain speed fairness of upload vs download
5. 70% of Gnutella (first P2P DSN) users uploaded no files at all

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Random peer selection | Optimised peer selection | Benefited robustness |
| 20-second rolling average of upload speed | Historical upload contribution | Simpler, less fair long-term |

---

## Breaks in Our Case

- **Torrent is used when two providers maintain redundancy** ≠ **our model where data owner uploads once and laxes**
  → Upload-once behaviour changes incentive dynamics entirely

- **Choking algorithm assumes peers are interdependent** ≠ **our peers are individually incentivised**
  → No choking algorithm required, but adversarial provider behaviour (data deletion, false audit responses) must still be explicitly designed against

- **Tracker is a central single point of failure** ≠ **our need for high availability**
  → Tracker must be replaced — Kademlia addresses this

- **20-second rolling window for scoring** ≠ **our need for long-term reliability signals**
  → Reliability scoring must operate over days or larger time spans

---

## Decisions Influenced

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]:** A random peer set is more robust than a structured one for handling peer churn

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection]:** Piece size 256 KB, sub-piece size 16 KB, with 5 requests lined up at all times

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection]:** The anti-snubbing mechanism can be modelled as heartbeat detection

- **[ADR-009](../decisions/ADR-009-reliability-scoring.md) [#8 Reliability Scoring]:** Reliability need not begin from zero; random optimistic assignment (equivalent to optimistic unchoking) can bootstrap a new peer's initial score

---

## Disagreements

- **"BitTorrent is an Auction" (Levin, 2008):** Critiques the tit-for-tat model.
  *Implication for us:* Not applicable — choking algorithms are not needed in our incentive model.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q01-1 through Q01-6.
