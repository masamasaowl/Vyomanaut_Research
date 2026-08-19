# ADR-083 — The Market Study Is a Governing Document, and Price Is Not Vyomanaut's Axis

**Status:** Proposed
**Track:** LTS
**Topic:** #0 Governance / #6 Provider Economics / #16 Positioning
**Amends:** the implicit assumption, present across `mvp.md` and the provider-facing framing, that
Vyomanaut competes as low-cost storage
**Research source:** `market-study.md`; M-01; ADR-081; ADR-082
**Establishes:** the empirical market track, running parallel to the LTS literature track

---

## Context

Two things happened in one session that the project's document set had no place to put.

First, the R-17 measurement (M-01) produced a **customer-acquisition constraint** as the output of a
confidentiality investigation: to hold the placement property, Vyomanaut must recruit Reliance Jio
customers at 0.38× their natural rate and small-ISP customers at 2.203× theirs. That is not a
cryptography finding and it does not belong in a reading-list domain.

Second, the unit-economics derivation (ADR-082) showed that the provider proposition is arithmetically
false at the current parameter set, and that the owner-facing price floor is **1.9× to 60.5×** the
cheapest Indian consumer cloud plan **from electricity alone**.

Neither result has a home. `requirements.md` states what the system must do; `architecture.md` states
how; neither states *for whom, at what price, against what alternative*. The corpus has 75+ paper
notes about mechanisms and none about buyers.

This ADR creates the home, fixes its rules, and records the one positioning conclusion the evidence
forces — while explicitly refusing to decide the one it does not.

---

## Decision

### 1. `market-study.md` sits beside the design documents and is governing

It is placed in `docs/system-design/` alongside `requirements.md`, `architecture.md`,
`data-model.md` and `interface-contracts.md`, and it has the same standing: **a design decision that
contradicts a measured number in the market study is a defect, not a trade-off.**

Its scope is deliberately narrow — supply base, unit economics, competitive price surface, demand
theses, and the forks these open. It does not contain roadmap, positioning copy, or forecasts.

### 2. The market track runs **parallel** to the literature track, not behind it

Domain P (repair security) and Domain K (threshold decoupling) proceed on their own sequence and are
not blocked by market work. The tracks intersect at exactly three parameters, recorded in
`market-study.md` §6:

| Parameter | Coupling |
| --- | --- |
| `provider_size_gb` | economics ↔ audit load (ADR-082 Decision 4) |
| `n/k = 3.5` | economics ↔ confidentiality threshold (`k` *is* the AONT-RS disclosure threshold) |
| ASN placement weight | acquisition funnel ↔ collusion resistance (ADR-081) |

**Neither track may move one of those three alone.** Everything else in either track is independent
and should proceed concurrently.

### 3. Provenance discipline applies to market numbers exactly as to literature

The LTS Literature Documentation Standard's four tags — `[P-num]`, `[DERIVED]`, `[INFERRED]`,
`[ASSUMED]` — are mandatory in `market-study.md` and in any ADR citing it. **An ADR may not rest on
an `[INFERRED]` or `[ASSUMED]` claim alone.**

Two additional rules, specific to market data:

- **Every number carries its date.** Indian telecom subscriber data, electricity tariffs and cloud
  pricing all move on quarterly or annual cycles. A market number without a date becomes a
  `[VENDOR-DEFAULT]` within two quarters, which is the failure ADR-077 exists to catch.
- **A number may be replaced only by a measurement.** A council may decide what to *do* about a
  number; it may not decide what the number *is*. This is the amendment rule, and it is the reason
  §7's forks are referred rather than resolved.

### 4. Price is not Vyomanaut's competitive axis — recorded as settled

`[DERIVED]` `market-study.md` §2.3: the owner-facing floor is **3.50 × provider raw cost**, and at
Indian metro marginal tariffs that is **1.9×** the cheapest Indian consumer cloud plan in the most
favourable physically-available scenario, **7.4×** in the realistic one, and **60.5×** if the machine
runs for Vyomanaut alone. Before provider profit, repair bandwidth, the microservice, the payment
rail, or margin.

**This is arithmetic at fixed compile-time constants, not a forecast.** It does not depend on
execution, scale, efficiency, or competitive response. `n = 56` and `k = 16` are frozen; the 3.50×
multiplier is not a cost curve that improves.

**Therefore:**

- **No Vyomanaut artefact may make a price-advantage claim against consumer cloud storage.** Not
  marketing copy, not `mvp.md`, not the pitch, not a comparison table. The claim is false and
  measurably so.
- **The supportable claim is the one ADR-022 Addendum A §5 already identified**: *computational
  confidentiality resting on client-held key material*, with data resident in India. That is a
  genuine claim and price is not part of it.
- **Provider-facing earnings claims are net-of-electricity at the household's marginal slab**
  (ADR-082 Decision 2). Gross and net differ in **sign**.

### 5. The positioning fork is referred to `/design-council` and is **not decided here**

`market-study.md` §4 identifies two demand theses that point in opposite directions:

- **§4.1 sovereignty/compliance** — the buyer needs Indian-resident, non-hyperscaler, client-keyed
  storage. Price-insensitive in the way §2.3 measures. Provider count becomes a **cost to minimise**.
- **§4.2 + §4.3 supply-side income** — the provider is the customer, and the UPI/INR payment rail is
  the one differentiator competitors cannot trivially copy, since it is regulatory and operational
  rather than technical. Provider count becomes the **growth metric**.

These are not compatible roadmaps and they imply different architectures.

**The council is convened only after Q-MKT-2 (measured wall power) resolves.** ADR-082 Decision 3
sets the allocation from that measurement, and the allocation determines whether §4.2 is even
available: at 70 GB the provider is cash-negative, so the income thesis does not exist until the
allocation moves. **Convening before the number lands would produce a judgement where an arithmetic
answer is pending**, which is the ADR-060 / Q66-2 failure pattern in a new domain.

Recorded as **Q-MKT-1**, and as `market-study.md` §7.2.

### 6. Two further forks are registered and sequenced

- **§7.1 — ISP-partnership acquisition vs placement independence.** The only tractable route to the
  2.203× tail over-recruitment is segmented acquisition through regional ISPs; that same channel
  concentrates providers inside single legal entities, which is what ADR-081 Decision 2's
  per-organisation cap exists to prevent. **Settles on Q-R17-4.**
- **§7.3 — whether `n/k = 3.5` survives commercially.** Domain C is closed on technical grounds;
  §2.3 reopens it on commercial ones. **Do not convene before R-30 is read**: `k = 16` *is* the
  AONT-RS disclosure threshold, so lowering it for cost reasons lowers the confidentiality threshold
  with it. If R-30 decouples the two, the expansion factor becomes tunable on durability grounds
  alone. **If R-30 fails, this fork is closed and 3.5× is permanent.**

### 7. Three measurements are promoted to the critical path

| ID | Measurement | Why it is upstream |
| --- | --- | --- |
| **Q-MKT-2** | Wall-power draw of the provider daemon on representative Indian desktop hardware | Every number in `market-study.md` §2.2–2.5; the assumed bracket is **32× wide** |
| **Q-MKT-3** | R-15 — Indian residential grid and broadband availability (CEA / state-utility outage data) | Redundancy parameters; whether a UPS becomes a provider prerequisite, turning a zero-capex proposition into a ₹3,000–6,000 capex one |
| **Q-R17-3** | TRAI ISP-wise wired-broadband series | The falsifier for `N_eff = 5.16`; the acquisition-funnel shape |

**Q-MKT-2 is first.** It is a wall meter and a weekend, it is upstream of ADR-082's replacement
value, and ADR-082's replacement value is upstream of Decision 5's council.

---

## Consequences

**For the document set.** One new governing document. One prohibited claim class. One amendment rule
that binds councils as well as sessions.

**For the roadmap.** The market track begins now and runs concurrently. Three measurements enter the
critical path, one of them (Q-MKT-2) cheap and blocking two ADRs and a council.

**For the product.** The project has been carrying an unexamined price-competition assumption. It is
now measured false, and the honest positioning — confidentiality and residency, not cost — is
narrower, more defensible, and unaffected by the 3.50× multiplier that made the price claim
impossible.

**What this does not settle.** Whether anyone buys the honest positioning. `market-study.md` §4.1's
sovereignty thesis has **no measurement behind it at all** — it is `[INFERRED]` from India's data
centre and localisation environment, and Decision 3 forbids an ADR resting on that alone. **The
demand side of Vyomanaut is currently unmeasured, and this ADR does not pretend otherwise.** Q-MKT-1
owns it, and it is the largest open question in the project that is not a cryptography question.

**Compounding supply constraints.** ADR-081 narrows the supply base by ASN diversity; ADR-082 narrows
it by allocation (a household sharing 3 TB is plausibly a different household from one sharing
70 GB). **Nobody has multiplied the two.** Doing so is the first task of the market track once
Q-MKT-2 and Q-R17-3 land.

---

## References

- `market-study.md` — the document this ADR establishes and governs
- `measurement-01-asn-diversity-india.md` (M-01) — the supply-side measurement
- ADR-022 Addendum A §5 — the supportable product claim, identified there, adopted here
- ADR-062 — two-track development; the market track is LTS-only
- ADR-077 — research-first triage; the dating rule extends it to market numbers
- ADR-081 — ASN diversity; the acquisition constraint
- ADR-082 — the allocation gate; upstream of Decision 5's council
- Paper 47 — Rosenblat & Stark; the basis for the net-of-electricity rule
- `lts-literature-standard.md` — the provenance tags extended here to market data
