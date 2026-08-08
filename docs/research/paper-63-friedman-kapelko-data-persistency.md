# Paper 63 — On the Data Persistency of Replicated Erasure Codes in Distributed Storage Systems

**Authors:** Roy Friedman (Technion), Rafał Kapelko (Wrocław University of Science and Technology), Karol Marchwicki
**Venue / Year:** Information and Computation 304 (2025) 105297, Elsevier
**Citations:** not verified in this pass
**Topics:** #3 Erasure Coding, #4 Repair Protocol, #5 Peer Selection (placement)
**ADRs produced:** ADR-003 (revised), ADR-029 (revised), ADR-005 (placement invariant — evidence added)
**Findings addressed:** F-28 (bootstrap durability regime), F-07 (storage-overhead misstatement — independent source)
**Reading list:** Band 0 #63; Domain E / R-16

---

## Problem Solved

Every durability number in the Vyomanaut corpus comes from Giroire (Paper 10), which models **disk failure with repair** and produces a LossRate per year. That is the wrong failure process. Vyomanaut's providers are people with desktops who stop participating: they uninstall, they sell the machine, they lose interest. The node does not fail — it *leaves and takes its data with it*. F-28 objected that the bootstrap regime was never modelled; the deeper problem is that the model in use was never the right shape.

This paper analyses exactly that process. It defines **data persistency** as the minimum number of nodes that must be removed — successively, independently, uniformly at random, each erasing what it stored — before at least one document becomes unrecoverable. It gives closed-form expressions in terms of the incomplete beta function, plus asymptotic bounds, for a general redundancy family `REC(p, p+q, r)`: each document split into `p` chunks, encoded into `p+q`, each chunk replicated `r` times, and all `r(p+q)` pieces distributed by one of two placement strategies.

This is the first analytic tool in the corpus that takes departure rather than failure as the primitive, and the first that gives an exact answer at small `N` rather than an asymptotic one.

---

## Key Findings

### The two placement strategies

- **Random** (Algorithm 1): every piece is placed on an independently and uniformly chosen node. Pieces of the same document may collide on one node.
- **Sequential** (Algorithm 2): pieces are laid down in order across a fixed partition of the node set, so each document's `r(p+q)` pieces land on `r(p+q)` distinct nodes.

The distinction is precisely the question of **whether one provider may hold more than one shard of the same file**. `architecture.md` already requires 56 distinct providers per file segment, which makes Vyomanaut the sequential case. Until now that invariant carried no numeric justification.

### The governing quantity

To lose a document, all `r` replicas of `q+1` distinct chunks must vanish. Everything scales in the single quantity `r(q+1)`.

| Placement | Expected persistency | Depends on `D`? |
| --- | --- | --- |
| Random | `N · Θ(D^(−1/(r(q+1))))` | **Yes** |
| Sequential | `Θ(N^(1 − 1/(r(q+1))))` | **No** |

The `D`-independence of the sequential strategy is called out by the authors as the surprising result, and it is the one that matters most to us.

### Exact forms

- **Random, exact** (Corollary 5): `E[X] = Σ_{ℓ=0..N} (1 − I_{(ℓ/N)^r}(q+1, p))^D`
- **Sequential, exact** (Theorem 9): `E[X] = (N+1) ∫₀¹ (1 − I_{x^r}(q+1, p))^{N/((p+q)r)} dx`, valid when `(p+q)r | N` and `D ≥ N/((p+q)r)`
- **Asymptotics** (Theorems 8 and 11) via Laplace's method, with gamma- and beta-function coefficients.

### Maximal persistency is at `p = 1`

For both strategies, persistency is maximised at `p = 1` — which is plain replication with `(1+q)r` copies. Read carelessly this says "replication beats erasure coding." It does not: the result holds at **fixed `q` and `r`**, i.e. at fixed piece count, not at fixed storage cost. See the comparison table below, where the correction runs strongly the other way.

### Non-uniform redundancy

§5.1 sketches the mixed case — `D_i` documents under `REC(p_i, p_i+q_i, r_i)` — and states that the closed-form and asymptotic results for a system holding a *mixture* need further theoretical work. This is exactly ADR-018's Hot/Cold band model, and the paper's own position is that it is unsolved.

---

## Substitution at Vyomanaut's parameters

Vyomanaut maps to `p = 16`, `q = 40`, `r = 1` — RS(16,56) with no shard-level replication. Governing quantity `r(q+1) = 41`.

Sanity check first: at `N = 56` with sequential placement, every document occupies all 56 nodes with one shard each, so exactly `q+1 = 41` removals are always required and `E[X]` must be exactly 41. Theorem 9 returns **41.00**. ✓

| `N` | Sequential `E[X]` (any `D`) | Random, `D=1` | Random, `D=10³` | Random, `D=10⁶` |
| --- | --- | --- | --- | --- |
| 56 | **41.00** (73.2% of the network) | 40.78 | 29.09 | 23.04 |
| 112 | 77.52 (69.2%) | 81.06 | 57.67 | 45.57 |
| 560 | 350.33 (62.6%) | 403.31 | 286.36 | 225.87 |
| 1,120 | 675.82 (60.3%) | — | — | — |
| 5,600 | 3,137.04 (56.0%) | 4,028.57 | 2,859.07 | 2,254.22 |

**Three things fall out.**

1. **The bootstrap network is far more persistent than F-28 feared, under independent departure.** At the ADR-029 gate exactly, 41 of 56 providers — 73.2% of the entire network — must depart before the first file becomes unrecoverable. That is a large margin, and it is the first number the bootstrap regime has ever had.

2. **The distinct-provider invariant is worth 35–55%, and its value grows with `D`.** At `N = 560, D = 10⁶`, sequential placement tolerates 350 departures against random placement's 226 — the invariant buys **55% more** tolerated departures. At `N = 5,600, D = 10⁶` it is 3,137 versus 2,254, **39% more**. Vyomanaut already enforces this, so the finding is confirmatory rather than corrective — but `architecture.md` states it as a mechanical consequence of RS(16,56) needing 56 holders, and it is in fact an independently load-bearing durability decision. It should be recorded as such, because "56 distinct providers" and "no provider holds two shards of one file" are separable in implementation and only the second is what buys this.

3. **The asymptotics fail in exactly the regime F-28 cares about.** Theorem 11 evaluated at `N = 56` gives 26.31 against the exact 41.00 — a 36% underestimate. The `O(N^{1−2/(r(q+1))})` error term is not small when `N` is small and `r(q+1)` is large. **Anyone applying this paper to the bootstrap regime must use the Theorem 9 integral, not the asymptotic.** Same failure mode as F-43 and NFR-044: a formula lifted without checking it at the anchor point.

### Cross-scheme comparison at equal governing quantity — corrects F-07's implication

F-07 correctly established that RS(16,56) costs 3.5× storage against 3× replication's 3.0×, and that ADR-003's claimed storage win does not exist. This paper supplies the missing other half of that argument, in a metric where the comparison is decisive:

| Scheme | Storage | `r(q+1)` | Sequential `E[X]` at `N = 10·(p+q)r` |
| --- | --- | --- | --- |
| 3× replication | 3.0× | 3 | 12.57 (of 30 nodes) |
| 4× replication | 4.0× | 4 | 20.58 (of 40 nodes) |
| **RS(16,56)** | **3.5×** | **41** | **350.33 (of 560 nodes)** |
| RS(16,32) | 2.0× | 17 | 123.02 (of 320 nodes) |
| RS(16,56), `r=2` | 7.0× | 82 | 885.42 (of 1,120 nodes) |

At roughly equal storage cost, RS(16,56) has a governing quantity **13.7×** that of 3× replication. To match 41 by replication alone would take 41 copies — 41× storage. ADR-003's conclusion was right; only its stated reason was wrong. The honest sentence is: *RS(16,56) costs marginally more storage than 3× replication and buys an order of magnitude more departure tolerance for it.*

### Bootstrap ASN arithmetic (feeds F-28 and ADR-029)

Combining `q+1 = 41` with ADR-014's 20% cap (`⌊56 × 0.20⌋ = 11` shards per ASN) at the ADR-029 gate of 5 ASNs:

| ASNs lost simultaneously | Shards removed (max) | Outcome |
| --- | --- | --- |
| 1 | 11 | survives |
| 2 | 22 | survives |
| 3 | 33 | survives |
| **4** | **44** | **file lost** |
| 5 | 55 | file lost |

**The bootstrap network survives any three of five ASNs failing together and loses data at four.** That is the concrete statement F-28 asked for and no document currently contains. It is a comfortable margin against realistic correlated-outage sizes, and a thin one against a scenario where the 5-ASN floor is only nominally satisfied — which is exactly what R-17 (Indian residential AS topology) exists to test.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Departure-with-erasure as the failure primitive | Disk failure with repair (Giroire) | Matches the P2P consumer failure mode exactly; gives up any notion of repair, so the result is a floor rather than a steady-state |
| Exact closed forms via incomplete beta, plus asymptotics | Simulation | Usable at any `N` including the bootstrap regime; the asymptotics are only trustworthy at large `N`, which the paper states and this reader had to verify |
| Two clean placement strategies | A realistic placement policy | Random and sequential bracket the real behaviour and make the distinct-node invariant measurable; neither models reliability-weighted assignment (ADR-005's Power of Two Choices) |
| Uniform `(p, q, r)` across all documents | Mixed redundancy | Tractable closed form; the mixed case is left explicitly open, and that is the ADR-018 case |

---

## Breaks in Our Case

- **No repair.** The model has nodes leave and never return, and nothing reconstructs. Vyomanaut has lazy repair (ADR-004) restoring redundancy from `s+r0 = 24` back to 56. `E[X]` is therefore a **floor**, not a prediction — it answers "how many providers can leave before the first file dies *if repair never runs*." That is not a hypothetical: it is precisely the microservice-extended-outage scenario, and it is a number worth having for exactly that. It is not a substitute for Giroire's steady-state LossRate.

- **Departures are independent and uniform.** This is the assumption F-28 objects to, and the paper does not relax it. The bootstrap arithmetic above patches over it by treating ASN loss as a worst case rather than a random one, but that is our substitution, not the paper's result. **Paper 63 does not close F-28.** It gives F-28 an exact independent-case baseline against which correlated departure can be measured, and it makes the size of the gap visible — which is more than the corpus had. R-16 and R-18 remain open.

- **The `p = 1` optimality result is easy to misread and will be misread.** Taken out of context it says replication maximises persistency. Anyone quoting it must carry the "at fixed `q` and `r`" qualifier, or ADR-003's central choice appears to be contradicted by a paper that in fact supports it strongly.

- **`E[X]` is about the *first* document to become unrecoverable.** It is a min over `D` documents — a worst-case, not an average. That is the right metric for a durability SLA and the wrong one for expected data loss volume. The paper offers nothing on how much is lost after the first loss.

- **The mixed-redundancy case is open, and it is our case from ADR-018 onward.** Hot and Cold bands with different `(k, r)` mean a system holding `D_hot` documents under one scheme and `D_cold` under another. §5.1 states the closed form for the mixture needs further study. Whatever ADR-018 settles on, this analysis will not extend to it without new work.

- **Theorem 9 requires `(p+q)r | N`.** Real provider counts are not multiples of 56. The result interpolates sensibly, but any operational dashboard computing "departures remaining before first loss" must either round or use the random-placement exact form as a conservative bound.

---

## Decisions Influenced

- **[ADR-029](../decisions/ADR-029-bootstrap-minimum-viable-network.md) [#1, #5]** `REVISED`
  Supplies the bootstrap durability number the readiness gate never had: at exactly 56 providers, 41 departures (73.2% of the network) under independent departure with no repair, and survival of any 3 of 5 simultaneous ASN losses. The 5-ASN floor is confirmed as sufficient rather than merely non-trivial, subject to those ASNs being genuinely independent — which is R-17's job.
  *Because:* the exact Theorem 9 integral is evaluable at `N = 56` and returns a checkable value (41.00, matching the trivial hand argument), so this is a computed result rather than an extrapolated one.

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `REVISED`
  Independently confirms the 3.5× storage ratio for F-07 (the paper states erasure-code storage cost as `(p+q)/p`), and replaces the deleted storage-efficiency claim with a defensible one: RS(16,56) buys a governing persistency quantity of 41 against 3× replication's 3, for 0.5× more storage.

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection]** `EVIDENCE ADDED`
  The one-shard-per-provider-per-file invariant is worth 39–55% more tolerated departures than collision-permitting placement at `D = 10⁶`. It should be stated as a durability requirement in its own right, not left as a side effect of needing 56 holders.

---

## Disagreements

- **Against Giroire (Paper 10) on what is being modelled.** Not a contradiction — a scope statement. Giroire models failure-with-repair and yields LossRate/year; this models departure-without-repair and yields tolerated-departure count. The corpus currently has only the first, and quotes it as though it covered provider churn. It does not. Both numbers should appear side by side in ADR-003, labelled with their failure processes.

- **Against ADR-003's original storage argument.** Already flagged by F-07; this paper is the primary source that settles it and supplies the replacement argument.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q63-1 and Q63-2.
