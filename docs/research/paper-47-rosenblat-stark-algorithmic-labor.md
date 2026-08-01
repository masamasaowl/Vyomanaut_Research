# Paper 47 — Algorithmic Labor and Information Asymmetries: A Case Study of Uber's Drivers

**Authors:** Alex Rosenblat, Luke Stark (Data & Society Research Institute)
**Venue / Year:** International Journal of Communication, Vol. 10 | 2016, pp. 3758–3784
**Citations:** not independently verified this session; a foundational, heavily-cited paper in gig-economy/algorithmic-management literature, still cited in 2025–2026 systematic reviews of the field
**Topics:** #20, #13
**ADRs produced:** none — directly informs the earnings-transparency design that will consume the ADR-016 addendum's new ledger fields

---

## Problem Solved

Your provider is paid, rated on a reliability score, and subject to a payout multiplier that can reduce their earnings — the exact three ingredients (payment, scoring, algorithmic control over income) this paper studies empirically in Uber's driver relationship. Based on a nine-month study of driver forums and Uber's own driver-facing communications, Rosenblat and Stark document precisely how a platform's information and power asymmetry over a paid participant erodes trust, even when the underlying system is not malicious — a direct, evidenced warning for the earnings and reliability-score screens your Provider app has not yet built.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Uber discloses ratings and some performance summaries, but keeps fare/surge calculation and matching logic opaque | Full disclosure of the earnings algorithm | Opacity protects the platform from gaming, but the paper documents this as the direct cause of driver distrust, forum-based reverse-engineering ("tips and tricks"), and a persistent sense of unfair treatment |
| Gamified, rhetorical framing ("entrepreneurship," "be your own boss") paired with significant indirect behavioral control | Presenting the relationship plainly as platform-managed work | Softens the perception of control in the moment, but the paper documents this rhetoric as a source of driver resentment once the gap between "independence" framing and actual control becomes apparent |

---

## Breaks in Our Case

- **Uber's drivers cannot see why a specific fare or incentive was calculated the way it was, and turn to driver forums to reverse-engineer the algorithm**
  ≠ **your ADR-016 addendum already records, per release, the gross amount and the exact multiplier applied — the data needed to show a provider precisely why their payout was smaller, if you choose to show it**
  → You are not starting from Uber's position of not having the data. The open question this paper sharpens is not "can we be transparent" but "how much of the reliability-score formula should be disclosed," given full disclosure risks gaming exactly as full opacity risks the distrust this paper documents. See Q47-1.

- **Uber pairs indirect algorithmic control with "entrepreneur"/"independence" rhetoric that the paper finds contradicts drivers' actual experience of being managed**
  ≠ **your `ux-finding.md` positioning already avoids an "independence"/"be your own boss" framing, describing providers plainly as paid participants in a marketplace with real rules (vetting, audits, reliability scoring)**
  → This is a difference worth preserving deliberately, not by accident — the paper's central finding is that the gap between rhetoric and actual control, not the control itself, is what erodes trust.

- **Uber's drivers experience unpredictable earnings with no advance visibility into demand/pricing logic**
  ≠ **your escrow and release-schedule model is deterministic and disclosed in `requirements.md`'s FR set — a provider can compute their own expected payout in advance**
  → This is a genuine, evidenced advantage over the gig-economy precedent worth stating plainly in the Provider app rather than assuming it's obvious; the paper shows unpredictability itself, independent of the amount, is a source of distress.

---

## Decisions Influenced

- **Earnings-transparency design consuming the ADR-016 addendum's new fields (not yet an ADR — forthcoming)**
  The Provider app's earnings screen should explain a reduced payout in plain terms ("your reliability score dropped below 95%, so ₹X was withheld and will roll into next month") rather than showing only a final number.
  *Because:* this paper documents, empirically, that opaque algorithmic earnings calculation — even when the platform is not acting maliciously — produces sustained driver distrust and defensive forum-based reverse-engineering. The data to avoid this (gross amount, multiplier applied) already exists per the ADR-016 addendum; not using it to explain the number would forfeit evidence-backed trust for no engineering reason.

- **Provider-facing tone/positioning (not yet an ADR — forthcoming, extends `ux-finding.md` §5)**
  Continue to describe providers plainly as paid marketplace participants subject to real, disclosed rules, not as "independent entrepreneurs running their own micro-business."
  *Because:* the paper's central finding is that rhetoric promising more independence than the system actually grants is itself the trust-eroding mechanism — a risk avoided by not making that promise in the first place, rather than by managing the gap after the fact.

---

## Disagreements

- **A reading of this paper could conclude that any algorithmic scoring of a paid participant is inherently exploitative and should be avoided or minimized.**
  *Implication for us:* the paper's own conclusions are narrower than that — it identifies *opacity and rhetorical mismatch* as the mechanism of harm, not scoring itself. Your reliability score and audit system are structurally necessary (Paper 05's honesty-verification problem has no substitute), so the actionable takeaway is disclosure design, not the elimination of scoring.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q47-1.
