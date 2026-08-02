# Paper 42 — BOINC: A System for Public-Resource Computing and Storage

**Authors:** David P. Anderson (UC Berkeley Space Sciences Laboratory)
**Venue / Year:** 5th IEEE/ACM International Workshop on Grid Computing (GRID) | 2004
**Citations:** not independently verified this session; foundational paper in volunteer/public-resource computing, still cited in current literature (see Anderson's own 2019 follow-up, arXiv:1903.01699)
**Topics:** #21, #20
**ADRs produced:** none — strengthens ADR-009; feeds the forthcoming Impact Analytics decision

---

## Problem Solved

By 2004, most of the world's computing and storage capacity had moved into hundreds of millions of ordinary PCs your organization does not own or administer, with almost no general-purpose software to harness it for a shared purpose. Anderson built BOINC so a small research group could stand up a public-resource computing project in about a week, without needing to solve device heterogeneity, churn, and zero administrative control themselves. This is the founding document of the category your provider network sits in: software that asks strangers, not employees, to donate a resource you don't control.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Asymmetric trust model — the project has no control over participants and cannot prevent malicious behavior | A managed Grid model with organizationally-owned, IT-administered resources | Public-resource computing reaches far more machines, but pushes all correctness/honesty enforcement onto the application layer (redundant computing, in BOINC's case) rather than administrative trust |
| Credit (a visible, numeric contribution score) as the primary retention mechanism | Relying on altruism/cause alone | Requires ongoing engineering investment (trickle messages, leaderboards) to keep credit visible during long-running work — not a one-time feature |
| Participant-set resource limits (idle/input-gated, disk, bandwidth) | A fixed, project-set resource policy | Better retention (the owner's claim on their machine is never contested) at the cost of unpredictable, participant-controlled available capacity |

---

## Breaks in Our Case

- **BOINC's participants are unpaid volunteers, motivated by credit and cause** ≠ **your providers are paid in rupees, motivated primarily by income**
  → Borrow BOINC's finding that visible, frequent, non-monetary progress feedback matters *in addition to* payment (feeds the Impact Analytics design), but do not assume altruism does any of the retention work that payment is supposed to do.

- **BOINC has no reputation, vetting, or economic penalty system — a bad result is simply discarded and recomputed elsewhere**
  ≠ **your system has a multi-month vetting period, a reliability score, and escrow penalties**
  → This is not a gap in BOINC's design to imitate. It is a direct consequence of computation being cheaply re-verifiable and storage not being — see Paper 05 (Storj) for why your audit/challenge protocol exists instead.

- **BOINC assumes projects "cannot prevent malicious behaviour" and does not try to (§1.1)**
  ≠ **your entire audit/scoring/escrow stack exists specifically to prevent and penalize dishonest providers**
  → BOINC's no-enforcement posture is acceptable there because a bad computational result costs almost nothing to discard. An undetected dishonest provider in your network causes real, costly data loss — do not read BOINC's simplicity here as a simpler alternative worth adopting.

---

## Decisions Influenced

- **[ADR-009](../decisions/ADR-009-background-execution.md) [#11 Background Execution]** `ACCEPTED — independently confirmed`
  Your desktop-only, idle- and AC-power-gated, resource-budgeted background execution policy is exactly what BOINC's participant-preference model (§3.4: control over input-activity gating, active hours, disk, and bandwidth) converged on independently, 20 years earlier and at far larger scale.
  *Because:* a resource-harvesting daemon that visibly competes with the owner's own use of their machine does not survive contact with real users — this is Anderson's finding, not just your own design assumption.

- **Impact Analytics (`ux-decisions.md` §8; formalized in ADR-044)**
  Your provider app's home screen should show frequently-updating, tangible progress — not just a monthly payout figure.
  *Because:* Anderson reports (§3.5) that participants "demand immediate gratification; they want to see their credit totals increase at least daily," and built BOINC's trickle-message mechanism specifically to solve this for workunits lasting months.

---

## Disagreements

- **Anderson's 2004 projection of ~1 billion Internet-connected PCs by 2015, implying continued PC-centric growth:** directionally correct on device count, but did not anticipate the shift of ordinary consumer computing toward mobile devices rather than PCs specifically.
  *Implication for us:* do not treat "PC owner" and "internet user" as interchangeable population figures at any future date — see the India device-ownership data behind `ux-decisions.md` §4, where PC ownership among internet users is falling, not rising.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q42-1.
