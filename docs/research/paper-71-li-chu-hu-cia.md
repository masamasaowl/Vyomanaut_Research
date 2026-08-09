# Paper 71 — CIA: A Collaborative Integrity Auditing Scheme for Cloud Data With Multi-Replica on Multi-Cloud Storage Providers

**Authors:** Tengfei Li, Jianfeng Chu, Liang Hu (Jilin University)
**Venue / Year:** IEEE Transactions on Parallel and Distributed Systems, Vol. 34, No. 1, January 2023
**Citations:** early-stage, post-2023
**Topics:** #2 Proof of Storage, #1 Coordination
**ADRs produced:** none — considered and not recommended for adoption
**Findings addressed:** none closed; corroborates ADR-002's existing centralised-auditor argument by contrast
**Reading list:** Domain A / adjacent to R-04 and R-05; does not satisfy either domain's accept criteria cleanly — see below

---

## Problem Solved

CIA addresses a real, but different, problem than Vyomanaut has. In a multi-replica, multi-cloud-provider deployment — several CSPs each holding a **full, identical copy** of the same file — Li, Chu & Hu remove both the third-party auditor *and* the expensive homomorphic-tag generation step by having the CSPs audit **each other**. Each CSP computes a weighted sum of its own raw data bytes over a jointly-negotiated, unpredictable challenge; if two CSPs hold identical data, their proofs must be byte-identical, so verification is a plain comparison, and the whole scheme uses **only hash functions** — no modular exponentiation, no pairings, no per-block tag storage. The paper's stated motivation is the two costs every RSA/BLS-tag scheme (including Papers 66–68) carries: expensive tag generation at upload, and an artificially small block size (traditionally 160 bits) driven by the size of the algebraic group the tag lives in.

This is genuinely useful evidence on tag-generation cost, and it is the reading list's most literal answer to the phrase "audit with only hash functions." It is documented here for that reason. It is not, on inspection, adoptable for Vyomanaut's actual storage model, for reasons detailed below.

---

## Key Findings

### The mechanism: 2PC secret negotiation, then a hash-committed weighted sum

Setup: the data owner hashes each block once (`h_i = H(b_i)`), keeps the hashes for arbitration, and sends one replica to each of `m` CSPs. An audit begins with `SecNego`, a two-phase-commit among the `m` CSPs holding replicas of the same file: each CSP commits to a random secret via its hash (`SecPrepare`), then reveals it (`SecSubmit`), and every CSP checks every other CSP's reveal against its earlier commitment. The commit-then-reveal structure — provably, under only pre-image and second-preimage resistance — prevents any single CSP from biasing the aggregated challenge after seeing others' contributions. The aggregated secret derives a challenge set via a PRF/PRP pair (`ChalGen`), each CSP computes `P = Σ b_{μj} · ν_j` directly over its own stored bytes (`ProofGen`), and proofs are exchanged via a second 2PC round (`ProofDistr`) to prevent a lazy CSP from simply copying another's already-computed proof. `ProofVerify` is equality comparison. `Arbitrate` — the data owner stepping in — triggers only on detected disagreement, using the locally-held per-block hashes from setup.

### No tag generation, and free block sizing

Because the "tag" is just the raw data itself, weighted and summed, there is no upload-time exponentiation step at all — the paper's own measurements show competing pairing-based multi-replica schemes (MuR-DPR, MDSS) costing 70–250 seconds of client CPU for tag generation on a modest dataset, against **zero** for CIA. Block size is unconstrained by any algebraic group order — the paper explicitly criticises the traditional 160-bit block ceiling this creates in RSA/BLS-tag schemes (which Paper 66's Ateniese scheme also carries) and demonstrates variable, application-chosen block sizes up to megabytes.

### Measured cost

At 1,000 challenged blocks and 5–10 replicas, CIA's combined `ProofGen`+`ProofVerify` cost is consistently under 0.03 seconds versus 3–30 seconds for the pairing-based comparators — described by the authors as "negligible," which the numbers support. The cost that does **not** shrink is communication: `SecNego` and `ProofDistr` are both all-to-all 2PC rounds among the `m` replica-holding CSPs, so total network traffic scales as `O(m² · (|hash| + |secret|))`. At `m = 10` replicas the authors measure ~37 Kbit total system-wide, ~3.7 Kbit per CSP — small in absolute terms, but quadratic, not constant, in the number of parties.

### Security argument, and its scope

Theorem 1 and Lemma 1 prove that (a) no CSP can bias the negotiated challenge into a favourable subspace, and (b) a CSP that has not preserved the intact data has only negligible probability of producing a matching proof, both reducing to hash pre-image/second-pre-image resistance. **Both proofs are stated against a single malicious CSP acting alone**, or a malicious CSP attempting to exploit the protocol's mechanics. Neither proof addresses the case where **all `m` CSPs collude simultaneously** to agree on a shared false answer — in that scenario there is no disagreement to trigger `Arbitrate`, and the scheme's guarantee is silent.

---

## Substitution at Vyomanaut's parameters — why this does not transfer

### The core assumption does not hold: Vyomanaut has no full replicas

CIA's entire verification mechanism — comparing `P_k` across CSPs for equality — depends on multiple parties holding **byte-identical** copies of the same data. Vyomanaut's durability strategy is RS(16,56) erasure coding (ADR-003): 56 providers each hold a **distinct, unique shard**, and no two providers' stored bytes are ever the same. There is no pair of parties in Vyomanaut's storage model for whom `ProofVerify`'s equality check is even meaningful. Porting CIA would require abandoning erasure coding for full replication — a far more expensive durability strategy that ADR-003 already evaluated and rejected on storage-overhead grounds. This is a structural incompatibility, not a parameter-tuning gap.

### The topology CIA depends on is the one Vyomanaut's own corpus already rejected

CIA removes the trusted third-party auditor specifically so that peer CSPs can audit each other. Paper 37 (Shelby, already in Vyomanaut's corpus and cited directly in ADR-002) formally proves that peer-to-peer provider auditing **without a trusted backstop collapses to universal dishonesty as the unique Nash equilibrium** — which is the stated reason ADR-002 centralises the auditor in the microservice in the first place. CIA's `Arbitrate` step is a backstop, but a conditional one: it activates only when CSPs *disagree*, and CIA's own security proofs do not model the case where every CSP is simultaneously dishonest and therefore never disagrees. That is precisely the collusion equilibrium Paper 37 warns about. Adopting CIA's topology would be a direct reversal of an already-evidenced architectural decision, not a neutral addition.

### What does carry over, at the idea level, not the mechanism level

CIA's central efficiency lesson — homomorphic tags carry a real, avoidable upload-time cost, and a scheme that avoids them can be dramatically cheaper — is already reflected in ADR-059's own choice. ADR-059 selected the PRF-based Shacham–Waters **private** scheme specifically because it needed no public-key arithmetic anywhere, for the same reason CIA gives. CIA does not add new information to that decision; it corroborates it from a different direction, using a different (and here, inapplicable) storage model.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Hash-only proofs over raw replica bytes | Homomorphic algebraic tags (RSA/BLS) | Zero tag-generation cost, unconstrained block size; only meaningful when two parties hold identical data |
| Peer CSPs audit each other, DO arbitrates on disagreement | A trusted third-party auditor | Removes one role from the trust model; reintroduces the collusion equilibrium a trusted auditor exists to avoid |
| 2PC commit-then-reveal for challenge negotiation | A single party unilaterally choosing the challenge | Provably unbiasable challenge generation among honest-but-competing peers; cost is `O(m²)` communication |

---

## Breaks in Our Case

- **`m` CSPs each hold a full, identical replica of the file** ≠ **56 Vyomanaut providers each hold a unique, non-overlapping erasure-coded shard**
  → `ProofVerify`'s equality comparison has no counterpart to compare against. This alone rules the mechanism out without a change to Vyomanaut's durability strategy that no other finding recommends.

- **Peer CSPs are commercial giants (Google, Amazon) with reputational stakes in not colluding** ≠ **Vyomanaut's providers are consumer desktops, individually low-stakes, exactly the population Paper 37's Nash-equilibrium argument is about**
  → The paper's own informal justification for trusting peer auditing — *"once the arbitration result is drawn, the falsifying CSPs' reputation will be seriously damaged"* — does not apply to anonymous, low-reputation, easily-replaced consumer providers. This is a population mismatch on top of the structural one.

- **All-to-all 2PC among `m` parties, `m` in the single digits in every measurement** ≠ **Vyomanaut's stripe width is 56, and a stripe-wide collaborative negotiation (if it were otherwise applicable) would be two orders of magnitude past anything measured here**
  → Even setting the replica-vs-shard mismatch aside, CIA's own quadratic communication scaling was never evaluated near Vyomanaut's actual fan-out.

---

## Decisions Influenced

- **No ADR produced.** CIA's core mechanism is not applicable to Vyomanaut's erasure-coded storage model, and its trust topology is one Paper 37/ADR-002 already evidenced against. This paper closes no open question and revises no decision.

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]** `STRENGTHENED — no change`
  By contrast, corroborates the existing centralised-auditor rationale: a concrete, recent (2023) instance of the peer-collaborative alternative, examined directly, does not close the collusion gap Paper 37 already identified, and does not apply to Vyomanaut's storage model regardless.
  *Because:* recording a considered-and-declined alternative is worth as much as recording an adopted one — it forecloses re-litigating this path without a change to the erasure-coding decision first.

---

## Disagreements

- **Against the paper's own framing that removing the TPA is a straightforward improvement.** Section 1.1's claim that peer CSPs auditing each other is safe because they are "independent and in competition" does not engage with the game-theoretic collusion argument Paper 37 makes formally and which is already load-bearing in Vyomanaut's own ADR-002. The paper is not wrong for its intended deployment (major commercial CSPs with public reputations); it is simply answering a different trust question than Vyomanaut has.

- **Against treating CIA's efficiency numbers as a reason to reconsider ADR-059's primitive.** CIA's near-zero computational cost looks dramatic next to RSA/BLS tag schemes, but ADR-059 already avoided that cost by choosing the PRF-based private scheme over exactly those alternatives, for the same underlying reason. CIA is not a cheaper option relative to what Vyomanaut already chose; it looks cheaper only relative to constructions Vyomanaut already rejected.

---

## Open Questions

No new open questions raised. This paper does not bear on any currently open Domain A question; it forecloses re-opening the peer-collaborative-auditing alternative without new evidence.
