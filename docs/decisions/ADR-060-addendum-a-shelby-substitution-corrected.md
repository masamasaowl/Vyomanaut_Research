# ADR-060 Addendum A — The SHELBY substitution, corrected

**Status:** Accepted — the correction is arithmetic and semantic; ADR-060's own decision is unchanged
**Track:** LTS
**Topic:** #2 Proof of Storage · #18 Economic Mechanism
**Amends:** ADR-060 § *"The economic constraint is looser under sampling, not tighter"*. The sampling rule, the 1% rate, the detection table, and the scale recomputation are **unchanged**.
**Research source:** Paper 37 (SHELBY, Crystal, Goren & Kominers, arXiv:2510.11866) re-read against the sampled-audit design — LTS reading list Band 0, item 4

---

## Context

ADR-060 contains a section deriving the escrow-to-gain ratio that Vyomanaut's economic mechanism must
satisfy under sampling. It reports:

> The reading list states that at 1% sampling SHELBY Condition (i) demands a slashing-to-gain ratio
> of **99×**, from `(1 − p_au)/p_au`. […] Evaluated at `c = 2,867`, the maximum is
> `3.438 × 10⁻⁴ · V`, attained as `t → 0`, against the small-`t` limit `1/c = 3.488 × 10⁻⁴`. So
> `S ≥ V / 2,909` — three orders of magnitude below the naive reading.

and flags it correctly as needing re-derivation against SHELBY's actual Theorem 1 statement (Q66-2).
That re-derivation is done. **Three errors, one of them structural. The conclusion survives all
three.**

### Error 1 — the wrong condition

SHELBY §2.4's parameter table defines `p_au` as *"probability that a **1 entry in an auditor
report** is selected for on-chain proof verification"* and `t_au` as *"slashing penalty for failing
**audit-the-auditor** verification."* Theorem 1 Conditions (i) and (ii) are both conditions on the
**auditing layer** — they discipline storage providers acting as auditors of one another.
Condition (iii), `r_st ≥ c_st`, is the only condition on storage.

SHELBY defines a separate parameter `t_st`, the storage-audit slashing penalty. **`t_st` does not
appear in Theorem 1 at all.** The paper derives no condition on the storage-side penalty; storage
honesty in its model comes from Condition (iii), Observation 1, and the majority-honest-auditor
indicator in the utility function.

**Vyomanaut has no auditor layer.** The microservice is the sole auditor; providers never rate each
other. Paper 37's own *Breaks* section records this as a structural escape from Proposition 1 and is
right to. The consequence it missed: **Conditions (i) and (ii) have no Vyomanaut referent.** They
are vacuous, not satisfied.

So the reading list's `99×`, ADR-060's `V/2,909`, and Paper 37's original `p_au ≈ 1` reading all
substitute a *storage sampling rate* into an *auditor inspection* condition. Three different answers
from one category error.

### Error 2 — the wrong gain quantity

ADR-060 writes gain as `G(t) = t·V` for an unnamed `V`. A storage provider's gain from deleting is
the **storage cost it avoids**, which is `t·C` where `C` is the per-period cost of the whole
holding. The owner's valuation of the data does not enter the provider's payoff function at all.
This also disposes of the first assumption Q66-2 flagged — gain is linear in the deleted fraction
by definition once it is the right quantity.

### Error 3 — a grid artefact

`h(t) = t·(1−t)^c / (1 − (1−t)^c)` is monotone decreasing on `(0, 1]`, so its supremum is its limit
at `t → 0`, which is `1/c`. A supremum attained as `t → 0` cannot be smaller than the `t → 0` limit,
yet ADR-060 reports `3.438 × 10⁻⁴` against a stated limit of `3.488 × 10⁻⁴`. Reading a log-scan of
`h`, `3.438 × 10⁻⁴` is `h(10⁻⁵)` — a scan that stopped four decades short of the limit.

## Decision

### 1. The corrected derivation

Deviating is unprofitable when

```
(1 − P)·(R − (1 − t)·C)  −  P·S    ≤    R − C
```

With `A = R − C` the honest per-period profit:

```
(1 − P)·(A + t·C) − P·S ≤ A
A + t·C − P·A − P·t·C − P·S ≤ A
t·C − P·(A + t·C + S) ≤ 0

        S  ≥  t·C·(1 − P(t))/P(t)  −  A
```

Maximising over the attacker's choice of `t`, with `P(t) = 1 − (1−t)^c` and `c = 2,867`:

```
sup_t [ t·(1 − P(t))/P(t) ]  =  1/c   (attained as t → 0)

        S  ≥  C/2,867  −  A
```

Verified numerically over a 3,000-point logarithmic grid from `10⁻¹⁵` to `1`:

| `t` | `h(t)` | `1/h(t)` |
| --- | --- | --- |
| `10⁻¹²` | `3.48797 × 10⁻⁴` | **2,867.0** |
| `10⁻⁶` | `3.48297 × 10⁻⁴` | 2,871.1 |
| `10⁻⁵` | `3.43819 × 10⁻⁴` | 2,908.5 ← the published figure |
| `10⁻⁴` | `3.01165 × 10⁻⁴` | 3,320.4 |
| `10⁻³` | `6.02068 × 10⁻⁵` | 16,609 |

**Hand check:** as `t → 0`, `(1−t)^c → 1 − ct`, so `h(t) → t/(ct) = 1/c`. The supremum equals the
sample count exactly. ✓

### 2. The discrete adversary, and the reconciliation of `99×` with `V/2,909`

`t` is not continuous — the smallest real deviation is one chunk, `t = 1/n = 3.487723 × 10⁻⁶`:

```
P(1/n) = 1 − (1 − 1/286,720)^2,867 = 0.00994949        ≈ the sample rate q = 0.01
(1 − P)/P = 99.5076
h(1/n) = 3.470551 × 10⁻⁴          ⟹  1/h = 2,881.4
```

```
S  ≥  99.5 × (one chunk's per-period storage cost)  =  C/2,881 − A
```

**The reading list's `99×` is arithmetically correct and was attached to the wrong quantity.** It is
`(1 − q)/q` at `q = 0.01` — the required ratio of escrow to the gain from deleting **one chunk**.
ADR-060's `V/2,909` is the same statement normalised to the whole holding, with a 1.5% slip. The
three-orders-of-magnitude disagreement was an artefact of one being per-chunk and the other
per-holding, and it should have triggered a re-read of the source rather than a publication.

### 3. Is it binding?

`C` is one day's marginal cost of storing 70 GB on a desktop the provider already owns —
electricity and amortised disk wear, order of single-digit rupees per day. `C/2,867` is order
**10⁻³ paise/day**, against ADR-024's 30-day rolling held-earnings escrow of order a month's
earnings. And `A ≥ 0` is Condition (iii).

**Slack by roughly six orders of magnitude. The participation constraint `r_st ≥ c_st` is the whole
game.** ADR-060's conclusion — *"sampling does not tighten Vyomanaut's economic constraint; it
loosens it, because a provider that cheats more is caught more"* — **stands.** Only the reasoning
behind it is replaced.

### 4. The multi-period assumption, discharged

Q66-2's second flagged assumption — detection evaluated over a single audit period — is
**conservative in the safe direction.** Over `d` periods an undetected cheater must survive `d`
independent draws: at `t = 0.001`, `P = 0.9440` per day, so surviving a week has probability
`0.056^7 ≈ 1.9 × 10⁻⁹`. Multi-period exposure strictly increases detection and strictly reduces the
required `S`. The single-period bound is an upper bound on what escrow must cover. ✓

**Q66-2 closes.**

---

## A second finding: ADR-060's chunk-granularity choice secures Observation 1

Not a correction. A property ADR-060 has and does not know it has.

SHELBY's Observation 1 — storing dominates on-the-fly reconstruction when `f_st·k·c_r > c_st` — rests
on a granularity assumption stated in §2.3 and conceded in footnote 6:

> *"Given that data in the system is read at the granularity of full chunks, this implies that
> on-the-fly reconstruction requires the auditee to read at least `k` chunks from `k` different SPs."*
>
> *"In specific cases the system supports data formats allowing non-complete chunk access. This is
> primarily used for crash recovery and involves high data volumes, making it impractical for
> on-the-fly audits of random chunks."*

**If sub-chunk reads were cheap, `k · c_r` would fall by the sampling ratio and Observation 1 would
weaken proportionally.** Reed-Solomon is symbol-wise linear: an attacker needing only symbol `α` of
a missing shard needs only symbol `α` from 16 peers, not 16 whole shards.

ADR-060's decision — *"Sample chunks. Prove chunks whole."*, with every block of every sampled chunk
challenged — makes partial reconstruction useless. To answer, the attacker needs **all** 256 blocks
of each sampled chunk, hence all 16 peer chunks in full. The rejected option in ADR-060's own table,
*"sample individual blocks across the whole holding"*, would have destroyed the property: the
attacker would need only the corresponding sectors from 16 peers, and the outsourcing cost would
fall by up to 256×.

**ADR-060 rejected block sampling on disk-seek grounds and got a security property for free.** That
is worth recording because the reasoning is not recoverable from the ADR as written, and a future
optimisation could undo it without knowing.

Paper 73 (Chen & Curtmola) supplies the same conclusion from the adversary's side. Its Theorem 3.2
proves the α-cheating adversary's optimal strategy is to spread its deletion uniformly across all
its data. Under whole-chunk challenges that strategy is catastrophic for the adversary — a uniform
deletion touching every chunk means **every** sampled chunk fails, and detection is 1. The
whole-chunk challenge forces the adversary out of its optimal strategy and into whole-chunk
deletion, which is precisely the regime Ateniese's bound governs. **ADR-060's detection model is
self-consistent, and it is self-consistent because of the granularity choice.**

## Consequences

**Positive:**

- Q66-2 closes with the working shown, per the LTS Literature Standard §3.3.
- Two contradictory published figures are reconciled as one result at two normalisations.
- ADR-024's Theorem 1 obligations shrink from three conditions to one — Condition (iii), the
  participation constraint. That is a smaller and clearer pre-launch checklist.
- The chunk-granularity choice acquires a stated security justification alongside its disk one, so a
  future block-sampling optimisation cannot be made without confronting it.

**Negative / trade-offs:**

- ADR-060's `V/2,909` figure is withdrawn and any downstream document citing it is wrong. Known
  consumers: `reading-list.md` §5 Domain A's `99×` remark, `answered-questions.md` Q41-2's closing
  sentence, Paper 37's original ADR-024 section. All three corrected this session.
- SHELBY's coalition results (Theorems 2, 3, Proposition 3) do **not** transfer either — they depend
  on a majority of honest peer auditors, which Vyomanaut does not have. Vyomanaut's resistance to a
  colluding provider set rests instead on ADR-060's seed-derived sampling being unpredictable, which
  is a different argument and is Domain G's R-50. That remains open and this addendum does not touch
  it.
- The derivation assumes a caught provider forfeits `A` — escrow seized and ejected. If ADR-024's
  graduated release multipliers (`0.95 / 0.80 / 0.65`) are what a failed audit actually triggers, the
  `−A` term is wrong and the bound needs re-deriving against the real penalty schedule. → Q37-2,
  reframed.

**Open constraints:**

- **Observation 1's granularity condition must be protected.** Any future range-read or partial-shard
  optimisation on the retrieve path must exclude the audit path explicitly, or the primary
  outsourcing defence weakens by up to the sampling ratio. This belongs in `internal/p2p`'s package
  doc as an invariant, not only here.
- **The detection table is conditional on the response deadline binding.** See ADR-014 Addendum A —
  Ateniese's bound is about possession, not about ability to answer, and in an erasure-coded system
  the prover can obtain what it deleted.
- ADR-060's own open constraints — F-03's payment-unit ruling, `audit_sample_rate` as a profiled
  field, Q67-1's cross-stripe union bound, and the ADR-030 vetting re-derivation — are **unchanged**
  and none is closed here.
- **Demo unaffected.** `ADR-062` freezes it; `Track: LTS`; not backported.

## References

- [Paper 37 — SHELBY](../research/paper-37-shelby-incentive-compatibility.md): §2.3 and footnote 6 (Observation 1's granularity condition), §2.4 (the parameter table that defines `p_au` and `t_au`), §3.2 Theorem 1, §5 (sufficient conditions, not tight bounds). Revised in place this session
- [Paper 73 — Chen & Curtmola](../research/paper-73-chen-curtmola-rdc-sr.md): Theorem 3.2, the α-cheating adversary's optimal strategy
- [Paper 66 — Ateniese, PDP](../research/paper-66-ateniese-pdp.md): the `P_X` detection bound, unchanged
- [ADR-060 — Sampled chunk audit schedule](ADR-060-sampled-chunk-audit-schedule.md): the section amended; sampling rule and rate unchanged
- [ADR-014 Addendum A](ADR-014-addendum-a-outsourcing-defence-restated.md): the companion finding on which the detection table depends
- [ADR-024 — Economic mechanism](ADR-024-economic-mechanism.md): Theorem 1 obligations reduced to Condition (iii)
- [ADR-012 — Payment basis](ADR-012-payment-basis.md): F-03 unchanged and still upstream
- [ADR-077 — Research-first triage](ADR-077-research-first-triage.md): this is the canonical Class-B case — a number published without its substitution
