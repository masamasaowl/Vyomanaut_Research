# Paper 74 — Towards Server-Side Repair for Erasure Coding-Based Distributed Storage Systems

**Authors:** Bo Chen (Stony Brook University), Anil Kumar Ammula, Reza Curtmola (New Jersey Institute of Technology)
**Venue / Year:** ACM CODASPY 2015, pp. 281–288
**Also published as:** an extended technical report `[10]` containing the proofs, the HAIL overview, and the implementation/experimental sections — **not obtained**
**Topics:** #2 Proof of Storage, #3 Confidentiality, #4 Repair Protocol, #19 Adversarial Provider Behaviour
**Track:** LTS
**Reading list:** Domain A / **R-47** — Band 0 — *"the erasure-coded case rather than network-coded — closer to RS(16,56)"*; reaches into Domain P / **R-28** — *"repair without any party assembling `k`"*
**ADRs produced:** ADR-078 (authenticator transport through repair)
**Findings raised:** F-LTS-14, F-LTS-15
**Questions closed:** **Q68-3** (constructively, subject to ADR-078's proof obligation)
**Questions raised:** Q74-1, Q74-2, Q78-1
**Triage score:** 10/10 (parameter reach 2 · trust model 2 · evidence 2 · actionability 2 · corpus delta 2)

---

## Provenance

Read: the full 8-page CODASPY proceedings version. **The proofs of Theorems 4.1 and 4.2, the HAIL
background, the encoding/decoding detail, and the entire experimental evaluation are deferred by the
authors to technical report `[10]`, which was not obtained.** Every claim depending on those is
tagged `[P §x, proof not read]` and no ADR in this session rests on one.

What *is* fully present in the conference version — and is what matters here — is the complete
protocol specification: Figures 1–4 give Setup, Challenge, Repair, and all seven component
algorithms in full pseudocode. The construction can be read, checked, and transplanted from the
version obtained. The security *proofs* cannot.

This is the closest paper in the corpus to Vyomanaut's actual code family. Read Paper 73 first for
the threat model, which this paper inherits by reference and does not restate.

---

## Problem Solved

HAIL is the only prior remote-data-checking scheme that gives strong adversarial guarantees over an
erasure-coded distributed store, and its repair phase is the worst possible one: to repair **a single
corrupt segment**, the data owner must download **all `n` segments**, reconstruct the entire file,
and recompute the lost segment `[P §1]`. Bowers et al. posed low-bandwidth repair for adversarial
erasure-coded storage as an **open problem**, and it stood unsolved `[P §1]`.

`RDC-EC` solves it. The owner's repair cost drops from `n` segments to **two**, with the servers
carrying the reconstruction between themselves `[P Table 1]`. Getting there required defeating a
specific attack: server-side repair naively needs the servers to know the information dispersal
matrix `M`, and an adversary that knows `M` can substitute a bogus file wholesale and pass every
challenge forever `[P §4.3]`. The paper's answer is to have the servers compute over **masked**
coefficients — algebraically useful, informationally empty.

For Vyomanaut the value is not the masking. It is that this paper is the only source in the corpus
that specifies, in full, **what a repair protocol looks like when the party doing the repair is not
the key holder** — and it does so with a homomorphic tag construction structurally identical to the
one ADR-059 already adopted. That construction is what closes Q68-3.

---

## Key Findings

### The cost table, and where Vyomanaut sits in it

`[P Table 1]`, for a file `F` of `k` segments encoded to `n`, single-server failure:

| | MR-PDP (replication) | HAIL (erasure) | RDC-NC (network) | **RDC-EC** |
| --- | --- | --- | --- | --- |
| Total server storage | `O(n·|F|)` | `O(n|F|/k)` | `O(2n|F|/(k+1))` | `O(n|F|/k)` |
| Repair communication | `O(|F|)` | `O(n|F|/k)` | `O(2|F|/(k+1))` | **`O(|F|/k)`** |
| Server computation (repair) | `O(1)` | `O(1)` | `O(ℓ)` | `O(k)` |
| Sub-file access | yes | yes | **no** | yes |

RDC-EC keeps erasure coding's optimal storage and sub-file access while paying network coding's
repair communication on the *owner's* link. The `O(k)` server computation is the price.

### The repair protocol, in the form that transfers

`[P §4.2, Figure 2]`. Client `C` has detected corrupt segment `b_y` at faulty server `S_y`.

1. `C` sends a fresh key `K` to all `n` servers; each returns a **repair tag** `t_i`.
2. `C` verifies all `n` tags at once, because *the `n` repair tags themselves form a valid
   integrity-protected dispersal codeword* — so a Reed-Solomon decode over the tags both verifies
   and **localises** any server that lied `[P §4.1, VerifyAllRepairTag]`.
3. `C` picks `k` healthy repair servers, computes intermediate coefficients `z_{i₁..i_k}` from `M`,
   and masks them: `Z_{i_j} = a·z_{i_j} + r·x_j` for fresh secrets `a, r, x_j` `[P §4.2 step 5]`.
4. Each repair server returns **two** partial segments: `Z_{i_j}·b_{i_j}` and `x_j·b_{i_j}`.
5. A randomly chosen **Aggregation Server (AS)** sums each set, yielding `b'_y = Σ Z_{i_j} b_{i_j}`
   and `b''_y = Σ x_j b_{i_j}`.
6. `C` unmasks: `b_y = a⁻¹(b'_y − r·b''_y) ± PRF correction` `[P §4.2, RepairOneSegment]`.

The algebra is worth stating because it is the whole trick. Since `Z_{i_j} = a·z_{i_j} + r·x_j`,

```
b'_y − r·b''_y  =  Σ (a·z_{i_j} + r·x_j)·b_{i_j}  −  r·Σ x_j·b_{i_j}
                =  a·Σ z_{i_j}·b_{i_j}
                =  a·b_y                                              (before the PRF term)
```

`[DERIVED]` — the identity is stated in the paper implicitly through `RepairOneSegment`; the
expansion above is mine and is a one-line check of it.

### The repair tag: publicly computable, privately verifiable

`[P §4.1]` The repair tag is `t_i = Σ_{j=1..ℓ} g_K(file_handle‖j) · b_{ij}` — a keyed random linear
combination of the segment's symbols. Two properties are named explicitly:

- **Publicly computable** — anyone holding the seed `K` can compute the tag for a segment they hold.
  No secret is needed to *produce* it.
- **Privately verifiable** — only the holder of `sk` can check it, because correctness is established
  by the tags collectively forming a valid dispersal codeword under the secret matrix `M`.

And the property that matters most: `[P §4.2, VerifyPartialRepairTag]` **the repair tag of a scaled
segment is the correspondingly scaled repair tag.** *"if `t_{i_j}` is the repair tag for the stored
segment `b_{i_j}`, then `x·t_{i_j}` will be the repair tag for its partial segment `x·b_{i_j}`,
considering both repair tags are generated based on the same key."*

That sentence is the finding. **The tag travels through the linear repair map with the data.** The
AS verifies `t' =? Σ Z_{i_j} t_{i_j}` on aggregates it cannot decompose, and if the check fails it
runs `VerifyPartialRepairTag` per contribution to name the liar.

### The attack that forces the masking

`[P §4.3]` If the adversary learns `M`, it can: recompute the parity from the real file; subtract to
extract the embedded secret random values; substitute a bogus file of the same size; re-embed those
secrets into the bogus parity; and pass every subsequent challenge. **The dispersal matrix is a
secret in HAIL and in RDC-EC, and revealing it is total compromise** `[P §4.3]`. Theorem 4.1 states
the probability the adversary learns `M` under RDC-EC is negligible `[P §4.3, proof not read]`.

This is where Vyomanaut and this paper part company, and the difference is favourable — see
Substitution §4.

### The Aggregation Server is explicitly trusted, and the paper says so

`[P §4.1, §4.2 step 8]` `C` **restores the AS's code component** before repair — removes the malware,
reinstalls clean software — and *"we assume the AS acts honestly until the end of the Repair phase."*
The AS is additionally handed `Z_{i₁..i_k}`, `x_{1..k}`, the `k` repair tags, and `K`.

`[DERIVED]` Because the AS knows `x_j` and receives `x_j·b_{i_j}`, it can compute
`x_j⁻¹·(x_j·b_{i_j}) = b_{i_j}` for every `j`. **The AS obtains all `k` source segments.** The
masking protects `M` from the *repair servers*; it does not hide the segments from the aggregator.

This is not a flaw in the paper — its threat model states the assumption plainly, and confidentiality
is declared orthogonal `[P §4.3 of Paper 73]`. It is a flaw in any reading of the paper that expects
it to close F-69. It does not.

But the *reason* the AS is given the coefficients is stated, and it is not incidental: it needs them
for `VerifyPartialRepairTag`, i.e. to **localise** a faulty contributor `[P §4.2, Figure 4]`. That
gives a clean statement of a design axis nobody in this project had articulated — see Substitution
§5.

### The field: everything in one algebra

`[P §4.2]` *"all the arithmetic operations are performed in `GF(2^w)`"* — the erasure code, the
masking, the PRF output, and the repair tags. `[P §1.2]` records the engineering consequence: *"To
achieve an appropriate security level, we extended Jerasure's coding functions to support 128-bit
symbols."*

They did not adopt a large field for coding reasons. They adopted it because **the tag must live in
the same field as the code for the homomorphism to transport, and a tag in `GF(2^8)` is not a tag.**
This is the paper's least-advertised and most consequential engineering decision, and it is the one
Vyomanaut has, so far, made the other way.

---

## Substitution at Vyomanaut's Parameters

### 1. The repair-cost table at RS(16,56)

`[DERIVED]` `|F|` here is one segment's worth of coded data. At Vyomanaut's parameters a repair of
one lost 256 KiB shard costs the coordinating party:

| Scheme | Owner/coordinator bytes per single-shard repair |
| --- | --- |
| HAIL shape (`n` segments) | `56 × 262,144` = **14.68 MB** |
| ADR-004 / ADR-076 as built (`k` segments to the repairer) | `16 × 262,144` = **4.19 MB** |
| RDC-EC shape (2 segments to the coordinator) | `2 × 262,144` = **524 KB** |

Vyomanaut already sits at the middle row — ADR-076 moved reconstruction to an elected provider, so
the microservice carries no shard bytes at all and the *repairer* carries 4.19 MB. RDC-EC's headline
saving is a saving Vyomanaut has already banked by a different route. **The `O(|F|/k)` result is not
what this paper gives us.**

### 2. The tag homomorphism, evaluated against ADR-059 — and the field mismatch

This is the load-bearing substitution.

ADR-059's authenticator, for file-global block index `i` with 64 sectors `m_{i1..i64}`:

```
σ_i  =  f_kprf(i)  +  Σ_{j=1..64} α_j · m_ij        (mod p),   p = 2^128 + 51
```

RS(16,56) reconstruction of shard `y` from any 16 surviving shards `S`, symbol-wise:

```
m_y  =  Σ_{j ∈ S} z_j · m_j                         (in GF(2^8), poly 0x11d)
```

`[DERIVED]` If both operations lived in the same field, the repair tag would transport exactly as
`[P §4.2]` states for `x·t_{i_j}`:

```
Σ_j z_j·σ_{i_j}  =  Σ_j z_j·f(i_j)  +  Σ_j z_j·(Σ_l α_l·m_{i_j,l})
                 =  Σ_j z_j·f(i_j)  +  Σ_l α_l·(Σ_j z_j·m_{i_j,l})
                 =  Σ_j z_j·f(i_j)  +  Σ_l α_l·m_{y,l}
                 =  Σ_j z_j·f(i_j)  +  (σ_y − f(i_y))

  ⟹   σ_y  =  Σ_j z_j·σ_{i_j}  +  [ f(i_y) − Σ_j z_j·f(i_j) ]
                └── computed by helpers ──┘   └── computed from keys alone ──┘
```

**The correction term `Δ = f(i_y) − Σ_j z_j·f(i_j)` depends only on `k_prf`, the block indices, and
the RS coefficients. It contains no chunk data.** So the party holding the key computes `Δ` without
seeing bytes, and the parties holding bytes compute `Σ z_j σ_j` without holding the key.

**It does not currently work, and the reason is a single parameter.** ADR-059 evaluates `σ` in
`Z_p` with 16-byte sectors read as big-endian integers; RS evaluates in `GF(2^8)`. Addition in `Z_p`
is integer addition with carry; addition in `GF(2^8)` is XOR. The two do not commute, so the
factorisation on line 2 above is invalid and the homomorphism does not transport. **This is F-LTS-14
and it is the entire reason Q68-3 has stood open.**

**The fix, and the hand check.** `8 | 128`, so `GF(2^8)` is a subfield of `GF(2^128)`. Represent
`GF(2^128) = GF(2^8)[Y]/(g(Y))` with `g` irreducible of degree 16 over `GF(2^8)`, and map a 16-byte
sector `(b_0 … b_15)` to `Σ b_i·Y^i`. Then `GF(2^128)` is a 16-dimensional `GF(2^8)`-vector space
with basis `1, Y, …, Y¹⁵`, and multiplying a sector by an embedded scalar `ι(z)`, `z ∈ GF(2^8)`, is
**by definition of scalar multiplication** the bytewise `GF(2^8)` multiplication of each of its 16
coordinates — which is exactly what the RS codec does.

*Hand check.* Take `z = 2` (the primitive element ADR-003's Vandermonde uses) and a sector whose
bytes are `(b_0 … b_15)`. RS produces `(2·b_0, …, 2·b_15)` bytewise in `GF(2^8)`. The field
embedding produces `ι(2)·Σ b_i Y^i = Σ (2·b_i) Y^i`, since `ι(2)` is a subfield element and
multiplication distributes over the `GF(2^8)`-basis. Identical coordinate vectors. ✓ The two
operations coincide, which is what the factorisation needs.

So: **change ADR-059's authenticator field from `Z_p`, `p = 2^128 + 51`, to `GF(2^128)` under a
subfield-compatible basis, and authenticators transport through repair by the code's own
linearity.** Tag size is unchanged at 16 bytes; per-chunk overhead stays 4,096 B = 1.5625%. That is
ADR-078.

### 3. Cost of the transported-tag repair protocol

`[DERIVED]` Per repaired 256 KiB chunk:

```
helper→repairer, tags   16 helpers × 256 blocks × 16 B  =  65,536 B   (alongside 4.19 MB of shard data — +1.56%)
repairer→microservice   256 blocks × 16 B                =   4,096 B   (aggregate tags only, no shard bytes)
microservice→repairer   256 blocks × 16 B                =   4,096 B   (Δ corrections)
microservice PRF work   256 blocks × 17 evaluations      =   4,352 HMAC-SHA-256 per chunk
```

**Hand check against ADR-060's own PRF accounting.** ADR-060 prices routine verification at 733,952
PRF evaluations per provider per day, and `reading-list.md` Domain A prices the whole fleet at
~8,494 HMAC-SHA-256/s — a fraction of one core. Repair-time `Δ` generation is 4,352 evaluations per
repaired chunk. Under ADR-076's `r0 = 8` gate, repair is rare by construction. Even at an
implausible 10⁴ chunk-repairs/day the added load is `4.35 × 10⁷` HMAC/day = **503/s**, i.e. 5.9% of
the routine audit load already budgeted. ✓ Negligible.

**The microservice never receives a shard byte.** ADR-076's central property survives intact.

### 4. Vyomanaut does not need the masking, for a reason worth stating

`[DERIVED]` The masking exists solely to keep `M` secret `[P §4.3]`, because in HAIL and RDC-EC the
secrecy of `M` *is* the integrity mechanism — parity symbols carry embedded MACs derived from it.
Vyomanaut's dispersal matrix is a **public** Vandermonde over `GF(2^8)` with `α = 2`
(`internal/erasure/rs_internal.go`), and its integrity mechanism is ADR-059's authenticators, which
are independent of `M`.

So `[P §4.3]`'s bogus-file attack does not apply: an adversary knowing Vyomanaut's `M` still cannot
produce authenticators, because those need `(k_prf, α₁…α₆₄)`, which `M` does not contain. `a`, `r`,
`x_j`, `GenRandom`, and the two-round structure all drop. Repair becomes a single round of
`z_j·shard_j` contributions.

**This is a real simplification and it costs something.** Dropping the masking is what makes the
aggregator able to decode — see §5.

### 5. The accountability/blindness axis, stated for the first time

`[DERIVED]` from `[P §4.2, Figure 4]`. The AS is given `(Z_{i_j}, x_j)` so it can run
`VerifyPartialRepairTag` and name a faulty contributor. Possession of those scalars is also exactly
what lets it invert each contribution and recover every source segment.

Generalised: **an aggregator that can attribute a bad contribution to a specific helper can also
invert that helper's contribution.** Localisation requires per-contribution checking; per-contribution
checking requires knowing the per-contribution scalar; knowing the scalar inverts the contribution.

The converse gives a candidate construction. Withhold `(Z_{i_j}, x_j)` from the aggregator and it
holds two linear combinations of 16 shards and no scalars. Two combinations is below `k = 16`, so
under AONT-RS it learns nothing. A separate party — one holding `a`, `r` — performs the unmask and
obtains **one** shard, also below `k`. **No party assembles `k`.** That is precisely R-28's accept
criterion, and it has been sitting inside a Band 0 paper.

**Do not treat this as a result.** It is `[INFERRED]`, the paper proves nothing about it (Theorem
4.1 is about `M`, not about hiding segments from the AS), and my own sketch already finds a leak:
the aggregator receives `Z_{i_j}·b_{i_j}` and `x_j·b_{i_j}` as *separate* vectors, so it can form
the sector-wise ratio `Z_{i_j}/x_j` — a constant across the whole shard — which is one equation
about the masking, and if the aggregator is itself a helper it knows one `b_{i_j}` outright and can
solve for that `x_j`. The construction needs the contributions to arrive **pre-aggregated**, which
means a chained or tree-shaped aggregation with its own failure modes. It goes to Domain P as a
lead, not to an ADR. → Q74-1.

### 6. What the field change costs

`[DERIVED]` ADR-059 chose `p = 2^128 + 51` for a stated reason: *"to obtain a byte-aligned 16-byte
sector. Every 16-byte sector is `< p` by construction, so no rejection sampling or padding is
required anywhere in the codec."* `GF(2^128)` keeps that property and improves on it — every 16-byte
string is a valid field element with no comparison at all, where `Z_p` requires each sector to be
checked against `p` in principle even though the bound makes it vacuous.

Arithmetic cost moves from 128-bit modular multiplication to `GF(2^128)` carry-less multiplication.
On any x86-64 with `PCLMULQDQ` — universal on the target hardware and the same instruction GCM uses
— this is **cheaper**, not more expensive. On a target without it, it is a software Karatsuba over
carry-less halves, comparable to the `Z_p` path. Neither has been measured. → Q78-1.

**The cost that is real is a proof obligation, not a cycle count.** Shacham & Waters state the
private scheme over `Z_p`. The security argument — a forged aggregate implies `Σ α_j Δμ_j = 0` for
some `Δμ ≠ 0`, which a randomly chosen secret `α` satisfies with probability `1/|F|` — is linear
algebra over a field and does not use any property of `Z_p` beyond being one. `[INFERRED]` it
carries to `GF(2^128)` at `2⁻¹²⁸`. RDC-EC instantiates a Shacham–Waters-style tag over `GF(2^w)`
`[P §4.1]` and RDC-NC over `GF(p)` `[Paper 75 §3.1.4 fn. 5]`, so both instantiations exist in the
literature. **This must be checked against Shacham & Waters Theorem 4.2 line by line before ADR-078
leaves `Proposed`.** Recorded as ADR-078's blocking constraint.

---

## What This Paper Rules Out

- **The claim that Q68-3 is a trilemma.** Q68-3 states three shapes of answer and calls all three
  unacceptable, on the premise that computing a tag for reconstructed content requires one party to
  hold the key and the content simultaneously. `[P §4.2]`'s scaled-tag property falsifies the
  premise for any tag linear in the content over the code's field. The question was not hard; it was
  asked in a field where the answer is false.
- **Microservice-side re-tagging as the only cheap option.** ADR-076 §4 removed that option on the
  grounds that the microservice no longer holds the reconstructed bytes. Under ADR-078 it does not
  need them — `Δ` is computed from keys and indices. The option ADR-076 closed reopens in a form
  that does not violate the constraint that closed it.
- **RDC-EC's masking machinery, for Vyomanaut.** `a`, `r`, `x_j`, `GenRandom`, `GenRepairServerCoefficient`'s
  matrix inversion, and the two-round structure all exist to protect a secret dispersal matrix.
  Vyomanaut's is public and its integrity does not rest on it. Adopting the masking would buy
  nothing unless the blind-aggregator line in §5 is pursued, in which case it is bought for a
  different property entirely.
- **The Aggregation Server as a model for a blind repairer.** `[P §4.2 step 8]` trusts it explicitly
  and hands it the scalars. Anyone citing this paper as evidence that server-side repair keeps the
  aggregator blind has misread it.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Server-side repair with a two-segment owner cost | HAIL's `n`-segment owner reconstruction | Owner cost falls ~28× at `n = 56`; owner must run a two-round protocol instead of one |
| Masked coefficients | Revealing `M` to enable server collaboration | Defeats `[P §4.3]`'s bogus-file substitution; requires fresh `a, r, x_j` per repair and a matrix inversion per event |
| A designated, temporarily-trusted Aggregation Server | Peer-to-peer aggregation among the `k` helpers | Enables fault localisation and one clean verification point; **the AS obtains all `k` segments** |
| Repair tags as a dispersal codeword | Per-server independent tags | `n` tags verify and localise in one RS decode; the tag scheme is welded to the code parameters |
| `GF(2^w)` throughout, with 128-bit symbols | A separate large-prime field for tags | The tag homomorphism transports through the code; required extending Jerasure, which stock libraries do not do |
| Erasure coding | Network coding | Keeps sub-file access and optimal storage; server repair computation rises from `O(1)` to `O(k)` |

---

## Breaks in Our Case

- **The information dispersal matrix `M` is secret and its secrecy carries the integrity guarantee**
  ≠ **Vyomanaut's Vandermonde matrix over `GF(2^8)` with `α = 2` is public in source, and integrity
  comes from ADR-059's authenticators** — **Costed (negative — a simplification)**
  → `[P §4.3]`'s attack does not apply. All masking machinery drops; repair is one round of
  `z_j·shard_j`. The cost of the simplification is that the aggregator can decode, which the masking
  would otherwise have been a starting point for hiding. → Q74-1.

- **`GF(2^w)` for code, masking, PRF and tags alike** ≠ **ADR-059 tags in `Z_p`, `p = 2^128 + 51`,
  while `internal/erasure` codes in `GF(2^8)`** — **Fatal as built, Costed once fixed**
  → This mismatch is why Q68-3 stood open. `GF(2^8) ⊂ GF(2^128)` since `8 | 128`, so a
  subfield-compatible `GF(2^128)` restores the homomorphism at unchanged tag size. → ADR-078,
  F-LTS-14.

- **The Aggregation Server is cleaned and trusted for the duration of the repair epoch, and told the
  masking scalars** ≠ **Vyomanaut has no party it can clean, and ADR-019 says the operator never sees
  plaintext** — **Open**
  → ADR-076 already ruled that the elected repairer obtains plaintext and that this is relocation,
  not resolution. This paper does not improve on that ruling; it independently arrives at the same
  shape (random per-event choice, trusted only for the event). The blind-aggregator variant in
  Substitution §5 is the only thing here that could improve on it, and it is unproven. → Q74-1,
  Domain P / R-28.

- **The owner is online and performs the final unmask with `a`, `r`** ≠ **ADR-004 makes the owner
  offline by design** — **Costed**
  → Under ADR-078 the final step is not an unmask, it is a `Δ` addition from key material the
  microservice already holds under ADR-059. The owner stays offline. If the blind-aggregator variant
  is ever pursued, the unmask lands on the microservice, which then obtains **one** shard — below
  `k = 16`, so below AONT-RS's disclosure threshold. That is survivable; it is not free, and it
  would need writing into ADR-019.

- **A repair event repairs one corrupt segment detected at one faulty server** ≠ **ADR-076's `r0 = 8`
  gate fires when a segment is down to 24 of 56 fragments, so a repair event reconstructs up to 32
  fragments at once** — **Costed**
  → The per-event `Δ` and tag traffic in Substitution §3 multiply by up to 32: 131 KB of tag traffic
  and 139,264 PRF evaluations per gated repair event. Still negligible against the 4.19 MB × 32 of
  shard data moving in the same event. No adaptation required; recorded so the number is not
  rediscovered.

- **`b = (n − k − 1)/2` servers corruptible per epoch, with the code component restorable at epoch
  end by removing malware** ≠ **Vyomanaut providers are consumer desktops the operator cannot touch;
  there is no "restore the code component" operation and no epoch boundary at which one occurs** —
  **Fatal**
  → Every guarantee in this paper that depends on periodic code restoration is unavailable. That
  includes the AS trust assumption and the mobile-adversary containment. Vyomanaut's substitute is
  ADR-058's keystore-derived provider identity plus ADR-008's reliability scoring, neither of which
  is a restoration. Do not port any `b`-bounded claim.

---

## Decisions Influenced

- **ADR-078 [#2 Proof of Storage · #4 Repair Protocol] `NEW — Proposed`**
  Transport authenticators through repair by the code's linearity. Change ADR-059's authenticator
  field from `Z_p (p = 2^128 + 51)` to `GF(2^128)` under a basis compatible with the `GF(2^8)`
  subfield embedding; have each repair helper contribute `ι(z_j)·σ_j` alongside `z_j·shard_j`; have
  the microservice supply `Δ = f(i_y) + Σ_j ι(z_j)·f(i_j)` from key material alone. **Q68-3 closes:
  a reconstructed shard carries a valid authenticator from the moment it is written, and no party
  ever holds both the key and the plaintext.**
  *Because:* `[P §4.2, VerifyPartialRepairTag]` states that the repair tag of a scaled segment is
  the scaled repair tag, and `[P §4.2]` puts the entire construction in one field so that it does.

- **ADR-059 [#2 Proof of Storage] `PARAMETER CHANGED via ADR-078`**
  `(p, s) = (2^128 + 51, 64)` becomes `(GF(2^128), 64)`. ADR-059's own justification for `p` —
  byte-aligned 16-byte sectors with no rejection sampling — is preserved and marginally improved.
  Its "Repair breaks tagging, and nothing here fixes it" open constraint is discharged. Its F-22
  concern is untouched: the microservice still holds a forgeable secret.
  *Because:* the field was chosen for codec convenience without the repair path in view, and it is
  the sole obstruction.

- **ADR-076 [#4 Repair Protocol] `EXTENDED — §4's Q68-3 narrowing is reversed`**
  ADR-076 §4 recorded that Q68-3 had lost the microservice-tagging option and reduced to two
  unacceptable choices, making R-47 the gating read for Proof of Storage. R-47 is now read and the
  option returns in a form compatible with ADR-076's own constraint: the microservice supplies `Δ`
  without receiving shard bytes. The elected-repairer protocol ADR-076 specifies must carry the tag
  aggregate and the `Δ` exchange.
  *Because:* `Δ` is a function of `(k_prf, indices, coefficients)` only — ADR-076 removed the
  microservice from the *data* path, and `Δ` is not in the data path.

- **ADR-003 [#3 Erasure Coding] `CONSTRAINT ADDED`**
  The `GF(2^8)` field and the `0x11d` polynomial are no longer purely a codec choice. Under ADR-078
  they are a parameter of the audit primitive, because the authenticator field must contain the code
  field as a subfield. If ADR-026's successor ever widens the code to `GF(2^16)` (LESS's requirement,
  per Q65-1), `16 | 128` still holds and `GF(2^128)` still works — but that is a coincidence worth
  recording rather than relying on.
  *Because:* the subfield relation `GF(2^w) ⊂ GF(2^128)` requires `w | 128`.

- **ADR-014 [#19 Adversarial Defences] `NO CHANGE — see Paper 73`**
  This paper's contribution to Defence 2 is inherited from Paper 73's threat model. Handled there.

---

## Falsifiers

1. **ADR-078 is void if Shacham & Waters' Theorem 4.2 does not carry to `GF(2^128)`.**
   The proof obligation: check that the extractability argument and the `1/|F|` forgery bound use
   only field properties, and that no step relies on `Z_p`'s ordering, its characteristic, or the
   integer representation of sectors. Characteristic 2 is the specific thing to look at — `2x = 0`,
   so any step that divides by 2 or relies on `x + x ≠ 0` fails. **Blocking on ADR-078.**

2. **The homomorphism is void if `internal/erasure` reconstruction is not a pure `GF(2^8)`-linear
   map of surviving shards.** Verify against `rs_internal.go`: `invertMatrix` and the Vandermonde
   construction must give `shard_y = M_y · M_S⁻¹ · shard_S` with coefficients in `GF(2^8)` and no
   non-linear step (no rank-deficiency fallback, no interleaving). If any shard position is handled
   specially — the systematic first `k` in particular — the coefficient vector differs but linearity
   holds; if there is any non-linear path, ADR-078 fails for that path.

3. **The `Δ` construction is void if publishing `Δ` leaks `k_prf`.** In the specified protocol the
   microservice computes `Δ` and returns `σ_y = σ' + Δ`, never publishing `Δ` itself. If an
   implementation instead sends `Δ` to the repairer, each repaired block yields one linear equation
   in 17 PRF outputs, and repeated repairs over overlapping index sets accumulate. Whether that
   system becomes solvable at realistic repair volumes is not derived here. **The protocol must
   forbid publishing `Δ`; if a design reason ever forces it, this becomes a live question.**

4. **The blind-aggregator lead (Substitution §5) is void if pre-aggregation cannot be enforced.**
   It requires the aggregator to receive only sums, never individual contributions. Any topology in
   which contributions arrive separately — which is the natural network implementation — restores
   invertibility. A chained or tree aggregation fixes it and introduces its own liveness problem
   under ADR-021's NAT constraints. → Q74-1.

5. **The negligible-cost claim in Substitution §3 is void if repair is not rare.** It is priced
   against ADR-076's `r0 = 8` gate. F-LTS-07 records that the gate **does not exist yet** and the
   system currently performs eager repair. Under eager repair, every nightly absence triggers
   reconstruction and the `Δ` load scales with churn, not with durability events. **ADR-078's cost
   argument depends on ADR-076 §1 being built first**, which is ADR-076's own stated order of work.

---

## Disagreements

- **HAIL (Bowers, Juels & Oprea):** posed low-bandwidth adversarial repair as an open problem and
  left it. This paper closes it for the erasure-coded case.
  *Implication for us:* Vyomanaut's ADR-004 assumed P2P repair was straightforward and specified it
  in one paragraph. The literature treated it as an open problem for four years. F-LTS-07 and
  F-LTS-08 — the gate that was never built and the microservice that decodes — are what that
  underestimate looks like in a codebase.

- **Paper 69 (Erway et al., DPDP), as cited in ADR-059's option table:** ADR-059 records DPDP as
  supplying *update verification* but not *update authority*, and calls the authority question "the
  actual blocker." That framing is now wrong, and it was wrong for a specific reason: it assumed the
  new tag must be *computed* by someone. Under `[P §4.2]`'s scaled-tag property the new tag is not
  computed, it is **transported**. Nobody exercises authority over it.
  *Implication for us:* ADR-059's assessment of Paper 69 stands as a description of DPDP; its
  conclusion about Q68-3 does not. Corrected in ADR-078's Context.

- **Paper 16 (AONT-RS, Resch & Plank):** its security section assumes the adversary is a set of
  shard holders, not a party performing a linear operation across shards. Nothing here contradicts
  it, but the blind-aggregator lead needs it re-read against a *linear-combination* adversary — one
  that holds two combinations of 16 shards rather than two shards. `reading-list.md` Domain P
  already flags this re-read as a must-read before searching outward; it is now specific.

---

## Corpus Delta

New to the corpus: a fully specified repair protocol in which the repairing parties are not the key
holders; the scaled-tag homomorphism; the accountability/blindness tension at the aggregator; and
the engineering precedent for a 128-bit-symbol erasure code adopted specifically so tags and code
share a field.

**Subsumes:** nothing. **Contradicts:** ADR-059's and Q68-3's shared premise that tagging
reconstructed content requires key-and-content co-location.

**Corrections applied this session:** ADR-059's open constraint on repair is discharged by ADR-078
(recorded in ADR-078, not by editing ADR-059, which stays `Proposed` on its own F-01 grounds).
ADR-076 §4's narrowing of Q68-3 is reversed in ADR-078's Context. Q68-3 moves to
`answered-questions.md`.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions **Q74-1**, **Q74-2**, **Q78-1**.
Question **Q68-3** is closed by ADR-078 and moves to [answered-questions.md](answered-questions.md).
