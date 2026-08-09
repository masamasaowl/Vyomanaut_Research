# ADR-059 — Verify audits with homomorphic authenticators

**Status:** Proposed — blocked on the F-01 design-council ruling
**Topic:** #2 Proof of Storage
**Supersedes:** ADR-002 (on approval)
**Superseded by:** —
**Research source:** Paper 66 (Ateniese, CCS 2007), Paper 67 (Juels & Kaliski, CCS 2007), Paper 68 (Shacham & Waters, Asiacrypt 2008), Paper 69 (Erway et al., TISSEC 2015 — evidence), Paper 70 (Armknecht et al., TCC 2021 — evidence, directly bears on F-22)

---

## Context

Vyomanaut's audit path currently specifies a response of `SHA-256(chunk_data || challenge_nonce)` against a nonce generated fresh at challenge time, verified by a microservice that never holds chunk data. Those three properties are jointly unsatisfiable (F-01). ADR-014 Defence 4 claims the microservice *"independently has the expected hash"*; ADR-030 states the opposite in plain text; `internal/audit/validate.go` documents the gap as *"a stated design property, not a gap to close."*

The failure mode this decision prevents is total and silent: **a provider can delete every chunk it holds, answer every challenge with 32 random bytes correctly signed, pass forever, and be paid.** Everything downstream — reliability scoring (ADR-008), payment release (ADR-012), repair triggering (ADR-004), the receipt log (ADR-015, ADR-017) — is gated on a check that cannot currently fail.

The corpus contained four deployed storage networks and zero proof-of-storage primitive papers. Papers 66–68 close that gap. Shacham & Waters' private-verification scheme meets all four properties Vyomanaut requires simultaneously: the verifier holds neither the data nor a precomputed answer; challenges are unbounded; the verifier is stateless; and the response is constant-size.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Status quo — `SHA-256(chunk_data ‖ nonce)`** | Already built; 32-byte responses; trivial provider cost | Unverifiable. Provides no guarantee of any kind. This is F-01 |
| **Merkle tree per chunk, root stored at upload** | Verifiable from a 32-byte root; no field arithmetic; simple to reason about | Response is `l` leaves plus `l·log₂(leaves)` authentication paths — at 256 leaves and 256 challenged, ~40 KB per chunk versus 1,040 bytes for the whole audit; no aggregation across chunks, so one receipt per chunk and F-02's row ceiling is untouched. Paper 69's rank-based authenticated skip list is the dynamic-update-capable version of this option — it adds authenticated insert/modify/delete in `O(log n)` and would let a reconstructed shard be re-tagged with a checkable proof of the update itself — but does not remove the per-chunk response-size problem, and its standard-model security guarantee requires transmitting the actual challenged indices, which is worse than this ADR's coefficient-vector bandwidth problem at Vyomanaut's audit volume (see Paper 69) |
| **Ateniese S-PDP (RSA homomorphic tags)** | Founding construction; publicly verifiable variant; data-format independent | Tag generation measured at 162 KB/s — ~25 s of client CPU per 4 MB stripe before upload; 128-byte tags; relies on the non-standard KEA1-r assumption whose extractor, per Paper 68, can never be implemented |
| **Juels–Kaliski sentinels** | Cheapest verification of any scheme — a byte comparison; smallest file expansion | Bounded `q` audits fixed at encoding time against ~1.05 M chunk-challenges per provider per year; requires a stateful verifier, which ADR-025's three-replica gossip cluster is not |
| **Shacham–Waters public verification (BLS)** | 20-byte query, 40-byte response — smallest of any PoR; third parties can audit, which answers F-22's recourse gap; already ROM-based, so seed-derived queries cost nothing | Pairing operations on every verification; the reading list's own R-04 reject-if flags this against the 5% CPU budget; larger authenticators |
| **Shacham–Waters private verification (PRF) — chosen** | Verifier needs only keys; unbounded challenges; stateless verifier; constant-size aggregate response across arbitrarily many chunks; no public-key arithmetic anywhere; full security proof against arbitrary adversaries | The verifier holds a symmetric secret and can therefore forge a proof (F-22); standard-model proof does not survive seed-derived queries; introduces field arithmetic and a new per-file key table |

## Decision

Replace the hash-challenge primitive with **Shacham & Waters' private-verification homomorphic authenticator scheme**, at the following parameters. Every value below is frozen at first authenticator generation and cannot be changed without re-tagging every stored chunk.

### Field and layout

```
p               = 2^128 + 51 = 340282366920938463463374607431768211507   (prime, verified)
sector          = 16 bytes, interpreted as a big-endian element of Z_p
s               = 64 sectors per block
block           = 1,024 bytes
blocks/chunk    = 256                     (256 KB chunk / 1,024 B)
```

`p` exceeds Paper 68's `λ = 80` recommendation deliberately, to obtain a byte-aligned 16-byte sector. Every 16-byte sector is `< p` by construction, so no rejection sampling or padding is required anywhere in the codec.

### Authenticators

Generated **client-side, at upload, after AONT-RS encoding**, over ciphertext bytes. For block `i` of a file, with sectors `m_i1 … m_i64`:

```
σ_i  =  f_kprf(i)  +  Σ_{j=1..64} α_j · m_ij   (mod p)
```

- `i` is the file-global block index: `i = (chunk_ordinal × 256) + block_ordinal`, unique within a file and **never reused**, including after repair (Paper 66 Remark 3; Paper 68 §3.1).
- `f` is HMAC-SHA-256 keyed with `k_prf`, output reduced mod `p`, domain-separated with the tag `"vyomanaut/por/prf/v1"`.
- Authenticators are stored alongside the chunk on the provider: **4,096 bytes per 256 KB chunk = 1.5625% overhead**, transmitted in the same chunk-upload stream and covered by the chunk's existing `content_hash`.

### Key material

Per file, generated client-side and uploaded to the microservice with the file manifest:

```
k_prf     32 bytes
α_1..α_64 64 × 16 = 1,024 bytes
```

These keys are independent of the AONT-RS encryption key and reveal nothing about plaintext — the tagged bytes are already ciphertext. Handing them to the microservice does not weaken ADR-019's zero-knowledge property. At 10⁶ files the table is ~1.06 GB.

For **synthetic vetting chunks** (ADR-030) the microservice generates the chunk data itself, so it generates the authenticators itself before discarding the data. ADR-030's admission that it cannot verify synthetic-chunk responses is therefore removed, not worked around.

### Challenge

A challenge is a **fresh 32-byte seed** and nothing else. Both parties derive from it, by SHA-256 in counter mode with domain separation:

1. the sampled chunk set (ADR-060 supplies the sampling rule and rate);
2. a coefficient `ν_i ∈ Z_p` for every block `i` of every sampled chunk.

Every block of a sampled chunk is challenged, so the prover's read is one contiguous 256 KB vLog record per sampled chunk.

Seed-derived queries are a **stated deviation** from Paper 68's private-verification scheme, which is proven in the standard model only when the query is transmitted. Transmission is not available to us: at ADR-060's rate the coefficient vector is 11.7 MB per provider per day. The deviation places the scheme in the random-oracle model. See Open constraints.

### Response

```
μ_j  =  Σ_{(i,ν_i)} ν_i · m_ij   (mod p),  j = 1..64
σ    =  Σ_{(i,ν_i)} ν_i · σ_i    (mod p)
```

Wire size: `65 × 16 = 1,040 bytes`, **constant regardless of how many chunks the challenge sampled**, plus the existing 64-byte Ed25519 provider signature.

### Verification

```
accept  iff  σ  ==  Σ_{(i,ν_i)} ν_i · f_kprf(i)  +  Σ_{j=1..64} α_j · μ_j   (mod p)
```

Computable from `(k_prf, α₁…α₆₄)` alone. The microservice never needs a byte of chunk data, and there is no state carried between audits — any cluster replica holding the file keys can issue and verify independently.

### Prohibited

The coefficients `ν_i` **must not** be set to 1 or omitted, at any parameter setting, for any efficiency reason. Paper 68 Appendix B gives a concrete attack: an adversary storing `n−1` of `n` blocks answers ~9% of coefficient-free queries correctly, and its state is information-theoretically consistent with any value for any block, so no extraction can recover anything. This is the same weakness that makes Ateniese's E-PDP unsafe. Add a compiler-visible guard analogous to `TestProfileShardSizeIsConstant`.

## Consequences

**Positive:**

- The audit verifies something for the first time. F-01 and F-32 close; ADR-014 Defence 4 becomes true rather than aspirational.
- Response size is constant in the number of chunks audited, which is what makes ADR-060's one-receipt-per-`(provider, file, day)` aggregation possible and is the mechanism behind F-02's fix.
- No new coordinated operation. The verifier is stateless, so the audit path adds nothing to ADR-013's six non-I-confluent operations and stays clear of F-35 and F-76.
- Zero-knowledge is preserved. Authenticator keys are independent of the encryption key and are computed over AONT-RS ciphertext.
- Challenge freshness is stronger than the current design, not weaker: the coefficient vector is fresh per audit, so no forward caching is possible. ADR-002's rejection of Storj v2's pre-generated challenges is honoured.
- Provider cost stays disk-bound. ~47 M modular multiplications per provider-day is sub-second against 0.6 minutes of disk time, confirming Paper 66's measured finding and F-40's independent one.

**Negative / trade-offs:**

- 1.5625% additional storage on every stored byte, on top of RS(16,56)'s 3.5×.
- The microservice holds a symmetric secret per file and can forge a passing proof, or fabricate a failure. This is F-22 restated and it is not closed here. The BLS variant closes it at the cost of pairings.
- Seed-derived queries put the scheme in the random-oracle model, losing the standard-model proof that is the private scheme's distinguishing advantage.
- A new per-file key table whose loss makes a file permanently unauditable. It needs the same backup posture as the payment ledger.
- Client-side upload cost rises: one HMAC and 64 modular multiplications per 1,024-byte block, ~16,384 multiplications per chunk. Unmeasured, and it lands on the data owner's machine in the upload path, which has no NFR (same gap Q65-1 names for RS encode).
- A new microservice CPU cost: ~734 k PRF evaluations per provider-audit at ADR-060's rate. Unbudgeted.
- Every wire format, receipt column, test fixture and stored authenticator depends on `(p, s)`. Changing either later means re-tagging the network.

**Open constraints:**

- **The F-01 council ruling has not happened.** This ADR stays `Proposed` until it does. The competing coherent option is a Merkle scheme, which is why it is priced in the Options table rather than dismissed.
- **Repair breaks tagging, and nothing here fixes it.** A shard that *migrates* intact keeps valid authenticators — tags bind to block content and index, not provider identity. A shard that is *reconstructed* is new bytes at the same stripe position with no valid authenticators, and only the owner holds the keys to generate them, and the owner is offline by design (ADR-004). This is R-03 and it is unsolved in all three papers. **A reconstructed shard is unauditable until this is answered.** Q68-3. Paper 69 (Erway et al., the canonical R-03 source) formalises the *update-verification* half of this problem — a claimed new block can be checked in `O(log n)` as the unique valid successor of the prior committed state — but does not answer *who is permitted to compute the new tag*, which is the actual blocker: giving the repairing provider the keys lets it forge future proofs for that position, and having the microservice see the reconstructed bytes to tag them itself contradicts ADR-021's pure-P2P repair model (*"no central entity fetches and re-encodes on behalf of others"*) and ADR-004's repair flow (surviving fragment holders reconstruct and push directly to the new provider, not via the microservice). The three shapes of answer in Q68-3 stand unchanged; Paper 69 only supplies a way to make whichever one is chosen provable after the fact rather than an unverified claim.

- **The random-oracle deviation must either be accepted explicitly by the council or resolved by moving to the BLS variant. Q68-1.** Paper 70 (Armknecht et al.) adds a third framing the council should see before ruling: **Fortress**, built directly on this ADR's already-chosen private PRF-based scheme, supplies a formal liability mechanism — the auditor cryptographically cannot lie about its own tag-generation parameters — at ~10% overhead over plain PSW, versus BLS's pairing cost on every audit. This does not remove the need for *some* trade-off (Fortress needs an unpredictable, publicly-reconstructible randomness source Vyomanaut does not yet have — see ADR-015 and Domain G's R-23), but it reframes Q68-1 from "accept the deviation or pay for pairings" to a three-way choice that also directly closes F-22, which neither of the original two options does on its own. Paper 72 (Zeng et al.) adds a further, independent point: this ADR's private-scheme choice already carries meaningfully less long-term quantum risk than the BLS alternative, since HMAC-SHA-256 and field arithmetic degrade gracefully under Grover's algorithm where pairings are fully broken by Shor's — a consideration absent from the original Q68-1 framing.
- Per-provider guarantee is possession, not retrievability. Retrievability is a property of the RS(16,56) stripe (Paper 67's outer code). ADR-002's successor must say this rather than let "PoR" carry a claim the per-provider primitive cannot support. Q66-1.
- `(p, s) = (2^128 + 51, 64)` must be frozen before the first authenticator is generated in any environment that will outlive a database reset.

## References

- [Paper 66 — Ateniese, Provable Data Possession](../research/paper-66-ateniese-pdp.md): homomorphic verifiable tags and blockless verification; index uniqueness (Remark 3); the disk-bound finding
- [Paper 67 — Juels & Kaliski, PORs](../research/paper-67-juels-kaliski-por.md): retrievability as the security goal; the bounded-`q` and stateful-verifier limitations that eliminate the sentinel family; the inner/outer code framework
- [Paper 68 — Shacham & Waters, Compact PoR](../research/paper-68-shacham-waters-compact-por.md): the adopted construction; the `s` tradeoff; Appendix B's coefficient attack; the open problem of short standard-model queries
- [Paper 69 — Erway, Küpçü, Papamanthou & Tamassia, Dynamic Provable Data Possession](../research/paper-69-erway-dpdp.md): authenticated update proofs; sharpens but does not close Q68-3; the standard-model property is unusable at Vyomanaut's audit volume
- [Paper 70 — Armknecht, Bohli, Karame & Li, Outsourcing Proofs of Retrievability](../research/paper-70-armknecht-opor.md): the Fortress construction; a costed liability mechanism for the primitive already chosen; closes F-22 conditional on a `GetRandomness`-equivalent
- [Paper 71 — Li, Chu & Hu, CIA](../research/paper-71-li-chu-hu-cia.md): considered and not recommended — assumes full replicas, which Vyomanaut does not have; corroborates the private-scheme choice by contrast, not directly relevant otherwise
- [Paper 72 — Zeng et al., PQ-Audit](../research/paper-72-zeng-pq-audit.md): not adoptable as specified (breaks the constant-response-size property); adds a quantum-resilience data point in the private-scheme's favour for Q68-1
- [ADR-002 — Proof of Storage](ADR-002-proof-of-storage.md): superseded on approval
- [ADR-060 — Sampled chunk audit schedule](ADR-060-sampled-chunk-audit-schedule.md): the sampling rule this challenge format assumes
- [ADR-014 — Adversarial Defences](ADR-014-adversarial-defences.md): Defence 4 requires correction
- [ADR-017 — Audit Receipt Schema](ADR-017-audit-receipt-schema.md): requires revision
- [ADR-030 — Synthetic Vetting Chunks](ADR-030-synthetic-vetting-chunks.md): the unverifiable-synthetic-chunk paragraph is removed
- [ADR-025 — Microservice Consistency](ADR-025-microservice-consistency-mechanism.md): stateless verification adds no coordinated operation
- [ADR-021 — P2P Transfer Protocol](ADR-021-p2p-transfer-protocol.md): the pure-P2P repair constraint that rules out the microservice tagging reconstructed shards directly
- [ADR-004 — Repair Protocol](ADR-004-repair-protocol.md): repair is performed by surviving fragment holders pushing to the new provider, not by the microservice — the party gap Q68-3 is actually about
- [ADR-015 — Audit Trail](ADR-015-audit-trail.md): candidate substrate for Paper 70's `GetRandomness`, contingent on the gossip mechanism Domain G's R-23 has not yet supplied
