# Paper 67 — PORs: Proofs of Retrievability for Large Files

**Authors:** Ari Juels (RSA Laboratories), Burton S. Kaliski Jr. (RSA Laboratories)
**Venue / Year:** ACM CCS 2007, pp. 584–597 (full version: Cryptology ePrint 2007/243)
**Citations:** ~4,000+ — the paper that names the primitive
**Topics:** #2 Proof of Storage, #3 Erasure Coding
**ADRs produced:** none directly; contributes the outer-code argument to ADR-059 and the `ε`-adversary framing to ADR-060
**Findings addressed:** F-01, F-02, F-32
**Reading list:** Domain A / R-01 — second of the three canonical sources

---

## Source provenance — read this before citing

**The PDF supplied for this slot is not Juels & Kaliski.** The file named `Juels_Kaliski_proofs_of_retrievability_large_files.pdf` contains **Bowers, Juels & Oprea, *Proofs of Retrievability: Theory and Implementation*, ACM CCSW 2009** — a different paper, by an overlapping author set, three venues and two years later. This is flagged rather than silently resolved, per project convention.

Consequently this note is assembled from three sources, and each claim below is tagged:

- `[BJO]` — stated in the supplied Bowers–Juels–Oprea paper, which describes the JK scheme in §1.1, §2, §3.3 and §4.2 in order to improve on it.
- `[SW]` — stated in Shacham & Waters (Paper 68), whose §1 and §2 recapitulate the JK model and whose Definition 2.1 is explicitly a variant of it.
- `[UNVERIFIED]` — from general knowledge of the primary paper and **not** checkable against any document in hand.

**No `[UNVERIFIED]` claim may be carried into an ADR, an NFR, or a parameter derivation until the CCS 2007 primary source is obtained.** The primary is worth obtaining: JK is the source of the `(ρ, δ)`-style detection bound that F-02 names as the fix, and the sentinel construction's exact query-budget arithmetic is not reproduced in either substitute source.

BJO itself is a strong candidate for its own research note — it supplies the inner/outer code framework that ADR-059's design rests on, and its Table 2 is a direct storage-and-communication comparison against Shacham–Waters. Proposed as **Paper 69**, pending your approval; it is not drafted here because you asked for the three core papers first.

---

## Problem Solved

Ateniese (Paper 66) proves the server *possesses* a sampled subset of blocks. That is not the property a data owner wants. Vyomanaut's promise to a data owner is that the file comes back, and possession of 99% of a compressed archive returns nothing.

Juels & Kaliski define **proof of retrievability**: a protocol after which the verifier is convinced that the *whole* file can be recovered, formalised by requiring that an extractor interacting with any prover which passes audits at a non-negligible rate can reconstruct the file `[SW]`. The insight the definition rests on is simple and is the one Vyomanaut needs: **checking that most of a file is stored is far easier than checking that all of it is — so encode the file redundantly first, and then "most" becomes "all"** `[SW]`.

For Vyomanaut this reframes the audit path. The current design treats the audit as the thing that guarantees retrievability. It cannot. The audit can only bound the corruption rate; retrievability comes from the code layered underneath it. Vyomanaut already has that code — RS(16,56) — but it sits at the wrong layer relative to where the audit operates, and nothing in the ADR set says so.

---

## Key Findings

### The construction: sentinels

The client applies an error-correcting code to the file, encrypts it, and embeds **sentinels** — pseudorandomly generated check values, indistinguishable from encrypted file blocks — at pseudorandomly chosen positions `[BJO]`. A challenge names a set of sentinel positions; the server returns those values; the client compares against what it can regenerate from its key. A server that has deleted a fraction of the file will, with high probability, have deleted sentinels too.

Two properties follow, and both matter here:

- **Sentinels are single-use.** Once a sentinel's position has been revealed in a challenge, the server knows it and can retain it selectively. Each sentinel supports exactly one challenge, so the scheme supports a **bounded number `q` of audits, fixed at encoding time** `[BJO]`, `[SW]`.
- **Sentinel indistinguishability requires an encrypted file.** The scheme *"can only be applied to encrypted files"* `[Paper 66, §3, describing JK]`.

BJO describes the alternative JK mechanism as well: append a collection of `q` MACs over subsets of blocks, instead of generating sentinel values from a PRF `[BJO §2]`. Same bounded-`q` limitation.

### The security model, and its two restrictions

JK's model is the one every later paper is measured against. `[SW]` gives its shape precisely: a secure scheme is one where, if a server can pass an audit, a special **extractor** algorithm interacting with that server can recover the file with high probability. `[SW]` states two things it strengthened when adapting it, both of which are Vyomanaut-relevant:

- JK's model permits **stateful verifiers**. `[SW]` argues verifiers should be stateless, because state is lost on verifier crash and cannot be delegated or distributed. Vyomanaut's verifier is a three-replica microservice cluster with eventually-consistent gossip membership (ADR-025, ADR-048) — a stateful audit counter shared across replicas is precisely the class of non-mergeable operation F-35 and F-76 already flag.
- `[SW]` names the sentinel scheme as lacking **both unbounded use and statelessness**, and declines to consider it further.

BJO adds a third restriction: JK's analysis includes a **block isolation assumption** — that the probabilities of file blocks being returned correctly are independent of one another `[BJO §5.1]`. BJO's own variant removes it.

### The `ε`-adversary framing

`[BJO §3.2]` formalises the adversary JK's analysis assumes: an **`ε`-adversary** answers correctly on at least a `1 − ε` fraction of the challenge space, with `ε` the maximum fraction of corrupted challenges at any query. The audit protocol then splits into two phases: **Phase I** bounds `ε` by challenge-response spot checks; **Phase II** extracts, relying on that bound.

`[BJO]` gives the Phase I detection bound directly: with challenges drawn uniformly, the probability that an adversary passes `q_c` challenges while not in fact being an `ε`-adversary is bounded by

```
λ  <  (1 − ε)^(q_c)
```

That is the `(fraction corrupted, detection probability)` relation R-02 asks for, in its cleanest form. It is the same relation as Ateniese's `P_X`, expressed from the adversary's side.

`[BJO §3.3.1, Remark]` also makes the temporal point that matters for a system auditing for years rather than once: a server may be honest and *then* turn adversarial, so challenges should be spread over time. Their worked example — Phase I tuned for `q_c = 50` detects the condition with probability at least `1 − λ` within 50 days of the server turning bad, at one challenge per day — is the exact shape of Vyomanaut's daily audit.

### Where JK sits against SW on cost

`[BJO §5.2]`, comparing at a 4 GB file, `10⁻⁶` security level, rate-0.9 outer code, `q = 10,000` precomputed challenges:

- JK-variant server storage overhead: `320 KB + 0.1 · |F|`; challenge `8 + κ` bytes; response `z` bytes (32 in their implementation).
- Matching SW on storage forces `ρ = 0.1`, `l = 169`, `s = 13,107` — SW's challenge becomes **464 bytes** and its response **36 KB**.
- Matching SW on communication instead forces `l = 14`, `s = 10`, `1/ρ = 2.96` — SW's server storage becomes **about twice the file size**.

`[BJO]`'s conclusion: for small `ε`, JK is cheaper on both storage and communication, *but* SW retains unbounded verifications and extraction for any `ε < 1`. Their measured encoding throughput for the JK variant is **~3 MB/s** in Java, with the outer error-correcting layer accounting for **61–67%** of it.

### `[UNVERIFIED]` — not checkable from the sources in hand

- The exact sentinel count and per-challenge sentinel batch size in the CCS 2007 parameterisation.
- JK's own numeric worked example of the `(ρ, δ)` bound.
- The precise permutation and encryption ordering in JK's `encode`, as distinct from BJO's SA-ECC re-implementation of it.

---

## Substitution at Vyomanaut's parameters

### The bounded-`q` limitation is disqualifying, and the arithmetic says so plainly

Vyomanaut audits continuously for the life of a stored file. Take the sampling rate ADR-060 proposes — 1% of a provider's chunks per day — and a provider holding 70 GB:

```
chunks           = 70 GB / 256 KB              = 286,720
sampled per day  = 1% × 286,720                =   2,867
per year         = 2,867 × 365                 = 1,046,455 chunk-challenges
```

A sentinel scheme must pre-embed one sentinel per future challenge, at encoding time, before it knows how long the file will be stored. There is no value of `q` that is both finite and sufficient for a storage product with no stated retention limit. **JK's construction is rejected for Vyomanaut on this ground alone**, which is the same ground `[SW]` rejects it on, and it is a structural rejection rather than a parameter one.

### What survives the rejection, and it is the important half

The *framework* survives entirely, and it is what ADR-059's security argument is built on.

`[BJO §3.3]` names two levels of error correction: an **outer code** applied to the file at encoding, static over its lifetime, and an **inner code** computed on the fly by the server in answering a challenge. Vyomanaut already has both — it has simply never named them:

| BJO layer | Vyomanaut's instantiation | Where it lives |
| --- | --- | --- |
| Outer code `ECC_out` | RS(16,56) across 56 providers, `k = 16`, tolerating 40 lost shards | ADR-003 |
| Inner code / `respond` | the audit response computed over challenged blocks | ADR-002, to be replaced by ADR-059 |

This mapping resolves the retrievability question the primitive papers leave open for us. **Vyomanaut's outer code is inter-provider, not intra-shard.** A single provider's shard carries no internal redundancy, so per-provider audits give PDP-strength detection, not PoR-strength extraction. Retrievability is a property of the stripe, delivered by RS(16,56), and detection is what the audit contributes to it.

That is a defensible architecture and it is arguably the right one — but it is not what ADR-002's title claims, and the distinction has never been written down. Recorded as Q66-1.

### One consequence the mapping makes visible

BJO's outer code has to be an **adversarial** error-correcting code — one that denies the adversary any advantage from choosing *which* symbols to corrupt over corrupting at random `[BJO §3.3, §4.2]`. They achieve it by permuting file blocks under a PRP before striping, and encrypting the parity blocks, so stripe boundaries are hidden from the server.

Vyomanaut's outer code has no such property and does not need the same one — its adversary is a provider that can corrupt only *its own* shard, not choose positions across the stripe. But the analogous question is live and is unaddressed: an adversary that can influence **placement** can choose which stripe positions it controls. That is F-34's colluding-ASN problem arriving from a second direction, and it means the placement controls in ADR-014 are load-bearing for the *coding* argument, not only the durability one.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Retrievability (extraction) as the security goal | Possession of sampled blocks | The guarantee finally matches what a data owner is buying; requires an outer erasure code, which Vyomanaut has |
| Sentinels — pseudorandom values hidden among blocks | Per-block authenticators | Verification is a byte comparison, no public-key arithmetic anywhere; each sentinel is consumed by one challenge |
| Bounded `q` audits fixed at encode time | Unbounded challenge capability | Small file expansion and very cheap verification; a hard expiry on the audit relationship, which is fatal for continuous storage |
| Encrypt-then-embed | Format-independent tagging | Sentinel indistinguishability holds; the scheme cannot be applied to plaintext files at all |
| Stateful verifier | Stateless | Simpler bookkeeping in the single-verifier case; unusable for a replicated verifier, which is what Vyomanaut has |

---

## Breaks in Our Case

- **`q` audits fixed at encoding time** ≠ **~1.05 M chunk-challenges per provider per year, for a retention period with no stated bound**
  → Reject the sentinel construction. Adopt the framework and take the primitive from Paper 68. This is the same call `[SW]` makes.

- **A stateful verifier that tracks which sentinels are spent** ≠ **a three-replica microservice cluster with 1 s gossip cadence, a 5 s healthy window and no leader election (ADR-025, ADR-048)**
  → A spent-sentinel counter is a sixth non-mergeable operation on top of ADR-013's six. Two replicas issuing challenges from divergent views would replay a spent sentinel and hand a deleting provider a free pass. The stateless-verifier requirement is not aesthetic here; it is what keeps the audit path out of F-35's blast radius.

- **The block isolation assumption** ≠ **a provider that deletes contiguous regions, or an entire vLog segment**
  → Vyomanaut's storage engine is a log-structured vLog with segment-granularity GC (ADR-023, ADR-046). The realistic corruption event is *not* independent per block — it is one segment gone. Any detection bound derived under block isolation overstates detection against segment loss. ADR-060's sampling is over whole chunks precisely so the sampled unit matches the deletion unit; that alignment is the mitigation and it should be stated as one.

- **The outer code lives inside the object, applied by the client before upload** ≠ **Vyomanaut's outer code lives across 56 providers and is applied by the client, but no single provider holds a decodable fraction of it**
  → This is the useful break rather than a problem. It means intra-shard redundancy is unnecessary, so Vyomanaut pays the outer-code cost once (in RS(16,56)) rather than twice. It also means per-provider PoR is unachievable and unnecessary, and the ADRs must stop implying otherwise.

- **A 4 GB single-owner archive audited at `q_c = 50` over 50 days** ≠ **286,720 chunks per provider, audited daily, across 56 providers per stripe**
  → The `(1 − ε)^(q_c)` bound applies per prover per file. At Vyomanaut's fan-out, the union bound across 56 provers matters and is not in either source: the probability that *some* provider in a stripe is an undetected `ε`-adversary is what governs stripe durability, not the per-provider figure. Not derived here; recorded as Q67-1.

- **`~3 MB/s` encoding throughput in Java, outer code 61–67% of it** ≠ **RS(16,56) at `k·m = 640`, 16× outside any published benchmark (Paper 65)**
  → BJO's outer code is `(255, 223, 32)` Reed–Solomon truncated — `k·m` of a few hundred at most. Their throughput figure cannot be carried across. It does corroborate the direction Paper 65 established: the outer code dominates client-side encode cost, and Vyomanaut has never measured its own. Q65-1 stands unchanged.

---

## Decisions Influenced

- **[ADR-059](../decisions/ADR-059-homomorphic-authenticator-audit.md) [#2 Proof of Storage]** `PROPOSED — evidence`
  Supplies the security goal (extraction, not possession) and the stateless-unbounded requirement that eliminates the entire sentinel and precomputed-challenge family. Does not supply the construction.
  *Because:* the bounded-`q` arithmetic above is decisive and is checkable from Vyomanaut's own parameters, independent of anything unverified in this note.

- **[ADR-060](../decisions/ADR-060-sampled-chunk-audit-schedule.md) [#2 Proof of Storage]** `PROPOSED — evidence`
  Supplies the `ε`-adversary model, the `λ < (1 − ε)^(q_c)` Phase I bound, the two-phase separation of *bounding the corruption rate* from *recovering the data*, and the turns-adversarial-later argument for spreading challenges over time.
  *Because:* it gives the audit schedule a stated security objective — bound `ε` below what the outer code tolerates — rather than a coverage target.

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `EVIDENCE ADDED`
  Names RS(16,56) as the outer code in a proof-of-retrievability sense, not only a durability sense. The erasure code is load-bearing for the *security* argument of the audit path, which ADR-003 does not currently say and which changes what a future code-family change (ADR-026, F-44) has to preserve.

- **[ADR-002](../decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]** `CORRECTION REQUIRED`
  The ADR uses "PoR" throughout for a per-provider mechanism that cannot deliver retrievability. Either the term changes or the claim does.

---

## Disagreements

- **Shacham & Waters (Paper 68) against JK's model, not its goal.** `[SW]` rules out state in key generation and verification, allows arbitrary rather than two-move proof protocols, and adds a public key. They note any stateless scheme secure in the original JK model is secure in theirs.
  *Implication for us:* build to SW's variant of the definition. The stateless requirement is one Vyomanaut needs on its own terms.

- **Bowers, Juels & Oprea (CCSW 2009) against JK's own analysis.** `[BJO §5.1]` states the JK challenge phase *"is only used to ensure an `ε`-adversary, but is not effectively useful to extract file blocks"*, and that JK's block isolation assumption is a strong one their variant removes. Their variant tolerates an error rate at least an order of magnitude higher than JK at the same outer-code overhead and security bound.
  *Implication for us:* the inner/outer framework is the durable contribution; JK's specific parameterisation was superseded by its own authors within two years.

- **Against the reading list's Domain A framing, mildly.** The entry says the corpus *"read the deployments and skipped the primitives."* True — but the deeper problem this paper exposes is that Vyomanaut has an outer code and never identified it as one. The missing piece was not only the primitive; it was the layering.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q66-1, Q67-1 and Q67-2.
