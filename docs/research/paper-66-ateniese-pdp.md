# Paper 66 — Provable Data Possession at Untrusted Stores

**Authors:** Giuseppe Ateniese, Randal Burns, Reza Curtmola, Joseph Herring (Johns Hopkins), Lea Kissner (Google), Zachary Peterson (JHU), Dawn Song (UC Berkeley / CMU)
**Venue / Year:** ACM CCS 2007, pp. 598–609 (extended version: IACR ePrint 2007/202)
**Citations:** ~5,000+ — the founding paper of the field
**Topics:** #2 Proof of Storage, #19 Adversarial Defences
**ADRs produced:** ADR-059 (proposed), ADR-060 (proposed); ADR-002 (supersession candidate), ADR-014 (correction), ADR-017 (schema revision), ADR-030 (correction)
**Findings addressed:** F-01, F-02, F-32, F-40
**Reading list:** Domain A / R-01 — the first of three canonical sources named in the entry

---

## Problem Solved

Vyomanaut's audit path asserts three properties that cannot hold simultaneously: the challenge nonce is fresh at challenge time, the response is a hash over the full chunk body, and the microservice never holds chunk bodies (F-01). `internal/audit/validate.go` states the consequence in a code comment — *"the microservice cannot verify that responseHash == SHA-256(chunkData || challengeNonce) because it never holds chunkData"* — and then calls it *"a stated design property, not a gap to close."*

This paper is the reason that framing is wrong. Ateniese et al. define **provable data possession**: a client that has deleted its local copy verifies that an untrusted server still holds a file, storing only O(1) metadata, transmitting O(1) per challenge, and having the server touch only a sampled subset of blocks. The primitive that makes it work is the **homomorphic verifiable tag** — a per-block authenticator that lets the server compress many block-tag pairs into one constant-size value the client can check against its keys alone. The client verifies without the data and without a precomputed answer, which is exactly the criterion the reading list names as non-negotiable for Domain A.

Two further contributions bear directly on F-02 and F-40. First, an explicit detection-probability formula relating sample size to corrupted fraction, which converts "audit everything daily" from a requirement into a parameter choice. Second, the measured finding that **PDP is bounded by disk I/O, not by cryptographic computation** — the same conclusion F-40 reached independently about the current design.

---

## Key Findings

### Homomorphic verifiable tags and blockless verification

Given block `m` at index `i`, the tag is `T(i,m) = (h(W_i) · g^m)^d mod N`, where `N = pq` is an RSA modulus, `d` is the client's private exponent, `g` generates the quadratic residues mod `N`, and `W_i = v || i` binds the tag to the index using a secret `v`. Two properties follow:

- **Blockless verification.** The server constructs a proof the client can check without holding any block.
- **Homomorphism.** `T(m_i)` and `T(m_j)` combine into a tag for `m_i + m_j`, so `c` tags compress into a single value.

The paper is explicit that aggregate signatures, multi-signatures, batch RSA and condensed RSA all **fail** to give blockless verification. The property is specific to this construction, not generic to signature aggregation.

### The two schemes, and why only one is safe

| | S-PDP | E-PDP |
| --- | --- | --- |
| Coefficients `a_j` | PRF-derived, fresh per challenge | all fixed to 1 |
| Server work | `c` exponentiations | 1 exponentiation |
| Guarantee | possesses **each** challenged block | possesses only **the sum** of challenged blocks |

E-PDP is the variant the authors implement and benchmark. Its weakness is not theoretical: Shacham & Waters (Paper 68, Appendix B) give a concrete attack in which a server stores `n−1` blocks instead of `n`, answers roughly `1/l` of all queries correctly, and no extraction procedure can recover any block. **The `a_j` coefficients are load-bearing.** Any Vyomanaut construction that omits per-challenge random coefficients to save arithmetic reproduces this hole.

### Detection probability as a function of sample size

For a file of `n` blocks with `t` deleted, challenging `c` distinct blocks:

```
1 − ((n − t)/n)^c  ≤  P_X  ≤  1 − ((n − c + 1 − t)/(n − c + 1))^c
```

Worked by the authors: at `t = 1% of n`, `c = 460` gives `P_X ≥ 99%` and `c = 300` gives `≥ 95%` — **independent of `n`**. Recomputed from the lower bound: `ln(0.01)/ln(0.99) = 458.2` and `ln(0.05)/ln(0.99) = 298.1`. The published figures are those values rounded up. The substitution checks out.

The constant-`c` property is the whole result. It is why a fixed audit budget can cover an arbitrarily large provider, and it is the direct answer to F-02.

### Cost, measured

- 4 GB file, `n = 1,000,000` 4 KB blocks, 1024-bit `N`: tags add **128 MB** (3.1% storage overhead); client stores **~3 KB** total.
- Challenge 168 bytes; response 148 bytes. Constant in file size.
- E-PDP verifies a 64 MB file in **0.4 s at 99% confidence** with sampling, versus 1.8 s without.
- Pre-processing is the bottleneck: **162 KB/s** tag generation (one exponentiation per block).
- Block size sweep (Fig. 6): the balance point sits at natural filesystem block sizes, **4–64 KB**; they choose 4 KB.
- *"the performance of PDP is bounded by disk I/O and not by cryptographic computation."*

### Index uniqueness across files (Remark 3)

Tag indices must be distinct not only within a file but **across all files** stored by the same client, or a tag can be replayed against a different block. The paper's suggested fix is prepending the file identifier: `TagBlock(pk, sk, m_i, id(F) || i)`.

---

## Substitution at Vyomanaut's parameters

### Tag storage at a 256 KB chunk

Ateniese's overhead is a fixed cost per block, so it is set by block size, not file size. At their 1024-bit modulus and 4 KB blocks the tag is 128 bytes per block = 3.1%. Vyomanaut's chunk is 256 KB, so:

| Block size | Blocks/chunk | Tag bytes/chunk (128 B tags) | Overhead |
| --- | --- | --- | --- |
| 4 KB | 64 | 8,192 | 3.125% |
| 16 KB | 16 | 2,048 | 0.781% |
| 64 KB | 4 | 512 | 0.195% |

Layered on RS(16,56)'s 3.5× that is 0.7–11% of an already-large expansion. Tolerable, but this is the RSA construction's number, not the number Vyomanaut should carry — Shacham & Waters do the same job at 16-byte authenticators over a 128-bit field (Paper 68), which is where ADR-059 lands.

### Detection at a chunk-sampling granularity

Applying the formula at Vyomanaut's actual unit — a provider holding 70 GB = **286,720 chunks** of 256 KB — sampling `c` whole chunks per day:

| Sample rate | `c` chunks/day | Bytes read | Disk time* | `t=0.1%` | `t=0.5%` | `t=1%` |
| --- | --- | --- | --- | --- | --- | --- |
| 0.1% | 286 | 0.075 GB | 0.1 min | 24.88% | 76.15% | 94.36% |
| 0.5% | 1,433 | 0.376 GB | 0.3 min | 76.16% | 99.92% | 99.9999% |
| **1%** | **2,867** | **0.752 GB** | **0.6 min** | **94.32%** | **99.9999%** | **~100%** |
| 2% | 5,734 | 1.503 GB | 1.2 min | 99.68% | ~100% | ~100% |
| 100% | 286,720 | 75.2 GB | **60.3 min** | 100% | 100% | 100% |

\* 10 ms seek + 256 KB at 100 MB/s per chunk.

The bottom row reproduces **F-40's ~1 h/day of HDD seeking exactly**, from an independent calculation. The 1% row is the same durability posture at **0.6 minutes** and 0.75 GB.

### SHELBY Condition (i) is far cheaper under sampling than the reading list assumed

The reading list states that at 1% sampling the required slashing-to-gain ratio becomes **99×**, from `(1 − p_au)/p_au`. That substitution treats `p_au` as a constant. It is not: under proportional sampling, detection probability grows with the amount cheated. Writing gain as `G(t) = t · V` (V = full value of the withheld storage) and detection as `P(t) = 1 − (1−t)^c`, the required slashing is

```
S  ≥  max_t  G(t) · (1 − P(t)) / P(t)
```

Evaluated numerically at `c = 2,867`, the maximum is `3.438 × 10⁻⁴ · V`, attained as `t → 0`, against the small-`t` limit `1/c = 3.488 × 10⁻⁴`. So:

```
S ≥ V / 2,909      (not 99 × G)
```

The 99× figure describes a scheme where cheating is all-or-nothing. Vyomanaut's is not — a provider that deletes more is caught more. **Sampling does not tighten the economic constraint here; it loosens it.** This should be re-derived against SHELBY's actual Theorem 1 statement before it is relied on, since the substitution above assumes gain is linear in the deleted fraction and that detection is evaluated over a single audit period. Recorded as Q66-2.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Probabilistic sampling | Deterministic full-file check | Constant challenge cost at any file size; a residual, quantified, non-zero miss probability that must be stated in provider-facing copy |
| Homomorphic tags at the server | MAC-per-block returned to the client | Response is constant-size instead of linear in blocks challenged; requires per-block pre-processing at 162 KB/s |
| RSA setting, arbitrary-size blocks | Prime-order / elliptic-curve group | Blocks need not be reduced mod a small prime; tags are ~1024 bits, far larger than an EC or field-element authenticator |
| S-PDP's per-challenge coefficients | E-PDP's coefficient-free aggregation | Possession of every challenged block instead of only their sum; `c` exponentiations instead of one |
| KEA1-r assumption for the short response | Sending the block sum as an integer | Constant response size; a non-standard assumption whose extractor, as Paper 68 notes, can never be implemented in the real world |
| Data-format independence | Requiring encrypted input | Works on the AONT-RS ciphertext Vyomanaut already stores, with no additional constraint |

---

## Breaks in Our Case

- **Ateniese's client is the verifier and holds `sk`** ≠ **Vyomanaut's verifier is the microservice, and the data owner is offline by design**
  → Authenticator keys must be generated client-side at upload and handed to the microservice. They are independent of the encryption key and reveal nothing about plaintext — the chunk is already AONT-RS ciphertext. This is the single most important structural adaptation and it is what makes the whole family usable here.

- **PDP proves possession, not retrievability** ≠ **Vyomanaut's product promise is that the file comes back**
  → The gap is closed by the *outer* code. RS(16,56) across 56 providers, not redundancy inside a shard, is what makes a shard's loss survivable. PDP supplies detection; the stripe supplies retrievability. State this explicitly rather than claiming PoR-strength for a per-provider primitive (see Paper 68 and Q66-1).

- **One file, one client, one server** ≠ **one chunk is one shard of one stripe of one file, held by one of 56 providers, with many chunks of the same file on the same provider**
  → Remark 3's index-uniqueness requirement becomes a real schema constraint. The tag index must be namespaced by `(file_id, chunk_ordinal, block_ordinal)` and never reused, including across repair, where a shard moves to a replacement provider. Repair re-tagging is R-03 and is not answered by this paper.

- **Synthetic vetting chunks are generated by the microservice** ≠ **the paper's client-tags-then-deletes model**
  → For synthetic chunks (ADR-030) the microservice *does* hold the data at generation time, so it can tag them itself before discarding. ADR-030's admission that it *"cannot verify the response hash directly (same as real chunks)"* is not forced — it is an artefact of the current primitive and disappears under this one.

- **1024-bit RSA and 162 KB/s tag generation** ≠ **a 5% CPU budget on a min-spec Indian desktop and client-side upload latency nobody has an NFR for**
  → Tag generation at 162 KB/s means a 4 MB stripe costs ~25 s of client CPU before upload. That is upload latency, on the data owner's machine, and it is unacceptable. It is also specific to RSA exponentiation-per-block; the PRF-based construction in Paper 68 replaces it with one PRF evaluation and `s` field multiplications per block. This break is the reason ADR-059 does not adopt Ateniese's construction directly.

- **E-PDP is the variant they benchmark** ≠ **the variant that is secure**
  → Every performance figure in §5.2 is E-PDP's. Do not carry E-PDP timings into an argument for a scheme with per-challenge coefficients; the coefficient-free version is the one Paper 68 breaks.

---

## Decisions Influenced

- **[ADR-059](../decisions/ADR-059-homomorphic-authenticator-audit.md) [#2 Proof of Storage]** `PROPOSED — SUPERSEDES ADR-002 on approval`
  Establishes the primitive class: per-block homomorphic authenticators, verified from keys alone, with fresh per-challenge coefficients. Ateniese supplies the concept and the security argument; the concrete parameters come from Paper 68.
  *Because:* this is the construction that makes the verifier's job possible at all, and F-01 is not closable without it.

- **[ADR-060](../decisions/ADR-060-sampled-chunk-audit-schedule.md) [#2 Proof of Storage]** `PROPOSED`
  Supplies the detection-probability formula the sampling rate is derived from, and the constant-`c` result that makes a fixed audit budget cover an arbitrary provider size.
  *Because:* F-02's ceiling is a row-count problem created by auditing every chunk daily; the formula shows that a 1% sample gives ~100% detection of a 1% deletion, at 1/100th the disk cost.

- **[ADR-014](../decisions/ADR-014-adversarial-defences.md) [#19 Adversarial Defences]** `CORRECTION REQUIRED`
  Defence 4's sentence *"Verified by the microservice which independently has the expected hash"* is false as written and must not survive into provider-facing copy. Correction text in the reply, not applied here.

- **[ADR-017](../decisions/ADR-017-audit-receipt-schema.md) [#2 Audit Receipt Schema]** `REVISION REQUIRED`
  `response_hash BYTEA(32) -- SHA256(chunk_data || challenge_nonce)` is not a checkable field. The receipt must carry a proof the verifier can evaluate.

- **[ADR-030](../decisions/ADR-030-synthetic-vetting-chunks.md) [#5 Peer Selection]** `CORRECTION AVAILABLE`
  The "microservice cannot verify" paragraph is removable for synthetic chunks specifically, since the microservice generates the data.

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]** `SUPERSESSION CANDIDATE`
  The audit-scalability paragraph's stated ceiling (100,000 providers × 10,000 chunks) both exceeds the stated Postgres limit and assumes a 2.5 GB provider. Both are corrected by ADR-060.

---

## Disagreements

- **Shacham & Waters (Asiacrypt 2008)**, Paper 68, on E-PDP: the coefficient-free variant guarantees only possession of the block *sum*, and they give an attack achieving ~9% success at standard parameters while storing `n−1` of `n` blocks. They also state plainly that Ateniese et al. do not prove extraction, only unforgeability, and that the KEA1-r extractor *"can never be implemented in the real world."*
  *Implication for us:* adopt the tag concept and the detection formula; do not adopt E-PDP, and do not treat Ateniese's Appendix A as an extraction proof.

- **Against ADR-002's own title.** ADR-002 is titled *"PoR Merkle Challenge"* and its consequences paragraph reasons about Merkle path verification, but neither the ADR nor the implementation builds a Merkle tree. A real Merkle scheme is a coherent alternative to the homomorphic one and is priced in ADR-059's Options table — but it is not what is currently specified, and it is not what ADR-014 describes either. F-01 stands until the council rules.

- **Against the "stated design property" comment in `internal/audit/validate.go`.** The inability to verify is not a property of proof-of-storage; it is a property of choosing a construction that predates the field's founding paper by nothing more than not having read it. The comment should be deleted with the code it documents.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q66-1 and Q66-2.
