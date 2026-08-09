# ADR-060 — Sample chunks; prove whole chunks

**Status:** Proposed — blocked on the F-03 payment-unit ruling and on ADR-059
**Topic:** #2 Proof of Storage
**Supersedes:** ADR-002 §Open constraints (audit scalability ceiling), on approval
**Superseded by:** —
**Research source:** Paper 66 (Ateniese, CCS 2007), Paper 67 (Juels & Kaliski, CCS 2007), Paper 68 (Shacham & Waters, Asiacrypt 2008)

---

## Context

Full daily audit of every chunk on every provider fails in three places at once, at roughly the same scale (F-02, F-40, F-58, F-59):

| Constraint | Bites at | Finding |
| --- | --- | --- |
| Postgres INSERT ceiling | 1,000 providers × 200 GB → 9,481/s, ×2 for two-phase = 18,963/s against a stated 5–10 k | F-02, F-58 |
| Provider disk | 70 GB provider = 286,720 chunks/day = **~1 h/day** of random HDD seeking | F-40 |
| Nonce guard index | 48 h retention at 1,000 × 70 GB = 573 M rows ≈ 30 GB hot index | F-59 |

`capacity.md` makes the first worse: the 5–10 k figure is *"derived from generic benchmark literature and not from the actual Vyomanaut schema."*

ADR-002 asserts full daily audit remains feasible to *"approximately 100,000 providers × 10,000 chunks."* That figure is wrong twice — 10⁹ rows/day is 11,574/s, already above its own stated ceiling; and 10,000 chunks is 2.5 GB, not a desktop provider sharing idle disk.

The failure mode this decision prevents is a design that stops working inside the year-one target and whose only escape hatch — sampling — the system currently has no formal basis for, because the detection-bound literature was absent from the corpus.

Papers 66–68 supply that basis. Ateniese gives detection probability as an explicit function of sample size, independent of total size. Juels & Kaliski give the `ε`-adversary framing that turns audit scheduling into a stated security objective. Shacham & Waters give a constant-size aggregate response, which is what allows many sampled chunks to collapse into one receipt row.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Status quo — every chunk, every day** | Deterministic coverage; simplest to explain; per-audit payment unit works unchanged (ADR-012) | ~1 h/day of provider disk; 3,319–16,593 INSERT/s at plausible year-one scale; 30 GB hot nonce index. Fails on all three axes |
| **Sample individual blocks across the whole holding** | Smallest possible read volume; closest to the papers' own parameterisation | 512 random 1 KB reads per file-audit is *worse* than the status quo for seeks — random small reads, not fewer reads. Defeats R-08 |
| **Sequential sweep of the whole holding on a multi-day cycle** | Best possible disk profile — 70 GB sequential in ~12 min | Detection latency equals the sweep period; a provider that deletes just after being swept is undetected for a full cycle. Also gives an adversary a predictable window, which is what ADR-002 rejected Storj v2's scheme for |
| **Sample whole chunks, challenge every block within each sampled chunk — chosen** | Sampled unit matches both the storage unit and the realistic deletion unit; each sampled chunk is one contiguous 256 KB read; one constant-size response covers the whole sample; detection derived from a published bound | Non-zero, quantified miss probability that must appear in provider-facing copy; makes provider income a function of sampling unless F-03 is settled first |

## Decision

**Sample chunks. Prove chunks whole.**

### Sampling rule

Each provider is audited once per day. For each file the provider holds, the microservice draws a fresh 32-byte challenge seed and derives from it a sample of that provider's chunks of that file at rate

```
audit_sample_rate = 0.01        (1% of the provider's chunks per file, per day)
```

Every block of every sampled chunk is challenged (ADR-059: 256 blocks per chunk). One challenge, one 1,040-byte response, one receipt row per `(provider, file, day)`.

The sample is derived from the seed, not stored, so the microservice retains no per-chunk challenge state and the nonce guard index (F-59) disappears entirely — there is one seed per `(provider, file, day)`, not one nonce per chunk.

### Detection, derived

Ateniese's bound, `P_X ≥ 1 − ((n−t)/n)^c`, at `n = 286,720` chunks (a 70 GB provider) with `c` chunks sampled:

| Sample rate | `c` chunks/day | Bytes read | Disk time* | `t = 0.1%` | `t = 0.5%` | `t = 1%` | `t = 5%` |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0.1% | 286 | 0.075 GB | 0.1 min | 24.88% | 76.15% | 94.36% | ~100% |
| 0.5% | 1,433 | 0.376 GB | 0.3 min | 76.16% | 99.92% | 99.9999% | ~100% |
| **1%** | **2,867** | **0.752 GB** | **0.6 min** | **94.32%** | **99.9999%** | **~100%** | **~100%** |
| 2% | 5,734 | 1.503 GB | 1.2 min | 99.68% | ~100% | ~100% | ~100% |
| 100% | 286,720 | 75.2 GB | **60.3 min** | 100% | 100% | 100% | 100% |

\* 10 ms seek + 256 KB at 100 MB/s per chunk. The bottom row reproduces F-40's ~1 h/day independently.

Substitution for the chosen row: `c = 0.01 × 286,720 = 2,867`; at `t = 0.01`, `1 − 0.99^2867 = 1 − 4.0 × 10^-13`. Ateniese's own worked example is the same formula at block granularity: `ln(0.01)/ln(0.99) = 458.2`, rounded to the published `c = 460` for 99% at `t = 1%`.

**1% daily sampling is chosen.** It detects a 1% deletion with certainty to twelve decimal places, a 0.5% deletion at 99.9999%, and a 0.1% deletion at 94.3% — per day, so within a week a 0.1% deletion is detected with probability `1 − 0.0568^7 ≈ 1 − 1.9 × 10^-9`. It costs 0.6 minutes of provider disk against 60.3.

### Scale, recomputed

| | Rows/day | Sustained INSERT/s |
| --- | --- | --- |
| Status quo, 1,000 providers × 70 GB | 286,720,000 | 3,319 |
| Status quo, 5,000 providers × 70 GB | 1,433,600,000 | 16,593 |
| **This ADR, 1,000 providers × 1,000 files** | **1,000,000** | **11.57** |
| This ADR, 5,000 providers × 5,000 files | 25,000,000 | 289 |

Row count is now driven by files-per-provider, not chunks-per-provider — a factor of ~287 at 1,000 files. F-02's ceiling is no longer reachable at any plausible V2 or V3 scale. F-58 and F-59 close with it.

### The economic constraint is looser under sampling, not tighter

The reading list states that at 1% sampling SHELBY Condition (i) demands a slashing-to-gain ratio of **99×**, from `(1 − p_au)/p_au`. That substitution treats `p_au` as constant. It is not: detection grows with the amount cheated. With gain `G(t) = t·V` and detection `P(t) = 1 − (1−t)^c`,

```
S  ≥  max_t  G(t)·(1 − P(t))/P(t)
```

Evaluated at `c = 2,867`, the maximum is `3.438 × 10⁻⁴ · V`, attained as `t → 0`, against the small-`t` limit `1/c = 3.488 × 10⁻⁴`. So `S ≥ V / 2,909` — three orders of magnitude below the naive reading. **Sampling does not tighten Vyomanaut's economic constraint; it loosens it,** because a provider that cheats more is caught more.

This must be re-derived against SHELBY's actual Theorem 1 statement before it is relied on. The substitution assumes gain is linear in the deleted fraction and that detection is evaluated over a single audit period. Q66-2.

### What is not decided here

**The payment unit.** ADR-012 pays per audit passed. Under sampling, income becomes a function of how often you were sampled — a microservice-controlled random variable, not a measure of service delivered (F-03). The clean answers are to pay per `chunk-GB-period` and use the audit as a *gate* rather than a *meter*, or to normalise per-audit payment by sampling rate. This ADR takes no position; it only notes that the receipt now carries the full derivable chunk set, so a gate-based unit is implementable without further schema change. **The schema follows the payment unit, so F-03 must be ruled on before the audit scheduler is finalised.**

## Consequences

**Positive:**

- Provider disk cost falls from ~60.3 min/day to ~0.6 min/day at the same durability posture.
- Sustained INSERT rate falls from 3,319/s to 11.6/s at 1,000 providers. F-02, F-58 close.
- The nonce guard index disappears — one seed per `(provider, file, day)` replaces one nonce per chunk. F-59 closes.
- Each sampled chunk is a single contiguous 256 KB vLog read, satisfying R-08 by construction rather than by a separate mechanism.
- The sampled unit is a whole chunk, which matches the realistic deletion unit under a log-structured vLog with segment-granularity GC (ADR-023, ADR-046). Detection bounds derived under a block-isolation assumption would have overstated detection against segment loss; sampling at chunk granularity avoids relying on that assumption.
- The audit now has a stated security objective — bound the corruption rate `ε` below what RS(16,56) tolerates — rather than a coverage target.

**Negative / trade-offs:**

- Coverage is probabilistic. A 0.1% deletion has a 5.7% chance of surviving any given day. This is a real, quantified weakening and it must appear in ADR-045's provider-facing and owner-facing copy, not be rounded to "we check your data every day."
- Provider income becomes sampling-dependent unless F-03 rules otherwise. Two providers storing identical data can earn different amounts in a month. That is a direct fairness problem for a product whose provider pitch is predictable income.
- Detection latency for very small deletions is days rather than hours.
- Row count now scales with files-per-provider, a quantity nobody has measured and which no ADR bounds. At 5,000 files/provider and 5,000 providers it is 289 INSERT/s — comfortable, but the growth axis has moved rather than disappeared.
- The receipt covers a set of chunks rather than one chunk, so per-chunk dispute resolution needs the seed and the derivation to be reproducible years later. The derivation function becomes a frozen contract.

**Open constraints:**

- **F-03 must be ruled on first.** The receipt schema follows the payment unit.
- `audit_sample_rate = 0.01` must be a `NetworkProfile` field with a compiler-enforced relationship to the durability target, alongside `TestProfileShardSizeIsConstant`, or the next profile silently re-derives its own detection probability — the same structural defect F-67 names for the confidentiality threshold.
- The per-provider bound above is per provider per file. The probability that *some* provider in a 56-shard stripe is an undetected `ε`-adversary is what governs stripe durability, and it is not derived in any of Papers 66–68. Q67-1.
- Depends on ADR-059. Constant-size aggregate responses are what make one receipt cover 2,867 chunks; under the current hash primitive this ADR is not implementable.
- The vetting path (ADR-030) uses the same rule at the same rate; the 10% storage cap means a vetting provider's sample is proportionally smaller and its `consecutive_audit_passes` counter now counts aggregated audits, not chunk-audits. The 80-pass ACTIVE threshold means something different under this ADR and must be re-derived, not carried across.

## References

- [Paper 66 — Ateniese, Provable Data Possession](../research/paper-66-ateniese-pdp.md): the `P_X` detection bound; sample size independent of file size; PDP is disk-bound
- [Paper 67 — Juels & Kaliski, PORs](../research/paper-67-juels-kaliski-por.md): the `ε`-adversary model; `λ < (1 − ε)^(q_c)`; bounding the corruption rate as a distinct phase from recovering the data; spread challenges over time against a server that turns adversarial later
- [Paper 68 — Shacham & Waters, Compact PoR](../research/paper-68-shacham-waters-compact-por.md): constant-size aggregate response across arbitrarily many sampled chunks
- [ADR-059 — Homomorphic authenticator audit](ADR-059-homomorphic-authenticator-audit.md): the primitive this schedule assumes
- [ADR-002 — Proof of Storage](ADR-002-proof-of-storage.md): the audit-scalability ceiling paragraph is superseded
- [ADR-012 — Payment per Audit](ADR-012-payment-basis.md): F-03, unresolved and upstream of this ADR
- [ADR-024 — Economic Mechanism](ADR-024-economic-mechanism.md): SHELBY Condition (i), re-derived above
- [ADR-030 — Synthetic Vetting Chunks](ADR-030-synthetic-vetting-chunks.md): the 80-pass threshold requires re-derivation
- [ADR-033 — Audit Receipts Partitioning](ADR-033-audit-receipts-partitioning.md): partition sizing assumptions change by ~287×
