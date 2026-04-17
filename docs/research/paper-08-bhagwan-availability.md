# Paper 08 — Understanding Availability

**Authors:** Ranjita Bhagwan, Stefan Savage, Geoffrey M. Voelker, UC San Diego
**Venue / Year:** IPTPS 2003 (Workshop on Peer-to-Peer Systems)
**Topics:** #5, #6, #8

---

## Problem Solved

1. Existing measurements and models did not capture the complex time-varying nature of availability in P2P environments
2. Studies availability in large P2P systems over the span of one week
3. Results affect the design of high-availability P2P services

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| 7-day trace | Longer observation period | Risky to form concrete decisions from this alone |
| Overnet study | Gnutella (unstructured) | Structured DHT behaviour; unstructured churn may differ |
| Measurement by HostID | IP address | Avoids inflation from dynamic IPs (Saroiu study used IPs — ~4× lower availability) |

---

## Breaks in Our Case

- **Study population is voluntary file-sharers with no financial stake** ≠ **our paid storage providers**
  → Our actual availability is higher than the 0.3 median proposed in the paper. Financial friction reduces churn.

- **6.4 joins/leaves per host per day on a DHT where joining/leaving is free with zero state consequences** ≠ **our model with financial consequences for churn**
  → Financial friction and the exit state machine significantly dampen this rate.

---

## Decisions Influenced

- **[ADR-009](../decisions/ADR-009-reliability-scoring.md) [#8 Reliability Scoring — DECIDED]:** The reliability score must weight recent behaviour more heavily but not discard long-term history entirely. Maintain three rolling windows per provider — 24 h, 7 d, 30 d — and compute a weighted combination.

- **[ADR-008](../decisions/ADR-008-polling-interval.md) [#6 Polling — DECIDED]:** Polling interval t must be set longer than the diurnal absence window. A provider offline for 6–8 hours (nighttime) must not trigger repair. Minimum safe value of t is therefore ≥ 8 hours, ideally 12–24 hours — confirms Blake & Rodrigues.

- **[ADR-007](../decisions/ADR-007-provider-exit-states.md) [#7 Provider Exit States — DECIDED]:** Two-component model (daily churn vs permanent departure) maps to four exit states. A lower-bound threshold is maintained; crossing it triggers repair regardless of t:
  1. **Accidental/temporary absence (downtime):** no repair, wait for t; decrease reliability score
  2. **Promised downtime:** no repair, wait for promised period p to end; then repair and penalise directly if broken
  3. **Permanent silent departure:** crossing a warning limit of time declared at session start → trigger repair, seize escrow earnings, sign node out of network
  4. **Announced departure:** node announces permanent exit → trigger repair, release pending escrow, sign node out

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection — CONFIRMED]:** Independence of randomly selected hosts is confirmed. Random selection within a filtered pool (vetted, geographically close) gives near-independent failure probabilities.

---

## Disagreements

- **Saroiu et al. (2002, Gnutella measurement study):** Their availability numbers are ~4× lower because they used IP address probing.
  *Implication for us:* Be cautious when reading papers earlier than 2003 that use IP-based probing.

- **Weatherspoon & Kubiatowicz (2002):** Modelled failures as independent.
  *Implication for us:* Keep a small safety margin in erasure parameters to account for correlated failures.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q08-1 through Q08-2.
