# Paper 75 — Remote Data Checking for Network Coding-Based Distributed Storage Systems

**Authors:** Bo Chen, Reza Curtmola (New Jersey Institute of Technology), Giuseppe Ateniese, Randal Burns (Johns Hopkins University)
**Venue / Year:** ACM CCSW 2010 (Cloud Computing Security Workshop, co-located with CCS), pp. 31–42
**Topics:** #2 Proof of Storage, #4 Repair Protocol, #19 Adversarial Provider Behaviour
**Track:** LTS
**Reading list:** Domain A / **R-47** — Band 0 — *"origin of the RDC-on-repair line; read for threat model"*
**ADRs produced:** none — ADR-072 Addendum A confirmed; ADR-078 supported
**Findings raised:** none new; supplies the missing citation for ADR-072 Addendum A
**Questions closed:** none
**Questions raised:** Q75-1
**Triage score:** 7/10 (parameter reach 1 · trust model 2 · evidence 1 · actionability 1 · corpus delta 2)

---

## Provenance

Read: the full 12-page CCSW proceedings version including Appendices A and B. Theorem 3.1's proof is
sketched in the body and given in an appendix that was read for structure; Theorem 3.2's proof is a
sketch in the body only. Claims resting on either are tagged.

This is the **origin** of the Chen–Curtmola remote-data-checking-on-repair line, and the reading
list assigned it for threat model, not for its scheme. That instruction is correct: network coding
is disqualified for Vyomanaut on a property the paper itself names, and the scheme does not
transfer. Two things in it do, and one of them supplies a citation an existing Vyomanaut ADR was
missing.

Ateniese is a co-author here, which links this paper directly to Paper 66. Note the same first
author and the same problem statement as Papers 73 and 74 — this is one research programme read in
reverse chronological order, and the earliest paper states the threat model the later two inherit
silently.

---

## Problem Solved

Prior remote-data-checking work optimised only the **prevention** phase — cheap periodic possession
checks — and left the **repair** phase to be paid at whatever the redundancy scheme cost `[P §1]`.
Over an archive's lifetime servers fail repeatedly, so repair dominates. Network coding reduces
repair traffic by orders of magnitude in a benign setting; nobody had shown it survives an
adversary.

Two attacks are specific to network coding and have no analogue in erasure coding `[P §1, §3.1]`:

- **Replay attack** — repair computes *new* coded blocks rather than restoring the same block, so
  block identity is not fixed. A malicious server can re-serve an old coded block with the right
  logical identifier, silently reducing the number of linearly independent combinations in the
  system until the recovery condition breaks, with the client unaware `[P §3.1.3]`.
- **Pollution attack** — a server behaves honestly during Challenge and contributes *corrupted*
  blocks during Repair. The corruption propagates into every block derived from it, and the client
  cannot check the encoding because it does not hold the original data `[P §3.1]`.

`RDC-NC` defeats both while keeping constant client storage. For Vyomanaut the value is in the
threat model and in one construction — the repair verification tag — not in the coding scheme.

---

## Key Findings

### Two tag families, for two different jobs

`[P §3.1.4]` The scheme uses **two independent** verification structures over the same data:

- **Challenge tags** `t_ijk`, one per *segment*, used during possession checking.
- **Repair tags** `T_ij`, one per *block*, used only during repair to prove the encoding was
  performed correctly.

They are not interchangeable and the paper is explicit about why: a challenge tag proves the server
*has* what it should; a repair tag proves the server *computed* what it should. Vyomanaut has the
first (ADR-059) and has never had the second.

### The proof of correct encoding

`[P §3.1.4, GenRepairBlock]` A helper asked to contribute a combination `ā_i = Σ_j x_ij·c̄_ij`
returns, alongside it, `τ_i = Σ_j x_ij·T_ij` — the same linear combination applied to its own
repair tags. The client checks

```
τ_i  =?  Σ_j x_ij·f_Kprf4(i‖j‖z…)  +  Σ_k λ_k·a_ik   (mod p)        [P §3.1.4 Repair step 1(f)]
```

and declares `S_i` faulty if it fails. This verifies that the helper applied the client-supplied
coefficients to the blocks it was supposed to hold — **without the client ever seeing those
blocks** `[P §3.1.4]`.

This is the same homomorphism Paper 74 states as `x·t_{i_j}` and the same one ADR-078 transports. It
appears here five years earlier, applied to a different job: **detecting a lying helper**, rather
than **carrying a tag forward**. Both fall out of the tag being linear in the block content over the
code's field.

### Logical identifiers, and why an index must be bound into the tag

`[P §3.1.1]` The paper states the requirement bluntly: *"a malicious server `S_i` could simply store
the blocks and tags of another (honest) server and successfully pass the integrity challenges. This
would reduce the overall redundancy across servers and will eventually lead to a state where the
file becomes unrecoverable, without the client's knowledge. To prevent this attack, the index of a
block must be embedded into the verification tags associated with the segments of that block."*

Blocks get logical identifiers `"i.j"` — server number, block number — embedded in every challenge
tag, and the identifier survives repair: when `S_3` fails, the blocks placed on the replacement
server keep identifiers `"3.1"`, `"3.2"` `[P §3.1.2]`. Placement is unconstrained; the identifier is
what is audited, and a discovery service maps identifier to holder.

**Vyomanaut has exactly this problem and fixed it without this citation.** ADR-072 Addendum A binds
`segment_id` and `shard_index` into the capability token, and ADR-059 makes the authenticator index
`i = (chunk_ordinal × 256) + block_ordinal`, *"unique within a file and never reused, including after
repair."* Both are correct. Neither cites a source for why, and the reason it matters is not
obvious from inside the design — it is a **durability** attack, not a confidentiality or integrity
one. This is the missing citation.

### Replay is only harmful when it creates linear dependence

`[P §3.1.3]` A replayed old block is harmful **only if it introduces additional linear dependencies**
among the blocks currently stored. Otherwise the recovery condition is unaffected. The mitigation is
therefore not to prevent replay but to deny the adversary the knowledge needed to *target* it: the
client stores the coding coefficients **encrypted** at the servers, so the adversary cannot tell
which old blocks would be dependent with which current ones and can only replay at random
`[P §3.1.3]`. Theorem 3.1 bounds the resulting advantage as negligible `[P §3.1.3, proof read for
structure]`.

The simple alternative — a per-server version counter in the tags — works but costs `O(n)` client
storage `[P §3.1.3]`, which the paper rejects as violating the outsourced-storage premise.

### Spot checking catches large corruption only

`[P §3.1.4]` *"A spot checking mechanism for the challenge phase is only effective in detecting
'large' data corruption."* Small corruption is handled by layering an error-correcting **server
code** beneath the network-coded blocks, applied before the tags are computed — and the helper
**excludes** the server-code portion when computing new blocks during repair; the client recomputes
it `[P §3.1.4]`. Designing a scheme where the server code can be computed together with the rest of
the block through network coding is left as future work.

### The disqualifying property, named by the authors

`[P §1, §2]` Network coding is **not systematic**: the input is not embedded in the output, so
reading any part of a file requires reconstructing the whole file. The authors are explicit that
network-coded storage *"really only makes sense for systems in which data repair occurs much more
often than read"* and confine its application to read-rarely archival workloads — regulatory
retention, medical image archives, data escrow.

---

## Substitution at Vyomanaut's Parameters

### 1. The read-rarely test, applied

`[DERIVED]` Vyomanaut's product is consumer file storage with an owner-facing retrieve path, a
`retrieve` CLI subcommand, and `ls`/`rm` semantics. Reads are the user-visible operation and the
one the product is judged on. Under `[P §2]`'s own criterion — repair must occur *much more often*
than read — Vyomanaut fails the test in the direction that disqualifies network coding.

ADR-076 sharpens this further and in the same direction. The `r0 = 8` gate exists precisely to make
repair **rare**. A code family whose only advantage is cheap repair, adopted into a system that has
just spent two milestones making repair infrequent, buys progressively less as that work succeeds
while paying its read penalty on every retrieval.

**Network coding is closed for Vyomanaut, on the authors' own criterion, not on ours.** This
confirms ADR-003 rather than revising it, and supplies the reason ADR-003 stated only as "sub-file
access."

### 2. The two-tag separation, evaluated against ADR-059 and ADR-078

`[DERIVED]` Vyomanaut has one tag family. Mapping this paper's two onto it:

| RDC-NC | Job | Vyomanaut equivalent |
| --- | --- | --- |
| Challenge tag `t_ijk` | prove possession of a stored segment | ADR-059's `σ_i` — present |
| Repair tag `T_ij` | prove a helper applied the right coefficients to the right blocks | **absent** |

Under ADR-078, `σ` does both jobs, because the same linearity that transports it also makes
`Σ_j ι(z_j)·σ_{i_j}` checkable. Concretely, a verifier holding `(k_prf, α₁…α₆₄)` and the coefficient
vector can check a helper's contribution `(z_j·shard_j, ι(z_j)·σ_j)` by the ordinary ADR-059
verification equation applied to the scaled pair.

`[DERIVED]` **This gives pollution-attack detection for free, and Vyomanaut currently has none.**
ADR-076's elected-repairer protocol has no mechanism by which a helper's contribution is checked; a
malicious helper contributing corrupted bytes produces a wrong reconstructed shard, which is written
to a replacement provider, carries a tag consistent with the *wrong* bytes under naive transport,
and then passes every subsequent audit. The corruption is permanent, undetectable by the audit path,
and consumes one of the 40 tolerated fragment losses without anyone noticing.

*Hand check on the "carries a consistent tag" claim.* If helper `j` contributes `(z_j·b'_j,
ι(z_j)·σ_j)` where `b'_j ≠ b_j` is corrupted but `σ_j` is the tag of the **real** `b_j`, then the
transported aggregate is the tag of the real reconstruction while the data is the corrupt one — so
the pair is **inconsistent** and per-contribution verification catches it. If instead the helper
contributes a self-consistent forgery `(z_j·b'_j, tag(b'_j))`, it needs `(k_prf, α)` to compute
`tag(b'_j)`, which it does not hold. ✓ Either way the check binds. **ADR-078 must specify this check
as mandatory**, not leave it implied.

### 3. Cost of per-contribution verification

`[DERIVED]` Per repaired chunk, per helper, the check is one ADR-059 verification over 256 blocks:
256 PRF evaluations plus 64 field multiplications per block. Across 16 helpers, per chunk:

```
16 × 256 = 4,096 PRF evaluations  +  16 × 256 × 64 = 262,144 field multiplications
```

Against the routine audit load already budgeted (733,952 PRF evaluations per provider per day,
~8,494 HMAC-SHA-256/s fleet-wide), a repaired chunk costs 4,096 — **0.56% of one provider-day's
verification**. Even under ADR-076's worst gated event, 32 fragments at once, it is 131,072 PRF
evaluations, well under one provider-day.

The open design question is **where** the check runs. On the repairer it is cheap and self-serving
(the repairer has no incentive to accept corrupt input, so it will run it honestly — but it also
cannot be *proved* to have run it). On the microservice it requires shipping 16 × 4,096 B = 65,536 B
of tags per chunk, which is 1.56% of the shard traffic and keeps the microservice out of the data
path. → Q75-1.

### 4. What does not need porting

`[DERIVED]` The logical-identifier machinery, the version counters, the encrypted coefficient
vectors, and Theorem 3.1's replay analysis are all consequences of network coding's **non-fixed file
layout** `[P §3.1.1]`. Vyomanaut's layout is fixed: shard `y` of segment `s` is always shard `y` of
segment `s`, and repair reconstructs *the same block* rather than a new one — which the paper itself
identifies as the erasure-coding case that makes index binding *"straightforward"* `[P §3.1.1]`.

`[DERIVED]` The replay attack in its network-coding form therefore does not exist here. Its
erasure-coding shadow does, and ADR-072 Addendum A and ADR-059's never-reused index already close
it. No work is created; a citation is supplied.

---

## What This Paper Rules Out

- **Network coding for Vyomanaut, permanently, on the authors' own criterion.** `[P §2]` confines
  it to read-rarely archival workloads. Vyomanaut is read-facing consumer storage, and ADR-076 is
  actively making repair rarer. This closes the question rather than deferring it.
- **Spot checking alone as protection against small corruption.** `[P §3.1.4]` states plainly that
  it is only effective against large corruption and layers a server code beneath it. ADR-060's
  detection table already shows the same shape — a 0.1% deletion survives a day with 5.7%
  probability — and this is the independent confirmation that the fix is an *inner code*, not a
  higher sampling rate. Vyomanaut has no inner code: RS(16,56) is across providers, and Q66-1
  already records that this makes the per-provider guarantee possession rather than retrievability.
  The two findings are the same finding arrived at from two directions.
- **Per-server version counters as a replay defence.** `O(n)` client state `[P §3.1.3]`. Not needed
  here anyway, but the rejection reasoning is worth having: any audit-side mechanism whose state
  grows with provider count is disqualified on the same grounds ADR-060 used to kill the nonce guard
  index (F-59).
- **Trusting a helper's contribution because the aggregate decodes.** `[P §3.1]`'s pollution attack
  is exactly the failure of that assumption, and MDS reconstruction from `k` shards will happily
  produce a wrong-but-well-formed shard from one corrupt input.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Network coding | Erasure coding | Repair traffic `O(2|F|/(k+1))` versus `O(n|F|/k)`; loses systematic layout, so every read reconstructs the whole file |
| Separate challenge and repair tags | One tag family | Each job gets the right primitive; two tag structures to store, generate and key |
| Encrypted coding coefficients | Per-server version counters in tags | Client storage stays `O(1)`; replay becomes untargetable rather than impossible |
| Logical identifiers decoupled from placement | Binding blocks to servers | Blocks migrate freely; requires a discovery service mapping identifier to holder |
| A server code beneath the coded blocks | Higher spot-check rates | Covers small corruption that sampling misses; the helper must exclude it during repair and the client recompute it |
| `GF(p)` symbols | `GF(2^w)` | Comparable decoding probability `[P §3.1.4 fn. 5]`; the choice is explicitly noted as not load-bearing |

---

## Breaks in Our Case

- **Repair computes a *new* coded block, so block identity is not fixed and replay is possible**
  ≠ **RS(16,56) repair reconstructs *the same* shard at the same stripe position** — **Fatal
  (favourably)**
  → The entire replay-attack apparatus is inapplicable. `[P §3.1.1]` says so directly about the
  erasure-coding case. Vyomanaut's residual — a provider serving another provider's shard to pass
  its own audit — is already closed by ADR-059's file-global index binding and ADR-072 Addendum A's
  position binding. **This paper is the citation those two decisions did not have.**

- **Non-systematic code; any read reconstructs the whole file** ≠ **Vyomanaut's product is a
  consumer file store where retrieval is the user-visible operation** — **Fatal**
  → Network coding closed for Vyomanaut. ADR-003 confirmed with a better-stated reason.

- **The client is online, holds `sk`, and personally verifies every helper's proof of correct
  encoding** ≠ **ADR-004 makes the owner offline; ADR-076 puts repair between providers with the
  microservice as bookkeeper** — **Costed**
  → The verification role maps to the microservice, which holds `(k_prf, α)` under ADR-059 and can
  run the check on tag aggregates without touching shard bytes. Cost derived in Substitution §3;
  placement is Q75-1.

- **Two independent tag families, one per job** ≠ **Vyomanaut has one, and no repair-verification
  mechanism at all** — **Open, and it is a live gap**
  → Under ADR-078's field change one family does both jobs. Without ADR-078, Vyomanaut has no
  pollution-attack defence and ADR-076's elected-repairer protocol will ship without one unless this
  is written into it. This is the second independent reason to adopt ADR-078 and it is not a
  confidentiality argument.

- **A server code beneath the coded data handles small corruption** ≠ **Vyomanaut has no inner code;
  RS(16,56) is the outer code across providers and a single shard has no internal redundancy** —
  **Open**
  → Q66-1's finding restated from a second source. Not created by this paper and not closed by it.
  Adding an inner code would pay the coding cost twice, which Q66-1 already identifies as the reason
  the current layering is probably right. Recorded so the third source does not get treated as a new
  discovery.

---

## Decisions Influenced

- **ADR-072 Addendum A [#2 Proof of Storage] `CONFIRMED — citation supplied`**
  Position binding (`segment_id`, `shard_index` in the capability token) and ADR-059's never-reused
  file-global block index are the erasure-coding form of `[P §3.1.1]`'s index-embedding requirement.
  The attack they prevent is a **durability** attack — a provider passes audits using another
  provider's data, redundancy silently falls, and the file becomes unrecoverable without the owner
  knowing — not the confidentiality or impersonation attack the capability-token framing suggests.
  *Because:* `[P §3.1.1]` names the failure mode and its consequence explicitly, which no Vyomanaut
  document currently does.

- **ADR-003 [#3 Erasure Coding] `CONFIRMED — network coding closed`**
  `[P §2]` confines network-coded storage to read-rarely workloads and states that online systems do
  not use it because they optimise for read. Combined with ADR-076 making repair rare, the only
  advantage network coding offers is one Vyomanaut is deliberately reducing the value of.
  *Because:* the disqualification is the authors', at the point where they scope their own
  contribution.

- **ADR-078 [#2 · #4] `SUPPORTED — second, independent reason`**
  `[P §3.1.4]`'s `τ_i = Σ x_ij·T_ij` proof-of-correct-encoding is the same homomorphism ADR-078
  transports, applied to helper verification instead of tag continuity. ADR-078 must therefore
  specify **mandatory per-contribution verification** of `(z_j·shard_j, ι(z_j)·σ_j)` pairs, not only
  the aggregate. Without it, the pollution attack lands and the reconstructed shard is permanently
  and undetectably wrong.
  *Because:* `[P §3.1]` — the client cannot check the encoding because it does not hold the original
  data, which is exactly ADR-076's microservice position.

- **ADR-076 [#4 Repair Protocol] `GAP NAMED — no change here`**
  The elected-repairer protocol as specified has no helper-verification step. `[P §3.1]`'s pollution
  attack applies directly. The fix is in ADR-078; the gap is recorded against ADR-076 so it is not
  lost if ADR-078 is rejected.
  *Because:* MDS reconstruction from `k` shards produces a well-formed shard from corrupt input, so
  "it decoded" is not evidence.

---

## Falsifiers

1. **The pollution-attack concern is void if no helper can influence its own contribution
   undetectably.** It cannot be: the helper computes `z_j·shard_j` locally and ships the result.
   Nothing in ADR-076 checks it. The finding stands unless a verification step exists in
   `internal/repair` that this reading of ADR-076 missed — check `executor.go` for any per-shard
   validation beyond `content_hash`. If `content_hash` is verified on each fetched shard *before*
   the linear combination, the attack is closed for **fetched** shards, and the residual is only for
   helpers that compute the combination themselves — which is precisely what ADR-076 moves toward.

2. **The network-coding disqualification is void if Vyomanaut's read profile is archival.** It is
   not, on the current product definition. If a future LTS product tier were deep-archive with
   contractual retrieval delays, `[P §2]`'s criterion would need re-running rather than assuming
   this conclusion. Flagged, not expected.

3. **The ADR-072 Addendum A confirmation is void if capability tokens are not enforced on the audit
   path.** Position binding in the *token* prevents a provider being issued a token for someone
   else's shard. It does not by itself prevent a provider *answering an audit* with someone else's
   bytes — that is prevented by ADR-059's index binding, separately. Both must hold. Verify that the
   audit verification equation uses the file-global index of the challenged provider's own assigned
   positions and not an index supplied in the response.

4. **Substitution §3's cost figures are void if per-contribution verification must run on the
   microservice.** They are computed as PRF counts, which are placement-independent, but the 65,536
   B/chunk of tag traffic is not. At ADR-076's 32-fragment gated event that is 2.1 MB per event to
   the microservice — still small, but it is traffic ADR-076 currently budgets at zero. → Q75-1.

---

## Disagreements

- **Papers 66–68, and by inheritance ADR-059/ADR-060:** all three treat the audit as a two-party
  protocol between a verifier and a prover, and none has a repair phase. This paper's central claim
  is that optimising prevention alone is the wrong objective because repair dominates the lifetime
  cost `[P §1]`.
  *Implication for us:* ADR-059 and ADR-060 are both scoped to the challenge phase, and both list
  the repair problem as an unresolved open constraint rather than as a co-equal design objective.
  The 2010 paper says that scoping is the mistake. F-LTS-07 and F-LTS-08 are what it costs.

- **Paper 74 (RDC-EC), on tag design:** the 2015 paper collapses to a single repair-tag family and
  reuses HAIL's integrity-protected dispersal code for challenge verification. The 2010 paper keeps
  two families. Neither is wrong — the difference tracks whether the file layout is fixed.
  *Implication for us:* Vyomanaut's layout is fixed, so the 2015 collapse is the right precedent,
  and ADR-078's single-family design follows it. The 2010 separation is what a system with mobile
  block identity needs, and Vyomanaut is not one.

- **Its own guideline on `GF(p)` versus `GF(2^w)` `[P §3.1.4 fn. 5]`:** the paper treats the choice
  as immaterial, arguing decoding probability is similar either way. For Vyomanaut it is the single
  most consequential parameter in the audit primitive, because it decides whether tags transport
  through repair. The footnote is correct about *decoding* and silent about *tag homomorphism across
  a code*, which is the property that matters. Recorded because it is exactly the kind of
  "immaterial" parameter that F-LTS-14 shows is not.

---

## Corpus Delta

New to the corpus: the prevention/repair holistic framing; the replay and pollution attack classes;
the proof-of-correct-encoding construction; and the explicit statement that index binding is
required or redundancy silently erodes.

**Confirms without changing:** ADR-003 (network coding closed), ADR-072 Addendum A (position
binding), Q66-1 (no inner code → possession not retrievability).
**Supports:** ADR-078, on grounds independent of Paper 74's.
**Corrections applied elsewhere:** none. ADR-072 Addendum A gains a reference; no text of it changes.

---

## Open Questions

See [open-questions.md](open-questions.md) — question **Q75-1**.
