# Paper 44 — Recalibrating Global Data Center Energy-Use Estimates

**Authors:** Eric Masanet, Arman Shehabi, Nuoa Lei, Sarah Smith, Jonathan Koomey
**Venue / Year:** Science, Vol. 367, Issue 6481 | 2020, pp. 984–986. DOI: 10.1126/science.aba3758
**Citations:** not independently verified this session; widely cited as the corrective reference against "data center energy is doubling" claims in subsequent literature and journalism
**Topics:** #20, #23
**ADRs produced:** none — narrows the claim behind the forthcoming Impact Analytics decision

---

## Problem Solved

Your Impact Analytics plan (`ux-decisions.md` §8) needs a specific, checkable environmental claim, not a generic "data centers are bad for the planet" narrative. Several widely-cited estimates claimed global data-center energy use was doubling every few years; this paper is a bottom-up recalibration showing that claim was methodologically weak. It is the correct primary source for grounding — or correcting — whatever number your app eventually shows a provider.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Bottom-up recalibration using workload, efficiency, and hardware-shipment data | Top-down extrapolation from earlier, simpler energy-growth models | More defensible per-unit figures, but the paper's scope is operational electricity only — it does not quantify embodied (manufacturing) carbon, which is the half of your actual claim that matters most |

---

## Breaks in Our Case

- **A plausible first-draft Impact Analytics pitch ("every file stored with us is one less server the world needs, and data centers are a growing climate problem") assumes runaway operational energy growth**
  ≠ **this paper's central finding is that global data-center electricity use stayed roughly flat from 2010–2018 despite a 550% increase in compute workload, due to efficiency gains (hyperscale consolidation, virtualization)**
  → Rewrite the claim around avoided *new hardware manufacturing and construction* specifically, not general data-center energy waste — this paper shows that framing was already outdated before your product existed.

- **The paper's 2010–2018 flat-energy finding**
  ≠ **widely reported post-2020 trends showing renewed data-center energy growth driven by AI training/inference, which this 2020 paper could not have anticipated**
  → Do not claim the industry's energy use is *currently* flat. The safer, evergreen version of the claim is the embodied-carbon/avoided-new-construction one, which does not depend on which way current-year hyperscaler energy trends are moving.

---

## Decisions Influenced

- **Impact Analytics claim (`ux-decisions.md` §8; formalized in ADR-044)** `CLAIM NARROWED`
  Your provider app's environmental-impact numbers should be framed specifically around avoided new-hardware manufacturing and avoided new dedicated-cooling construction, not a general claim about data-center energy waste or growth.
  *Because:* this paper is the primary rigorous source showing the general "data centers are an exploding energy problem" framing was already a largely-debunked claim as of 2020 — using it risks your own sustainability messaging repeating exactly the category of error this paper was written to correct.

- **Impact Analytics copywriting rule (not yet an ADR — forthcoming)**
  Any specific number shown to a provider (grams of CO2 avoided, kg of hardware not manufactured) must be traceable to a bottom-up, per-unit calculation, not a top-down industry-wide growth statistic.
  *Because:* the latter is exactly the category of claim this paper shows to be unreliable at the industry level, let alone at the level of one user's individual contribution.

---

## Disagreements

- **This paper's own scope is operational electricity only; it does not attempt to quantify embodied carbon** (manufacturing, raw materials, end-of-life) of data-center hardware, which is the half of your argument most relevant to "avoided new-hardware manufacturing."
  *Implication for us:* a separate embodied-carbon source is required before any specific number is shown to a user — see Q44-1.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q44-1.
