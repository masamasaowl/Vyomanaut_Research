# ADR-082 — The 70 GB Provider Allocation Fails the Unit-Economics Gate

**Status:** Proposed
**Track:** LTS
**Topic:** #6 Provider Economics / #2 Storage Sizing / #9 Audit Load
**Amends:** the `provider_size_gb = 70` compile-time constant and every document that treats it as
settled; ADR-046 (storage engines, sizing assumptions); ADR-060 (audit sampling, which is derived
from it)
**Research source:** `market-study.md` §2 — `docs/research/data/market/unit_economics.py`
**Depends on:** Q-MKT-2 (measured wall power) before this ADR leaves `Proposed`

---

## Context

`provider_size_gb = 70` sits in the standing Vyomanaut parameter set as a compile-time constant,
alongside `RS(16,56)` and the 256 KiB chunk. Unlike those two it has no derivation. `reading-list.md`
§2 lists it among the fourteen `[UNDERIVED]` parameters across nine ADRs, and ADR-077 governs the
tag.

The market study derived it, for the first time, from the provider's actual cost structure. **The
constant fails by between one and two orders of magnitude**, and it fails in a way that makes the
project's stated provider proposition — income from idle disk space — arithmetically false.

This ADR records the arithmetic and opens the parameter. It does not set a new value, because the
input that would set it (Q-MKT-2, measured wall power) has not been measured.

---

## The arithmetic

All working reproducible from `unit_economics.py`. FX ₹87/USD is `[ASSUMED]` and flagged wherever
used.

### A. The multiplier

```
expansion = n/k = 56/16 = 3.50×
owner-facing cost floor = 3.50 × (provider cost per raw GB-month)

Storj RS(29,80) = 2.76×   →  Vyomanaut carries a 1.27× worse multiplier
```

### B. Provider marginal electricity, at the fixed 70 GB allocation

```
cost per raw GB-month = W × 0.720 kWh × tariff / 70
```

`[P-num]` Indian domestic tariff, national average ₹5.5/unit FY 2025-26; metro marginal slabs
₹7–9/unit; Delhi loses its subsidy entirely above 400 units and jumps to ~₹9/unit.

At the metro marginal slab (₹7/unit):

| scenario | W | kWh/month | ₹/month | **₹ per raw GB-month** |
| --- | --- | --- | --- | --- |
| (a) I/O only, PC on anyway | 2 | 1.44 | 10.08 | **0.144** |
| (b) dedicated HDD, PC on anyway | 8 | 5.76 | 40.32 | **0.576** |
| (c) dedicated desktop 24/7 | 65 | 46.80 | 327.60 | **4.680** |

Hand check, row (b): `8 × 720 / 1000 = 5.76 kWh`; `5.76 × 7 = ₹40.32`; `40.32 / 70 = ₹0.576`. ✓

### C. The owner floor against the market

`[P-num]` Google One 2 TB annual, India: ₹6,500/year → `6500 / 12 / 2000 = ₹0.2708 per GB-month`.
The cheapest Indian consumer benchmark.

| scenario | raw | **× 3.5 = owner floor** | **vs Google One 2 TB** |
| --- | --- | --- | --- |
| (a) | 0.144 | **0.504** | **1.9×** |
| (b) | 0.576 | **2.016** | **7.4×** |
| (c) | 4.680 | **16.380** | **60.5×** |

**Electricity alone. No provider profit, no repair bandwidth, no microservice, no payment-rail fee,
no margin.** (F-MKT-01.)

### D. The provider's net position

`[P-num]` Storj node payout: **$1.50/TB-month** for raw shared space, stated as inclusive of the
expansion factor — the node is paid for bytes held; the network absorbs the multiplier.
`[DERIVED]` = **₹0.1305 per raw GB-month**.

At 70 GB, revenue = `70 × 0.1305 = ₹9.13/month`:

| scenario | revenue | electricity | **net** |
| --- | --- | --- | --- |
| (a) I/O only, PC on anyway | ₹9.13 | ₹10.08 | **−₹0.95** |
| (b) dedicated HDD | ₹9.13 | ₹40.32 | **−₹31.19** |
| (c) dedicated desktop | ₹9.13 | ₹327.60 | **−₹318.46** |

**There is no operating point at 70 GB where the provider profits.** The best case — the machine was
running regardless and Vyomanaut costs only extra I/O — is still negative. (F-MKT-02.)

### E. Break-even and motivating allocations

Break-even against marginal electricity at a Storj-rate payout, ₹7/unit:

```
GB_breakeven = (W × 0.720 × 7) / 0.1305
```

| scenario | break-even | vs 70 GB |
| --- | --- | --- |
| (a) | 77 GB | **1.1×** |
| (b) | 309 GB | **4.4×** |
| (c) | 2,510 GB | **35.9×** |

Motivating allocation, target net **₹500/month**, owner price ₹1.00/GB-month, 60% of revenue to the
provider → provider rate `1.00 × 0.60 / 3.50 = ₹0.1714` per raw GB-month:

```
GB_motivating = (500 + electricity_cost) / 0.1714
```

| scenario | allocation needed | vs 70 GB |
| --- | --- | --- |
| (a) | 2,975 GB | **42.5×** |
| (b) | 3,152 GB | **45.0×** |
| (c) | 4,828 GB | **69.0×** |

**The economically coherent allocation is approximately 3 TB.** (F-MKT-03.)

### F. Why the constant fails

`[INFERRED]` The provider's dominant cost is a **fixed host cost** — the machine's power draw —
amortised across the allocation. At 70 GB it is amortised across almost nothing. Storj's node
economics close because a typical node shares multiple terabytes, driving the per-GB share of a fixed
8 W host down by roughly two orders of magnitude.

The constant was chosen without that denominator in view. This is not an error of arithmetic; it is
a parameter set by intuition about *disk a household can spare* rather than by *cost a household
incurs*, and the two differ by ~45×.

---

## Decision

**1. `provider_size_gb = 70` is withdrawn as a settled constant and re-tagged `[UNDERIVED]` under
ADR-077.**

It may not be cited as a design premise in any new document until Decision 3 lands. Documents that
already cite it are not rewritten — the demo track is frozen under ADR-062 and 70 GB is a legitimate
*demo* value — but the LTS track carries no commitment to it.

**2. A provider-facing earnings claim is binding only if it is net of measured electricity at the
household's marginal slab.**

Gross-of-power earnings figures are prohibited in provider-facing copy, onboarding, the Wails GUI's
earnings display, and any marketing artefact. §D shows gross and net differ in *sign*, not degree.
Paper 47 (Rosenblat & Stark) documents this exact failure mode in the corpus already; this clause
makes avoiding it a rule rather than an intention.

**Corollary:** the earnings display must model the household's **marginal** tariff slab, not the
national average. §B's spread between ₹3.0 and ₹9.0 is 3×, and Delhi's subsidy cliff means a
household near 400 units can see its *whole bill* reprice. A flat national-average assumption is a
`[VENDOR-DEFAULT]` in the exact sense ADR-077 prohibits.

**3. The replacement allocation is set by measurement, not by council.**

The blocking input is **Q-MKT-2**: measured wall-power draw of the provider daemon on representative
Indian desktop hardware, at idle and under audit load. §B's 2 W / 8 W / 65 W are `[ASSUMED]` bracket
values chosen to span the physical range, not measurements. The bracket is 32× wide and the answer
moves the allocation by 32×.

Until Q-MKT-2 lands, **no numeric replacement is adopted.** The LTS-track planning band is
**300 GB – 3 TB**, per §E, and is explicitly a band, not a value.

**4. The allocation is an audit-load parameter and must be raised as one.**

`[DERIVED]` At `chunk = 256 KiB`, 70 GB gives `n = 286,720` chunks, and `audit_sample_rate = 0.01`
gives `c = 2,867` chunks/day per provider. Raising the allocation to 3 TB multiplies `n` by ~44.7 to
roughly **12.8 million chunks**, and at a fixed sample rate `c` rises to approximately **128,000
chunks/day per provider**.

That is a 44.7× increase in audit work per provider, and it lands on ADR-059's homomorphic
authenticator path and ADR-060's schedule, both of which were sized against 2,867.

**Therefore: any allocation change is a joint change to `provider_size_gb` and `audit_sample_rate`,
and neither may move alone.** A session that raises the allocation without re-deriving the sample
rate has moved the audit load by 44.7× silently, which is the shape of failure `reading-list.md`
§2.6 exists to prevent.

**5. This ADR does not decide the product's positioning.**

§C shows Vyomanaut cannot win on price. Whether it should therefore become a sovereignty product
(where allocation matters less) or a supply-side income product (where Decision 3's answer is
existential) is `market-study.md` §7.2, and it is referred to `/design-council`. **Do not convene
that council before Q-MKT-2.** The fork is not decidable without the number.

---

## Consequences

**Immediate.** One compile-time constant re-opened and re-tagged. One prohibition on provider-facing
copy that binds before any such copy exists — deliberately, because it is cheaper now than after.
One joint-change rule coupling storage sizing to audit sampling. One measurement (Q-MKT-2) promoted
to the critical path for the LTS track's economics.

**For the demo track.** Nothing. ADR-062 freezes the demo, 70 GB is a demo value, and the demo makes
no earnings claim. This ADR is LTS-only.

**For the audit subsystem.** Decision 4 makes ADR-059 and ADR-060 downstream of a market parameter
for the first time. That coupling is real and was previously invisible.

**What this does not settle.** Whether a 3 TB allocation is *available* on Indian consumer desktops
at all. `[INFERRED]` a household willing to share 3 TB is a different household from one willing to
share 70 GB — plausibly a NAS owner or an enthusiast rather than a typical desktop user — and that
would narrow the supply base already measured at 47.8 M wireline connections (market study §1.2), on
top of the ASN-diversity constraint from ADR-083. **The two supply constraints compound and nobody
has multiplied them yet.** Q-MKT-2 and Q-R17-3 are both upstream of doing so.

---

## Limits of this decision

- **§B's power figures are `[ASSUMED]`.** The three scenarios bracket a physical range; none is
  measured. Decision 3 exists because of this and no numeric replacement is adopted.
- **FX at ₹87/USD is `[ASSUMED]`.** Every Storj comparison is denominated in it. A 15% FX move
  changes §D's revenue figures by 15% and does not change any sign: scenario (a) at ₹87 is −₹0.95,
  and would need FX at roughly ₹96/USD to reach break-even at 70 GB — which is a currency bet, not a
  business model.
- **The ₹500/month target in §E is a judgement, not a measurement.** It is a plausible motivating
  amount for an Indian household, not a surveyed one. The *break-even* column (§E upper table) does
  not depend on it and is the more defensible of the two.
- **Storj's $1.50/TB is used as a payout benchmark, not a target.** Vyomanaut may pay more. Paying
  more raises the owner floor in §C proportionally, since the owner floor is `3.5 ×` the provider
  rate — which is the trap: **every rupee added to provider income costs the owner ₹3.50.**

---

## References

- `market-study.md` §2 — the derivation, in full, with the market comparison table
- `docs/research/data/market/unit_economics.py` — re-runnable working
- ADR-046 — storage engines; sizing assumptions downstream of this constant
- ADR-059, ADR-060 — audit authenticators and schedule; coupled by Decision 4
- ADR-062 — two-track development; this ADR is LTS-only
- ADR-077 — `[UNDERIVED]` / `[VENDOR-DEFAULT]` triage; `provider_size_gb` re-enters the register
- ADR-083 — the supply-side placement constraint that compounds with this one
- ADR-083 — establishes `market-study.md`'s standing and the amendment rule
- Paper 47 — Rosenblat & Stark; the basis for Decision 2
