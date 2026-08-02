# ADR-044 — Impact Analytics: Scope the Environmental Claim to Avoided Hardware, Not General Data-Center Waste

**Status:** Proposed
**Topic:** #23 Environmental Impact & Sustainability Messaging
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 44 (Masanet et al.)

---

## Context

`ux-decisions.md` §8 commits to an "Impact Analytics" feature showing providers a defensible environmental claim. Paper 44 (Science, 2020) found that global data-center electricity use stayed roughly flat from 2010–2018 despite a 550% workload increase, directly undercutting the popular "data centers are an exploding energy problem" framing several informal sustainability pitches rely on. This decision fixes the scope of the claim before any copy or number is written.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| A general "you're helping fight data-center energy waste" claim | Simple, emotionally resonant copy | Directly contradicted by Paper 44's own finding — repeats the category of claim the paper was written to correct |
| **A narrower claim: avoided new-hardware manufacturing and avoided new dedicated-cooling construction, specifically** | Defensible against the best available evidence; does not depend on which way current-year hyperscaler energy trends are moving | Requires a separate, currently-unsourced embodied-carbon-per-unit figure before any specific number can be shown (Q44-1) — cannot ship a precise number yet, only the qualitative claim |

## Decision

Impact Analytics copy is scoped to the claim that using existing idle hardware avoids the embodied carbon and material cost of manufacturing new dedicated storage hardware and building new purpose-built cooling infrastructure — not a general claim about data-center energy waste or growth. No specific quantitative figure (grams of CO2, kg of hardware) ships until a bottom-up, per-unit embodied-carbon source is identified and verified (tracked as Q44-1); until then, the feature ships with qualitative language only.

## Consequences

**Positive:** the product's sustainability messaging is defensible against the best current evidence rather than repeating a debunked narrative; avoids a credibility risk if a user or journalist checks the claim against Paper 44 or similar sources.

**Negative:** the Impact Analytics feature cannot show a specific number at launch — only qualitative framing — until the embodied-carbon sourcing work is done.

**Open constraints:** Q44-1 (a defensible per-unit embodied-carbon figure) must be resolved before any quantitative version of this feature ships.

## References

- Paper 44 — Recalibrating Global Data Center Energy-Use Estimates (Masanet et al., Science 2020)
