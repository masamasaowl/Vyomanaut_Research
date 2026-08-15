# Research Topic List v3 — LTS Track

**Track:** LTS. **Nothing in this document applies to the demo**, which is frozen at `demo-v1.0.0`
under ADR-062. Where a demo-track ADR (063, 064, 070–073) appears below, it appears only as
*evidence about a decision the LTS must make for itself* — never as a demo work item.

**Supersedes** `reading-list-v2.md` for all forward planning. v2 is **not deleted**: it is the
record of the ADR-001–058 interrogation and remains the provenance for R-01–R-46. Retag it
`Superseded by reading-list-v3.md — retained as the ADR-001–058 interrogation record`.

**Built from** a pass over ADR-059 → ADR-073, Papers 66–72, `build_part4.md`'s LTS milestone map,
and the open-questions register as of Q-LAB-1. v2 was written against ADR-058 and a single-track
project; both of those premises are now false.

---

## §1 — Why v2 needs restructuring, in one paragraph

v2 ordered everything by **rewrite risk** — how expensive a gap becomes to learn late — because at
the time there was one artifact and one dependency graph. ADR-062 split the project in two and
`build_part4.md` states the consequence explicitly: *"Milestone 19 is fixed. Everything after it is
provisional and will be renumbered and reordered according to what the demo runs and the
reading-list-v2 research produce."* That sentence makes the reading list an **input to the LTS
milestone order**, not a parallel activity. Rewrite risk is now the wrong axis, because on the LTS
track there is nothing to rewrite yet — the axis is **which LTS milestone cannot start until this
is known**. Everything below is re-ordered on that basis, and §3 is the binding table.

Three secondary things also changed. Papers 66–72 closed R-01–R-04 as *reading* gaps and opened
four council questions plus one hard blocker (Q68-3) that v2 has no topics for. ADR-065's
measure-then-freeze `LaunchGate` pattern gives several v2 "research" items a real home that is not
the reading list. And ADR-069's simulation harness, ADR-070's thirteen seam findings, and
ADR-066–068's observability work opened three areas the corpus has **zero** coverage of — which is
the same shape of problem Domain A was, and it was Domain A that turned out to matter most.

---

## §2 — Corrections listed for approval, not applied

Six findings surfaced while reading ADR-059→073 against each other. Per the project's own
convention these are flagged, not patched. Numbered in a separate namespace (`F-LTS-NN`) so they do
not collide with the F-01–F-79 interrogation series or ADR-070's `F-070-NN`; renumber into the main
series when accepted.

| # | Finding | Where it bites |
| --- | --- | --- |
| **F-LTS-01** | **ADR-072's security argument does not survive ADR-073.** ADR-072 dropped `file_id` from the capability-token signing input on the stated ground that `chunk_id` is *"256 bits of fresh, microservice-generated randomness … never reused across files."* ADR-073 makes `chunk_id` exactly `SHA-256(chunk_data)`, computed **client-side**. ADR-073 says it does not reopen ADR-072's *decision*, and it is right — but the *argument* recorded for that decision is now false as written. Token-to-assignment binding now rests on ciphertext uniqueness, which rests on AONT key freshness, which is **F-47's** RNG assumption. | The decision is probably still correct. It must be re-argued on the LTS track from the premise that actually holds, before real owner data. Couples Domain N to the upload path for the first time. |
| **F-LTS-02** | **The server persists a `chunk_id` it cannot verify.** Under ADR-073 the microservice never sees `chunk_data`, so it cannot check the client's submitted hash. Provider-side `0x02 CHUNK_ID_MISMATCH` correctly enforces the binding at write time — but nothing prevents two files, or two *owners*, submitting the same 32-byte value, and no document states what a provider does when a second assignment arrives for a `chunk_id` it already stores. Unreachable on the demo; a namespace-aliasing question at LTS scale with adversarial owners. | New **Domain H**. |
| **F-LTS-03** | **ADR-069's synthetic tier cannot answer an ADR-059 challenge.** ADR-069 §2 has synthetic providers derive `chunk_bytes = PRF(node_seed ‖ chunk_id)` so audit responses are "indistinguishable from a real provider's." Under ADR-059 a valid response is `(μ_j, σ)`, and `σ` requires the per-chunk authenticators — 4,096 B/chunk, generated client-side from `(k_prf, α₁…α₆₄)`. A synthetic node can regenerate `m_ij`, so it can compute `μ`. It cannot compute `σ`. Either the harness generates and holds authenticators, or the microservice hands out file keys — which changes the trust model the simulation is supposed to be testing. | The Scale Simulation milestone sits *after* the Proof of Storage milestone in `build_part4.md`'s map, so this is discovered during a 10,000-node run unless fixed first. Sharpens Q-SIM-1. |
| **F-LTS-04** | **ADR-060 removes ADR-061's main reason for existing.** ADR-061 chose the flat split partly because *"low per-file audit sample counts at production cadence make the weighting noisy."* Under ADR-060 one receipt covers ~2,867 chunks per `(provider, file, day)` — the sample count rises by ~287×, and the noise objection largely evaporates. Meanwhile ADR-060 states income becomes a function of sampling unless F-03 rules otherwise. Two Accepted/Proposed ADRs now point in opposite directions on the same question. | Sequence is **F-03 first, then re-open ADR-061 for LTS**. Domain J gains R-63. |
| **F-LTS-05** | **Four post-ADR-062 ADRs carry no `Track:` line.** ADR-062 §3: *"Every ADR from ADR-062 onward carries a `**Track:**` line directly under `**Status:**`. An untagged ADR fails CI check 10's extended form."* ADR-065, 066, 067 and 068 have none. ADR-059, 060 and 061 predate the rule but govern LTS-only mechanisms and should be retro-tagged. | Session 19.0.1's `TestEveryADRHasTrackTag` fails on its first run. Two minutes to fix; better fixed than discovered. |
| **F-LTS-06** | **ADR-062's frozen ADR ceiling is arithmetically wrong.** §6 fixes CI check 10's demo ceiling at **ADR-064**, on the reasoning that 065+ are LTS. But ADR-070, 071, 072 and 073 are all `Track: DEMO`, all record demo-track decisions, and 070/072/073 are implemented in demo source. Either the ceiling moves to **ADR-073** (with 065–069 individually excluded), or check 10 fails at the M18 freeze. | Bites exactly at M18 sign-off, which is the worst possible moment. |

Two of these (F-LTS-01, F-LTS-02) are **design questions** and belong to a council. Four
(F-LTS-03 → F-LTS-06) are **prescriptive fixes** needing only approval.

---

## §3 — The new ordering axis: research binds to LTS milestones

`build_part4.md` fixes M19 and leaves the rest provisional. This table is this document's answer to
the ordering question it poses. Read it as: *no milestone in the left column should open until the
domains in the right column have produced a ruling or a documented decision to proceed without one.*

| LTS milestone (`build_part4.md`) | Research precondition | Status of that precondition |
| --- | --- | --- |
| **19 — LTS Foundation** | None. Deliberately. | Clear to run. The one research-adjacent item is **Q-M19-2** (measured vs assumed 128 relay slots), which Session 19.1.3 already measures. |
| **Proof of Storage** (ADR-059/060) | **Domain A′** — Q68-3 (repair-time tag authority) is a *hard* blocker, not a caveat. **Domain B′** — Q67-1 (stripe-level detection) and Q68-2 (verifier cost). | **Not met.** R-03 was never actually closed; Papers 69/70 narrowed it. R-47–R-49 are new. |
| **Confidentiality** (F-69, F-34) | **Domain P**, **Domain K** | **Not met.** R-27–R-31 unstarted. This is the only milestone in the map with *no ADR at all* behind it. |
| **Population Re-basing** | **Domain D** | **Not met.** Still resting on 1999–2003 measurements. |
| **Audit at Fleet Scale + Transparency Log** | **Domain B′**, **Domain G** (incl. new R-50) | **Not met**, and G is now load-bearing for two decisions, not one (Q70-1). |
| **Repair & Erasure Optimisation** | **Domain C**; blocked upstream on **Q62-1** | Q62-1 is a *specification* question answerable today from ADR-004's text. Until it resolves, the whole domain may be worth **zero**. |
| **Payments Hardening** | **Domain J**, and **F-03** ruled | F-03 is upstream of ADR-060, ADR-061 and ADR-012 simultaneously (F-LTS-04). |
| **Production Hardening** | **Domain L**, **Domain M** | R-32 is the high-value item; R-37 resolves F-75's dilemma. |
| **Observability & Launch Gates** | **Domain O** *(new)* | Zero corpus coverage. |
| **Desktop GUI Shell** | Domains from the ADR-038–052 cluster | Low priority, correctly. Unchanged from v2. |
| **Scale Simulation** | **Domain S** *(new)*, plus **F-LTS-03** fixed | A number produced without S is a number that cannot be defended in a paper or to an investor. |
| **GA Launch Readiness** | **Domain Q** *(new)* runs across all of it | ADR-070 found thirteen defects by running one seam once. Nobody has counted how many seams there are. |

**The three that gate the most:** Domain A′ (two milestones), Domain G (two milestones and two open
questions), Domain Q (every milestone, continuously).

---

## §4 — How to search, upgraded

v2's workflow stands: this document names a **topic** → search → triage document → Technical
Researcher drafts `paper-NN`. What changes is the precision of each step, because the reported
failure mode is a search aimed at a settled engineering choice, and the fix for that is structural,
not more keywords.

### 4.1 — Per-database query syntax, paste-ready

Generic keyword strings return generic results. Each index has a field syntax; use it.

```
IEEE Xplore  ("Document Title":TERM_A) AND ("Abstract":TERM_B) AND ("Publication Year":2018-2026)
             — supports NEAR/n. Use ("Abstract":repair NEAR/5 bandwidth) for phrase-adjacency.

ACM DL       Title:(TERM_A) AND Abstract:(TERM_B)
             — then filter by "Publication: Proceedings of ..." to pin a venue.

ScienceDirect  TITLE-ABSTR-KEY(TERM_A AND TERM_B) AND NOT TITLE-ABSTR-KEY(POLLUTANT)
             — 8-boolean-connector limit; split long queries rather than truncating.

USENIX       site:usenix.org "TERM_A" "TERM_B"   (via a general web index — usenix.org's own
             search is weak; the proceedings are open-access so full text is indexed)

DBLP         author-first, for closed subfields: dblp.org/pid/... → scan the full publication list.
             Faster and more complete than keyword search whenever one canonical author exists.
```

### 4.2 — Standing exclusion set

Append to **every** query in Domains A′, B′, C, E, G, P, K. These term families are what turned four
of thirteen erasure-coding hits into cost-arbitrage papers, and the 2023–2026 literature has added
two more floods:

```
NOT (blockchain OR "smart contract" OR token OR incentive-layer)     ← swamps all PoR/audit queries
NOT ("federated learning" OR "model aggregation")                    ← swamps all "distributed" queries
NOT ("cloud-of-clouds" OR "vendor lock-in" OR "storage class" OR "tier pricing" OR "cost model")
NOT (FPGA OR GPU OR "P4 switch" OR SmartNIC)                         ← providers have none of these
```

### 4.3 — The four-part filter (replaces v2's two-part)

v2 used **Accept if / Reject if**. Two failure modes survived that filter and both are expensive:

- the **near miss** — a paper that satisfies "accept if" on its abstract and turns out to solve an
  adjacent problem, discovered three hours into a full read;
- the **unsubstituted paper** — a paper that is genuinely on-topic and whose result silently does
  not hold at `(n=56, k=16, s=64, r₀=8)`. This is the F-43 / NFR-044 failure mode exactly.

So every topic below carries four fields:

| Field | Purpose |
| --- | --- |
| **Accept if** | What the paper must *contain*. Unchanged in spirit from v2. |
| **Reject if** | What disqualifies it outright. |
| **Adjacent, not this** | The specific near-miss class this query is known to attract. Read the abstract *for this* first — it is faster to disqualify than to qualify. |
| **Substitution test** | A one-line arithmetic check at Vyomanaut's real parameters that the paper's result must survive before it is worth drafting. **Run it during triage, in the triage document, showing the working** — this is the standing rule (F-43, NFR-044) moved upstream from ADR-drafting to shortlisting, which is where it is cheap. |

### 4.4 — Triage scorecard

Score each shortlisted abstract 0–2 on five axes; **draft at ≥ 7, discard at ≤ 4, and put 5–6 in a
"mine the introduction" pile** (read the related-work section for citations, do not draft a
`paper-NN`). This makes shortlisting reproducible across sessions rather than a judgement call each
time.

| Axis | 0 | 1 | 2 |
| --- | --- | --- | --- |
| **Parameter reach** | result stated only at parameters ≥ 4× from ours | within 4× | covers or brackets our parameters |
| **Trust model** | assumes a trusted coordinator/placement authority | partially trusted | untrusted providers, untrusted operator |
| **Evidence type** | proposal, no evaluation | simulation only | implementation + measurement |
| **Actionability** | would require a codec/protocol replacement | a parameter change | adoptable behind an existing interface |
| **Corpus delta** | duplicates a paper we have | overlaps one | genuinely new mechanism or measurement |

### 4.5 — Citation-chain rule

For **closed subfields** — secure regenerating codes, PoR-on-repair, transparency-log gossip,
ramp secret sharing — forward-citation search from a known anchor beats keyword search, because the
subfield's own vocabulary is small and inconsistently used. Procedure: take the anchor named in the
domain's **Must-read** table → Semantic Scholar / Google Scholar "cited by" → filter to 2015+ →
scan titles only. Budget one hour. This is how R-27, R-28, R-47 and R-50 should be run; keyword
search is for the *open* subfields (D, E, O, S).

Tags: **[GAP]** nothing in the corpus touches it · **[STALE]** covered by expired measurement data ·
**[OPEN-Q]** the reading is done and a council question remains · **[NEW]** first appears in v3.

---

# Band 0 — Drafting queue, no searching required

**Papers 62–65 stand exactly as v2 lists them** (LESS, Friedman–Kapelko, Hitchhiker, BiLP-BX). One
correction and one addition:

- **Papers 62 and 64 are now conditional on Q62-1.** v2 lists them as ready-to-draft. Q62-1 (does
  `r₀=8` mean single-fragment repair never happens?) is a specification question answerable from
  ADR-004's own text plus a ruling. If the answer is "never," both papers document a 0% saving.
  **Settle Q62-1 before drafting either.** Cost of getting this wrong: two full paper drafts.
- **New, ready to draft on sight — Chen & Curtmola, "Remote data integrity checking with
  server-side repair"** (*Journal of Computer Security* 25(6), 2017), with its predecessor
  Chen, Ammula & Curtmola, "Towards server-side repair for erasure coding-based distributed storage
  systems" (CODASPY 2015). This is **the** literature for Q68-3 and it is not in the corpus. Its
  entire premise is the problem ADR-059 records as unsolved: previous RDC schemes require the data
  owner to download, re-encode and re-upload to repair a faulty server, and these papers remove
  that requirement. See Domain A′ / R-47.

---

# Band 1 — Gates an LTS milestone

## Domain A′ — Audit continuity across repair `[OPEN-Q]` `[GAP]` — **START HERE**

**ADRs:** 002, 004, 014, 015, 017, 021, 030, 059, 060 · **Findings:** F-01, F-02, F-32 ·
**Questions:** Q66-1, Q68-1, Q68-3, Q69-1, Q70-1

**Domain A is re-scoped, not closed.** v2's status note is right that R-01–R-04's canonical sources
have been read (Papers 66–72) and that F-01 and Q68-1 are now council questions rather than reading
gaps. It is wrong about R-03. ADR-059's own Open constraints put it plainly: *"A reconstructed shard
is unauditable until this is answered."* Repair is routine, not exceptional. That is not a narrowed
question; it is an unsolved one with an unread literature behind it, and it gates the entire Proof
of Storage milestone.

Three topics, all new. R-01, R-02 and R-04 are retired into the answered record.

**Must-read**

| Work | Why |
| --- | --- |
| **Chen & Curtmola, "Remote data integrity checking with server-side repair"** (J. Computer Security 25(6), 2017) | Directly Q68-3. Removes the owner from the repair loop, which is the exact constraint ADR-004 imposes and ADR-059 cannot satisfy. |
| **Chen, Ammula & Curtmola, "Towards server-side repair for erasure coding-based distributed storage systems"** (CODASPY 2015) | The erasure-coded case specifically, rather than the network-coding case. Closer to RS(16,56) than anything else named here. |
| **Chen, Curtmola, Ateniese & Burns, "Remote data checking for network coding-based distributed storage systems"** (CCSW 2010) | The origin of the RDC-on-repair line. Read for the threat model, not the construction. |
| **Bowers, Juels & Oprea, HAIL** (CCS 2009) | Dispersal-level PoR *across* servers. This is Q67-1's shape — the corpus currently has the authors' CCSW 2009 paper as a Paper 67 substitute, which is a different result. |
| **Le & Markopoulou, "NC-Audit"** (NetCod 2012) · **Omote & Thao, "MD-POR"** | Multi-source direct repair with auditing. Read only if R-47 sends you toward a coding change. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-47** | **Tag generation for reconstructed shards without the owner** | A party that is *not* the key holder can produce authenticators a verifier accepts, **or** the scheme proves the trilemma's cost explicitly. Must name who holds what during repair. | Requires the owner online at repair time — that is the assumption ADR-004 removes. | Dynamic-update PDP (Paper 69 territory): it verifies an update *claimed by the key holder*. The gap is authority, not verification. |
| **R-48** | **Detection composition across a 56-prover stripe** | Gives `P(more than n−k provers are simultaneously corrupt-and-undetected)`, or the tools to derive it, **with a stated independence assumption**. | Per-prover bounds restated — we have three papers' worth. | Multi-replica PDP (MR-PDP and descendants): replicas are *identical*, our 56 shards are distinct. Wrong composition. |
| **R-49** | **Verifier-side cost of aggregate audit at fleet scale** | Reports verifier CPU as a function of blocks challenged, and treats the `s`-vs-response-size trade-off as a *tunable*, not a fixed choice. | Prover-side optimisation; the prover is disk-bound and settled. | Batch auditing across *owners* (Domain B′ / R-05). Related, different bottleneck. |

**Search strings**

```
R-47  IEEE  ("Document Title":repair) AND ("Abstract":"remote data checking" OR "provable data
            possession") AND ("Abstract":"server-side" OR delegat* OR "without the client")
      ACM   Title:(repair OR reconstruction) AND Abstract:("proof of retrievability" AND
            (delegated OR "third party" OR "server side"))
      chain forward-cite Chen & Curtmola 2017; filter 2017+
R-48  IEEE  ("Abstract":"proofs of retrievability") AND ("Abstract":"multiple servers" OR
            dispersal OR "across servers") AND ("Abstract":detection)
      ACM   Title:(HAIL OR "distributed proofs") AND Abstract:(erasure AND corruption AND bound)
R-49  ACM   Abstract:("homomorphic authenticator" OR "aggregate verification") AND
            Abstract:(verifier AND (cost OR throughput OR "CPU"))
      SD    TITLE-ABSTR-KEY(("integrity auditing") AND (scalability OR throughput) AND verifier)
```

**Substitution tests**

- **R-47:** the scheme must work when the repairing party holds `k=16` shards and the verifier holds
  `(k_prf, α₁…α₆₄)` for a file it has never seen the bytes of. If the paper's repair party is the
  storage *operator*, ask what breaks under ADR-021's pure-P2P constraint before drafting.
- **R-48:** substitute `n=56`, `k=16`, tolerance 40 lost shards, per-prover per-day miss probability
  `4.0 × 10⁻¹³` at `t=1%` (ADR-060's own table). If the paper's bound assumes independence, state
  the common-mode cases (shared daemon release, shared storage-engine bug) it therefore excludes.
- **R-49:** 733,952 HMAC-SHA-256 evaluations + the same count of modular multiplications, per
  provider, per day, per file-audit, × 1,000 providers. Any paper that does not survive four orders
  of magnitude of that is a "mine the introduction" paper.

**Two things this domain must not spend budget on.** Q66-1 (possession vs retrievability) is a
documentation ruling answerable today — ADR-002's title says PoR, the per-shard primitive gives
possession, and one of the two must change. Q68-1 is a three-way council decision (accept the RO
deviation / BLS / Fortress) whose inputs are all now on the table; more reading will not improve it.
**Both should go to a single council session together with F-01 and Q68-3.**

---

## Domain G — Transparency logs and public randomness `[GAP]` — **PROMOTED from Band 3**

**ADRs:** 015, 017, 032, 059 · **Findings:** F-22, F-23, F-68 · **Questions:** Q70-1

v2 placed this in Band 3 ("cheap to change now, expensive later"). Paper 70 moved it. Fortress's
liability mechanism — the thing that closes F-22 without paying BLS pairing costs on every audit —
requires an unpredictable, publicly-reconstructible randomness source, and ADR-015's Transparent
Merkle Log is the only candidate Vyomanaut has. **R-23 is now a dependency of the Proof of Storage
milestone as well as its own**, which is what Q70-1 exists to make explicit rather than accidental.

R-23, R-24, R-25 carry forward from v2 with their search strings intact. One new topic.

**Must-read**

| Work | Why |
| --- | --- |
| **Chuat, Szalachowski, Perrig, Laurie & Messeri, "Efficient gossip protocols for verifying the consistency of certificate logs"** (CNS 2015) | R-23's canonical source; v2 already names it in the search string but it has never been read. |
| **Crosby & Wallach, "Efficient Data Structures for Tamper-Evident Logging"** (USENIX Sec 2009) | R-24. The history-tree construction, which is the answer to F-23/F-68's "tamper-evidence over a store that physically permits UPDATE." |
| **Meiklejohn et al., "Think Global, Act Local: Gossip and Client Audits in Verifiable Data Structures"** (2020) | The modern restatement, and the closest to Vyomanaut's actual topology (many light clients, one operator). |
| **Tomescu et al., "Transparency Logs via Append-Only Authenticated Dictionaries"** (CCS 2019) | Read for cost, if the log has to carry per-provider lookups rather than only a root. |
| **The Go checksum database (`sum.golang.org`) and Sigsum** | Deployed, non-blockchain transparency logs with a gossip story. Grey literature, but LTS Session 19.0.1 already *requires* `sum.golang.org` verification — the project is a consumer of one of these before it is a builder of one. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-50** `[NEW]` | **Public randomness beacons and reconstructible unpredictability** | Produces a value that is unpredictable before a deadline, publicly reconstructible after it, and verifiable by a party that trusts nobody — and states its liveness dependency. | Requires a threshold of *staked* participants, or a chain for finality. | Verifiable Delay Functions on their own: a VDF gives delay, not distribution. We need both, and the paper must say which it provides. |

```
R-50  ACM   Title:("randomness beacon" OR "distributed randomness") AND
            Abstract:(unpredictab* AND verifiab*)
      IEEE  ("Abstract":"public randomness") AND ("Abstract":bias-resistant OR "unbiasable")
      also  NIST Randomness Beacon (interoperability spec) · drand / League of Entropy
            (operational, deployed, and free to consume — price *consuming* before *building*)
```

**Substitution test (R-50):** ADR-060 draws one 32-byte seed per `(provider, file, day)`. At 1,000
providers × 1,000 files that is 10⁶ beacon-derived values per day. A beacon emitting one value per
30 s cannot supply them directly — the design must be *one beacon value per epoch, expanded
locally*, and the paper must survive that expansion without losing unpredictability. Show the
expansion in the triage note.

**The cheap move to price first:** consume `drand` rather than build. Fortress needs
`GetRandomness` to be unpredictable and publicly reconstructible; it does not require Vyomanaut to
operate the source. If consuming an external beacon is acceptable, R-50 becomes an integration
question and Q70-1's re-prioritisation largely dissolves. **Establish that before searching.**

---

## Domain P — Confidentiality-preserving repair `[GAP]` — **BLOCKER, unchanged**

**ADRs:** 004, 019, 020, 022, 026, 055 · **Finding:** F-69

Unchanged from v2 in substance — R-27, R-28, R-29 stand as written, and nothing in ADR-059→073
touches F-69. What has changed is that it now has a **named milestone** in `build_part4.md` with no
ADR behind it at all, and that ADR-059 adds a second reason to care: under the chosen scheme, the
party performing a repair assembles `k=16` shards *and* would need authenticator keys to re-tag
them. Domain P and Q68-3 are the same physical moment in the protocol seen from two directions.

**Must-read**

| Work | Why |
| --- | --- |
| **Pawar, El Rouayheb & Ramchandran, "Securing dynamic distributed storage systems against eavesdropping and adversarial attacks"** (IEEE Trans. Inf. Theory, 2011) | The secrecy-capacity result for regenerating codes. R-27's anchor; run the citation chain from here, not from keywords. |
| **Shah, Rashmi, Kumar & Ramchandran, "Distributed storage codes with repair-by-transfer…"** (IEEE Trans. Inf. Theory, 2012) | R-28 exactly: a helper contributes without any party assembling `k`. |
| **Rawat, Koyluoglu, Silberstein & Vishwanath, "Optimal locally repairable and secure codes for distributed storage systems"** (IEEE Trans. Inf. Theory, 2014) | Secrecy *and* repair locality together, with the storage cost stated. |
| **Goparaju, El Rouayheb, Calderbank & Poor, "Data secrecy in distributed storage systems under exact repair"** (2013) | Same first author as Paper 22 — the citation chain is already half-walked. |
| **Resch & Plank, AONT-RS** — *already Paper 16* | v2's "cheap first move" stands: re-read the security section against a repair-path adversary **before** searching outward. |

**Add to R-27/R-28's reject-if:** *Adjacent, not this* — secure regenerating codes against an
**eavesdropper on links** are the well-populated case; Vyomanaut's adversary is the **helper node
itself**, which is a different (and less-studied) model. Screen abstracts for which one they treat.

**Substitution test:** any secrecy result must be evaluated at `(n=56, k=16, d=55)`. A construction
whose secrecy overhead is stated as "ℓ symbols sacrificed" needs `ℓ` computed at those values and
compared against RS(16,56)'s existing 3.5× expansion before it is called adoptable.

---

## Domain K — Decoupling confidentiality from reconstruction `[GAP]` — **BLOCKER, unchanged**

**ADRs:** 014, 022, 029, 031 · **Findings:** F-34, F-67

R-30 and R-31 stand as written. One addition to the must-read list that v2's search strings would
probably not surface, because it predates the vocabulary:

**Must-read**

| Work | Why |
| --- | --- |
| **Krawczyk, "Secret Sharing Made Short"** (CRYPTO 1993) | The computational-secret-sharing construction that AONT-RS descends from, and the first statement of the shape R-30 is looking for: privacy threshold decoupled from reconstruction threshold at bounded cost. Read this before any 2020s ramp-scheme paper. |
| **Blakley & Meadows (1984); Yamamoto (1986)** — ramp secret sharing | The primitive itself. Old, and old is fine for mechanisms. |
| **Cidon et al., "Copysets: Reducing the Frequency of Data Loss in Cloud Storage"** (USENIX ATC 2013), and **"Tiered Replication"** (ATC 2015) | R-31. Constrains *which sets of nodes* may hold a stripe — precisely the "collusion-resistant placement under a diversity budget" shape, arrived at from durability. The math transfers; the objective changes. |

**Substitution test (R-30):** the fix must raise the confidentiality threshold above `k=16` while
leaving `k=16` as the reconstruction threshold, at a storage cost stated as a multiple of the
existing 3.5×. Compute it. A scheme costing another 2× is not adoptable and should be triaged out
during shortlisting, not after a full read.

**v2's closing instruction stands and is worth repeating:** whatever the council rules on F-34, the
fix lands as a **compiler-enforced `NetworkProfile` invariant** alongside
`TestProfileShardSizeIsConstant` — F-67 is structural, so a one-off fix returns with the next
profile.

---

## Domain B′ — Audit at fleet scale, re-scoped `[GAP]`

**ADRs:** 002, 012, 023, 033, 059, 060 · **Findings:** F-02, F-03, F-40, F-58, F-59 ·
**Questions:** Q66-2, Q67-1, Q68-2

**ADR-060 closed most of v2's Domain B by arithmetic.** Sampling at 1% takes INSERT/s from 3,319 to
11.6, removes the nonce index entirely, and takes provider disk from 60.3 min/day to 0.6. F-02,
F-58, F-59 close with it. R-06, R-07 and R-08 are therefore **retired** — R-08 in particular is now
satisfied *by construction* (each sampled chunk is one contiguous 256 KB vLog read), which is the
best possible outcome for a research topic.

What remains is not what v2 listed. R-05 survives, re-aimed. Everything else moved to Domain A′
(R-48, R-49) or became a derivation:

- **Q66-2 is a derivation, not research.** ADR-060 §"The economic constraint" already re-derives
  SHELBY Condition (i) under proportional sampling and gets `S ≥ V/2,909` against the reading
  list's `99×` — three orders of magnitude in the *opposite* direction. ADR-060 flags that this
  needs re-derivation against SHELBY's actual Theorem 1 (Paper 37, in the corpus, unread since the
  sampling design existed). **That is one afternoon with a paper we already own.** It is the single
  highest value-per-hour item in this entire document, and it is not a search.

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-05** | **Batch and aggregate verification across owners** | One verification covers many chunks across **many owners' key sets** — the axis ADR-060 left growing (rows now scale with files-per-provider, a quantity nobody bounds). | Batches within one owner's data; ADR-059's aggregate response already does that. | Batch *signature* verification (Ed25519 batching). Useful separately for the receipt path, not this. |

```
R-05  ACM   Abstract:("batch auditing" OR "batch verification") AND Abstract:("multiple owners"
            OR "multi-user" OR "cross-user") AND Abstract:(storage OR cloud)
            + standing exclusion set (blockchain floods this one badly)
```

**Substitution test (R-05):** at 5,000 providers × 5,000 files ADR-060 gives 289 INSERT/s. A batching
scheme is only worth adopting if it changes the *growth axis*, not the constant. State which.

---

# Band 2 — Invalidates a load-bearing parameter

## Domain D — Churn, availability, lifetime `[STALE]` — **HIGHEST LEVERAGE, unchanged**

**ADRs:** 005, 006, 007, 009, 010, 047, 053, 057 · **Findings:** F-06, F-15, F-16, F-72

R-12 → R-15 stand exactly as v2 writes them. Nothing in ADR-059→073 touched this and nothing will
until it is measured. It has its own LTS milestone ("Population Re-basing") which is the correct
recognition of its size.

**Must-read / where to actually look** — this domain's problem is that the *academic* literature is
thin and the *operational* data is not:

| Source | Why |
| --- | --- |
| **ProbeLab / Nebula IPFS network measurement reports** (2023–2025) | The modern successor to Paper 20's data, from the same measurement lineage, and updated continuously. Grey literature; cite the dataset and the methodology, not a paper. |
| **Storj published node-churn and node-retirement statistics** | R-13's *incentivised* population, which Paper 20's 87.6%-under-8h figure explicitly is not. |
| **Klein & Moeschberger, *Survival Analysis: Techniques for Censored and Truncated Data*** | R-14. This is a textbook problem, not a research problem — right-censored MTTF from a truncated observation window is solved, in a different field. Do not search for it as a distributed-systems topic. |
| **TRAI performance reports; M-Lab and Ookla open datasets** | R-15 and R-17's India-specific inputs. Research note, not a `paper-NN`. |

**Substitution test (R-12/R-13):** the paper must report a distribution whose median session length
can be compared against the **72-hour departure threshold** and the **24-hour poll**. A mean is not
enough — the entire objection to Bolosky is that `σ=1.9 h` on a nightly absence is implausible for
consumer hardware, and a mean would have hidden that.

---

## Domain E — Correlated failure and the bootstrap regime `[GAP]`

**ADRs:** 001, 003, 004, 005, 008, 014, 029, 055 · **Findings:** F-28, F-34 · **Questions:** Q63-1

R-16 → R-19 stand. Two must-reads that v2's search strings would very likely miss, both canonical
and both absent from a corpus that has Dalle and Nath but not these:

**Must-read**

| Work | Why |
| --- | --- |
| **Ford et al., "Availability in Globally Distributed Storage Systems"** (OSDI 2010) | The reference measurement of correlated failure in a real fleet, with statistical models for the effect of placement and redundancy choices. Absent from the corpus. R-16's anchor. |
| **Haeberlen, Mislove & Druschel, "Glacier: Highly durable, decentralized storage despite massive correlated failures"** (NSDI 2005) | A *decentralised* system designed explicitly against massive correlated failure — the closest structural analogue to Vyomanaut in the entire literature, and it is not in the corpus. |
| **Cidon et al., Copysets** (ATC 2013) — *shared with Domain K* | One measurement campaign, two domains. |

**Substitution test (R-16):** run at the ADR-029 gate — **56 providers, 5 ASNs, 20% cap**. Any
durability model that assumes failure domains are abundant relative to stripe width is describing a
regime Vyomanaut is never in at launch. `5 × 20% = 100%`: the cap is exactly saturated and any two
ASNs is 40% of the network. State that substitution in the triage note before drafting.

**Campaign note (unchanged and still right):** R-17 pairs with R-15, R-22 and the Band 6 NAT items.
One measurement pass, five answers. Sources are TRAI, M-Lab, Ookla and CEA/state-utility outage
data — a research note, not a `paper-NN`.

---

## Domain C — Wide stripes, repair I/O, small objects `[GAP]`

**ADRs:** 003, 004, 026 · **Findings:** F-29, F-30, F-43, F-44 · **Questions:** Q62-1, Q62-2, Q64-1

R-09 → R-11 stand. **The whole domain is gated on Q62-1** and `build_part4.md` says so: *"ADR-026 is
escalated to council; the entire single-block-repair literature is downstream of a repair-policy
question ADR-026 never asked."* Under `r₀=8`, if all repair gates on `available ≤ 24`, single-block
repair is ~0% by construction and every candidate family delivers a 0% saving. **Do not spend
search budget here until Q62-1 rules.** One new topic, which is *not* gated on Q62-1 because it is
about redundancy policy rather than code family:

**Must-read**

| Work | Why |
| --- | --- |
| **Kadekodi, Rashmi & Ganger, "Cluster storage systems gotta have HeART"** (FAST 2019), and **"Tiger: Disk-adaptive redundancy without placement restrictions"** | R-64. Adapts redundancy to *observed* failure rates per device population rather than a fixed worst case. Vyomanaut has per-provider reliability scores and measured throughput and feeds neither into stripe construction — this is the literature for doing so. |
| **Li, Yang & Lee, "Repair Pipelining for Erasure-Coded Storage"** (USENIX ATC 2017) | Repair *latency* rather than repair bandwidth. Relevant if ADR-055's emergency-eject path ever needs a bounded repair time. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-64** `[NEW]` | **Redundancy adaptation from observed reliability** | Varies `(k, n)` or placement per device population from measured failure behaviour, and states the transition cost when a population is re-classified. | Requires a homogeneous fleet with a known AFR curve — Vyomanaut's providers are heterogeneous by definition. | Hot/cold *access*-based tiering (Papers 59–61, and Q59-1). Same shape, different signal: this is reliability-driven, not hotness-driven. |

```
R-64  IEEE  ("Abstract":"disk-adaptive" OR "adaptive redundancy") AND ("Abstract":"failure rate"
            OR AFR) AND ("Publication Year":2018-2026)
      ACM   Title:(redundancy AND (adaptive OR heterogeneous)) AND Abstract:("failure rate")
      chain forward-cite HeART (FAST 2019)
```

**Substitution test (R-64):** Vyomanaut's population signal is `mv_provider_scores`, not a
manufacturer AFR curve. A scheme that needs a per-model reliability history cannot run here. State
what signal the paper needs and whether Vyomanaut has it.

**Correction, unchanged from v2 and still outstanding:** ADR-026's Clay rejection states
`α ≥ 40^16`; the standard MSR bound at `(56, 16, 55)` gives `40² = 1,600` — a 164-byte sub-chunk.
Clay very likely stays ruled out on I/O grounds. Recompute, show the substitution, and change the
argument.

---

# Band 3 — Structural stability `[NEW BAND]`

**This band did not exist in v2 and is the largest single addition in v3.** Its justification is
empirical rather than theoretical: ADR-070 ran *one* multi-stage seam, live, for the first time, and
found **thirteen defects**, every one invisible to a passing unit-test suite, several
self-documented as known gaps years earlier. ADR-070's own conclusion is the right generalisation —
*"a component correctly implementing its own documented contract, defeated by a neighbouring
component that either didn't exist yet or disagreed with it at the boundary."*

The LTS bar in ADR-062 is *"outperforms every competitor; thousands of desktops."* Nothing in
Bands 1–2 gets you there, because they are about whether the design is *correct*, and this band is
about whether the implementation is *true to it* and whether the numbers you publish about it can
be defended. The corpus has zero coverage of any of it.

---

## Domain Q — Deterministic simulation, fault injection, seam verification `[GAP]` `[NEW]`

**ADRs:** 069, 070, 071, 072, 073 · **Findings:** F-070-1 … F-070-13, F-LTS-03

**Must-read**

| Work | Why |
| --- | --- |
| **Yuan et al., "Simple Testing Can Prevent Most Critical Failures: An Analysis of Production Failures in Distributed Data-Intensive Systems"** (OSDI 2014) | Read this first, before any of the others. Its central finding — that a large majority of catastrophic failures come from incorrect handling of *non-fatal* errors, and that most are reproducible by simple tests — is ADR-070's thirteen findings restated as a general result four years before they happened. It also tells you which tests to write. |
| **Alvaro, Rosen & Hellerstein, "Lineage-driven Fault Injection"** (SIGMOD 2015) | Derives *which* faults to inject from the lineage of a successful outcome, rather than injecting randomly. The principled version of what ADR-070 did by hand. |
| **The FoundationDB paper** (SIGMOD 2021) | Deterministic simulation testing as the primary correctness strategy for a storage system. The closest thing to a blueprint for what Vyomanaut's `--sim-tier` harness could become. |
| **Kingsbury & Alvaro, "Elle: Inferring Isolation Anomalies from Experimental Observations"** (VLDB 2020), and the Jepsen reports | Vyomanaut runs Postgres with RLS, a gossip cluster, and six non-I-confluent operations under ADR-013. Nothing currently checks the resulting histories for anomalies. |
| **Leesatapornwongsa et al., "SAMC: Semantic-Aware Model Checking"** (OSDI 2014); Gunawi et al., "FATE and DESTINI" (NSDI 2011) | Systematic state exploration for distributed protocols — the alternative to injection when the state space is small enough, which for the provider lifecycle it is. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-51** | **Deterministic simulation of a distributed system's own implementation** | Runs the *real* code under a controlled scheduler and clock, with reproducible seeds; states what it can and cannot make deterministic (DNS, disk, wall-clock). | Tests a *model* of the system — ADR-069 already rejects that option for exactly this reason. | Property-based testing of a single component. Useful, and not the seam problem. |
| **R-52** | **Fault injection targeted at inter-component seams** | Chooses injection points from the system's own dataflow or lineage rather than uniformly; reports bugs-found-per-run, not just coverage. | Chaos-engineering-as-practice papers with no selection principle — random injection at Vyomanaut's state-space size finds nothing. | Network-partition testing alone. ADR-070's findings were auth-header, encoding and identity mismatches, none of which a partition surfaces. |
| **R-53** | **Lightweight formal specification of lifecycle state machines** | A specification of a multi-stage state machine that is *checked against the implementation*, not only against itself; reports the cost of keeping the two in sync. | Full refinement proofs — the cost is not payable and the payoff is in the wrong place. | Verifying a consensus protocol. Vyomanaut's problem is `PENDING_ONBOARDING → VETTING → ACTIVE → DEPARTED` and who writes which column, not agreement. |

```
R-51  ACM   Title:("deterministic simulation" OR "simulation testing") AND
            Abstract:(distributed AND (reproducib* OR deterministic) AND implementation)
      USENIX site:usenix.org "deterministic simulation" testing distributed storage
R-52  ACM   Title:("fault injection") AND Abstract:(distributed AND (systematic OR
            "lineage" OR targeted)) AND Abstract:(bugs OR failures)
            + NOT ("fault injection" AND (hardware OR "soft error" OR radiation))   ← heavy pollutant
R-53  IEEE  ("Abstract":"model checking") AND ("Abstract":"state machine") AND
            ("Abstract":implementation AND conformance)
      ACM   Abstract:(TLA+ OR "P language" OR stateright) AND Abstract:(industrial OR "case study")
```

**Substitution tests**

- **R-51:** Vyomanaut's simulation target is `--sim-count` / `--sim-tier` (ADR-069) with **real
  Postgres** in the loop. Determinism stops at the database. Any paper that assumes an all-in-process
  system needs its assumption stated against that boundary before drafting.
- **R-52:** the thirteen `F-070-N` findings are the benchmark. Ask of every candidate technique:
  *would it have found F-070-4 (a missing `Authorization` header) and F-070-11 (a cache TTL with no
  refresh loop)?* If the answer is no for both, it is the wrong technique for this system.
- **R-53:** the provider lifecycle has ~5 states and ~8 transitions, spread across
  `cmd/microservice`, `internal/scoring`, `internal/vettingchunk` and `cmd/provider`. That is small
  enough to specify exhaustively — which is exactly why the failure was *distribution* across
  packages, not state-space size. Judge candidates on whether they check the *seam*, not the machine.

**The cheapest item in this domain is not literature.** ADR-070's own closing rule — *"a milestone
that adds a new stage to a multi-stage provider lifecycle must be verified against the stage
immediately before and after it, live, before the milestone is considered complete"* — should become
a **VERIFY-block requirement in the build skill**, machine-checked, the same way `FILES:`/`VERIFY:`
already are. Do that before any of R-51–R-53.

---

## Domain S — Scale-claim validity `[GAP]` `[NEW]`

**ADRs:** 065, 068, 069 · **Findings:** F-LTS-03 · **Questions:** Q-SIM-1, Q-LAB-1, Q-M19-2

ADR-069 is a good design and it makes a claim that needs defending: 10,000 coordination-plane nodes
on one 32 GB desktop, with a mixed population of real, lightweight and synthetic providers. §5 is
right that *"simulation runs are never a substitute for physical-fleet runs"* and that tier
composition must be recorded with every result. What no document says is **how to get from a
mixed-fidelity run to a defensible number** — a sampling error, a confidence interval, a statement
of what the synthetic tier cannot exercise. ADR-069 records one such limit (repair) honestly. There
will be others, and finding them by publishing a wrong number is expensive.

This matters beyond engineering: ADR-065's rationale explicitly names *"the minimum evidentiary
standard for the research paper."*

**Must-read**

| Work | Why |
| --- | --- |
| **Jansen, Tracey & Goldberg, "Once is Never Enough: Foundations for Sound Statistical Inference in Tor Network Experimentation"** (USENIX Security 2021) | The single best fit in this document. It is about drawing conclusions from *sampled* network simulations, argues that single-run results are not meaningful without significance analysis, and supplies both modelling methodology and analysis methods, demonstrated over 420 simulations. Substitute "sampled Tor network" for "three-tier synthetic population" and it is written about ADR-069. |
| **Shadow** (Jansen et al.) — the simulator itself, and its scalability/accuracy work | A discrete-event simulator that runs *real application binaries*. The design point ADR-069 is reaching for, already built and validated. |
| **Gouveia et al., "Kollaps: decentralized and dynamic topology emulation"** (EuroSys 2020) | Topology emulation at scale without a central emulator, which is the constraint a single 32 GB desktop imposes. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-54** | **Fidelity of mixed real/synthetic node populations** | Quantifies what a reduced-fidelity node *fails* to reproduce, and gives a method for bounding the induced error, not just a caveat. | Pure emulation-vs-hardware comparison with no reduced-fidelity tier. | Digital-twin literature. Different objective — fidelity to a *physical* plant, not to a protocol. |
| **R-55** | **Statistical inference from sampled network experiments** | Gives a procedure for confidence intervals and required run counts over a *sampled* network, and states how the sampling model is validated. | Single-run performance evaluations, which is what it is arguing against. | Benchmarking-methodology papers about microbenchmark variance. Right instinct, wrong scale. |

```
R-54  ACM   Title:(emulation OR simulation) AND Abstract:(fidelity AND scalab*) AND
            Abstract:(network OR "distributed system")
      IEEE  ("Abstract":"scaled-down" OR "reduced fidelity") AND ("Abstract":validat*)
R-55  USENIX site:usenix.org "statistical" experimentation simulation "confidence interval" network
      ACM   Abstract:("sampling error" AND (simulation OR experimentation) AND network)
      chain forward-cite "Once is Never Enough" (USENIX Sec 2021)
```

**Substitution tests**

- **R-54:** ADR-069's three tiers are Real (full daemon, ~150 MB), Lightweight (transport + protocol,
  in-memory chunk store) and Synthetic (protocol surface, one 32-byte seed). A method that requires
  the reduced tier to be *statistically similar* to the real one fails here — synthetic nodes are
  behaviourally identical on the coordination plane by construction and absent on the storage plane
  entirely. That is not noise; it is a structural hole, and the method must handle holes.
- **R-55:** ADR-069's harness is meant to measure NFR-043's Postgres ceiling. Ask what run count the
  method requires to state that ceiling with a confidence interval, and whether that many runs fit
  in the harness's wall-clock budget. If not, the ceiling is a point estimate and must be published
  as one.

**Also fix F-LTS-03 before this domain is worth anything** — a synthetic tier that cannot answer the
audit primitive the LTS actually ships is not measuring the system.

---

## Domain O — Operational readiness `[GAP]` `[NEW]`

**ADRs:** 065, 066, 067, 068 · **Questions:** Q20-1, Q-M18-4, Q-M18-5

ADR-067's context paragraph contains the argument for this whole domain: *"A runbook nobody is paged
into is a document, not a control,"* and the five runbooks with no alert were the five covering
third-party and scheduled failure modes — the ones least diagnosable from first principles at 3 a.m.
ADR-068 adds the sharper case: relay slot exhaustion is simultaneously the **nearest** ceiling
(binds at N≈570 under 45% CGNAT) and the **slowest to remediate** (procure, deploy, warm — days),
and it had no instrumentation at all.

The corpus has four deployed storage networks, seventy-two papers, and nothing on operating one.

**Must-read**

| Work | Why |
| --- | --- |
| **Beyer et al., *Site Reliability Engineering*, ch. 4 (SLOs) and ch. 6 (Monitoring); *The SRE Workbook*, ch. 5 (Alerting on SLOs)** | Multi-window, multi-burn-rate alerting is the standard answer to ADR-068's "70% warning / 85% critical" shape, and it is derived rather than picked. Read before the thresholds are frozen. |
| **Rob Ewaschuk, "My Philosophy on Alerting"** | Short, and it is the source of the "every page must be actionable" rule that ADR-067's bijection is an implementation of. |
| **The clinical alert-fatigue literature** (e.g. Ancker et al., *BMC Med. Inform. Decis. Mak.*, 2017) | The only field with real *measured* data on what happens to responders as alert precision falls. Cross-field, deliberately: computing has opinions here, medicine has numbers. |
| **Inventory theory: base-stock and `(s, S)` policies under stochastic demand with procurement lead time** | ADR-068's relay problem is not a monitoring problem, it is a **reorder-point** problem — demand is provider growth, lead time is days, stockout is a provider that cannot join. Textbook (Zipkin, *Foundations of Inventory Management*). Naming it correctly gives the 70%/85% numbers a derivation instead of a judgement. |
| **Prometheus/OpenMetrics naming conventions** (spec, not paper) | ADR-066's grammar should be checked against the upstream convention it is approximating, including how the ecosystem handles the dimensionless case ADR-066 has to invent. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-56** | **SLO-based and burn-rate alerting** | Derives alert thresholds from an error budget and a target detection time, and reports the precision/recall trade-off explicitly. | Anomaly-detection-by-ML papers — Vyomanaut needs derivable thresholds a human can defend in a runbook, not a model. | APM vendor whitepapers. No methodology. |
| **R-57** | **Capacity provisioning under long procurement lead time** | Sets a reorder point from demand growth **variance** and lead time, not from a utilisation percentage. | Autoscaling papers — the whole difficulty is that relay capacity does *not* autoscale. | Datacenter capacity planning at fleet scale. Wrong lead time, wrong granularity, wrong cost structure. |
| **R-58** | **On-call and runbook efficacy** | Measures time-to-mitigate as a function of runbook presence/quality, or measures the cost of low-precision alerts on responders. | Incident post-mortem collections with no measurement. | Postmortem *culture* literature. Valuable, not a control. |

```
R-56  ACM   Abstract:("service level objective" OR "error budget") AND Abstract:(alert*)
      IEEE  ("Abstract":alerting AND ("false positive" OR precision)) AND ("Abstract":monitoring
            AND (threshold OR "burn rate"))
            + NOT ("intrusion detection" OR "network security")     ← dominates the word "alert"
R-57  SD    TITLE-ABSTR-KEY(("reorder point" OR "base stock") AND "lead time" AND capacity)
      IEEE  ("Abstract":"capacity planning") AND ("Abstract":"lead time" OR provisioning delay)
R-58  ACM   Abstract:("on-call" OR runbook OR "incident response") AND Abstract:(measure* OR
            empirical OR "time to")
      also  the alert-fatigue literature in PubMed — search "alert fatigue" override rate
```

**Substitution tests**

- **R-56:** ADR-068 fires warning at 70% for 1h and critical at 85% for 15m, on relay slots. Compute
  the detection time each gives against §27.5's growth model, and compare against the *actual* lead
  time to warm a new relay node. If the warning does not precede the ceiling by more than the lead
  time, the threshold is decorative.
- **R-57:** at 3 relay nodes = 384 slots, 45% CGNAT binds at N≈570 and the stated operational rule
  is "provision the 4th node before N=400." That is a reorder point of ~70% with an unstated demand
  variance. Substitute a growth rate and a lead time and check whether 400 survives.
- **R-58:** nine runbooks, nine-plus alerts, one operator. Any study assuming a rotating on-call team
  of six is describing a different failure mode.

**Q20-1 closes on ADR-068's `vyomanaut_cluster_relay_dependent_providers` gauge**, not on literature.
The 30%-vs-45% CGNAT question is worth ±280 providers of runway and it is now a measurement with a
mechanism. That is a research item that correctly stopped being one — record it as answered when the
gauge exists.

---

# Band 4 — Cheap to change now, expensive later

Domains **L**, **M**, **N** and **I** carry forward from v2 with their topics, search strings and
accept/reject criteria intact. Three amendments and one new domain.

## Domain L — Single-writer election `[GAP]` — must-reads added

**R-32 remains the high-value item** and v2's reasoning stands: if reservation-style bounded counters
work, the count of coordinated operations drops below six and ADR-013's central trade-off improves
materially.

| Work | Why |
| --- | --- |
| **Balegas et al., "Extending Eventually Consistent Cloud Databases for Enforcing Numeric Invariants"** (SRDS 2015), and "Putting Consistency Back into Eventual Consistency" (EuroSys 2015) | R-32 precisely: preserving a `≥ 0` floor by *reservation* rather than by coordinating every operation. Escrow debit is the textbook case. |
| **Burrows, "The Chubby Lock Service"** (OSDI 2006) | R-26's lease semantics and sequencer/fencing model, from the system that defined them. |
| **Kleppmann, "How to do distributed locking"** (2016, grey literature) | The clearest short statement of why a lease without a fencing token is not a lock — which is F-35/F-76 exactly. |

## Domain M — Local secrets `[GAP]` — must-reads added

| Work | Why |
| --- | --- |
| **Blocki, Harsha & Zhou, "On the Economics of Offline Password Cracking"** (IEEE S&P 2018) | R-34, and it is stated in the currency the decision needs: guesses per rupee at given parameters. ADR-020's `t=3/m=64 MB` is an interactive setting facing an offline attack. |
| **Bonneau, "The Science of Guessing"** (IEEE S&P 2012) | R-35. Reports a *distribution* of guessing difficulty over ~70M real passwords — so "what fraction of owners are actually protected" becomes answerable. |
| **Jarecki, Krawczyk & Xu, "OPAQUE: An Asymmetric PAKE Protocol Secure Against Pre-Computation Attacks"** (Eurocrypt 2018) | R-36. Removes the oracle rather than raising its cost. |
| **Windows DPAPI `CryptProtectData` documentation; macOS Keychain ACLs; libsecret** | R-37. Primary sources, not literature. The requirement is **no elevation** (ADR-042) and **unattended start** (ADR-047, ADR-051) — check both against the actual API guarantees. |

**v2's cheapest option stands and should be priced first:** do not store `pointer_ciphertext`
server-side at all. It trades ADR-020's Scenario 4 recovery for removing the entire oracle. Council
question, but it belongs on the table before four search topics are funded.

## Domain N — AEAD nonce handling and RNG failure `[GAP]` — **now upstream of the upload path**

R-38 and R-39 stand, and **F-LTS-01 raises this domain's priority**. Under ADR-073, `chunk_id` is a
content hash of AONT-RS ciphertext, and the capability token binds to it. Ciphertext uniqueness now
carries a *second* load: not only ADR-019's confidentiality but the token-to-assignment binding
ADR-072 relies on. A repeated AONT key `K` no longer merely leaks — it produces colliding
`chunk_id`s across files.

| Work | Why |
| --- | --- |
| **Heninger, Durumeric, Wustrow & Halderman, "Mining Your Ps and Qs"** (USENIX Security 2012) | R-39 exactly: measured, in-the-wild frequency of entropy failure producing duplicate keys. Turns "theoretically fragile" into a probability. |
| **Ristenpart & Yilek, "When Good Randomness Goes Bad: Virtual Machine Reset Vulnerabilities"** (NDSS 2010) | The VM-cloning and snapshot-restore case specifically — the scenario ADR-019's `nonce = 0` is most exposed to. |
| **Gueron & Lindell, AES-GCM-SIV** (CCS 2015) and **RFC 8452**; **libsodium's XChaCha20-Poly1305** | R-38. Both properties matter: 192-bit collision margin *and* what survives a nonce repeat. |

## Domain H — Content-addressed identity under adversarial clients `[GAP]` `[NEW]`

**ADRs:** 072, 073 · **Findings:** F-LTS-01, F-LTS-02

Small domain, entirely created by ADR-073. `chunk_id = SHA-256(chunk_data)` is a **content address**,
which is a different object from the opaque identifier ADR-072's security argument assumed. Content
addresses have well-known properties — equality across files is observable, the namespace is
client-controllable, and the server cannot verify a submission it never sees the input to — and
those properties have a literature.

**Must-read**

| Work | Why |
| --- | --- |
| **Harnik, Pinkas & Shulman-Peleg, "Side Channels in Cloud Services: Deduplication in Cloud Storage"** (IEEE S&P Magazine, 2010) | The canonical statement of what a client learns from a content-addressed namespace it can probe. Short. |
| **Bellare, Keelveedhi & Ristenpart, "Message-Locked Encryption and Secure Deduplication"** (Eurocrypt 2013) | The formal treatment of content-derived keys and identifiers, including exactly which security properties survive. |
| **Keelveedhi, Bellare & Ristenpart, "DupLESS: Server-Aided Encryption for Deduplicated Storage"** (USENIX Security 2013) | The mitigation shape, if one is needed. |
| **Douceur et al., "Reclaiming space from duplicate files in a serverless distributed file system"** (ICDCS 2002) | Convergent encryption's origin. Read for the assumption list. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-59** | **Adversarial submission into a content-addressed namespace** | Treats the identifier as **client-supplied and server-unverifiable**, and states what an attacker gains by choosing collisions or squatting an existing address. | Assumes the server hashes the content itself — that is the case Vyomanaut structurally cannot be in (ADR-019: the service never sees plaintext, and under ADR-073 never sees ciphertext at assignment time either). | Hash-collision resistance as a cryptographic property. SHA-256 is fine; the question is namespace *policy*, not the hash. |
| **R-60** | **Deduplication side channels under client-side encryption** | Quantifies what an observer learns from identifier equality across owners, under a randomised-key encryption scheme. | Assumes convergent encryption (deterministic keys) — AONT-RS uses a fresh random `K`, so the naive attack does not apply and a paper that only treats that case is not about us. | Deduplication *efficiency* work. We do not dedup and do not want to. |

```
R-59  ACM   Title:("content-addressed" OR "content addressable") AND Abstract:(adversar* OR
            malicious OR "namespace")
      IEEE  ("Abstract":"client-side hash" OR "client-supplied identifier") AND ("Abstract":
            attack OR forgery OR collision)
R-60  ACM   Title:(deduplication) AND Abstract:("side channel" OR "information leakage") AND
            Abstract:(encrypt*)
            + NOT (efficiency OR "space savings" OR "chunking algorithm")
```

**Substitution test:** under ADR-073, `chunk_id` is `SHA-256` over **AONT-RS ciphertext with a fresh
per-segment random key**. Two identical plaintexts therefore produce different `chunk_id`s. So the
classic convergent-encryption attacks do **not** apply — and that is a property worth stating
explicitly in an ADR, because it is currently an accident of ADR-019 rather than a documented
invariant of ADR-073. **The residual risk is the reverse direction:** if `K` ever repeats (Domain N,
F-47), `chunk_id`s collide across files and capability tokens become transferable. Compute the
collision probability under the actual RNG assumptions before deciding this domain is small.

## Domain I — Coordination DoS, reputation gaming, relay `[GAP]`

R-40, R-41, R-42 carry forward. **R-42 is promoted within the domain** — ADR-068 establishes relay
capacity as the binding constraint, so relay sizing is no longer a background item. Note the split:
R-42 asks *how much relay capacity is needed under burst*; R-57 (Domain O) asks *when to buy it*.
They are different questions with different literatures and should not be merged.

| Work | Why |
| --- | --- |
| **Hoffman, Zage & Nita-Rotaru, "A Survey of Attack and Defense Techniques for Reputation Systems"** (ACM Computing Surveys, 2009) | R-41's map. Read for the attack taxonomy against *time-windowed* scores specifically — Vyomanaut's three-window design is the structure these exploit. |
| **Juels & Brainard, "Client Puzzles"** (NDSS 1999) | R-40's anchor. Our clients are *registered*, which is a lever most DoS work does not have — read for what registration buys. |

---

# Band 5 — Economics and compliance

## Domain J — Payments, Sybil bound, carbon ledger `[WEAK]`

R-43 → R-46 stand as v2 writes them, including the correct observation that **R-44 is not academic
literature** (RBI PA/PG guidelines and Razorpay's escrow/Route documentation are the primary
sources) and that **R-46 is rescoped** to subtract accelerated replacement rather than only add
avoided datacenter hardware. Two additions.

**Must-read**

| Work | Why |
| --- | --- |
| **Gupta et al., "Chasing Carbon: The Elusive Environmental Footprint of Computing"** (HPCA 2021) and **"ACT: Designing Sustainable Computer Systems with an Architectural Carbon Modeling Tool"** (ISCA 2022) | R-46 and Q44-1. ACT is a *tool*, which is what "a defensible, citable per-terabyte figure" actually requires — a number with a stated model behind it, not a number from a vendor report. |
| **Tannu & Nair, "The Dirty Secret of SSDs: Embodied Carbon"** (HotCarbon 2022 / SIGEnergy 2023) | Per-device embodied carbon for **storage** specifically, with an HDD-vs-SSD comparison. This is Q44-1's missing half, and it is six pages. |
| **CarbonClarity** (HPCA 2024) and the storage-emissions call-to-research literature | Embodied-carbon estimates carry large uncertainty. ADR-044's claim needs error bars, not a point estimate, or it will not survive a hostile reading. |

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-63** `[NEW]` | **Time-weighted attribution of periodic revenue under mid-period reassignment** | Prorates a fixed periodic charge across parties whose tenure changed mid-period, with an **idempotent** settlement rule. | Requires recomputation after the fact — ADR-061's `DEPOSIT` idempotency key `SHA-256(provider_id‖file_id‖billing_period)` carries no amount component, so a corrected split has no retry path. | Revenue-recognition accounting standards. Right concept, wrong granularity, and they assume a mutable ledger. |

```
R-63  SD    TITLE-ABSTR-KEY((proration OR "time-weighted") AND allocation AND (billing OR
            settlement)) AND NOT TITLE-ABSTR-KEY(blockchain)
      ACM   Abstract:("cost allocation" AND (fairness OR proportional) AND (dynamic OR churn))
```

**Substitution test (R-63):** ADR-061's own recorded trade-off is that *"a provider who takes over a
shard via mid-period repair reassignment receives that period's full per-shard credit even though a
different provider served most of the period."* Quantify it: at ADR-004's repair frequency and a
one-month billing period, what fraction of `(provider, file, period)` tuples involve a mid-period
handover? If it is under ~1%, F-LTS-04 resolves as "flat split, documented," and this topic closes
without a search. **Run that count before funding R-63.**

**F-41 remains the sharpest item in this domain** and is unchanged: ADR-024 §5 justifies seizure as
cost recovery while §7 says the operator bears no repair cost. A forfeiture defended by a cost the
same document denies will not survive a consumer complaint. That is a document-consistency fix, not
a search.

---

# Band 6 — Multi-year lifetime `[NEW BAND]`

The demo is frozen and disposable. **The LTS is, by name, meant to be maintained for years**, and
ADR-062's bar — *"iterated indefinitely"* — creates obligations no v2 domain covers, because v2 was
written for a system with a launch date rather than a lifetime.

## Domain T — Cryptographic agility and post-quantum sequencing `[GAP]` `[NEW]`

**Questions:** Q72-1 · **Related:** ADR-059 (private scheme's quantum posture), Paper 72

Q72-1 states the risk correctly and then correctly declines to act on it: *"this question exists so
that 'post-quantum' is not answered piecemeal by whichever ADR happens to be open when the
requirement lands."* That is a design-process requirement — **crypto-agility** — and it is a cheap
property to build in and an expensive one to retrofit. Ed25519 appears in provider signing, receipt
signing, registration, heartbeat and capability tokens; ADR-059 freezes `(p, s)` before the first
authenticator is generated *"in any environment that will outlive a database reset."*

This domain is **two topics and a design rule**, not a large campaign. Keep it small deliberately.

| # | Topic | Accept if | Reject if | Adjacent, not this |
| --- | --- | --- | --- | --- |
| **R-61** | **Crypto-agility as a protocol property** | Specifies how a deployed protocol carries an algorithm identifier and negotiates a migration **without a flag day**, and states the cost of the version field. | Assumes coordinated simultaneous upgrade — Vyomanaut cannot upgrade thousands of desktop daemons at once (ADR-051 is a two-phase updater, not a flag day). | Hybrid TLS key exchange. Solves a *confidentiality-now* problem; ours is a *signature/authenticator* problem. |
| **R-62** | **PQ signature cost on consumer hardware** | Reports signing/verification time and signature size for NIST-standardised schemes (ML-DSA / FIPS 204, SLH-DSA / FIPS 205) on consumer-class CPUs. | Reports only key sizes, or only server-class benchmarks. | PQ *KEM* benchmarks (ML-KEM). Different primitive, different cost profile. |

```
R-61  ACM   Title:("cryptographic agility" OR "algorithm agility") AND Abstract:(protocol AND
            migration AND deploy*)
      IEEE  ("Abstract":"crypto agility") AND ("Abstract":"backward compat*" OR transition)
R-62  IEEE  ("Abstract":"ML-DSA" OR Dilithium OR SPHINCS) AND ("Abstract":benchmark OR
            performance) AND ("Abstract":"embedded" OR "commodity" OR "consumer")
      also  NIST FIPS 203/204/205 (primary), and the pqm4 benchmark project
```

**Substitution tests**

- **R-61:** every wire format in IC §3.1, §4.1 and §4.3 must be checked for whether it carries an
  algorithm identifier. If it does not, adding one later is a protocol version bump — the same
  problem ADR-073 Option (C) was killed by (*"Frame 1's byte layout has no spare field"*). **Do this
  audit at M19, when the interfaces are being touched anyway.** It costs one byte per frame now and
  a version negotiation later.
- **R-62:** Ed25519 signatures are 64 bytes. ML-DSA-44 signatures are ~2.4 KB. Substitute into a
  1,040-byte audit response and a 262,252-byte upload frame, at ADR-060's audit volume, and state
  the bandwidth delta before anyone treats this as a drop-in.

**The design rule is the deliverable, not the papers.** If the M19 audit finds no algorithm
identifiers, the recommendation is a one-line ADR adding a version byte to the signing-input domain
prefixes — the mechanism ADR-059 already uses (`"vyomanaut/por/prf/v1"`). That pattern is already in
the codebase; it just is not applied consistently.

---

# Band 7 — Carried forward, and what is no longer research

## 7.1 — Items that moved out of the reading list entirely

ADR-065's `LaunchGate` struct plus `scripts/benchmarks/results/{id}.{profile}.json` is a **better
home** than a reading list for anything whose answer is a measured number on a named machine. Move
these there and stop tracking them as research:

| Was | Now | Where it lives |
| --- | --- | --- |
| **R-20** — EC library throughput | Q65-1's measurement 1 | `LaunchGate` entry, min-spec bench machine. Paper 65 already read; nothing more to search. |
| **R-21** — symmetric crypto on low-end hardware | Q-M18-1 | `LaunchGate`. SHA-256 over 70 GB/day is already settled by calculation at ≤0.54% of one core. |
| **R-22** — duty cycle and drive lifetime | Feeds R-46's second half | Half measurement, half literature. **Keep the literature half in Domain F**; move the measurement to the campaign in §5.2. |
| **Argon2id parameterisation** (ADR-020) | R-34 gives the policy; the benchmark gives the setting | `LaunchGate` + Domain M. |
| **RocksDB/Badger HDD compaction** (ADR-023, ADR-046, Q23-1, Q27-1, Q49-1) | Never was literature | `LaunchGate`, post-launch. |

**Domain F is therefore reduced to R-22's literature half only.** v2 was right that it is *"largely a
collection problem, not a discovery problem"* — ADR-065 supplies the collection mechanism, so the
domain mostly dissolves. That is a win, not a loss.

## 7.2 — Council questions, not reading gaps

Answerable now, from material already held. **These consume no search budget and several are
blocking.** Listed in the order they should be convened:

| Item | Why it is not research | Blocks |
| --- | --- | --- |
| **F-01 + Q68-1 + Q68-3 + Q66-1** — one session | All four inputs are read (Papers 66–72). Q68-1 is a three-way choice; Q66-1 is a terminology ruling; Q68-3 needs R-47's literature *and* a ruling — bring the literature to the same session. | ADR-059, ADR-060, the entire Proof of Storage milestone |
| **F-03** — the payment unit | ADR-060: *"the schema follows the payment unit, so F-03 must be ruled on before the audit scheduler is finalised."* | ADR-060, ADR-061 (F-LTS-04), ADR-012 |
| **Q62-1** — does `r₀=8` mean single-fragment repair never happens? | A specification question answerable from ADR-004's own text. | Domain C entirely, Band 0's papers 62 and 64, Q62-2, Q64-1 |
| **Q66-2** — SHELBY re-derivation | Paper 37 is in the corpus. This is arithmetic with the working shown. | ADR-060's approval, ADR-024 |
| **F-LTS-01, F-LTS-02** | Design questions arising from ADR-072/073 read together. | LTS upload path |
| **F-34 / F-69 council** (carried from v2) | Unchanged. | The Confidentiality milestone |
| **`k_hot` derivation** (ADR-018), **secrets-manager product choice** (ADR-027), **namespace runbook** (ADR-029) | Unchanged from v2's list of "three things that look like research and are not." | — |

## 7.3 — Carried, unchanged priority

vLog GC fine/coarse reclaim (ADR-023) — real divergence between `build.md` Session 5.1.5 and
ADR-023's prose; independent of everything above and runnable in parallel by a second reader.
Upload straggler / Storj `o` (ADR-003) `[V3]`. Adaptive polling from score history (ADR-008) `[V3]`
— downstream of Domain D. Post-graduation monitoring of forced-ceiling providers (ADR-053, Q57-2).
ISP data-plan sync (ADR-056) `[V3]`. Background execution continuity (ADR-057, Q58-1). Desktop shell
cluster (ADR-038–041, 049–052) — **re-tagged in v2 and the re-tag stands**: low-because-cheap-to-
reverse, not low-because-well-evidenced. Hot/Cold band parameters (ADR-018) — **stop searching**;
Papers 59–61 closed the literature question and Q59-1 is a product call. DHT necessity at small N
(ADR-001) — **suspended**, subordinate to F-09/F-10.

---

# §5 — Recommended sequence

## 5.1 — The critical path

```
NOW, in parallel with M17/M18 (demo work — this does not compete for the same hours)
  ├── Council session 1:  F-01 · Q68-1 · Q68-3 · Q66-1        ← bring R-47's literature to it
  ├── Council session 2:  F-03 (payment unit)                 ← unblocks ADR-060, ADR-061, ADR-012
  ├── Council session 3:  Q62-1 (r₀=8 repair policy)          ← may close Domain C at zero cost
  ├── Derivation:         Q66-2 against Paper 37              ← one afternoon, already-owned paper
  ├── Approvals:          F-LTS-03 … F-LTS-06                 ← four prescriptive fixes
  └── Read:               Chen & Curtmola 2017 + CODASPY 2015 ← Band 0, no search needed

M19 opens (LTS Foundation — needs no research)
  └── Audit every wire format for an algorithm identifier (R-61's substitution test) while the
      interfaces are open anyway. One byte now, a version negotiation later.

THEN, gating the Proof of Storage milestone
  R-47 → R-48 → R-49        A′  audit continuity across repair
  R-50                      G   randomness beacon — price consuming drand before building
        ↓
  R-27 → R-28 → R-29        P   confidentiality-preserving repair    ┐ both independent
  R-30 → R-31               K   decouple the two thresholds          ┘ of A′; run in parallel
        ↓
  R-12 → R-13 → R-14 → R-15 D   churn and lifetime      [one campaign — see 5.2]
  R-16 → R-17 → R-18 → R-19 E   correlated failure, small N
        ↓
  R-51 → R-52 → R-53        Q   seam verification       [starts now, never stops]
  R-54 → R-55               S   scale-claim validity    [before the first published number]
  R-56 → R-57 → R-58        O   operational readiness
        ↓
  R-23 → R-24 → R-25        G   transparency logs
  R-26 → R-32 → R-33        L   single-writer election  [R-32 highest value]
  R-34 → R-37               M   local secrets           [R-37 resolves F-75's dilemma]
  R-38 → R-39               N   nonce handling          [now upstream of the upload path]
  R-59 → R-60               H   content-addressed identity
  R-40 → R-41 → R-42        I   DoS, gaming, relay
  R-43 → R-46 · R-63        J   economics, compliance, carbon
  R-61 → R-62               T   crypto agility          [small, keep it small]
  R-09 → R-11 · R-64        C   erasure optimisation    [ONLY IF Q62-1 says single-fragment exists]
```

**If only one domain gets attention: A′.** Same answer as v2, different content — the reading is
done, but Q68-3 means every repaired shard is currently unauditable, and repair is routine.

**If only two: A′ and P.** Unchanged reasoning, now reinforced: Domain P and Q68-3 are the same
moment in the protocol viewed from confidentiality and from auditability. A council that rules on
both together will produce a better answer than two councils ruling separately.

**If only three: add Q.** This is a change from v2, which said D. D is still where the corpus is
dangerously stale — but D produces *better parameters* for a system, and Q produces *a system that
does what its documents say*. ADR-070 is the evidence: thirteen defects on one seam, each one a
component that passed its own tests. At "thousands of desktops," an unfound seam is not a tuning
error, it is an outage.

## 5.2 — Three measurement campaigns

Each answers several questions from one setup. Name them, schedule them once, do not re-derive their
scope every time a question comes up.

| Campaign | One setup, these answers | Notes |
| --- | --- | --- |
| **India network measurement** | R-17 (AS-level topology), R-18 (outage duration/blast radius), Q14-1 (UDP block rate), OQ-002/Q20-1 (CGNAT fraction), NAT/CGNAT items from v2's Band 5 | Sources are TRAI, M-Lab, Ookla, CEA/state-utility outage data. Research note, not `paper-NN`. **Q20-1's half now closes on ADR-068's gauge instead** — remove it from the campaign's scope and save the effort. |
| **One min-spec Indian desktop** | Q65-1 (RS(16,56) encode/decode, and the 32-fragment repair case), R-21/Q-M18-1 (SHA-256 without SHA-NI), Argon2id parameterisation, RocksDB/Badger HDD compaction, R-22's duty-cycle half | v2 said *"measure once, answer three."* With ADR-065's `LaunchGate` and result-JSON schema it is now measure once, answer **five**, with a durable artifact per measurement. |
| **Provider cohort survival** | Q08-1, Q20-3, Q05-4, Q21-1, Q29-1, Q33-1, and R-14's method applied to real data | Cannot start until there is a cohort. **R-14 (survival analysis from truncated observation) should be read *before* the cohort exists**, so the instrumentation records what the method needs. Reading it after is the classic ordering error. |

## 5.3 — How to run this without it stalling again

The stated problem is pace, not direction. Five things, in order of how much they help:

1. **Separate the queues.** Right now "research" means five different activities competing for the
   same hours: council rulings, derivations from owned papers, literature search, paper drafting,
   and measurement. §7.2 alone contains **seven blocking items that need no reading at all**. They
   are stalled behind a search queue they do not belong in. Split them and the critical path
   shortens immediately.
2. **Cap the search budget per topic.** Two hours of searching, then either a shortlist or a written
   "nothing found, here is what I tried." A recorded null result is a deliverable — v2's own note
   that `"libp2p QUIC NAT traversal 2026"` returned nothing useful is one of its more valuable
   lines, because nobody has to repeat it.
3. **Triage in batches of ten, draft in batches of one.** The scorecard in §4.4 makes shortlisting
   mechanical, which is what makes it batchable. Drafting a `paper-NN` is not batchable and should
   not be attempted in parallel.
4. **Model allocation, extended from the established build pattern.** Sonnet for triage passes
   against the §4.4 scorecard and for first-draft `paper-NN` structure. Opus for the substitution
   tests, for council sessions, and for any cross-document read (this document is that pattern's
   output). The scorecard exists partly so the triage step can be delegated confidently.
5. **Every research artifact carries a `Track:` line.** Same rule as ADR-062 §3, same reason.
   Everything in this document is `Track: LTS`; when a topic produces a `paper-NN`, tag it, so that
   when someone opens the frozen demo in two years they can tell which research was behind it and
   which came after.

## 5.4 — The standing rule, restated because it has now earned a third instance

**Any formula imported from a paper carries its substitution inline.** F-43 (`40^16` vs `40²`) and
NFR-044 (an interpolation formula wrong at its own anchor) were the first two. The third is
**Q66-2**: `(1 − p_au)/p_au = 99×` was written into the reading list treating `p_au` as constant,
and ADR-060's re-derivation under proportional sampling gets `S ≥ V/2,909` — a factor of ~288,000 in
the opposite direction, from the same paper. Nobody was careless; the substitution just was not
shown, so the error had nowhere to surface.

§4.3's **substitution test** field moves that rule upstream from ADR-drafting to shortlisting, which
is the cheapest place it can live. That is the single most useful thing in this document.
