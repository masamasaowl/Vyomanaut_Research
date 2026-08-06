# Paper 57 — Data Empowerment and Protection Architecture (DEPA) and the Account Aggregator Framework

**Authors:** NITI Aayog (policy framework); Reserve Bank of India (Account Aggregator Master Directive); DigiSahamati Foundation
**Venue / Year:** NITI Aayog DEPA framework, 2020 (ongoing); RBI Account Aggregator regulations, live in financial sector; retrieved 2026
**Citations:** not applicable — government policy framework and regulatory documentation, not an academic paper
**Topics:** #20 Client-Facing UX & Copy (new — provider dashboard), #5 Peer Selection (provider persona / device conditions)
**ADRs produced:** ADR-056

---

## Problem Solved

The proposed provider-dashboard feature assumes Vyomanaut can query a provider's ISP for their remaining mobile/broadband data plan, to warn before stored data risks exceeding it. This note checks whether that query is actually possible in India today, under what legal and technical framework, and for which sectors. It matters because the entire feature design changes depending on the answer: a live, consented API integration is a very different engineering problem than an estimate built entirely from data Vyomanaut's own client can already observe.

---

## Trade-offs

| Chosen (by DEPA/RBI) | Over | Consequence |
|---|---|---|
| Sector-by-sector rollout (financial sector first, live; health and telecom named as future sectors) | A single cross-sector consent framework at launch | A third party today can get standardized, consented API access to a user's *banking* data through Account Aggregators; there is no equivalent regulated pathway for *telecom* usage data yet |

---

## Breaks in Our Case

- **The feature brief assumes a queryable ISP data-plan API exists or can be obtained via user consent, similar to how banking Account Aggregators work today**
  ≠ **DEPA's own roadmap documentation explicitly names telecom as a sector planned for *future* rollout, after health — it is not live.** Consent Managers under DEPA are currently operational specifically as Account Aggregators regulated by RBI for financial data (bank statements, investments); no equivalent regulated, standardized API exists for a third party to pull a consumer's mobile/broadband data-plan balance from an Indian telecom operator today.
  → The dashboard cannot be built as "sync with the ISP" in V3. Two real alternatives exist: (a) client-side estimation — the provider daemon measures its own upload/download volume from OS-level network interface counters, which requires no ISP cooperation but only knows what *this app* has consumed, not the plan's total remaining balance; or (b) ask the provider to manually enter their known plan size/reset date, and combine that with (a) to estimate remaining headroom. Neither requires waiting on DEPA's telecom rollout.

- **The brief frames this as primarily a technical integration problem**
  ≠ **it is currently a regulatory-availability problem — the technical shape of a future telecom Consent Manager API is not yet published, because the sector hasn't been opened**
  → There is nothing to integrate against yet. This is not a "build vs. buy" or effort-estimation question; it is a "not available at any effort level" answer for the current period. Re-evaluate if/when telecom Consent Managers go live.

---

## Decisions Influenced

- **[ADR-056](../decisions/ADR-056-isp-data-plan-sync-deferred.md) [#20 Client-Facing UX & Copy]** `PROPOSED`
  Defer true ISP-API data-plan sync; ship a client-side usage estimate (own traffic, OS-level counters) plus a manually-entered plan size as the V3 interim design.
  *Because:* the regulated API pathway this feature assumed does not exist for telecom data in India as of this research pass, and there's no committed date for it.

---

## Disagreements

- One could argue that individual bilateral partnerships with major operators (Jio, Airtel, Vi) could substitute for a standardized DEPA rollout, since large platforms have historically negotiated direct data-sharing deals outside formal consent-manager frameworks.
  *Implication for us:* possible in principle, but this is a business-development undertaking disproportionate to a storage network's scope at V3, and each such deal would be operator-specific rather than a single integration — this note does not recommend pursuing it and ADR-056 does not assume it.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q57-1.
