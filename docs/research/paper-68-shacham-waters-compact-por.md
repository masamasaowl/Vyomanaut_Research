# Paper 68 — Compact Proofs of Retrievability

**Authors:** Hovav Shacham (UC San Diego), Brent Waters (UT Austin)
**Venue / Year:** Asiacrypt 2008, LNCS 5350, pp. 90–107
**Citations:** ~4,500+ — the construction the field settled on
**Topics:** #2 Proof of Storage, #19 Adversarial Defences
**ADRs produced:** ADR-059 (proposed, supersedes ADR-002 on approval); ADR-060 (proposed)
**Findings addressed:** F-01, F-02, F-32, F-40, F-58, F-59
**Reading list:** Domain A / R-01 — third of three, and the one that closes the domain

---

## Problem Solved

Ateniese (Paper 66) gives a verifier that can check without the data but proves only possession, and its efficient variant proves less than that. Juels & Kaliski (Paper 67) give the right security goal but a construction that expires after `q` audits and needs a stateful verifier. Vyomanaut needs both halves at once: a microservice that verifies from keys alone, forever, with fresh challenges, statelessly across three gossip-synchronised replicas.

Shacham & Waters deliver exactly that. Two constructions — one from PRFs, secure in the standard model with private verification; one from BLS signatures, secure in the random-oracle model with public verification — each producing a **constant-size aggregate response**, an unbounded number of challenges, a stateless verifier, and the **first full proof of security against arbitrary adversaries in the Juels–Kaliski model**. They also supply the negative result that decides one of Vyomanaut's implementation choices: an explicit attack showing what breaks if the per-challenge coefficients are dropped.

This is the paper that closes Domain A. ADR-059's construction is its private-verification scheme.

---

## Key Findings

### The private-verification construction

The file is erasure-coded, then split into `n` blocks of `s` **sectors** each, every sector an element of `Z_p`. The client holds `α₁ … α_s ∈ Z_p` and a PRF key `k_prf`. For each block `i`:

```
σ_i  =  f_kprf(i)  +  Σ_{j=1..s} α_j · m_ij      (mod p)
```

A challenge is an `l`-element set `Q = {(i, ν_i)}` — `l` block indices drawn without replacement from `[1, n]`, each with a coefficient `ν_i` drawn with replacement from a set `B ⊆ Z_p`. The prover returns

```
μ_j  =  Σ_{(i,ν_i)∈Q} ν_i · m_ij     for j = 1..s
σ    =  Σ_{(i,ν_i)∈Q} ν_i · σ_i
```

and the verifier accepts iff

```
σ  ==  Σ_{(i,ν_i)∈Q} ν_i · f_kprf(i)  +  Σ_{j=1..s} α_j · μ_j
```

Everything on the right is computable from the keys. **The verifier never touches a byte of the file and never stores a precomputed answer.** That is the reading list's non-negotiable accept criterion for Domain A, met exactly.

### The storage/response tradeoff parameter `s`

One authenticator per block, `s` sectors per block. Storage overhead beyond the erasure code is `(1 + 1/s)×`; the response is `(1 + s)` field elements. `s = 1` recovers the introduction's simplified scheme and the Ateniese parameterisation. This single knob is what lets Vyomanaut pick its own point on the curve, and it is the reason this construction fits where Ateniese's fixed 1024-bit-tag-per-block does not.

### Parameter guidance

- Private verification: `p` a `λ`-bit prime, `λ = 80` typical. Public verification: `p` a `2λ`-bit prime over a Barreto–Naehrig curve.
- Conservative choice guaranteeing extraction against **any** adversary: `ρ = 1/2`, `l = λ`, `B = {0,1}^λ`.
- Relaxed: for a 1-in-10⁶ error tolerance, `B` may be 22-bit strings and `l = 22`.
- Extraction succeeds from any adversary answering an `ε` fraction of queries provided `ε − ρ^l − 1/#B` is non-negligible. The careful analysis lets challenge coefficients be 80 bits rather than the 160 proposed in the Ateniese ePrint — smaller coefficients mean cheaper multiplications.

### The three-part proof, and which part Vyomanaut actually relies on

1. **Unforgeability** (Theorems 4.1, 4.2) — the verifier rejects unless `μ_j = Σ ν_i m_ij` is correctly computed. Cryptographic. Different for each of the three schemes.
2. **Extractability** (Theorem 4.3, Lemma 4.5) — a well-behaved `ε`-admissible prover yields a `ρ` fraction of encoded blocks in `O(n/(ε−ω))` interactions, `ω = 1/#B + (ρn)^l/(n−l+1)^l`. Combinatorial and linear-algebraic.
3. **Retrievability** (Theorem 4.8) — a `ρ` fraction of blocks of a rate-`ρ` code reconstructs the file. Coding theory.

Parts 2 and 3 are identical across all three schemes; only part 1 differs. **Vyomanaut relies on part 1 only.** Part 3's erasure code sits *across providers* in our architecture (Paper 67), not inside a shard, so parts 2 and 3 are supplied by RS(16,56) rather than by this construction. This is why the `ω` parameterisation — which requires `l ≪ n` — does not bind on us, and it is the single most important thing to understand before reading ADR-059.

### The attack that decides the coefficient question — Appendix B

With `B = {1}` (coefficients omitted, response `μ = Σ_{i∈I} m_i`), the adversary picks a random index `i*`, draws `τ_i ∈ {−1,+1}` for each `i ≠ i*`, stores `m'_i = m_i + τ_i·m_i*`, and **discards `m_i*` entirely**. It answers `Σ_{i∈I} m'_i` when `i* ∉ I` and `Σ_{i∈I\{i*}} m'_i` otherwise. Both cases succeed with probability at least `1/l`.

The result: the adversary stores `n−1` blocks instead of `n`, answers a `1/l` fraction of all queries — non-negligible, **~9% at standard parameters** — and its stored knowledge is *information-theoretically consistent with any value for any block*. No extraction procedure can recover a single block.

This is the concrete reason Ateniese's E-PDP is unsafe and the concrete reason a Vyomanaut implementation must not "optimise away" the coefficient vector to save arithmetic or bandwidth. It is not a proof artefact.

### Deriving the query without transmitting it — the one real caveat

A conservative-parameter request is `λ·(⌈lg n⌉ + λ)` bits. The paper offers two ways to shrink it and is explicit about the standing of each:

- **Random-oracle model:** the verifier sends a short `2λ`-bit seed from which the prover generates the full query. Sound, and free for the BLS scheme which is already ROM-based.
- **Standard model:** unresolved. *"Obtaining short queries in the standard model is the major remaining open problem in proofs of retrievability."* Footnote 14 specifically rejects Ateniese's proposal of having the prover expand PRF keys sent by the verifier, since the PRF security definition assumes keys stay secret.

Vyomanaut cannot transmit the query in full (see the substitution below), so it must expand from a seed, which means the private-verification scheme's standard-model proof does not transfer. That is a real, named deviation and it is carried into ADR-059 as an open constraint rather than glossed.

### The public-verification construction

`σ_i = (H(name‖i) · Π u_j^{m_ij})^α`, verified by a pairing check against `v = g^α`. Anyone with the public key can audit. **Shortest query and response of any PoR: 20 bytes and 40 bytes at the 80-bit security level.** Secure under Computational Diffie–Hellman in bilinear groups, in the ROM.

---

## Substitution at Vyomanaut's parameters

### Field and layout

Sectors must be elements of `Z_p`. Choosing 16-byte sectors requires `p > 2^128`; the smallest such prime is

```
p = 2^128 + 51 = 340282366920938463463374607431768211507      (verified prime)
```

This is above the paper's `λ = 80` recommendation for private verification, deliberately — it buys a byte-aligned 16-byte sector, which removes an entire class of layout bugs from the codec, at the cost of ~60% more arithmetic per multiplication. At the volumes below that cost is not visible.

At a 256 KB chunk = 16,384 sectors:

| `s` | Block size | Blocks/chunk | Authenticator bytes/chunk | Storage overhead | Response size |
| --- | --- | --- | --- | --- | --- |
| 16 | 256 B | 1,024 | 16,384 | 6.250% | 272 B |
| 32 | 512 B | 512 | 8,192 | 3.125% | 528 B |
| **64** | **1,024 B** | **256** | **4,096** | **1.5625%** | **1,040 B** |
| 128 | 2,048 B | 128 | 2,048 | 0.781% | 2,064 B |

**`s = 64` is the choice.** 1.5625% on top of RS(16,56)'s 3.5× is a 0.055× increase in total stored bytes — invisible against the coding overhead itself. Going to `s = 128` halves it again but doubles the response to 2 KB for no benefit that matters. Going to `s = 16` quadruples the storage cost to buy a 272-byte response, and audit response bandwidth is not a constrained resource here.

### The challenge cannot be transmitted, so it must be derived

ADR-060 samples whole chunks and challenges every block within a sampled chunk. At 2,867 sampled chunks and 256 blocks each, `l = 733,952`, so the coefficient vector alone is

```
733,952 × 16 bytes  =  11.7 MB
```

per provider per day. Against the 100 Kbps background bandwidth budget (ADR-009, subject to F-04's unit ambiguity) that is not transmittable under any reading of the unit. The challenge must therefore be a **32-byte seed**, with the sampled chunk list and every `ν_i` derived from it by SHA-256 in counter mode.

This is the ROM deviation named above, and it must be recorded as such: the private-verification scheme's standard-model security proof does not survive it. The mitigation available is to move to the BLS public-verification scheme, which is ROM-based by construction and therefore loses nothing — at the cost of pairings on the verification path. Priced in ADR-059's Options table and left open as Q68-1.

### Why `l = n` per sampled chunk is safe here, and why it would not be in the paper

Challenging every block of a sampled chunk makes `ω = 1/#B + (ρn)^l/(n−l+1)^l` vacuous — with `l = n` the second term exceeds 1 and the extraction bound says nothing. That is fine, because Vyomanaut does not use the extraction bound. Part 1 (unforgeability) holds for **any** `l`, and part 1 is the whole of what is needed: a passing response certifies that `μ_j` was correctly computed over every block of every sampled chunk, which is deterministic possession of those chunks. Retrievability then comes from RS(16,56) across providers.

The gain from `l = n` per chunk is disk locality — the challenged region is exactly one contiguous 256 KB vLog record. This is R-08's requirement (*"the challenge is answerable from a contiguous region"*) satisfied by construction rather than by a separate mechanism, and it is why the sampling unit is a chunk rather than a block.

### Provider-side cost per audit

Per sampled chunk: one 256 KB sequential read, then `256 × 64 = 16,384` multiply-adds mod `p` for the `μ` vector plus 256 for `σ`. At 2,867 chunks/day that is ~47 M modular multiplications — sub-second on any desktop, against 0.6 minutes of disk time. **The audit remains disk-bound, exactly as Ateniese measured and as F-40 found independently.** The primitive change does not move the bottleneck; ADR-060's sampling does.

### Verifier-side cost per audit

The verifier evaluates `l` PRF outputs and `l` multiplications for `Σ ν_i f_kprf(i)`, plus `s = 64` multiplications for `Σ α_j μ_j`. The `l`-sized term is the cost, and it is the same 733,952 PRF evaluations the prover derives — roughly 12 MB of SHA-256 output per provider-audit. At 1,000 providers that is manageable but not free, and it is a new microservice CPU cost with no NFR. Recorded as Q68-2.

### Key material held by the verifier

Per file: `s = 64` values `α_j` (1,024 bytes) plus a 32-byte PRF key. At 10⁶ files, ~1.06 GB — a new table, and one whose loss is unrecoverable for auditing purposes. It is also a **symmetric** secret: the microservice can forge a passing proof for a file it holds keys for. That is F-22's operator-trust problem restated, and it is the strongest argument for the BLS variant.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Homomorphic authenticators aggregated into one value | Returning `λ` block-authenticator pairs | Response falls by a factor of `λ`; requires the linear structure that makes the coefficient attack possible if coefficients are dropped |
| Private verification (PRF, standard model) | Public verification (BLS, ROM) | No pairings, no public-key arithmetic, cheapest verification; the verifier can forge, and third-party audit is impossible |
| `s` sectors per block as an explicit knob | Fixed one-sector blocks | Storage overhead and response size trade continuously; a parameter that must be frozen before any authenticator is generated, because changing it re-tags everything |
| Erasure code applied before tagging | Tagging raw file blocks | Retrievability rather than possession; encoding cost, and a coding layer that must be adversarial-erasure-tolerant or Reed–Solomon |
| Coefficients `ν_i` drawn from a large `B` | `B = {1}` | Extraction is provable; without it the Appendix B attack stores `n−1` blocks and passes ~9% of queries |
| Full security proof against arbitrary adversaries | Intuitive extraction argument | The guarantee actually holds; the proof constrains the parameterisation (`ε − ρ^l − 1/#B` non-negligible), which is a constraint Vyomanaut sidesteps only because its outer code is elsewhere |

---

## Breaks in Our Case

- **The client is the verifier and holds `(k_prf, α₁…α_s)`** ≠ **the microservice is the verifier and the data owner is offline by design**
  → Keys are generated client-side at upload and handed to the microservice. They are independent of the AONT-RS encryption key and reveal nothing about plaintext. This is the structural adaptation that makes the family usable, and it is also what makes the verifier able to forge — accepted for V2, flagged for Domain G.

- **The erasure code is applied inside the object, so a `ρ` fraction of *its own* blocks reconstructs it** ≠ **RS(16,56) spans 56 providers; a single shard has no internal redundancy**
  → Parts 2 and 3 of the proof are not used. Vyomanaut takes unforgeability from this paper and retrievability from the stripe. Per-provider guarantee is PDP-strength; per-stripe guarantee is PoR-strength. Say so in ADR-002's successor rather than letting "PoR" carry a claim it cannot support.

- **Short queries in the standard model are an open problem** ≠ **Vyomanaut's query is 11.7 MB if transmitted**
  → Expand from a 32-byte seed under SHA-256, accept the random-oracle assumption, and record it. The alternative that loses nothing is the BLS scheme, which is ROM-anyway. This is the one place where the adopted scheme is weaker than the paper as written, and it should not be buried.

- **One file, one server, one static object** ≠ **repair moves a shard to a replacement provider without the owner online (ADR-004)**
  → The authenticators travel with the shard bytes and remain valid — the tags are over block contents and indices, not over provider identity, so a shard migrating intact needs no re-tagging. But a *reconstructed* shard is new bytes at the same stripe position and has no valid authenticators, and the keys needed to generate them are the owner's. Nothing in this paper addresses it. This is R-03, it is unsolved here, and it is a hard blocker on the repair path that ADR-059 must state rather than assume away.

- **Index uniqueness within one file** ≠ **many chunks of one file on one provider, and shard positions reused across stripes**
  → Index must be `(chunk_ordinal × 256) + block_ordinal`, globally unique within a file, and never reused after repair. Same requirement as Ateniese Remark 3, arriving from a second source, which makes it a schema invariant rather than a note.

- **`λ = 80`, 80-bit primes, 20-byte responses** ≠ **16-byte byte-aligned sectors over a 128-bit prime**
  → A deliberate over-provision for layout sanity, costing ~60% more per modular multiplication on a path measured in sub-seconds against 0.6 minutes of disk. Recorded so that nobody later "corrects" it back to 80 bits and breaks the sector alignment.

- **The paper's cost model counts field operations** ≠ **F-40's finding, and Ateniese's measurement, that this workload is disk-bound**
  → Do not tune the primitive. Tune the sampling rate. ADR-060 is where the win is; ADR-059 only makes verification possible.

---

## Decisions Influenced

- **[ADR-059](../decisions/ADR-059-homomorphic-authenticator-audit.md) [#2 Proof of Storage]** `PROPOSED — SUPERSEDES ADR-002 on approval`
  Adopts the private-verification construction with `p = 2^128 + 51`, 16-byte sectors, `s = 64`, 1,024-byte blocks, 256 blocks per 256 KB chunk, 1.5625% authenticator overhead, 1,040-byte responses, per-file key material `(k_prf, α₁…α₆₄)` held by the microservice, and mandatory per-challenge coefficients derived from a fresh 32-byte seed.
  *Because:* it is the only construction in Domain A that is simultaneously verifiable-without-data, unbounded, stateless and constant-response — the four properties F-01, ADR-025 and ADR-002 respectively demand.

- **[ADR-060](../decisions/ADR-060-sampled-chunk-audit-schedule.md) [#2 Proof of Storage]** `PROPOSED`
  Supplies the reason the sampling unit is a whole chunk (`l = n` per chunk gives contiguous reads and deterministic per-chunk possession without needing the extraction bound) and the constant-size response that makes one receipt per `(provider, file, day)` possible.

- **[ADR-014](../decisions/ADR-014-adversarial-defences.md) [#19 Adversarial Defences]** `CORRECTION REQUIRED`
  Defence 4's verification claim becomes true for the first time under ADR-059, but its stated mechanism is wrong and must be replaced, not annotated.

- **[ADR-017](../decisions/ADR-017-audit-receipt-schema.md) [#2 Audit Receipt Schema]** `REVISION REQUIRED`
  `response_hash BYTEA(32)` becomes a 1,040-byte proof `(μ₁…μ₆₄, σ)`; `challenge_nonce BYTEA(33)` becomes a 32-byte seed; `chunk_id` becomes a derivable chunk set. Full field list in the reply.

- **[ADR-025](../decisions/ADR-025-microservice-consistency-mechanism.md) [#1 Microservice Cluster]** `EVIDENCE ADDED`
  The stateless-verifier property means the audit path adds **no** new coordinated operation to ADR-013's six. Any replica holding the file keys can issue and verify a challenge independently. This is a positive result for F-35 and worth recording there.

---

## Disagreements

- **Against Ateniese et al. (Paper 66), on E-PDP.** SW state that Ateniese do not show extraction, that the only place extractability is addressed is a short intuitive paragraph in their Appendix A, and that their repeated use of PRFs is never applied in a security reduction — *"compelling evidence that a rigorous security proof was not provided."* Appendix B then breaks the E-PDP-like scheme concretely.
  *Implication for us:* take the detection formula and the disk-bound finding from Paper 66; take the construction from here.

- **Against the PDP model as a product guarantee.** SW argue directly that guaranteeing 90% of blocks is unsatisfactory — *"how happy a user would be were 10% of a file containing accounting data lost"* — and that a scheme guaranteeing only a sum of blocks offers nothing usable.
  *Implication for us:* Vyomanaut's answer is that the outer code is the stripe. That answer is sound but it has never been written down, and until it is, ADR-002's use of "PoR" is unsupported.

- **Bowers, Juels & Oprea (CCSW 2009) against SW's practical positioning.** BJO argue that SW's ability to extract for any `ε < 1` is of limited practical benefit, because an adversary with `ε > 1/2` is detected almost immediately in a Phase-I spot check; and that at small `ε` their JK variant is cheaper on both storage and communication (their Table 2 and §5.2).
  *Implication for us:* the counter-argument does not reach Vyomanaut, because the property we buy from SW is not high-`ε` extraction — it is unbounded stateless verification, which BJO's variant does not have.

- **Against a possible reading of this note.** Adopting SW does not make the audit cheap. It makes it *correct*. Every scaling number in F-02, F-58, F-59 and F-40 is settled by ADR-060's sampling, not by this construction; a per-chunk-daily audit under SW is exactly as unaffordable as it is today.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q66-1, Q68-1, Q68-2 and Q68-3.
