# Paper 09 — Feasibility of a Serverless Distributed File System Deployed on an Existing Set of Desktop PCs

**Authors:** William J. Bolosky, John R. Douceur, David Ely, Marvin Theimer, Microsoft Research
**Venue / Year:** ACM SIGMETRICS 2000
**Topics:** #1, #5, #6, #11

---

## Problem Solved

1. Measures feasibility using 51,662 machines at Microsoft over five weeks
2. Clearly states it will not work on consumer desktops without optimisations — only fine-tuned replication and downtime vs departure separation helps
3. Promises availability failures of just 0.3 to 3.0 (order of 1) per user per 1000 days

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Simple replication | Erasure coding | Simpler design; higher storage overhead |
| Lazy update | Immediate write propagation | Reduces write traffic (32 MB/hr → 7 MB/hr) but weakens consistency |
| Replica placement by availability score (greedy algorithm) | Random placement | Better reliability; requires scoring infrastructure |

---

## Breaks in Our Case

- **Focuses on file storage as done on a usual PC** ≠ **we are building a hyperscale cloud platform with file updates and duplicate file checks**
  → Direct comparison of reliability numbers should be treated as upper bounds only

- **Machines in a professional corporate environment** ≠ **home users we are targeting**
  → 95% uptime and 290-day lifetime are upper bounds; consumer behaviour is worse

- **Security ensured is much lower — nodes are employees of the same company** ≠ **adversarial public network**
  → Cannot rely on social trust mechanisms

- **50% free disk space assumed (year 2000)** ≠ **modern consumer storage is tighter**
  → Free space assumptions must be recalibrated

---

## Decisions Influenced

- **[ADR-007](../decisions/ADR-007-provider-exit-states.md) [#7 Provider Exit — CONFIRMED]:** Promised downtimes and announced departures skyrocket reliability because machines warn before leaving, providing time for reconstruction.

- **[ADR-007](../decisions/ADR-007-provider-exit-states.md) [#7 Provider Exit — DECIDED, thresholds]:** Replication trigger times per exit type:
  1. **Accidental/temporary absence (0–24 h):** wait until 72 h before declaring permanent silent departure
  2. **Promised downtime:** provide options from 0–72 h; exceeding declares permanent silent departure
  3. **Permanent silent departure:** after 72 h, perform replication
  4. **Announced departure:** immediate replication

- **[ADR-009](../decisions/ADR-009-background-execution.md) [#11 Background Execution — DECIDED]:** Median CPU load of 1–2% and disk load means we can occupy 5–10% of CPU for background audits and transfers without hindering user experience.

- **[ADR-010](../decisions/ADR-010-desktop-only.md) [#5 Peer Selection — TIER MODEL]:** Provider tier model (before collapse to single tier — see ADR-011):
  - **Tier 1: NAS providers** — MTTF: 290–380 days; Availability: 0.95; Storage: 30–70% of free disk; Payment: Premium
  - **Tier 2: Standard desktop** — MTTF: 180–290 days; Availability: 0.7; Storage: 30–40% of free disk; Payment: Standard
  - **Tier 3: Mobile users** — MTTF: 120–180 days (minimum needed); Availability: 0.3–0.5; Storage: 10–15% of free disk; Payment: Per service

---

## Disagreements

- **Blake & Rodrigues (2003):** Stated 24 h acceptable downtime, whereas Bolosky suggests a bimodal figure.
  *Implication for us:* Use Bolosky's bimodal distribution for trigger thresholds (desktop: 72 h).

- **Bhagwan et al. (2003):** Found availability to be 0.3, whereas Bolosky suggests 0.9 due to corporate environment.
  *Implication for us:* Our providers sit between these — financially-incentivised but home users.

---

## Quantitative Results (bimodal downtime distribution)

Bolosky's measured downtime distribution:
- **Nightly:** µ=14 h, σ=1.9 h → 99.7% return within 20 h
- **Weekend:** µ=64 h, σ=2.1 h → 99.7% return within 70 h

This informs the 72 h repair trigger threshold: it safely exceeds weekend absence without waiting for a true departure.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q09-1 through Q09-3.
