# Paper 73 — Remote Data Integrity Checking with Server-Side Repair

**Authors:** Bo Chen (Michigan Technological University, work done at NJIT/Stony Brook), Reza Curtmola (New Jersey Institute of Technology)
**Venue / Year:** Journal of Computer Security, Vol. 25, No. 6, 2017, pp. 537–584
**Also published as:** a preliminary version at ACM CODASPY 2014 (`RDC-SR` only; `ERDC-SR` is new to the journal version)
**Topics:** #2 Proof of Storage, #4 Repair Protocol, #19 Adversarial Provider Behaviour
**Track:** LTS
**Reading list:** Domain A / **R-47** — Band 0, no search required — accept criterion *"a non-key-holder produces authenticators a verifier accepts, **or** the trilemma's cost is proved explicitly. Must name who holds what during repair"*
**ADRs produced:** ADR-014 Addendum A (outsourcing defence restated)
**Findings raised:** F-LTS-12, F-LTS-13
**Questions closed:** none
**Questions raised:** Q73-1, Q73-2
**Triage score:** 8/10 (parameter reach 1 · trust model 2 · evidence 2 · actionability 2 · corpus delta 1)

---

## Provenance

Read: the full journal article, 48 pages, from the PDF supplied. All sections including the
appendices' experimental tables were available. Proofs of Theorems 3.2, 4.2 and 5.x are in the
paper's own appendices and **were read for statement and structure, not verified line by line** —
claims resting on them are tagged accordingly.

This is the replication-based member of the Chen–Curtmola server-side-repair line. It is *not* the
paper closest to Vyomanaut's code family; Paper 74 (RDC-EC, erasure coding) is. It is read first
because it establishes the threat model — the ROTF attack and the α-cheating adversary — that
Paper 74 inherits without restating, and because its §2.1 contains the empirical result that turns
out to matter most to Vyomanaut.

Triage score is recorded for completeness; this is a Band 0 item drafted by instruction, not by
score.

---

## Problem Solved

Every prior distributed remote-data-checking scheme repairs corruption by making the **data owner**
do the work: download enough redundancy to reconstruct, re-derive the lost object, upload it to a
fresh server `[P §1]`. For a replicated archive that is linear in replica size on the owner's own
connection, and the owner's connection is the worst link in the system. Chen & Curtmola propose
**server-side repair**: the servers collaborate to regenerate a lost replica over their own premium
interconnect, and the owner is demoted to a lightweight *repair coordinator* `[P §1.1]`.

Doing that safely requires giving the servers the means to generate replicas — which immediately
creates an attack the owner-driven design did not have. If servers can make replicas on demand, an
economically motivated provider can simply **not store** its replica and manufacture it when
challenged. The paper names this the **replicate-on-the-fly (ROTF) attack** `[P §1.1]` and the rest
of the work is about making replica generation expensive enough in wall-clock time that ROTF cannot
finish inside the audit deadline.

For Vyomanaut the value is not either scheme. It is that this paper is the only source in the corpus
that takes a **timing-based possession argument seriously enough to measure whether it works** — and
finds that the previously published version of that argument collapses in practice. ADR-014
Defence 2 is a timing-based possession argument. It has never been measured, and it is specified
against a wire format ADR-059 and ADR-060 subsequently removed.

---

## Key Findings

### The BDS network-delay model, and its empirical demolition

Benson, Dowsley & Shacham proposed proving geographic replica placement purely by response deadline:
challenge a server, and if the answer arrives too fast to have come from a peer at another
datacentre, the server must hold the data locally `[P §2.1]`. The admissible challenge size follows
from

```
T_i  ≤  min(t_i) + min(t_ij) + min(t_j) − 2·max(t_i)                      [P §2.1 eq. 1]
```

where `T_i` is the permitted protocol execution time, `t_i` the auditor↔server delay and `t_ij` the
server↔server delay. This works only if the servers have no fast private path between them —
BDS's Assumption 2.

Chen & Curtmola measured it on Amazon S3 and Assumption 2 is false `[P §2.1]`. Inter-datacentre
bandwidth ran **11–36 MB/s**, against **under 1 MB/s** from a point outside the datacentres, and
propagation delay between N. California and Oregon was **11 ms** `[P-num, Tables 8–9]`. Substituting
Virginia↔N. California into the inequality with 4 KB blocks and a measured per-random-block access
time of `x ≈ 30 ms` gives

```
x·c ≤ 80 + 0.3c   →   c ≤ 2.66 blocks per protocol execution           [P-num §2.1]
```

against the **460 blocks** Ateniese requires for 99% confidence at 1% corruption `[P-num §2.1]`.
The pure-timing argument permits an audit roughly 173× too small to mean anything.

This is the paper's most transferable result and it is a negative one.

### The fix: make the cheat expensive, not the honest path fast

Their repair of the model does not tighten the deadline. It **adds a term the attacker must pay**:

```
T_i + 2·max(t_i)  ≤  min(t_i) + min(t_ij) + min(t_j) + min(t_R)          [P §2.2 eq. 2]
```

where `t_R` is the time to *generate* the missing replica. Because replicas are differentiated
per-server, a cheat cannot be answered by relaying a peer's identical copy; the cheat must be
manufactured, and manufacturing takes measurable time `[P §2.2]`. This lets them drop BDS's
Assumption 2 entirely — the attacker may have an arbitrarily fast private network and still lose.

**Replica differentiation is the precondition for the whole approach** `[P §1.1, Insight 1]`. With
identical replicas, colluding servers store one copy and redirect challenges, and no deadline
recovers the property.

### The α-cheating adversary, and its optimal strategy

A provider contracted to hold `t` replicas but using only `α·t·|F|` storage is an **α-cheating
adversary**, `1/t ≤ α ≤ 1` `[P §3.2.2]`. The model is explicitly *economic*: the adversary cheats
only where cheating is undetectable and profitable, and the paper states plainly that against a
fully malicious adversary indifferent to its own cost, **no solution exists** for building a
reliable `t`-replica system `[P §3.2]`.

**Theorem 3.2** `[P §3.2.3, proof in Appendix C, not verified]`: the best data-distribution strategy
for an α-cheating adversary is to store **an equal α fraction at every server**, not full copies at
some and nothing at others. Under that strategy a challenged server already holds, on average, an α
fraction of the `c` challenged blocks and needs to manufacture only `(1 − α)·c` of them.

This is the finding that sets the deadline. The defender is not sizing against a total cheat; it is
sizing against a `(1 − α)` sliver, and the sliver shrinks as the adversary gets stealthier.

### The parameter guidelines, and the honest caveat inside them

The masking factor `η` and the deadline `τ` are derived, not chosen `[P §4.2]`:

```
η  ≥  x / ((1 − α)·s·t_prf)                     masking factor
τ  =  c·x + 2·t_i                               audit deadline
```

`x` is the measured per-block contribution to proof generation and must come from measurement — the
paper offers three acquisition routes (owner measures it, CSP publishes it, trusted third party
certifies it) and warns that the CSP must not be able to distinguish measurement traffic from
ordinary traffic, or it can poison `x` `[P §4.2]`.

Then the caveat that matters more than the formula: *estimating `x` is not trivial*, because random
block access on S3 is occasionally very slow, and **in those cases benign and malicious behaviour
cannot be told apart** `[P §4.2]`. Their measured distribution over 240 executions puts 95% of `x`
in **0.025–0.034 s** `[P-num §4.2]` — a 1.36× spread inside the 95th percentile, in a datacentre.
Their prototype separates the populations cleanly (95% of benign below threshold, 100% of
adversarial above) `[P-num §6.1]`, but that separation is bought by the artificially inflated `t_R`,
not by the deadline being tight.

### ERDC-SR, and why it is not the part that transfers

`RDC-SR` defeats a **static** α-cheater with fixed compute. A **dynamic** one that grows its compute
budget over time defeats it, and simply raising `η` to cover the horizon makes Setup and Repair
quadratically expensive. `ERDC-SR` instead uses a **β-butterfly encoding** so each replica block
depends on many original blocks, forcing an on-the-fly cheat to compute a large intermediate
dependency cone rather than a single block `[P §5.1.1]`. It is an order of magnitude cheaper than
scaling `η` `[P-num §6.2]`, and it imposes a **minimum file size** — at least 60 MB for a two-year
protection interval, at least 220 MB for five years `[P-num §5.2.2]`.

Vyomanaut does not need this machinery. It gets `t_R` for free from RS(16,56), for the reason the
next section derives.

---

## Substitution at Vyomanaut's Parameters

### 1. Vyomanaut satisfies Insight 1 by construction, and does not need masking

`[DERIVED]` The paper spends its entire construction manufacturing replica differentiation and an
artificial `t_R`, because replication produces identical objects. RS(16,56) produces **56 distinct
shards per segment** and AONT-RS ensures no shard is a function of fewer than `k = 16` others. There
is nothing to relay: no peer holds a copy of shard `y`. Insight 1 holds without a masking factor,
and `η` has no Vyomanaut analogue.

The correct analogue of `t_R` is therefore not "time to run a masking PRF" but **time to fetch
`k = 16` peer shards and run an RS decode-and-re-encode**. That is a real, unavoidable cost imposed
by the code family, and it is far larger than anything `η` can manufacture.

### 2. The ROTF cost at ADR-060's chosen row, α = 0

`[DERIVED]` Under ADR-059/060 an audit samples `c = 2,867` chunks and challenges **every block of
every sampled chunk**. The honest prover therefore reads whole chunks:

```
honest bytes = c · L = 2,867 × 262,144 B          = 751,565,  248 B = 0.752 GB     ✓ matches ADR-060
honest time  = c · (seek + L/D)
             = 2,867 × (0.010 s + 262,144/100e6 s)
             = 2,867 × 0.01262144 s               = 36.19 s = 0.603 min            ✓ matches ADR-060
```

A ROTF prover holding none of it must reconstruct each sampled chunk, and each reconstruction needs
`k = 16` peer chunks at the same segment offset:

```
ROTF bytes = c · k · L = 2,867 × 16 × 262,144 B   = 12,025,  044,  992 B = 12.03 GB  (11.20 GiB)
```

Against provider downlink `B`, ignoring peer upload limits and RTT (both of which help the defender,
so this is the attacker-favourable direction):

| Downlink | ROTF transfer time | Ratio to honest 36.19 s |
| --- | --- | --- |
| 10 Mbit/s | 9,620 s = 2.67 h | **266×** |
| 50 Mbit/s | 1,924 s = 0.53 h | **53×** |
| 100 Mbit/s | 962 s = 16.0 min | **27×** |
| 300 Mbit/s | 321 s = 5.4 min | **8.9×** |

**Hand check.** SHELBY's own version of this ratio (Paper 37, Observation 1) is `40·c_r/c_st ≈ 16`
at `k = 10` `[P-num, Paper 37 §5]`. Vyomanaut's `k = 16` and its whole-chunk challenge give a ratio
in the same family but larger, which is the direction `k = 16 > 10` predicts. ✓

So against a **total** cheat the margin is enormous, and ADR-014 Defence 2 is correct in spirit.
That is not the case the paper says to size against.

### 3. The α-cheater collapses it — the crossover

`[DERIVED]` Theorem 3.2's optimal adversary spreads its deletion uniformly and reconstructs only
`(1 − α)·c` chunks. Timing detection requires the cheat to take longer than honest:

```
(1 − α)·c·k·L / B   >   c·(seek + L/D)
(1 − α)             >   B·(seek + L/D) / (k·L)
(1 − α)             >   B × 0.01262144 / 4,194,304
(1 − α)             >   B × 3.00918 × 10⁻⁹                        B in bytes/s
```

Below that fraction the cheat finishes **inside honest timing** and the deadline sees nothing:

| Provider downlink | Deletable fraction invisible to timing | Equivalent α |
| --- | --- | --- |
| 10 Mbit/s | **0.376%** | α ≥ 0.99624 |
| 25 Mbit/s | **0.940%** | α ≥ 0.99060 |
| 50 Mbit/s | **1.881%** | α ≥ 0.98119 |
| 100 Mbit/s | **3.762%** | α ≥ 0.96239 |
| 300 Mbit/s | **11.28%** | α ≥ 0.88716 |

**Hand check.** At 100 Mbit/s the honest read is 0.752 GB in 36.19 s. The invisible cheat fetches
`0.03762 × 12.03 GB = 0.4525 GB` at 12.5 MB/s = 36.2 s. Equal, as the crossover requires. ✓

Cross-referencing ADR-060's own detection table: a 1% deletion is stated as detected with
probability `1 − 4.0 × 10⁻¹³`. At 50 Mbit/s and above, a 1% deletion is **timing-invisible**, and
the cryptographic response is *correct* because the prover fetched the bytes. **The detection
probability is a probability of the prover not having the data. It is not a probability of the
prover being unable to answer.** That distinction is F-LTS-12.

### 4. Where the deadline actually lives now

`[DERIVED]` ADR-014 Defence 2 specifies `deadline = (chunk_size / p95_throughput) × 1.5`, worked as
`(256 KB / 500 KB/s) × 1.5 = 768 ms`, plus a "latency floor" anomaly detector at `0.3×` the same
quantity. Both are written against a per-chunk challenge returning chunk-sized evidence.

Under ADR-059 the response is **1,040 bytes, constant, regardless of how many chunks were sampled**,
and under ADR-060 there is **one challenge per (provider, file, day)** covering 2,867 chunks. There
is no per-chunk response to time. `chunk_size / p95_upload_throughput` is now a quantity with no
referent: the wire carries 1,040 bytes in every case, so upload throughput no longer bounds anything
and the formula returns 768 ms for a response whose honest generation takes 36 s. **The deadline as
specified is roughly 47× shorter than the honest path it is supposed to admit.** That is F-LTS-13.

The paper's own formula, translated, is what the deadline should have been:

```
τ  =  c·x + 2·t_i                                                     [P §4.2]

    where c = 2,867 sampled chunks
          x = per-chunk contribution = seek + L/D + (field arithmetic per chunk)
          t_i = microservice↔provider network delay
```

with the arithmetic term from ADR-059 at ~16,384 modular multiplications per chunk, which the ADR
itself prices as sub-second across the whole audit against 36 s of disk.

### 5. What the deadline cannot be tightened to

`[DERIVED]` The paper's caveat — benign tail latency destroys separation `[P §4.2]` — is much worse
here than on S3. Their 95% spread of `x` was 1.36× *in a datacentre*. Vyomanaut's provider is an
Indian consumer desktop running as a background process under ADR-025's 50 ms p99 gate, with a
storage engine doing its own compaction, and the honest read is 2,867 random seeks rather than one.
Setting `τ` tight enough to catch a 1% α-cheat at 50 Mbit/s means setting it within a few percent of
the honest median. No measured distribution supports that, and none exists.

**The conclusion is not that the deadline should be tightened. It is that the deadline cannot carry
this weight and something else must.** See Falsifiers.

---

## What This Paper Rules Out

- **Pure network-delay possession proofs, as a family.** `[P §2.1]` measures the BDS model's
  admissible challenge size at 2.66 blocks against a required 460 and the assumption it rests on is
  false against a real provider. Any future Vyomanaut design that proposes to establish possession
  from response timing alone is answered by this measurement, not by argument.
- **Manufacturing `t_R` for Vyomanaut.** RDC-SR's masking factor and ERDC-SR's butterfly encoding
  both exist to create a cost RS(16,56) already imposes. Adopting either would be paying twice.
  ERDC-SR additionally imposes a 60 MB minimum file size for a two-year horizon `[P-num §5.2.2]`,
  which is incompatible with a consumer product whose median upload is not bounded below.
- **Identical-replica redundancy, at any scale, for this project.** `[P §1.1]` Insight 1's collapse
  argument holds. This closes nothing open — Vyomanaut has never proposed replication — but it
  supplies the citation ADR-003 lacked for why the alternative was never viable under an untrusted
  provider, as distinct from why it was too expensive.
- **The "fully malicious adversary" framing for the audit path.** `[P §3.2]` states that no solution
  exists for a `t`-replica system against an adversary indifferent to its own resource cost. The
  economic framing is not a convenience; it is load-bearing. ADR-014's five defence classes do not
  say which adversary they are sized against, and after this paper that omission is visible.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Server-side repair with the owner as coordinator | Owner-driven download-reconstruct-upload | Owner cost falls from `O(replica)` to `O(1)`; a whole new attack class (ROTF) is created and must be separately defeated |
| Time-consuming replica generation as the ROTF deterrent | Cryptographic binding of replica to server identity | Works without new primitives; converts a security property into a *measurement* property, so it degrades silently when the measurement is wrong |
| Economically-motivated adversary model | Fully Byzantine adversary | Makes the problem solvable at all `[P §3.2]`; guarantees evaporate against an attacker with non-economic motives |
| `RDC-SR`'s controllable masking factor | `ERDC-SR`'s butterfly encoding | Applies to files of any size; defeated by an adversary that buys more compute over the protection interval |
| `ERDC-SR` for dynamic adversaries | Scaling `η` to the horizon (`SRDC-SR`) | An order of magnitude cheaper in Setup and Repair `[P-num §6.2]`; imposes a 60–220 MB minimum file size |
| Deadline derived from measured `x` | Deadline chosen by inspection | The parameter is defensible; it requires an adversary model (`α`) the defender must commit to in advance |

---

## Breaks in Our Case

- **Replicas are identical objects that must be artificially differentiated** ≠ **RS(16,56) shards are
  distinct by construction and AONT-RS makes any `k−1` of them jointly uninformative** — **Costed
  (zero)**
  → Insight 1 is satisfied for free. Masking factor `η`, `ERDC-SR`'s butterfly encoding, and the
  minimum-file-size constraint all drop. The `t_R` term is supplied by the code family instead. No
  adaptation needed beyond recording *why* the term exists, which ADR-014 currently does not.

- **The adversary is a single CSP holding all `t` replicas and coordinating internally over premium
  links** ≠ **Vyomanaut's 56 shard holders are independent, mutually untrusted, and separated by
  consumer NAT and Circuit Relay v2** — **Open**
  → Cuts both ways and neither direction has been evaluated. A lone Vyomanaut provider cannot mount
  ROTF at all: it has no authorised path to another provider's shard, since download requires an
  ADR-072 capability token issued by the microservice. But nothing prevents a *colluding set* from
  serving each other raw shard bytes off-protocol over libp2p, and Chen & Curtmola's adversary is
  precisely a colluding set. The capability gate raises the cost against a lone cheat and does
  nothing against the modelled one. Whether the relay-mediated topology makes 16 concurrent
  cross-provider fetches slower than the table in §2 assumes is unmeasured. → Q73-1.

- **The deadline is set from a measured per-block access time `x` and an explicitly chosen adversary
  strength `α`** ≠ **ADR-014 Defence 2 sets it from `p95_upload_throughput × 1.5` with no adversary
  parameter anywhere** — **Fatal, as specified**
  → ADR-014 Defence 2 implicitly assumes `α = 0` — a total cheat — which Theorem 3.2 says is the
  adversary's *worst* strategy, never its chosen one. It must be restated with an explicit `α` and
  against ADR-059's actual wire format. → ADR-014 Addendum A.

- **Their `x` is measured inside an AWS datacentre with a 1.36× spread at the 95th percentile**
  ≠ **Vyomanaut's honest path is 2,867 random seeks on a consumer desktop under a background CPU
  gate, competing with the user's own workload and the storage engine's compaction** — **Open**
  → The tail-latency caveat the paper raises `[P §4.2]` is qualitatively more severe here. No
  distribution has been measured. Until one is, any `τ` is `[VENDOR-DEFAULT]`-class. → Q73-2, and a
  LaunchGate measurement.

- **Confidentiality is declared orthogonal — "replicas can be encrypted and our approaches applied on
  top"** `[P §4.3]` ≠ **Vyomanaut's confidentiality problem is caused by repair itself (F-69), not
  by storage** — **Fatal**
  → The orthogonality claim holds for replication, where repair regenerates a masked copy from a
  masked copy and no party sees plaintext. It fails for AONT-RS, where any repairer assembling
  `k = 16` obtains a decodable package. Nothing in this paper touches F-69. Paper 74 gets closer.

---

## Decisions Influenced

- **ADR-014 [#19 Adversarial Provider Behaviour] `ADDENDUM A — DEFENCE 2 RESTATED`**
  Defence 2's deadline formula, its worked 768 ms example, and its `0.3×` latency-floor detector are
  all computed against a per-chunk challenge/response that ADR-059 and ADR-060 replaced with a
  single 1,040-byte aggregate per `(provider, file, day)`. The formula's inputs no longer exist. The
  Addendum restates the deadline in the paper's own form, `τ = c·x + 2·t_i`, adds the missing
  adversary parameter `α`, and records that Defence 2 is a *secondary* signal behind the
  reconstruction-cost argument rather than the primary defence ADR-014 presents it as.
  *Because:* `[P §2.1]`'s measurement shows a timing-only possession argument admitting 2.66 blocks
  where 460 are needed, and `[P §3.2.3]` Theorem 3.2 shows the rational adversary attacks at the
  `(1 − α)` sliver where §3's substitution puts the margin at or below zero.

- **ADR-060 [#2 Proof of Storage] `CONSTRAINT ADDED — not an approval change`**
  ADR-060's detection table is a bound on *possession*, not on *ability to answer*. In a
  distributed erasure-coded system the prover can obtain what it deleted. The table therefore holds
  only while the response deadline binds, and §3 of this note shows the current deadline does not.
  Recorded as an open constraint on ADR-060 via ADR-014 Addendum A rather than as a change to
  ADR-060's chosen sampling rate, which is unaffected.
  *Because:* `[DERIVED]` at 50 Mbit/s a 1% deletion is timing-invisible, and ADR-060 states that
  same 1% deletion is detected with probability `1 − 4.0 × 10⁻¹³`.

- **ADR-003 [#3 Erasure Coding] `CONFIRMED — supporting citation supplied`**
  `[P §1.1]` Insight 1 supplies the adversarial argument against replication that ADR-003 made on
  storage-cost grounds alone. Under untrusted providers, identical replicas collapse to one stored
  copy plus redirection regardless of price.
  *Because:* the collapse is structural, not economic — no deadline recovers the property once
  replicas are identical.

- **ADR-076 [#4 Repair Protocol] `EVIDENCE — no change`**
  ADR-076 moved repair execution provider-side and accepted that the elected repairer obtains
  plaintext, arguing a rotating per-event exposure beats a permanent network-wide one. This paper
  reaches the same structure independently from a different starting point — its Aggregation Server
  is chosen at random per repair event and trusted only until the end of that epoch `[P §4.2 of
  Paper 74; §1.1 here]`. Convergent design, no decision changed.

---

## Falsifiers

1. **§3's crossover table is void if the honest read is not disk-bound at 12.62 ms/chunk.**
   The table assumes ADR-060's own 10 ms seek + 256 KB at 100 MB/s. If real provider storage is
   SSD-backed at the median, honest time collapses toward 7 s and the timing-invisible deletable
   fraction *falls* by ~5×, which strengthens the defence. If real storage is a contended 5,400 rpm
   consumer HDD, honest time rises and the invisible fraction rises with it. → owned by Q73-2 and
   the same LaunchGate measurement Q23-1 already names.

2. **F-LTS-12 is void if a provider cannot obtain peer shards at all.**
   The whole argument assumes a colluding provider can fetch 16 shards of a segment it holds one
   shard of. If ADR-072 capability tokens were enforced end-to-end *and* providers had no
   off-protocol libp2p path to each other's shard bytes, ROTF would be impossible rather than
   merely expensive. Verify against `internal/p2p` and IC §4.4.1: is there any code path by which a
   peer serves shard bytes to a requester that is not the microservice or a token-bearing owner? →
   Q73-1.

3. **The `α = 0` reading of ADR-014 Defence 2 is void if the deadline was ever intended to be
   evaluated per audit rather than per chunk.** ADR-014's worked example (`768 ms` for one 256 KB
   chunk) and its latency-floor detector are both per-chunk, and ADR-017's `response_latency_ms` is
   a single scalar per receipt. Check whether any implementation in `internal/audit` computes a
   per-chunk deadline; if it does, the finding is a specification/implementation divergence rather
   than a specification defect.

4. **Theorem 3.2's optimal-strategy result is void for Vyomanaut if deletion granularity is not the
   chunk.** The theorem is proved for uniform block-level distribution. Under a log-structured vLog
   with segment-granularity GC (ADR-023, ADR-046) the provider may not be *able* to delete at fine
   granularity — its cheapest deletion unit may be a whole vLog segment containing many chunks. That
   would push the adversary toward coarse deletion, which timing catches easily. → this is the same
   question ADR-060 raised for its own detection model and it now has a second consumer.

5. **The entire ROTF concern is void if the audit response cannot be produced from freshly fetched
   bytes.** It can: ADR-059's response is a pure function of the challenged block contents and the
   seed. Nothing binds the response to *when* the bytes arrived. A construction that bound the
   response to a provider-local, non-transferable secret would close this by primitive rather than
   by deadline — and none of Papers 66–73 supplies one for the erasure-coded case.

---

## Disagreements

- **Benson, Dowsley & Shacham (the BDS model), as used by any timing-only possession scheme:** this
  paper measures the model's central assumption on a real provider and finds it false `[P §2.1]`.
  *Implication for us:* ADR-014 Defence 2 is a BDS-shaped argument and inherits the criticism
  directly. It is not rescued by Vyomanaut's providers being consumer machines rather than
  datacentres — a colluding set on the same ISP has exactly the fast private path BDS assumes away.

- **Paper 37 (SHELBY), Observation 1:** SHELBY argues the outsourcing attack is deterred by *cost*
  (`f_st·k·c_r > c_st`), evaluated over a monthly epoch. Chen & Curtmola argue it must be deterred
  by *time*, within a single audit, because an economically motivated adversary that is willing to
  pay the bandwidth simply pays it.
  *Implication for us:* the two are not alternatives, they are different failure horizons. SHELBY's
  condition says a *permanent* ROTF strategy is unprofitable in the long run; Chen & Curtmola's says
  an *opportunistic* one succeeds in the short run if the deadline is loose. Vyomanaut needs both
  and currently argues only the first — ADR-014's Defence 2 reference to Paper 37 cites the cost
  argument while implementing a timing mechanism, and never reconciles them. ADR-014 Addendum A does.

- **ADR-059's own framing of Defence 4:** the ADR states that with homomorphic authenticators
  *"without the chunk data the provider cannot compute a passing response."* True and unchanged. But
  it is a statement about the *bytes*, not about *who stored them*, and this paper is the reason
  that distinction is not pedantic.

---

## Corpus Delta

New to the corpus: an *empirical* refutation of timing-only possession, an economic adversary model
with an optimal-strategy theorem, and the observation that server-side repair creates the attack it
must then defeat. Nothing in Papers 01–72 covers any of the three; Paper 37 is the closest and
argues the opposite horizon.

**Corrections applied elsewhere this session:** none to earlier paper notes. ADR-014 receives
Addendum A. ADR-060 receives a constraint via that addendum, not a revision. Paper 37 is revised in
place for an unrelated reason (Band 0 item 4) and the disagreement recorded above is added to it.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions **Q73-1** and **Q73-2**.
