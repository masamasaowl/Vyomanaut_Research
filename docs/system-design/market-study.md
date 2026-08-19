# Vyomanaut Market Study

**Status:** Living document. **Track:** LTS. **Standing:** governing, alongside `requirements.md`,
`architecture.md`, `data-model.md` and `interface-contracts.md`.
**Governed by:** ADR-081 (this document's standing and amendment rule)
**Established:** August 2026, from the R-17 / M-01 measurement.

---

## §0 — What this document is for, and what it is not

Vyomanaut's design documents answer *can we build it*. This one answers *should anyone buy it, and
from whom would we buy the supply*. It exists because the R-17 measurement (M-01) produced a result
that was not a cryptography result at all: meeting the confidentiality property requires **declining
the customers most likely to sign up**. A constraint of that shape cannot live in a reading-list
domain. It is a market constraint discovered by a security investigation, and it needs a home where
product decisions can see it.

**Three rules govern this document.**

1. **Every number carries its source and its date.** Indian telecom, tariff and cloud pricing all
   move on quarterly cycles. A number without a date is a `[VENDOR-DEFAULT]` waiting to happen —
   `reading-list.md` §2 already found fourteen of those across nine ADRs, and ADR-077 exists to stop
   the fifteenth.
2. **Provenance tags are the same four the LTS Literature Standard defines** — `[P-num]` measured
   and published, `[DERIVED]` computed here with working shown, `[INFERRED]` a judgement,
   `[ASSUMED]` a value nobody has measured. **An ADR may not rest on `[INFERRED]` or `[ASSUMED]`
   alone.**
3. **This document does not resolve forks.** Where the market evidence opens an architectural
   decision, it states the fork, prices both sides, and refers it to `/design-council`. §7 is the
   standing referral list.

This study runs **parallel** to the technical literature track, not behind it. Domain P (repair
security) and Domain K (threshold decoupling) continue on their own sequence. The two tracks meet at
exactly three parameters — provider allocation, expansion factor, and ASN placement weight — and §6
is where that intersection is recorded.

---

## §1 — The supply side: who can actually be a provider

*Source: M-01 `measurement-01-asn-diversity-india.md`; decisions in ADR-081.*

### 1.1 The recruitable population is not the Indian Internet

`[P-num]` APNIC Labs, India, 23 Jun 2026: the two largest eyeball ASNs are **AS55836**
(RELIANCEJIO-IN, 294,443,636 users, 49.85%) and **AS45609** (BHARTI-MOBILITY, 140,143,441 users,
23.73%). Together **73.58%** of Indian Internet users.

`[DERIVED]` Both are mobile. Vyomanaut providers are mains-powered desktops with persistent storage
(ADR-057 sleep inhibition, ADR-046 storage engines, the 100 KB/s background budget). **The two ASNs
holding three-quarters of India's Internet users contribute approximately zero candidate providers.**

Every published "Indian AS diversity" statistic is computed over this population and is therefore
answering a different question than Vyomanaut's.

### 1.2 The addressable supply base

`[P-num]` TRAI, June 2026 — India's **wireline** subscriber base is **47.82 million**, and it is
*shrinking*: down 1.67% month-on-month from 48.64 M in May, driven by BSNL losing 796,408 fixed lines
in a single month.

| Operator | wireline subs (Jun 2026) | share | source |
| --- | --- | --- | --- |
| Reliance Jio | 15,625,342 | **32.67%** | `[P-num]` TRAI |
| Bharti Airtel | 11,990,437 | **25.07%** | `[P-num]` TRAI |
| BSNL | 6,637,405 | 13.88% | `[P-num]` TRAI |
| Vodafone Idea | 783,861 | 1.64% | `[P-num]` TRAI |
| all others | 12,782,955 | 26.73% | `[DERIVED]` residual |

`[P-num]` Urban households hold **89.13%** of wireline connections; rural wireline tele-density is
**0.58%** against urban **8.41%** (TRAI, May 2026).

**Market read** `[INFERRED]`: the supply base is urban, fixed-line, and 47.8 M households at the
absolute ceiling — before filtering for desktop ownership, always-on tolerance, and willingness. It
is *not* the 1.29 billion-connection number that Indian telecom headlines describe. Any go-to-market
model anchored on India's mobile scale is anchored on the wrong population by a factor of ~27.

### 1.3 Supply concentration is the binding constraint, not supply count

`[DERIVED]` M-01 §3.4 — 49 independent fixed-line operators exist, but the **effective number of
independent placement domains is `N_eff = 1/HHI = 5.161`**, against ADR-081's requirement of 8.
Top-3 combined: **71.63%**.

`[DERIVED]` M-01 §5.3 — holding every domain under the placement ceiling requires the tail
over-recruited **2.203×** and Reliance Jio recruited at **0.38×** its natural rate.

**This is the market study's first hard constraint, and it is a customer-acquisition constraint
wearing a cryptography costume.** Stated plainly: *Vyomanaut must refuse roughly six out of ten Jio
customers who try to become providers, and must find 2.2× more small-ISP customers than the market
naturally supplies.* Any acquisition funnel that optimises for volume violates the confidentiality
property. See ADR-081 Decisions 1 and 2.

`[DERIVED]` M-01 §5.4 — reaching 8 usable domains by untargeted enrolment needs **427 providers**
against ADR-029's bootstrap gate of 56: a **7.6× fleet multiplier**. Targeted, ISP-segmented
acquisition is not an optimisation here. It is the only route.

### 1.4 Multi-ASN ownership

`[P-num]` ACT / Atria Convergence operates **four** ASNs (AS24309, AS55577, AS18209, AS131269).
`[DERIVED]` Under a per-ASN 20% cap that admits `4 × 11 = 44` of 56 shards — **78.6% of a stripe to
one company, against a disclosure threshold of 16, with no collusion at all.** ADR-081 Decision 2
moves enforcement to per-organisation as a result.

**Commercial consequence** `[INFERRED]`: an ISP-partnership acquisition channel — the obvious way to
solve §1.3 — is *also* the fastest way to concentrate a stripe inside one legal entity. The channel
that fixes the distribution problem creates the collusion problem. This is a genuine fork; see §7.1.

---

## §2 — The unit economics: the number that governs everything

*Working: `docs/research/data/market/unit_economics.py`. Decisions in ADR-080.*

### 2.1 The expansion factor is the whole story

```
RS(k=16, n=56)  →  expansion = 56/16 = 3.50×
```

`[DERIVED]` One GB of owner data occupies **3.50 GB** of provider space. Therefore:

> **owner-facing cost floor = 3.50 × (provider cost per raw GB-month)**

This is not a tuning parameter. `n = 56` and `k = 16` are compile-time constants (ADR-003, ADR-022),
and the audit, repair and placement subsystems are all sized against them.

`[DERIVED]` For comparison, Storj's RS(29,80) gives **2.76×**. Vyomanaut carries a **1.27× worse
storage multiplier than the closest deployed comparator**, before any other cost is counted.

### 2.2 The provider's marginal electricity cost

`[P-num]` Indian domestic tariffs, FY 2025-26 / 2026: national average **₹5.5/unit**; range ₹0
(Tamil Nadu subsidised first 100 units) to ₹10.19/unit (Maharashtra above 500 units); metro slabs
commonly ₹7–9/unit at household consumption levels.

`[P-num]` **The subsidy cliff matters more than the average.** Delhi households crossing 400
units/month lose the state subsidy entirely and pay ~₹9/unit on *every* unit above it, against ~₹4.50
effective below the cliff. `[INFERRED]` Vyomanaut's incremental consumption is therefore charged at
the household's **marginal** slab, and can itself trigger the cliff — a household at 380 units that
adds a dedicated always-on desktop is pushed over it. The relevant price is never the average tariff.

`[DERIVED]` Cost per **raw** provider GB-month at the ADR-fixed **70 GB** allocation, electricity
only, zero provider profit:

| provider scenario | ₹3.0 subsidised | ₹5.5 national avg | **₹7.0 metro marginal** | ₹9.0 post-cliff |
| --- | --- | --- | --- | --- |
| (a) I/O only, PC on anyway — 2 W | 0.062 | 0.113 | **0.144** | 0.185 |
| (b) dedicated HDD, PC on anyway — 8 W | 0.247 | 0.453 | **0.576** | 0.741 |
| (c) dedicated desktop 24/7 — 65 W | 2.006 | 3.677 | **4.680** | 6.017 |

Working, row (b) at ₹7: `8 W × 720 h / 1000 = 5.76 kWh`; `5.76 × ₹7 = ₹40.32/month`;
`₹40.32 / 70 GB = ₹0.576 per raw GB-month`. ✓

### 2.3 The owner-facing floor against the Indian market

`[P-num]` Indian retail cloud storage, ₹ per GB-month:

| product | ₹/GB-month | source |
| --- | --- | --- |
| **Google One 2 TB, annual (₹6,500/yr)** | **0.271** | Google One India |
| Google One 2 TB, monthly (₹650) | 0.325 | Google One India |
| Storj legacy, $4/TB-mo | 0.348 | Storj docs `[ASSUMED FX ₹87/USD]` |
| Storj Regional Workflows, $10/TB-mo (from 1 Jul 2026) | 0.870 | Storj docs |
| Google One 200 GB (₹210) | 1.050 | Google One India |
| Google One 100 GB (₹130) | 1.300 | Google One India |
| iCloud+ 50 GB (₹75) | 1.500 | Apple India |
| Google One Lite 30 GB (₹59) | 1.967 | Google One India |

`[DERIVED]` Applying the 3.50× expansion at the metro marginal slab:

| provider scenario | raw ₹/GB-mo | **owner floor (×3.5)** | **vs Google One 2 TB** |
| --- | --- | --- | --- |
| (a) I/O only, PC on anyway | 0.144 | **0.504** | **1.9×** |
| (b) dedicated HDD, PC on anyway | 0.576 | **2.016** | **7.4×** |
| (c) dedicated desktop 24/7 | 4.680 | **16.380** | **60.5×** |

**Read this carefully. That is electricity alone — before the provider earns one rupee, before
repair bandwidth, before the microservice, before Razorpay's cut, before margin.** In the most
generous scenario physically available — the desktop is on anyway, Vyomanaut costs only the extra
I/O — the owner-facing floor is still **1.9× the cheapest Indian consumer cloud plan**.

**F-MKT-01: Vyomanaut cannot win on price against hyperscale consumer cloud in India, in any
scenario, at any tariff, at the fixed parameter set.** This is arithmetic, not pessimism.

### 2.4 The provider loses money

`[P-num]` Storj pays node operators **$1.50/TB-month** for raw shared space, and states this is
"inclusive of the expansion factor" — the node is paid for bytes it holds, and the network absorbs
the multiplier. `[DERIVED]` `$1.50/TB = ₹0.1305 per raw GB-month` at ₹87/USD `[ASSUMED]`.

`[DERIVED]` A Vyomanaut provider at the fixed 70 GB allocation, paid at Storj's rate, metro marginal
slab:

| scenario | revenue/mo | electricity/mo | **net** |
| --- | --- | --- | --- |
| (a) I/O only, PC on anyway | ₹9.13 | ₹10.08 | **−₹0.95** |
| (b) dedicated HDD, PC on anyway | ₹9.13 | ₹40.32 | **−₹31.19** |
| (c) dedicated desktop 24/7 | ₹9.13 | ₹327.60 | **−₹318.46** |

**F-MKT-02: at the fixed 70 GB allocation, an Indian provider is cash-negative in all three
scenarios, including the one where the machine was going to be on regardless.** The proposition
"earn income from idle disk space" is, as currently parameterised, false.

Note that (a) is the *best* case and it is still negative. There is no operating point at 70 GB where
the provider profits at a competitive payout rate.

### 2.5 The break-even allocation — the design-influencing number

`[DERIVED]` Allocation at which a Storj-rate payout merely covers marginal electricity (₹7/unit):

| scenario | break-even allocation | vs the 70 GB constant |
| --- | --- | --- |
| (a) I/O only, PC on anyway | 77 GB | **1.1×** |
| (b) dedicated HDD, PC on anyway | 309 GB | **4.4×** |
| (c) dedicated desktop 24/7 | 2,510 GB | **35.9×** |

`[DERIVED]` Allocation needed for a *motivating* provider income — target net **₹500/month**,
assuming the owner pays ₹1.00/GB-month and 60% of revenue reaches the provider (Storj's stated
node-share ratio), giving a provider rate of `₹1.00 × 0.60 / 3.50 = ₹0.1714` per raw GB-month:

| scenario | allocation needed | vs the 70 GB constant |
| --- | --- | --- |
| (a) I/O only, PC on anyway | 2,975 GB | **42.5×** |
| (b) dedicated HDD, PC on anyway | 3,152 GB | **45.0×** |
| (c) dedicated desktop 24/7 | 4,828 GB | **69.0×** |

**F-MKT-03: the 70 GB provider allocation is between 1.1× and 36× too small to break even, and
between 42× and 69× too small to motivate.** The economically coherent allocation is approximately
**3 TB**, not 70 GB.

This is the market study's central design finding, and it is a *fixed compile-time constant* that
`reading-list.md` §2 already lists among the underived parameters. ADR-080 acts on it.

**Why the constant fails:** the provider's dominant cost is a *fixed* host cost — the machine's power
draw — amortised across the allocation. At 70 GB it is amortised across almost nothing. Storj's node
economics work because a typical node shares multiple terabytes, driving the per-GB share of a fixed
8 W host cost down by two orders of magnitude. Vyomanaut chose the allocation without this
denominator in view.

---

## §3 — Reliability: the requirement the supply base cannot meet

`[P-num]` Storj's node operator terms require: **≥99.3% uptime per month**, ≥5 Mbps upstream,
≥25 Mbps downstream, ≥2 TB monthly bandwidth.

`[DERIVED]` 99.3% of a 30-day month permits **5.04 hours** of downtime.

`[INFERRED]` An Indian residential desktop on grid power with no UPS will not hold that in most of
the country, and the project has no measurement to say otherwise. Domain D's **R-15** ("Indian
residential broadband availability — uptime, outage duration, power-availability data for IN
residential") is the reading-list item that owns this, and it is **unread**. `reading-list.md` names
CEA and state-utility outage data as the source.

**This is the highest-value unread item in the corpus for market purposes**, because it prices two
things at once: the redundancy the erasure code must carry (which sets `n/k`, which sets §2.1, which
sets everything), and whether a UPS becomes a *de facto* provider prerequisite — turning a
zero-capex proposition into a ₹3,000–6,000 capex one.

`[INFERRED]` If a UPS is required, F-MKT-02 worsens: the provider now has payback-period arithmetic
before the first rupee, and at −₹0.95/month best-case revenue the payback period is undefined.

**Open as Q-MKT-3.** Do not derive redundancy parameters from Bolosky or from Storj's NAS population
until R-15 lands; both describe different power regimes.

---

## §4 — The demand side: where value might actually be

§2 rules out price. That does not rule out the product; it rules out one axis. Three candidate demand
theses, with what is known and what is not.

### 4.1 Sovereignty and compliance

`[P-num]` India's data-centre and localisation environment is materially real: **245 operational data
centres**; **86% of India's 1,000 most-visited websites** servable from within India; Union Budget
2026–27 introduced a **tax holiday until 2047** for eligible foreign companies providing cloud
services from India-based data centres (APNIC Economy Report, 17 Aug 2026).

`[INFERRED]` The thesis: buyers who need data resident in India, outside hyperscaler control, with
client-held keys, are not price-sensitive in the way §2.3 measures. Vyomanaut's actual product on
this thesis is **computational confidentiality resting on client-held key material** — the claim
ADR-022 Addendum A §5 already established is the true one.

**What is not known:** whether such buyers exist at consumer or prosumer scale, or only at
enterprise scale where Vyomanaut's provider pool (residential desktops, §3) is disqualifying. **No
measurement.** Q-MKT-1.

### 4.2 The provider as the customer

`[INFERRED]` The thesis: the income proposition is the product, and data owners are the input cost.

§2.4 says this thesis is currently false — providers are cash-negative — and §2.5 says it needs a
~45× allocation change to become true. It is not eliminated; it is **priced**, and the price is
ADR-080.

**Caution flagged by the corpus itself:** Paper 47 (Rosenblat & Stark, algorithmic labour) is already
in the corpus. A platform that pays participants below their marginal cost while presenting it as
income is the exact failure mode that paper documents. **A provider-facing earnings claim must be
net of measured electricity at the household's marginal slab, not gross.** This is a requirement, not
a preference; ADR-081 Decision 4 makes it binding.

### 4.3 Payment rail as differentiator

`[P-num]` Storj pays node operators in STORJ tokens. Vyomanaut pays in **rupees over UPI**
(ADR-035, Razorpay).

`[INFERRED]` For an Indian provider, "₹X to your bank account via UPI" versus "STORJ tokens you must
bridge to an exchange" is a materially different product, and it is the one genuine structural
advantage identified in this study that competitors cannot trivially copy — it is regulatory and
operational, not technical. Paper 48 (crypto wallet UX) and Paper 57 (DEPA / Account Aggregator) are
the corpus's existing anchors.

**Consequence** `[INFERRED]`: the differentiator sits on the **supply** side, not the demand side.
That is evidence for §4.2's thesis and against §4.1's — the opposite of what §2 alone suggests. The
fork is real; see §7.2.

---

## §5 — Findings register

| ID | Finding | Evidence | Acts on |
| --- | --- | --- | --- |
| **F-MKT-01** | Vyomanaut cannot win on price against Indian consumer cloud in any scenario | §2.3 — best case 1.9×, worst 60.5× Google One 2 TB, electricity alone | ADR-081 |
| **F-MKT-02** | At 70 GB the provider is cash-negative in all scenarios, including PC-on-anyway | §2.4 — −₹0.95 to −₹318.46/month at Storj payout rates | ADR-080 |
| **F-MKT-03** | The 70 GB allocation is 1.1–36× below break-even and 42–69× below motivating | §2.5 — coherent allocation ≈ 3 TB | ADR-080 |
| **F-MKT-04** | The confidentiality property imposes a customer-acquisition constraint: decline ~62% of Jio applicants, over-recruit the tail 2.2× | M-01 §5.3; ADR-081 | ADR-081, §7.1 |
| **F-MKT-05** | The addressable supply base is 47.8 M shrinking wireline connections, 89% urban — not India's mobile scale | §1.2 `[P-num]` TRAI Jun 2026 | §7 planning |
| **F-MKT-06** | The 3.5× expansion factor is 1.27× worse than the closest deployed comparator and is a compile-time constant | §2.1 | ADR-080, §7.3 |
| **F-MKT-07** | Reliability requirements comparable to Storj's 99.3% permit 5.04 h downtime/month; whether Indian residential grid+broadband meets it is **unmeasured** | §3; R-15 unread | Q-MKT-3 |

---

## §6 — Where the market track and the technical track intersect

Exactly three parameters are shared. Changing any of them changes both tracks, and neither track may
change them unilaterally.

| Parameter | Technical owner | Market consequence | Status |
| --- | --- | --- | --- |
| **Provider allocation (70 GB)** | ADR-046, audit sampling (`c = 2,867 chunks/day`), `n = 286,720` chunks | §2.5 — the single largest lever on provider viability; ~45× off | **ADR-080 opens it** |
| **Expansion factor `n/k = 3.5`** | ADR-003, ADR-022, Domain C | §2.1 — multiplies every owner-facing price by 3.5 | Frozen for demo; LTS-open |
| **ASN placement weight** | ADR-014, ADR-081 | §1.3 — sets the acquisition funnel's shape | **ADR-081 Decision 2** |

`[INFERRED]` Note the coupling that makes ADR-080 non-trivial: raising the allocation to ~3 TB
multiplies `n` (chunks per provider) by ~45, and the audit sample count `c = audit_sample_rate × n`
rises with it. At `audit_sample_rate = 0.01`, `c` goes from 2,867 to roughly 128,000 chunks/day per
provider. **The allocation is not a free parameter; it is an audit-load parameter wearing a storage
costume.** ADR-080 records this and does not pretend otherwise.

---

## §7 — Forks referred to `/design-council`

This document prices forks. It does not resolve them. Each entry states the fork, both sides, and
what evidence would settle it.

### 7.1 — ISP-partnership acquisition vs placement independence

The only tractable route to §1.3's 2.2× tail over-recruitment is segmented acquisition through small
and regional ISPs. That same channel concentrates providers inside single legal entities — exactly
what ADR-081 Decision 2's per-organisation cap exists to prevent, and exactly what ACT's four ASNs
(§1.4) demonstrate is already possible without any partnership at all.

**Side A:** partner with regional ISPs; solve the distribution problem; accept concentration risk
managed by the per-organisation cap.
**Side B:** direct-to-consumer acquisition only; preserve independence; accept the 7.6× fleet
multiplier and a far slower path to eight domains.

**Settles on:** Q-R17-4 (is an ASN→organisation mapping maintainable?) and a decision about whether
per-organisation caps are enforceable against a partner who is also a distribution channel.

### 7.2 — Sovereignty product vs supply-side income product

§2 rules out price. §4.1 and §4.3 point in *opposite* directions: the compliance thesis puts the data
owner at the centre and tolerates a small expensive supply base; the income thesis puts the provider
at the centre and requires ADR-080's ~45× allocation change plus a demand base large enough to pay
for it.

**This is the product's identity, and it changes the architecture.** The compliance thesis makes
provider count a cost to minimise; the income thesis makes it the growth metric. They are not
compatible roadmaps.

**Settles on:** Q-MKT-1 (does a sovereignty-motivated Indian buyer exist below enterprise scale?) and
ADR-080's outcome.

### 7.3 — Whether `n/k = 3.5` survives contact with the market

§2.1 shows the expansion factor multiplies every owner-facing price. Domain C ("what survives of
erasure optimisation") is marked `[CLOSED except two]` on technical grounds. §2.3 reopens it on
commercial grounds: a code with expansion 2.5× would cut the owner floor by 29%.

**This must not be reopened casually.** `n = 56` and `k = 16` are load-bearing across ADR-003,
ADR-004, ADR-014, ADR-022, ADR-059, ADR-076, ADR-078 and ADR-081, and `k = 16` *is* the AONT-RS
disclosure threshold — lowering it lowers the confidentiality threshold with it, which is Domain K's
entire problem restated.

**Settles on:** whether Domain K's R-30 (privacy threshold above reconstruction threshold) succeeds.
If R-30 lands, `k` and the disclosure threshold decouple and the expansion factor becomes tunable on
durability grounds alone. **If R-30 fails, this fork is closed and 3.5× is permanent.** Council
should not convene on 7.3 before R-30 is read.

---

## §8 — Open questions

| ID | Question | Blocks | Priority |
| --- | --- | --- | --- |
| **Q-MKT-1** | Does a sovereignty-motivated Indian storage buyer exist below enterprise scale, and what will they pay per GB-month? | §7.2; the entire demand thesis | **High** |
| **Q-MKT-2** | What is the true marginal power draw of a Vyomanaut provider daemon on representative Indian desktop hardware, measured at the wall? §2.2's 2/8/65 W are `[ASSUMED]` bracket values, not measurements | Every number in §2.2–2.5 | **High** |
| **Q-MKT-3** | R-15 — Indian residential broadband and grid availability. Uptime, outage duration, CEA/state-utility outage data | §3; the redundancy parameters; whether a UPS is a prerequisite | **High** |
| **Q-MKT-4** | R-13 — Storj's published node-churn statistics for the *incentivised* population, and Indian node presence within it | Whether §2.4's negative economics are already visible as absent Indian nodes | Medium |
| **Q-MKT-5** | Indian fixed-broadband FUP and upload-speed distribution: can a provider sustain repair egress under a consumer plan's fair-use policy? | Repair bandwidth cost; ADR-076's budget | Medium |
| **Q-MKT-6** | What is the correct INR/USD assumption? §2 uses ₹87/USD `[ASSUMED]` throughout; every competitor comparison is denominated in it | Precision of §2.3–2.5, not their direction | Low |

**Note on Q-MKT-2.** It is the cheapest of the three High items — a wall meter and a weekend — and it
is upstream of every economic number in this document. It should be done first.

---

## §9 — Data and reproducibility

- `docs/research/data/market/unit_economics.py` — §2 in full, re-runnable
- `docs/research/data/R-17/` — `analyse.py`, `asn_in_fixed.csv`, `analysis_output.txt` for §1
- `docs/research/measurement-01-asn-diversity-india.md` — the supply-side measurement, its threats to
  validity and its falsifiers
- ADR-081, ADR-080, ADR-081 — the decisions this document forced

**Amendment rule (ADR-081):** a number in this document may be replaced only by a measurement, and
the replacement carries its own date and source. A number may not be replaced by a council judgement
— a council may decide what to *do* about a number, not what the number is.
