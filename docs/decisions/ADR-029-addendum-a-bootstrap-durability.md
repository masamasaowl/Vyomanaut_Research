# ADR-029 — Addendum A: Bootstrap Durability, Quantified (Paper 63)

**Applies to:** [ADR-029 — Bootstrap Minimum Viable Network](./ADR-029-bootstrap-minimum-viable-network.md)
**Status of parent ADR:** Accepted — **unchanged.** No threshold in Part A is altered by this addendum.
**Added research source:** [Paper 63 — Friedman, Kapelko & Marchwicki](../research/paper-63-friedman-kapelko-data-persistency.md)
**Findings addressed:** F-28 (partially — see *What this does not close*)

> **Insert location:** after Part A's readiness-conditions table, before *"Why shard count is not the
> bootstrap metric."* Recorded as an addendum rather than a rewrite because it adds evidence for
> thresholds that were chosen correctly, and changes none of them.

---

## Why this addendum exists

ADR-029 fixes the upload gate at **≥ 56 vetted providers across ≥ 5 distinct ASNs**. Both numbers were derived structurally — 56 because RS(16,56) needs 56 distinct shard holders, 5 because fewer makes the 20% ASN cap vacuous. Neither was ever checked against a durability model.

F-28 objected precisely here: *"Nobody has run the durability numbers for the regime N ∈ [56, ~500]. That regime is where the product spends its first year."* Giroire (Paper 10) models node **failure with repair running** and produces a steady-state LossRate. The bootstrap concern is different — a small network where providers **leave and erase**, and where failure independence is weakest exactly when data is first accepted.

Paper 63 models that process directly.

## The mapping

`REC(p, p+q, r)` with `p = 16`, `q = 40`, `r = 1`. Everything scales in `r(q+1) = 41` — the number of distinct chunks that must vanish entirely for a document to become unrecoverable.

Vyomanaut places one shard per distinct provider (`architecture.md`: *exactly 56 distinct providers per file segment*), which is Paper 63's **sequential** placement strategy. For that strategy, expected persistency **does not depend on the number of documents stored** — a result the authors single out as the surprising one, and the one that most directly answers F-28's worry about the first year.

## Result at the gate

Theorem 9 evaluated at `N = 56`:

```
E[X] = (N+1) ∫₀¹ (1 − I_{x}(q+1, p))^{N/((p+q)r)} dx
     = 57 × ∫₀¹ (1 − I_x(41,16)) dx
     = 57 × E[Beta(41,16)]  =  57 × 41/57
     = 41.00
```

Hand check: at exactly 56 providers each holding one of 56 shards, removing 41 is necessary and sufficient, deterministically. The integral returns exactly that. ✓

| Providers `N` | Departures tolerated before the **first** file is unrecoverable | As a share of the network |
| --- | --- | --- |
| **56 — the gate** | **41** | **73.2%** |
| 112 | 77.5 | 69.2% |
| 560 | 350.3 | 62.6% |
| 1,120 | 675.8 | 60.3% |
| 5,600 | 3,137.0 | 56.0% |

**At the moment the network first accepts data, 73.2% of it must walk away, erasing as it goes and with repair never running, before one file is lost.** Independent of how many files are stored. That is the number the readiness gate has been missing.

## ASN failure tolerance at the gate

Combining `q + 1 = 41` with ADR-014's cap (`⌊56 × 0.20⌋ = 11` shards per ASN) at ADR-029's 5-ASN floor:

| ASNs failing simultaneously | Shards removed (worst case) | Outcome |
| --- | --- | --- |
| 1 | 11 | survives (30 shards of margin) |
| 2 | 22 | survives (19 shards of margin) |
| 3 | 33 | survives (8 shards of margin) |
| **4** | **44** | **file lost** |
| 5 | 55 | file lost |

**The bootstrap network survives any three of five simultaneously failing ASNs and loses data at four.** This is the concrete statement F-28 asked for. The 5-ASN floor is confirmed as *sufficient*, not merely non-vacuous — but the margin at three ASNs is 8 shards, which is thin enough that the floor's value depends entirely on those five ASNs being genuinely independent.

That is R-17's job (Indian residential AS topology and CGNAT concentration). If the real answer is that a consumer provider pool in India spans only 6–8 meaningfully independent ASNs, then satisfying the ≥ 5 condition nominally and satisfying it substantively are different things, and the gate needs a stronger independence test than counting ASN numbers.

## Two cautions for anyone reusing this

1. **Do not use the asymptotic form near the gate.** Paper 63's Theorem 11 evaluated at `N = 56` gives **26.31** against the exact **41.00** — a 36% underestimate. The error term is `O(N^{1−2/r(q+1)})`, which is not small when `N` is small and `r(q+1) = 41` is large. Any readiness dashboard or capacity note must use the Theorem 9 integral. This is the same failure mode as F-43 and NFR-044: a formula lifted from a paper without checking it at its own anchor point.

2. **Theorem 9 requires `(p+q)r | N`.** Real provider counts are not multiples of 56. Interpolate, or use the random-placement exact form (Corollary 5) as a conservative lower bound — at `N = 560, D = 10⁶` that gives 226 against sequential's 350, so it is genuinely conservative and never optimistic.

## What this does not close

**F-28 remains open.** Paper 63 assumes departures are **independent and uniformly random**. That is exactly the assumption F-28 disputes for a 56-provider, 5-ASN network where "every file lands on essentially every provider." The ASN table above patches around it by treating ASN loss as an adversarial worst case rather than a random one — but that substitution is ours, not the paper's result.

What has changed is that F-28 now has an exact independent-case baseline, so the size of the correlation penalty is measurable rather than merely feared. **R-16** (durability under correlated failure at small scale) and **R-18** (AS-level outage duration and blast radius) remain the open items, and R-17 now has a specific number to test against.

**The model also assumes repair never runs.** `E[X] = 41` is a floor describing a prolonged microservice outage, not a steady state. It is not a replacement for Giroire's LossRate; both belong in ADR-003, labelled with their failure processes.

## Consequential note for ADR-005

Paper 63's random-versus-sequential comparison prices an invariant Vyomanaut already enforces but has never justified numerically. At `D = 10⁶` documents, permitting one provider to hold two shards of the same file costs **55%** of tolerated departures at `N = 560` (226 vs 350) and **39%** at `N = 5,600` (2,254 vs 3,137).

`architecture.md` states the 56-distinct-providers requirement as a mechanical consequence of RS(16,56) needing 56 holders. It is not only that: *"56 distinct providers"* and *"no provider holds two shards of one file"* are separable in implementation, and only the second buys this. ADR-005 should record the one-shard-per-provider-per-file rule as an independently load-bearing durability constraint, so that no future assignment optimisation trades it away for placement flexibility.

## References added

- [Paper 63 — Friedman, Kapelko & Marchwicki](../research/paper-63-friedman-kapelko-data-persistency.md): closed-form expected data persistency for `REC(p, p+q, r)` under random and sequential placement; the sequential result is independent of document count; `E[X] = 41` at the ADR-029 gate; the asymptotic form must not be used at small `N`
