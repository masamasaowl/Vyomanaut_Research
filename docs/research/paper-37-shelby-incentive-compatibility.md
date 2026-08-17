# Paper 37 — Rationally Analyzing SHELBY: Proving Incentive Compatibility in a Decentralized Storage Network

**Authors:** Michael Crystal (Stanford University), Guy Goren (Aptos Labs), Scott Duke Kominers (Harvard University and a16z crypto)
**Venue / Year:** arXiv:2510.11866 | October 2025
**Topics:** #2, #18, #19
**ADRs produced:** none originally — ADR-002 and ADR-024 confirmed. **On revision:** ADR-060 Addendum A, ADR-014 Addendum A (jointly with Paper 73)

> **Revised 2026-08-17 (LTS Band 0, item 4 — re-derivation, not a new note).**
> The paper was re-read against the sampled-audit design that did not exist when this note was
> written. Two claims are **withdrawn**: the mapping of SHELBY's `p_au` onto Vyomanaut's audit
> sampling rate, and the conclusion drawn from it that Theorem 1 Condition (i) collapses to the
> participation constraint at `p_au ≈ 1`. Both were category errors — `p_au` is an
> *audit-the-auditor* inspection rate in a two-layer auditing scheme, and Vyomanaut has no auditor
> layer. The correct condition for a storage provider's deletion deviation is derived fresh in
> §Substitution below and is **not** any of `99×`, `V/2,909`, or the participation constraint alone.
> **Downstream documents corrected in the same session:** ADR-060's *"The economic constraint is
> looser under sampling"* section, via ADR-060 Addendum A. **Q66-2 closes.**
> Documents that consumed the withdrawn claims and are corrected: ADR-060 (§ noted above),
> `reading-list.md` §5 Domain A's `99×` remark, `answered-questions.md` Q41-2's closing sentence.
> The paper's own conclusions are unchanged — the error was ours, not the authors'.

---

## Problem Solved

Decentralized storage protocols consistently make claims of incentive compatibility but almost none prove them formally. Without proof, a system that distributes storage across many nodes may still fail to decentralize it in any meaningful sense, because self-interested providers have a dominant strategy to shirk. This paper provides the first formal game-theoretic proof that a deployed DSN protocol — SHELBY, from Aptos Labs and Jump Crypto — achieves full incentive compatibility under natural parameter conditions. The core mechanism: infrequent but perfectly verifiable on-chain audits of auditors discipline high-frequency cheap off-chain peer audits. For Vyomanaut, the paper's primary value is establishing a formal proof that the equilibrium Vyomanaut's economic design relies on actually exists — **and, on re-reading, establishing precisely which of its conditions Vyomanaut is and is not subject to.**

---

## Key Findings

**Proposition 1 — Off-chain audits alone collapse to universal shirking:**
If the audit mechanism relies solely on off-chain peer audits with no trusted backstop, the unique pure-strategy Nash equilibrium is full dishonesty: every storage provider stores nothing, performs no audits, and reports all other providers as having passed. The proof is tight — reporting 1 for others is strictly dominant (free reward, no cost), so all providers do it; given that all pass regardless, there is no incentive to store data or audit. This is the formal statement of the "who audits the auditor" problem. Any P2P storage system that relies only on peer-issued audit reports without a trusted verifier operates in this equilibrium.

**Observation 1 — Storing dominates on-the-fly reconstruction when fst × k × cr > cst:**
A provider who skips storage might attempt to reconstruct a chunk on demand during a challenge by fetching k chunks from k other providers. This is only viable if the reconstruction cost is less than the storage cost. With k=10, fst > 3, and cst/cr ≈ 2.5 (SHELBY's real-world parameters), storing always dominates reconstruction. The condition fst × k × cr > cst is the formal analogue of Vyomanaut's response deadline constraint.

> **Added on revision.** Observation 1 rests on a stated granularity assumption that was not recorded
> the first time and that turns out to be load-bearing for Vyomanaut. §2.3 argues reconstruction
> costs `k · c_r` *"given that data in the system is read at the granularity of full chunks"*, and
> footnote 6 concedes that SHELBY's format does support non-complete chunk access, dismissing it as
> impractical *"for on-the-fly audits of random chunks"* because of the data volumes involved.
> **If sub-chunk reads were cheap, `k · c_r` would fall by the sampling ratio and Observation 1
> would weaken proportionally.** ADR-060's decision to challenge *every block of every sampled
> chunk* — taken on disk-seek grounds, to make each sampled chunk one contiguous 256 KB vLog read —
> independently forces full-chunk reconstruction and therefore secures Observation 1 by
> construction. The rejected alternative in ADR-060's options table, *"sample individual blocks
> across the whole holding"*, would have destroyed it. See ADR-060 Addendum A §2.

**Theorem 1 — Three conditions for honest equilibrium uniqueness:**
With a backstop of occasional perfectly verifiable on-chain audit-of-auditor inspection (probability pau per **reported success**), the honest strategy is the unique Nash equilibrium if:
(i) slashing discourages false reporting: tau ≥ ((1 − pau)/pau) × rau + (1/(ε × pau)) × cau
(ii) audit rewards outweigh audit costs: rau ≥ (1/(1 − ε)) × cau
(iii) storage rewards exceed storage costs: rst ≥ cst

> **Added on revision — the correction that matters.** Read the parameter table in §2.4, not the
> conditions in isolation. `p_au` is *"probability that a **1 entry in an auditor report** is
> selected for on-chain proof verification"*, and `t_au` is *"slashing penalty for failing
> **audit-the-auditor** verification."* Conditions (i) and (ii) are both conditions on the
> **auditing layer** — they discipline storage providers acting as *auditors of each other*.
> Condition (iii) is the only condition on storage itself, and it is the bare participation
> constraint.
>
> SHELBY defines a separate parameter `t_st`, *"slashing penalty for on-chain storage audit
> failure"* (§2.2). **`t_st` does not appear anywhere in Theorem 1.** The paper does not derive a
> condition on the storage-side slashing penalty at all: storage honesty in its model is secured by
> (a) Condition (iii), (b) Observation 1, and (c) the fact that a majority of honest auditors will
> report a non-storing provider as failed, which cuts off `r_st` via the indicator
> `1{|{j : ρ = 1}| > N/2}` in the utility function (§3).
>
> **Vyomanaut has no auditor layer.** The microservice is the sole auditor and issues every
> challenge itself; providers never rate one another. That is recorded below under *Breaks* as a
> structural escape from Proposition 1, and it is correct — but it also means **Conditions (i) and
> (ii) have no Vyomanaut referent at all.** They are vacuous, not satisfied.
>
> The prior version of this note mapped `p_au` onto "probability of microservice challenge on any
> given chunk ≈ 1" and concluded Condition (i) collapses. `reading-list-v2` then mapped it onto the
> 1% sampling rate and got `99×`. ADR-060 re-derived and got `V/2,909`. **All three substitute into
> a condition about auditors, using a rate about storage.** The correct derivation is in
> Substitution below and it is a different inequality entirely.

**Theorems 2 and 3 — Coalition resistance up to N/2:**
Without commitment power among coalition members, the honest strategy is robust against any coalition of fewer than N/2 providers — no joint deviation can improve all members' payoffs. With commitment power, a coalition can avoid internal auditing costs but cannot suppress data storage, because non-coalition members (the majority) will audit truthfully and honest audits cannot be faked. The maximum collective gain from a coalition of size |J| with commitment power is bounded by |J|² × cau — a function of wasted audit costs, not stolen storage rewards.

**Proposition 3 — Vector commitments close the commitment-power gap:**
Adding a vector commitment over successful audit responses to the on-chain audit submission eliminates the residual coalition gain even when coalition members have full commitment power. A false 1 report that cannot be backed by a stored inclusion proof is slashed on inspection. This modification adds minimal overhead (one Merkle root per audit report) and converts the strong equilibrium from "near honest" to "exactly honest."

**Back-of-envelope calibration:**
Using SHELBY's real-world parameters (rst ≈ $30/TB/month, cst ≈ $5/TB/month, cr ≈ $2/TB, cau ≈ $10⁻⁷/audit, rau ≈ $5×10⁻⁷/audit, pau ≈ 2×10⁻⁴, tau ≳ $1000, fst ≈ 4 audits/chunk/month), the conditions in Theorem 1 are comfortably satisfied with large margins. The expected penalty for a false 1 report (pau × tau ≈ $0.2) exceeds the reward (rau ≈ $5×10⁻⁷) by five orders of magnitude. The reconstruction check comes out at 40·cr/cst ≈ 16. This is not a tight boundary — the design operates deep in the safe regime.

> **Added on revision.** The paper states explicitly in §5 that these are *"sufficient conditions and
> a design blueprint"*, not tight bounds, and lists four limitations: Nash rather than
> dominant-strategy incentive compatibility; a **static** model with no dynamic strategy space; Sybil
> resistance treated as a separate composition problem whose combined margin is *"an open task"*; and
> `p_au` trading off against chain cost and finality. The static-model limitation is the one that
> bears on Vyomanaut's re-derivation — it is why the single-period substitution below is the
> conservative one.

---

## Substitution at Vyomanaut's Parameters — the Q66-2 re-derivation

### The right question

Vyomanaut's deviation of interest is not "an auditor reports 1 without auditing." It is **"a storage
provider deletes a fraction `t` of its holding and hopes sampling misses it."** SHELBY does not state
a condition for that, so it is derived here in SHELBY's own shape, and the working is shown.

### Setup

Per audit period (one day), for one provider:

```
n  = 286,720          chunks held (70 GB)
c  = 2,867            chunks sampled                (ADR-060, rate 0.01)
C                     per-period cost of storing the whole holding
R                     per-period revenue for the whole holding
A  = R − C            honest per-period profit                    (Condition (iii): A ≥ 0)
S                     escrow seized on detection
t                     fraction of chunks deleted
P(t) = 1 − (1 − t)^c  probability the audit samples at least one deleted chunk   (ADR-060 / Ateniese)
```

`P(t)` is the right detection function because ADR-059's response is a **single aggregate over every
block of every sampled chunk** — one missing chunk fails the whole audit. Not a per-chunk race.

### Derivation

Deviating is unprofitable when the deviant's expected payoff does not exceed the honest one:

```
(1 − P)·(R − (1 − t)·C)  −  P·S    ≤    R − C

(1 − P)·(A + t·C)  −  P·S          ≤    A                       [substituting A = R − C]

A + t·C − P·A − P·t·C − P·S        ≤    A

t·C − P·(A + t·C + S)              ≤    0

                                  S    ≥    t·C·(1 − P(t))/P(t)  −  A
```

Maximising over the attacker's free choice of `t`:

```
S  ≥  max_t [ t·C·(1 − P(t))/P(t) ]  −  A
```

**This is the same *shape* ADR-060 wrote, with two corrections.** The gain term is `t·C` — the
storage cost the provider *avoids* — not `t·V` for an undefined `V`. And the honest profit `A` is
subtracted, because a caught provider forfeits it too.

### Evaluating the maximum

Let `h(t) = t·(1 − t)^c / (1 − (1 − t)^c)`. As `t → 0`, `(1 − t)^c → 1 − c·t`, so

```
h(t)  →  t · 1 / (c·t)  =  1/c
```

`h` is monotone decreasing on `(0, 1]` — checked numerically over a 3,000-point logarithmic grid
from `10⁻¹⁵` to `1`:

| `t` | `h(t)` | `1/h(t)` |
| --- | --- | --- |
| `10⁻¹²` | `3.48797 × 10⁻⁴` | 2,867.0 |
| `10⁻⁶` | `3.48297 × 10⁻⁴` | 2,871.1 |
| `10⁻⁵` | `3.43819 × 10⁻⁴` | **2,908.5** |
| `10⁻⁴` | `3.01165 × 10⁻⁴` | 3,320.4 |
| `10⁻³` | `6.02068 × 10⁻⁵` | 16,609 |
| `10⁻²` | `3.06 × 10⁻¹⁵` | `3.3 × 10¹⁴` |

```
sup_t h(t)  =  1/c  =  1/2,867  =  3.487967 × 10⁻⁴            attained as t → 0

⟹   S  ≥  C/2,867  −  A
```

**Hand check.** The supremum of a function attained as `t → 0` must equal its limit at `t → 0`,
which is `1/c` by the expansion above. ✓ ADR-060 reported the maximum as `3.438 × 10⁻⁴` = `1/2,909`
*"attained as `t → 0`"* — but `1/2,909 < 1/2,867`, so a maximum cannot both be that value and be
attained at `t → 0`. Reading across the table, `3.438 × 10⁻⁴` is `h(10⁻⁵)`: a grid artefact from a
scan that stopped four decades short of the limit. The error is small in magnitude (1.5%) and
structural in kind — it is the second time in this project's history that a substitution from this
paper was published without its working, which is why the LTS Literature Standard now requires the
working inline.

### The discrete adversary, and where `99×` came from

`t` is not continuous: the smallest real deviation is one chunk, `t = 1/n = 3.487723 × 10⁻⁶`.

```
P(1/n)  =  1 − (1 − 1/286,720)^2,867  =  0.00994949         ≈ the sample rate q = 0.01, as expected
(1 − P)/P  =  99.5076
h(1/n)  =  3.470551 × 10⁻⁴            ⟹  1/h  =  2,881.4
```

```
S  ≥  99.5 × (one chunk's per-period storage cost)  =  C/2,881  −  A
```

**`reading-list-v2`'s `99×` is arithmetically right and was attached to the wrong quantity.** It is
`(1 − q)/q` at `q = 0.01`, and it is the required ratio of escrow to the gain from deleting **one
chunk** — not to the value of the data, not to total revenue, and not an instance of SHELBY's
Condition (i), which is about auditors. ADR-060's `V/2,909` is the same statement normalised to the
whole holding, with a 1.5% arithmetic slip and an undefined `V`. **Both reconcile to
`S ≥ C/c − A`, and the three-orders-of-magnitude disagreement between them was an artefact of one
being per-chunk and the other per-holding.**

### Is `S ≥ C/c − A` binding?

`[DERIVED]` `C` is one day's marginal storage cost for 70 GB on a desktop the provider already owns
— electricity and amortised disk wear, of the order of single-digit rupees per day. `C/2,867` is
therefore of the order of **10⁻³ paise per day**, against ADR-024's 30-day rolling held-earnings
escrow, which is of the order of a month's earnings. And `A ≥ 0` is Condition (iii), which ADR-024's
storage rate is designed to satisfy.

**The escrow constraint is not binding by roughly six orders of magnitude, and the participation
constraint `r_st ≥ c_st` is the whole game.** That is the same qualitative conclusion ADR-060
reached — *"sampling does not tighten Vyomanaut's economic constraint; it loosens it, because a
provider that cheats more is caught more"* — and it survives the correction. The reasoning behind it
did not.

### The two assumptions Q66-2 flagged, discharged

**"Gain is linear in the deleted fraction."** It is, and more cleanly than assumed. The provider's
gain is avoided *storage cost*, which is linear in bytes not stored by definition. The non-linear
quantity would be the *owner's* loss, which does not enter the provider's payoff at all. The
assumption is not merely acceptable; framing the gain as `t·V` was the error.

**"Detection is evaluated over a single audit period."** It is, and this is **conservative in the
safe direction**. Over `d` periods an undetected cheater must survive `d` independent draws:
`(1 − P(t))^d`. At `t = 0.001`, `P = 0.9440` per day, so surviving a week has probability
`0.056^7 ≈ 1.9 × 10⁻⁹`. Multi-period exposure strictly increases detection and therefore strictly
*reduces* the required `S`. The single-period bound is an upper bound on what escrow must cover. ✓

**Q66-2 closes.** → ADR-060 Addendum A.

### What is *not* derived here

The above is a single-provider condition. It says nothing about a **coalition** deleting
correlated shards of the same stripe, which is the durability question, and it inherits SHELBY's own
static-model limitation. Theorems 2 and 3 bound coalition gain in SHELBY's setting, where honest
auditors are the majority. Vyomanaut's auditor is a single microservice, so the coalition analysis
does not transfer either — its resistance to a colluding provider set comes from ADR-060's sampling
being seed-derived and unpredictable, which is a different argument that Domain G's R-50 is about.
That remains open and is not affected by this re-derivation.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Occasional on-chain audit-of-auditor verification | Fully on-chain auditing of every storage event | Drastically lower verification cost; incentive compatibility at reasonable cost; requires calibrating `t_au`, `p_au` correctly |
| Peer-to-peer off-chain audits backstopped by on-chain spot checks | Trusted central auditor | Removes the need for a trusted auditing oracle; replaces it with a need for a trusted on-chain verification mechanism |
| Majority voting on peer audit reports | Single auditor authority | Sybil-resistant report aggregation; collusion bounded but not eliminated below the N/2 threshold |
| Nash equilibrium as solution concept | Dominant-strategy incentive compatibility | Achievable at reasonable cost; dominant strategy remains an open goal the authors name |
| Static model | Dynamic reputation model | Within-period guarantees without relying on reputation or discounting; dynamic strategy space not captured |
| Cost-based reconstruction deterrence (Observation 1) | Cryptographic binding (Filecoin Seal) | No new primitive; depends on a chunk-granularity read assumption the paper flags in footnote 6 |

---

## Breaks in Our Case

- **SHELBY uses a blockchain as the trusted on-chain verifier — the verification is fully public and trustless** ≠ **Vyomanaut's "on-chain" equivalent is the hardened microservice, which is trusted but not publicly verifiable in V2**
→ The microservice plays the role of the on-chain verifier in Theorem 1. For incentive compatibility to hold, it must be the incorruptible backstop — unable to be bribed or co-opted by a coalition of providers. The (3,2,2) quorum (ADR-025) and append-only audit log (ADR-015) are the structural guarantees that make it a credible verifier. The V3 Transparent Merkle Log upgrade (ADR-015) would bring Vyomanaut closer to SHELBY's public verifiability guarantee. **On revision:** Paper 70's Fortress liability model is the more direct answer to this gap, and Domain G's R-50 supplies the randomness dependency both need.

- **SHELBY's providers audit each other directly — every provider acts as both auditee and auditor** ≠ **Vyomanaut's audit challenges are issued exclusively by the microservice; providers never rate each other**
→ Proposition 1's collapse applies to systems where peer-issued reports are the only mechanism. Vyomanaut avoids this entirely: the microservice is the sole auditor. This is a structural escape from Proposition 1, not a parameter calibration. **On revision — the consequence that was missed:** the same structural escape makes **Theorem 1 Conditions (i) and (ii) inapplicable**, since both are conditions on auditor behaviour. Vyomanaut inherits Condition (iii) and Observation 1 and nothing else from Theorem 1. Every prior substitution into Condition (i) in this project was therefore substituting into a condition that does not bind here.

- **SHELBY's reconstruction cost argument (Observation 1) uses `k = 10` chunks from `k` different providers, at full-chunk read granularity** ≠ **Vyomanaut's `k = 16`, and the deterrent is enforced in the time domain by a response deadline rather than in the cost domain**
→ Both aim at the same target. SHELBY's is cost-based over a monthly epoch; Vyomanaut's is deadline-based within a single audit. **On revision:** these are *different failure horizons*, not alternatives — Paper 73 makes the point directly. SHELBY's condition rules out a *permanent* outsourcing strategy; it says nothing about an *opportunistic* one that succeeds inside a loose deadline. ADR-014 Defence 2 cites Observation 1 while implementing a timing mechanism, and never reconciles the two. ADR-014 Addendum A does.

- **SHELBY's slashing is enforced on-chain automatically and transparently** ≠ **Vyomanaut's escrow seizure is enforced by the payment microservice at the 72 h departure threshold**
→ **Revised.** The prior version of this break mapped `t_au` onto held-earnings seizure and asked whether threshold-based binary seizure satisfies Condition (i). It does not need to: `t_au` is the *audit-the-auditor* penalty, which has no Vyomanaut referent. The parameter Vyomanaut's escrow actually corresponds to is `t_st`, the storage-audit slashing penalty — which **Theorem 1 does not constrain at all**. The condition Vyomanaut's escrow must satisfy is `S ≥ C/c − A`, derived above, and it is slack by about six orders of magnitude.

- **SHELBY's model has N providers each independently auditing and storing, all in a single epoch** ≠ **Vyomanaut has a staged model: vetting (4–6 months), then full participation, with a graduated reliability score**
→ The paper's static model applies cleanly to Vyomanaut's post-vetting steady state. The vetting period is an entry cost not modelled in the paper, which further raises the cost of Sybil-style identity attacks — and §5 names Sybil composition as an explicit open task, so this is a place where Vyomanaut's design is ahead of the analysis rather than behind it.

---

## Decisions Influenced

- **ADR-002 [#2 Proof of Storage]** `CONFIRMED`
Proposition 1 provides the formal justification for why PoR challenges backed by a trusted verifier are necessary and not merely sufficient. Without the microservice as a trusted backstop verifying challenge responses, the unique equilibrium is every provider faking every audit — a provable collapse, not an edge case.
*Because:* Proposition 1's proof shows that ρ = 1 for all j is strictly dominant regardless of actual auditing.

- **ADR-014 [#19 Adversarial Defences]** `ADDENDUM A — jointly with Paper 73`
Observation 1 formally proves the outsourcing attack is deterred when `f_st·k·c_r > c_st`. **Revised:** Observation 1 is a *cost* argument over a monthly epoch and ADR-014 Defence 2 is a *timing* argument within one audit; citing the former as justification for the latter is a horizon error, and Paper 73 supplies both the empirical case against timing-only arguments and the adversary model (`α`) that Defence 2 lacks. Observation 1 remains valid and remains satisfied at Vyomanaut's parameters — by a wide margin, and by construction, because ADR-060 challenges whole chunks (see ADR-060 Addendum A §2).
*Because:* §2.3 footnote 6 makes Observation 1 conditional on chunk-granularity reads, which ADR-060 enforces for unrelated reasons.

- **ADR-024 [#18 Economic Mechanism]** ~~`PARAMETER CONDITIONS ADDED`~~ → **`WITHDRAWN AND REPLACED`**
> **Withdrawn, not deleted.** The prior version mapped `p_au ≈ 1` ("microservice challenges every chunk every 24 h"), concluded Condition (i) simplifies to `t_au ≥ 0`, and reported this as *"a significantly weaker requirement than SHELBY faces."* The mapping was wrong: `p_au` is the audit-the-auditor inspection rate, and Vyomanaut has no auditors to inspect. The conclusion happened to point in the right direction — the constraint is slack — for a reason that was not the stated one.

  **Replacement.** From Theorem 1, Vyomanaut inherits **Condition (iii) only**: `r_st ≥ c_st`, the participation constraint, which ADR-024's storage rate must satisfy and which is now the sole Theorem 1 obligation on the design. The escrow condition is not from SHELBY at all; it is derived in Substitution above as `S ≥ C/c − A` = `S ≥ C/2,867 − A`, slack by roughly six orders of magnitude against ADR-024's 30-day rolling escrow.
  *Because:* §2.4's parameter table defines `p_au` and `t_au` as audit-the-auditor quantities, and `t_st` — the storage-side penalty, the one Vyomanaut's escrow corresponds to — appears nowhere in Theorem 1.

- **ADR-060 [#2 Proof of Storage]** `ADDENDUM A — arithmetic and semantics corrected`
ADR-060's *"The economic constraint is looser under sampling, not tighter"* section is corrected: the maximum is `1/c = 1/2,867`, not `1/2,909`; the gain term is avoided storage cost `t·C`, not `t·V`; honest profit `A` is subtracted; and the whole substitution moves out of Condition (i), which does not apply. ADR-060's *conclusion* — sampling loosens rather than tightens the constraint — stands.
*Because:* derived above with working shown, per the LTS Literature Standard §3.3.

- **ADR-008 [#8 Reliability Scoring]** `CONFIRMED`
Proposition 1 confirms that the three-window rolling score in a single authoritative microservice is the only viable architecture: distributed peer scores collapse to the fully dishonest equilibrium. The centralisation of the scorer is the structural requirement for escaping Proposition 1, not a compromise.
*Because:* any mechanism where providers self-report on each other produces the collapse equilibrium.

---

## Falsifiers *(added on revision)*

1. **`S ≥ C/c − A` is void if a caught provider does not forfeit `A`.** The derivation assumes
   detection ends the relationship — escrow seized and provider ejected. If ADR-024's actual penalty
   is graduated (reduced release multiplier rather than seizure and ejection), the `−A` term is
   wrong and the bound must be re-derived against the real penalty schedule. Check ADR-024's
   `0.95 / 0.80 / 0.65` release multipliers against what a failed audit actually triggers.

2. **The single-period bound stops being conservative if the provider can re-enter cheaply.** It is
   conservative because a cheater faces repeated draws. A cheater that can be ejected and re-register
   as a fresh identity faces one draw per identity. The vetting period (ADR-053) is the mechanism
   that prices that, and SHELBY §5 names Sybil composition as an open task. The bound holds only
   while re-entry costs more than `C/c` — trivially true today, and worth stating because it is the
   assumption that would break first.

3. **Observation 1 is void for Vyomanaut if sub-chunk reads become possible on the retrieve path.**
   Footnote 6 is explicit. ADR-060's whole-chunk challenge secures it, but a future range-read
   optimisation on `internal/p2p` for partial retrieval would reintroduce the cheap path unless the
   audit path is explicitly excluded from it. → recorded in ADR-060 Addendum A's open constraints.

4. **Conditions (i) and (ii) become live if Vyomanaut ever adds peer auditing.** They are vacuous
   only because the microservice is the sole auditor. Any V3 design that distributes auditing —
   including a partial one — reinstates the whole of Theorem 1 and Proposition 1's collapse
   equilibrium as the failure mode. This is the standing reason Q07-4's deferral of EigenTrust-style
   distributed reputation should not be quietly reversed.

---

## Disagreements

- **Filecoin whitepaper (Paper 29) and most other DSN whitepapers:** essentially all existing DSN protocols assert incentive compatibility without formal proof. Filecoin states a desired property but does not prove it; Storj discusses "incentive alignment" informally.
*Implication for us:* Vyomanaut's economic design should be verifiable against Condition (iii) before launch. **On revision:** that is now a *smaller* checklist than previously believed — one inequality, not three.

- **EigenTrust (Paper 24):** assumes peer-to-peer interaction data exists and that the pre-trusted set anchors convergence. Proposition 1 shows this is insufficient — without a trusted verifier backstop, peer trust scores collapse to all-1 regardless of honesty.
*Implication for us:* the deferral of EigenTrust-based distributed reputation to V3 (Q07-4) is further validated.

- **This project's own prior reading of this paper, twice:** `reading-list-v2` and ADR-060 both substituted a storage-sampling rate into an auditing condition, arrived at answers differing by three orders of magnitude, and neither reconciled the gap.
*Implication for us:* the disagreement was the signal. Two derivations from one paper that differ by 1,000× should have triggered a re-read of the source before either was published — which is now `reading-list.md` §1's research-first trigger, and the concrete case behind the LTS Literature Standard's requirement that a `[DERIVED]` claim show its working.

---

## Corpus Delta *(added on revision)*

No new paper. The corpus gains a corrected reading of one it already held, a worked derivation of
the storage-deviation condition SHELBY does not state, and a reconciliation of two previously
contradictory numbers (`99×` and `V/2,909`) as the same result at two normalisations.

---

## Open Questions

See [open-questions.md](open-questions.md).
**Q66-2 is closed** by the re-derivation above and moves to [answered-questions.md](answered-questions.md).
Q37-1 and Q37-2 remain open; **Q37-2's framing is amended** — it asked whether threshold-based binary
seizure satisfies Condition (i), which does not apply. The live version of that question is whether
ADR-024's graduated release multipliers are consistent with the `−A` forfeiture term derived above
(Falsifier 1).
