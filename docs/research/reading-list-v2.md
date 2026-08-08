# Research Topic List v2 — Consolidated

**Replaces `reading-list-v2.md` and `reading-list-v2-increment-2.md` in full.** Single document from here.

Built from a complete pass over **ADR-001 to ADR-058**, recorded in four interrogation documents
([001–015](./adr-001-015-design-interrogation.md), [016–030](./adr-016-030-design-interrogation.md),
[031–045](./adr-031-045-design-interrogation.md), [046–058](./adr-046-058-design-interrogation.md)) —
79 findings, of which 22 point at literature the corpus does not contain.

Ordered by **rewrite risk**: how expensive it becomes to learn this late. Not by build order, and no
longer by parameter dependency — the gaps that matter sit *underneath* Accepted ADRs, not in front of
unbuilt ones.

---

## The three structural findings

Everything below is downstream of these. Each is a case where the system does not currently have a
property its own documents claim.

| | Finding | Property claimed | What is actually true |
| --- | --- | --- | --- |
| **1** | **F-32** — audits verify nothing | ADR-014: *"Verified by the microservice which independently has the expected hash"* | ADR-030: *"The microservice cannot verify the response hash directly (same as real chunks — it never holds the chunk data)"*. A provider can delete every chunk, answer with random bytes, and be paid. |
| **2** | **F-69** — repair reconstructs plaintext | ADR-019/020/022: zero-knowledge | AONT-RS discloses at k=16. RS repair of one lost shard requires assembling k=16. Every repair event hands a decodable package to whoever performs it. No collusion needed. |
| **3** | **F-34** — two ASNs recover plaintext | ADR-022: *"the 20% ASN cap… keeps the threshold safe"* | Safe for one group (11 < 16). Two colluding groups is 22 ≥ 16. Durability margin 29 shards; confidentiality margin **5**. |

They are related but not the same problem. (1) is a missing primitive. (2) is a property of RS repair
that the code family was never chosen to avoid. (3) is one threshold doing two jobs with only one of
them costed. Domains A, P and K exist for them respectively, and are the first three below.

---

## The corpus problem, restated

Three distinct diseases, and only one is about age.

**Old is fine for mechanisms.** Kademlia (2002), Dynamo (2007), Bailis (2015), AONT-RS (2011),
Reed–Solomon, Condor (1988), Lazear (1979). These describe constructions and arguments. They do not
expire. Re-reading them against 2026 literature produces nothing. Leave them.

**Old is fatal for populations.** Bolosky (2000), Saroiu (2002), Blake & Rodrigues (2003) measured
*who was on a network and how they behaved*. That population is gone. Three 20-plus-year-old
measurement papers currently underwrite the 72-hour departure threshold, the 24-hour polling
interval, and the background bandwidth budget. Concentrated in Domain D.

**Missing regardless of age.** The corpus contains four deployed storage networks and **zero**
proof-of-storage primitive papers. Domain A's canonical sources are 2007–2008; recency is not the
criterion there, presence is. Same for transparency-log gossip, wide-stripe repair, and secure
regenerating codes.

A fourth pattern emerged late and is worth naming: **eight consecutive ADRs (038–045) each rest on
exactly one source**, four on the same vendor documentation. Those decisions are mostly cheap to
reverse — but low-priority-because-well-evidenced and low-priority-because-cheap-to-reverse are
different claims, and only the second is true there.

---

## How to use this list

Workflow: this document names a **topic** → Karma searches ScienceDirect / IEEE Xplore / ACM DL /
USENIX → promising abstracts return as a triage document → the Technical Researcher drafts `paper-NN`.

Each entry carries **search strings** (paste-ready), **accept if**, and **reject if** — the last
because the most common failure is a search aimed at a settled engineering choice rather than an open
question. The previous list records `"libp2p QUIC NAT traversal 2026"` returning nothing useful for
exactly that reason.

**Database notes.** ACM DL and IEEE Xplore hold the 2007–2012 crypto primitives (CCS, Asiacrypt,
S&P); ScienceDirect will not have them. ScienceDirect is strongest for 2024–2026 systems work (FGCS,
JSA, Computer Networks, JPDC). USENIX (FAST, OSDI, NSDI, ATC) is open-access — search usenix.org
directly, not through a paywalled index. FAST in particular is where Domains C and P live.

**Search hygiene.** Four of thirteen hits in the erasure-coding pass were multi-cloud cost-arbitrage
papers. That term family — *cloud-of-clouds, vendor lock-in, storage class, tier pricing* — pollutes
erasure-coding searches. Prefer "stripe width" and "redundancy adaptation" over "tiering."

Tags: **[GAP]** nothing in the corpus touches it · **[STALE]** covered by expired measurement data ·
**[WEAK]** Accepted ADR with a documented open constraint · **[V3]** deferred functionality.

---

## Band 0 — Drafting queue, no searching required

Four papers already triaged and ready. **Papers 62–65.**

| # | Paper | Why |
| --- | --- | --- |
| **62** | **LESS: I/O-Efficient Repairs** (FAST 2026, Cheng/Li/Hu/Lee) | Layers extended sub-stripes *over* Reed–Solomon — cuts bytes read and seek count on single-block repair without abandoning RS. Addresses F-29 and F-40. RS-compatible means adoptable, not merely interesting. |
| **63** | **Data Persistency of Replicated Erasure Codes** (Information and Computation 2025, Friedman/Kapelko/Marchwicki) | Closed-form persistency when nodes *leave and erase their data* — Vyomanaut's actual failure mode, not disk failure. Gives an analytic tool for the small-N bootstrap regime (F-28) that Giroire does not. |
| **64** | **Hitchhiker** (Rashmi et al., SIGCOMM 2014) | ADR-026 commits V3 chunk layout, pointer file schema and audit design to the piggybacking family on one band in one survey's Figure 8. The primary paper has never been read, and F-43 shows the *rejection* of Clay rested on a misapplied formula. |
| **65** | **BiLP-BX bit-matrix / XOR scheduling** (ESWA 2026, Chen/Hu/Zhang) | 442.8% encoding throughput over Jerasure vs Cerasure's 377.8%. The only concrete input to the unmeasured 5% CPU budget (F-27). Read for the comparison table, not the optimisation method. |

**Also triaged, lower priority:** *A Comprehensive Repair Scheme* (Computer Networks 2023 — constructs
stripes from device heterogeneity coefficients; Vyomanaut has per-provider reliability scores and
measured throughput and feeds neither into stripe construction) and *ZJC* (JSA 2026 — constant repair
bandwidth with two helper blocks; a serious challenger to wide RS, read for whether the interleaved
local-group structure survives at n=56 under untrusted providers).

**Rejected, do not re-surface:** *Effeclouds*, *Online Dynamic Replication for OSN* (multi-cloud cost
arbitrage — wrong economics), *InaPR* (needs P4 switches), *CURM* (update-heavy workload; Vyomanaut is
write-once), *LocalityCache* (redundant with POCache/Paper 61). *HCFR* — mine the introduction for the
small-file citation trail, do not draft (the solution is an FPGA).

---

# Band 1 — Blocks an Accepted ADR

## Domain A — Proof of Retrievability primitives `[GAP]` — **START HERE**

**ADRs:** 002, 014, 015, 017, 030 · **Findings:** F-01, F-02, F-32

The corpus read four systems that implement proof-of-storage and none of the papers that define it.
ADR-030 states the consequence in plain text. Everything in the audit, receipt, scoring, payment and
repair-trigger path is gated on a check that cannot currently fail.

Canonical sources are 2007–2008. Get them regardless of age, then the last five years to see what
survived.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-01** | Foundational PDP / PoR constructions | `Ateniese provable data possession untrusted stores` · `Juels Kaliski proofs of retrievability large files` · `Shacham Waters compact proofs of retrievability` | It defines what the verifier stores, what the challenge selects, and detection probability as a function of sample size | It's a blockchain storage-proof paper — those are downstream deployments, we have those |
| **R-02** | Sampling-based audit with formal detection bounds | `probabilistic verification outsourced storage detection probability` · `spot checking data possession sampling guarantee` · `sentinel based proof of retrievability` | It gives an explicit (fraction corrupted, detection probability) relation we can set a sampling rate from | It assumes a verifier that already holds the data |
| **R-03** | Proof transfer on repair | `dynamic provable data possession update` · `authenticated data structure proof of storage update` · `proof of storage data migration node replacement` | Auditability survives a shard moving to a replacement provider without the owner online — ADR-004's repair makes this not-quite-write-once | Pure dynamic-update work with no migration case |
| **R-04** | Publicly verifiable / third-party audit | `public verifiability third party auditor cloud storage` · `privacy preserving public auditing outsourced data` | Verification without the data owner online — the constraint that breaks ADR-014's design | Bilinear pairings at a cost the 5% CPU budget can't carry (note and move on) |

**Non-negotiable accept criterion for the whole domain:** the construction must work when the verifier
holds **neither the data nor a precomputed per-challenge answer**, and must support **challenge-time
fresh nonces**. That is what separates a usable scheme from the current one.

---

## Domain P — Confidentiality-preserving repair `[GAP]` — **NEW, BLOCKER**

**ADRs:** 004, 019, 020, 022, 026, 055 · **Finding:** F-69

Repairing one lost shard requires assembling `k=16` shards. AONT-RS discloses plaintext at `k=16`.
The two thresholds are the same number, so **every repair event reconstructs a decodable package**
for whoever performs it — routinely, by design, with no adversary.

ADR-026 defers regenerating codes to V3 on *bandwidth* grounds and never notices they also carry the
property the system already claims to have. That makes this domain a possible input to the V3
code-family decision (F-44), not only a fix for a false claim.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-27** | Secure regenerating codes | `secure regenerating codes eavesdropper` · `information theoretic security distributed storage repair` · `secrecy capacity regenerating codes` | It bounds what a party observing repair traffic learns, and states the storage cost of closing it | Pure MDS construction with no secrecy analysis |
| **R-28** | Repair without full reconstruction | `repair by transfer distributed storage` · `exact repair without decoding erasure code` · `helper node partial repair no reconstruction` | A helper contributes to reconstruction without any party assembling `k` shards — the mechanism that would fix this | Requires a trusted repair coordinator; that's the party we're trying to keep blind |
| **R-29** | AONT under partial exposure | `all-or-nothing transform security analysis partial` · `AONT-RS security proof adversary threshold` · `Resch Plank AONT-RS analysis` | It analyses what an adversary with `k−1` slices learns, and whether systematic-RS layout leaks more than the AONT proof assumes | A new AONT construction with no threat analysis of the deployed one |

**Cheap first move:** Paper 16 (Resch & Plank) is already in the corpus, read for the *mechanism*.
Re-read its security section against a repair-path adversary before searching outward.

---

## Domain K — Decoupling confidentiality from reconstruction `[GAP]` — **BLOCKER**

**ADRs:** 014, 022, 029, 031 · **Findings:** F-34, F-67

One threshold, two jobs. `k=16` was chosen by durability and repair economics (Giroire).
Confidentiality inherited it by accident, and the 20% ASN cap — designed as a durability and
adversarial-placement control — is doing confidentiality work it was never sized for.

F-67 makes it structural rather than a one-off: the confidentiality threshold is `DataShards`, the
placement allowance is `floor(TotalShards × ASNCapFraction)`, both `NetworkProfile` fields with no
expressed relationship. Every future profile silently re-derives its own collusion threshold.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-30** | Separate privacy and reconstruction thresholds | `ramp secret sharing threshold storage dispersal` · `information dispersal algorithm confidentiality threshold` · `secret sharing erasure code separate privacy threshold` | The privacy threshold can exceed the reconstruction threshold at bounded storage cost — exactly the shape of the fix | Pure threshold cryptography with no storage-overhead analysis |
| **R-31** | Collusion-resistant placement under a diversity budget | `collusion resistant data placement distributed storage` · `placement diversity constraint secret sharing nodes` · `adversarial coalition size storage placement bound` | It models an adversary controlling *several* failure domains — every placement paper in the corpus assumes one | It assumes a trusted placement authority with full topology knowledge |

**Whatever the council rules on F-34, the fix must land as a compiler-enforced profile invariant**
alongside `TestProfileShardSizeIsConstant`, or it returns with the next profile.

---

## Domain B — Audit verification at fleet scale `[GAP]`

**ADRs:** 002, 012, 023, 033 · **Findings:** F-02, F-03, F-40, F-58, F-59

Full daily audit of every chunk fails in three places at once, all at roughly the same scale:

| Constraint | Bites at | Finding |
| --- | --- | --- |
| Postgres INSERT ceiling | ~1,000 providers × 200 GB → 9,481/s, ×2 two-phase = **18,963/s** vs a stated 5–10k | F-02, F-58 |
| Provider disk | 70 GB provider = 286,720 chunks/day = **~1 h/day** of HDD seeking | F-40 |
| Nonce guard index | 48 h retention at 1,000 × 70 GB = 573 M rows ≈ **30 GB** hot index | F-59 |

capacity.md's own analysis makes the first worse: the 5–10k figure is *"derived from generic benchmark
literature and not from the actual Vyomanaut schema"* — two 64-byte signatures, a 33-byte unique
index, RLS evaluation, two-phase write. Real ceiling is likely below 5,000/s.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-05** | Batch and aggregate verification | `batch auditing multiple owners cloud storage` · `aggregate signature verification storage integrity` · `homomorphic verifiable tags batch audit` | One verification covers many chunks or many providers — the only way row count falls without cutting coverage | Batches within one owner's data; our bottleneck is across providers |
| **R-06** | Audit scheduling under a verification budget | `audit scheduling coverage constraint distributed storage` · `verification cost amortization integrity checking scale` | Audit frequency treated as resource allocation against a durability objective | Database write-throughput tuning — different bottleneck, number already known |
| **R-07** | Disk-efficient verification at rest | `low overhead integrity verification storage scrubbing schedule` · `sequential scan versus random sampling data verification` | Compares sequential sweep against random spot checks on cost *and* detection latency — a sequential sweep reads 70 GB in ~12 min versus an hour of seeking | Assumes an enterprise array with idle spindles |
| **R-08** | Verification with sub-chunk locality | `proof of retrievability sequential access pattern` · `audit challenge locality disk friendly verification` | The challenge is answerable from a contiguous region — **read this *with* Domain A, not after**; if we're respecifying the challenge format anyway, disk layout is free to choose now | Requires the verifier to hold the data |

**Two arithmetic prerequisites before searching, using material the corpus already has:**

1. **SHELBY Condition (i) scales as `(1 − p_au)/p_au`** (ADR-024 §7). At daily full audit `p_au ≈ 1`
   it is trivially satisfied; at 1% sampling the required slashing-to-gain ratio is **99×**. That
   equation bounds which sampling rates are admissible at all. Substitute it before R-05/R-06.
2. **F-03: the payment unit must be settled first.** ADR-012 pays per audit passed; sampling makes
   income a function of how often you were sampled. Ruling on that changes what R-05/R-06 are for.

---

## Domain C — Wide stripes, repair I/O, and small objects `[GAP]`

**ADRs:** 003, 004, 026 · **Findings:** F-29, F-30, F-43, F-44

ADR-003's own reference note concedes `n=56` is *"3× larger than any surveyed production system"* and
then treats it as settled. And `s=16 × 256 KB` makes the minimum efficiently-coded object **4 MB** —
a 200 KB file expands ~70×, with no file-size distribution stated anywhere.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-09** | Wide-stripe repair and combined locality | `wide stripe erasure coding repair` · `combined locality wide stripes repair bandwidth` · `ECWide erasure coding wide stripes` — start with **LESS** (Paper 62) | Targets `n ≥ 30` explicitly; reports repair cost as a function of stripe width | An LRC construction with no wide-stripe evaluation — LRC was ruled out in ADR-026 on topology grounds |
| **R-10** | Piggybacking code framework | `piggybacking framework erasure codes reconstruction` · `Hitchhiker erasure code data center reconstruction` — start with **Paper 64** | Reports savings as a function of `(n,k)` — we need to know whether 25–45% holds at `(56,16)` or was measured at `(14,10)` | Another survey; Paper 19 is the single source F-44 objects to |
| **R-11** | Small-object erasure coding | `small file erasure coding storage overhead` · `object bundling erasure code small objects` · `sub-stripe packing erasure coded small files` | Quantifies the crossover size below which replication beats coding — the missing input to a small-file policy | FPGA/GPU acceleration (see HCFR triage) |

**Correction, not research:** ADR-026's Clay rejection states `α ≥ 40^16`. The standard MSR bound at
`(n=56, k=16, d=55)` gives `40² = 1,600` — a **164-byte** sub-chunk. Clay very likely stays ruled out,
but on I/O grounds, not tractability. Different argument, different shelf life: tractability never
improves; the I/O argument weakens as providers move to SSDs. Recompute and show the substitution.

---

# Band 2 — Invalidates a load-bearing parameter

## Domain D — Churn, availability, and lifetime estimation `[STALE]` — **HIGHEST LEVERAGE**

**ADRs:** 005, 006, 007, 009, 010, 047, 053, 057 · **Findings:** F-06, F-15, F-16, F-72

Where the staleness problem concentrates. The 72-hour threshold, the 24-hour poll, and the entire
mobile-deferral argument rest on Bolosky's 1999 measurements of **corporate desktops on the Microsoft
campus LAN**. σ=1.9 h on a nightly absence is implausible for consumer hardware anywhere.

Two separate defects this domain also fixes:

- **F-15:** vetting certifies **audit success rate** via a Jeffreys prior, and that certificate is
  used to justify an **MTTF** assumption. Different random variables. Estimating lifetime from a
  truncated window is a solved problem in the wrong field.
- **F-72:** ADR-047's logon-trigger autostart means the daemon **stops at logoff and does not run
  before login**. An MTTF of 180–380 days is being applied to a daemon whose real duty cycle is
  *time-a-human-is-logged-in*, and nobody has measured that. Windows Update reboots make this
  routine, not exotic.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-12** | Modern consumer/edge device availability | `edge device availability measurement churn 2024` · `volunteer computing node uptime distribution measurement` · `consumer device session length distributed system measurement` | 2018 or later, consumer or edge hardware, reports a *distribution* not a mean | Datacenter or cloud-VM availability — wrong population, that's the point |
| **R-13** | Churn under financial incentive | `incentivised storage network node retention measurement` · `paid participation peer-to-peer churn effect` · `Storj Filecoin node operator retention analysis` | Separates *incentivised* from *volunteer* churn — Paper 20's 87.6%-under-8h figure is explicitly unincentivised and is used as if it generalises | Token-price analysis; we need retention behaviour |
| **R-14** | Lifetime estimation from truncated observation | `survival analysis censored data node lifetime distributed system` · `Kaplan-Meier reliability estimation partial observation` · `hazard rate estimation short observation window` | A method for estimating MTTF from a vetting-length window with right-censoring — literally the vetting problem | Hardware-failure-rate work (we have Schroeder, Paper 32); we need *departure*, not *failure* |
| **R-15** | Desktop session and logon duty cycle | `desktop login session duration measurement` · `workstation power state uptime measurement study` · `automatic update reboot frequency measurement Windows` | Reports time-logged-in as a fraction of time-powered-on — the number F-72 needs | Enterprise fleet-management vendor reports without methodology |

---

## Domain E — Correlated failure and the bootstrap regime `[GAP]`

**ADRs:** 001, 003, 004, 005, 008, 014, 029, 055 · **Findings:** F-28, F-34

Q07-4 recurs across four ADRs, has been *"deferred to Phase 2B"* since the beginning, and has never
been researched. ADR-029 fixes the bootstrap gate at ≥56 providers / ≥5 ASNs — precisely the regime
the durability model was never run against, and where F-34's confidentiality margin is thinnest
(5 ASNs × 20% = 100%, so the cap is exactly saturated and any two ASNs is 40% of the network).

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-16** | Durability under correlated failure at small scale | `correlated failure durability model small scale storage` · `failure domain diversity minimum nodes erasure coding` · `placement constraint feasibility limited failure domains` | Models the regime where failure domains are scarce relative to stripe width — the bootstrap problem exactly | Assumes a large fleet with abundant independent domains |
| **R-17** | AS-level topology of Indian residential broadband | `India autonomous system topology residential ISP` · `Indian broadband AS concentration measurement` · `CGNAT deployment measurement developing markets` | Tells us how many *genuinely independent* ASNs a consumer pool in India can span — if the real answer is 6–8, the 5-ASN gate has almost no headroom | Global AS-topology work with no India breakout |
| **R-18** | AS-level outage measurement | `AS level outage measurement Internet duration` · `ISP outage correlation measurement study` · `country level Internet disruption measurement India` | Outage **duration** and **blast radius** distributions to size the 20% cap against | BGP hijack / routing-security — different threat |
| **R-19** | Decentralised failure-domain inference | `failure correlation inference without central coordinator` · `distributed failure domain detection peer to peer` · `latency based topology inference failure correlation` | Peers infer shared-fate from observable signals (RTT, path overlap) without a trusted map — Q07-4 as originally posed | Requires a global view; that's what we're avoiding |

**R-19 unblocks four stalled constraints at once** (ADR-001, 008, 014, 055) and is the input to
ADR-055's emergency-eject trigger. **R-16 and R-17 are what affect launch.**

**Campaign note:** R-17 pairs with R-15, R-22 and the NAT/CGNAT items in Band 5. One measurement pass,
five answers. Sources are likely TRAI performance reports, M-Lab and Ookla open datasets, and
CEA/state-utility outage data rather than academic literature — treat as a research note, not a
`paper-NN`.

---

## Domain F — Provider-side compute and I/O budget `[GAP]`

**ADRs:** 009, 019, 022, 023 · **Findings:** F-27, F-40, F-53

ADR-009 has the thinnest evidence base of all 58 — one paper, from 2000, measuring *ambient* desktop
load rather than Vyomanaut's workload. The real cost is RS(16,56) encode/decode over GF(2⁸) with 40
parity shards, SHA-256 over every chunk on every audit, ChaCha20-Poly1305 on transfer, a WebView2 GUI
process, and (F-40) about an hour a day of random seeking. None of it estimated.

Largely a collection problem, not a discovery problem.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-20** | EC library throughput on commodity CPUs | `Reed-Solomon encoding throughput SIMD comparison` · `erasure coding library performance ISA-L Jerasure benchmark` — start with **Paper 65** | Reports MB/s at a stated `(k,m)` on a named consumer-class CPU | GPU or FPGA — our providers have neither guaranteed |
| **R-21** | Symmetric crypto on low-end hardware | `ChaCha20-Poly1305 throughput ARM x86 comparison` · `SHA-256 hardware acceleration availability consumer CPU` | Covers CPUs without AES-NI / SHA extensions — the min-spec Indian desktop case | Server-class benchmarks only |
| **R-22** | Duty cycle and consumer drive lifetime | `hard drive workload rate lifetime consumer drives` · `duty cycle drive wear annualised failure rate` · `HDD workload rating reliability measurement` | Relates sustained random-read load to AFR or expected lifetime — feeds **both** F-40 and the second half of Domain J's carbon ledger | Enterprise-drive datasheets; wrong hardware class |

**Pair with an empirical task.** Argon2id benchmarking (ADR-020), the HDD compaction benchmark
(ADR-023, re-run for Badger per ADR-046), and the daemon CPU budget all need one min-spec Indian
desktop. Measure once, answer three.

---

# Band 3 — Cheap to change now, expensive later

## Domain G — Transparency logs without blockchain `[GAP]`

**ADRs:** 015, 017, 032 · **Findings:** F-22, F-23, F-68

The product claims a non-crypto mechanism reproduces blockchain's three functions. Function 3 (public
dispute resolution) is deferred to V3 — so at launch the system offers strictly *weaker* verifiability
than the thing it replaced.

ADR-032 sharpens rather than closes this. It is the best security work in the corpus (test #10: grant
the app `DELETE`, retry, observe `DELETE 0`). But `vyomanaut_migrator` must hold `BYPASSRLS` to
refresh materialised views, so the maintenance identity can delete audit receipts **by design**. The
property is tamper-*resistance against the request path*, not tamper-*evidence against the operator*.

The V3 design has its own hole: Certificate Transparency is correctly cited, but CT's security rests
on **gossip between verifiers** to detect split views. A daily published root with no gossip lets an
operator serve different roots to different providers indefinitely.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-23** | Split-view detection and log gossip | `certificate transparency gossip split view detection` · `transparency log consistency proof gossip protocol` · `Chuat efficient gossip transparency log` | Specifies how verifiers exchange views and what operator equivocation looks like when caught | CT policy/ecosystem rather than mechanism |
| **R-24** | Tamper-evident logging over a mutable store | `tamper evident logging history tree` · `Crosby Wallach efficient data structures tamper evident logging` · `verifiable log backed map append only database` | Makes tampering **detectable** in a store that physically permits UPDATE — F-23's `audit_result` carve-out and F-68's `BYPASSRLS` are the same problem | Requires append-only storage hardware or a chain |
| **R-25** | Client-verifiable audit against an untrusted operator | `client verifiable storage audit without trusted server` · `accountability protocol untrusted service provider` | The provider can prove a countersignature was *withheld* — the missing recourse in F-22 | Assumes a trusted third party we'd then have to be |

---

## Domain L — Single-writer election in a leaderless cluster `[GAP]`

**ADRs:** 013, 025, 048 · **Findings:** F-35, F-76

ADR-025 imports Dynamo's leaderless membership and routes six non-I-confluent operations to *"the
single authoritative payment/assignment service"* without saying which replica that is or what happens
when it dies.

ADR-048 names the mechanism: `ResponsibleReplica()`, a pure function of gossip membership —
eventually consistent at a 1-second cadence, 5-second healthy window, 10-second client cache. ADR-048
authenticates it against *forgery*; it does nothing about *divergence*. Two replicas with genuine but
different views each compute a different responsible replica for the same key. For escrow debit — a
bounded counter with a `≥ 0` floor — that is a double-release path with no fencing token.

**Dynamo is leaderless because everything in Dynamo merges.** Escrow debit and the 20% ASN cap do not.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-26** | Lease-based election without external consensus | `lease based leader election fencing token` · `single writer failover without consensus service` · `distributed lock correctness fencing storage` | Specifies the fencing that makes a stale writer's writes *rejected*, not merely unlikely | Requires Raft/Paxos as a dependency — that option is understood; we need the alternatives priced |
| **R-32** | Safety of bounded counters under partition | `bounded counter CRDT reservation escrow` · `non-negative counter replicated safety partition` · `escrow balance replicated invariant preservation` | Preserves a `≥ 0` floor without coordinating every operation — a reservation scheme would take some escrow debits off the coordinated path entirely | A general CRDT survey; we have Bailis (Paper 11) |
| **R-33** | Conflict resolution on the quorum read path | `vector clock reconciliation quorum key value store` · `version vector conflict resolution metadata store` | Specifies what a client does with divergent versions — ADR-025 adopts R+W>N without the versioning that makes it meaningful | Dynamo itself (Paper 12) — read; we need what came after |

**R-32 is the high-value one.** If reservation-style bounded counters work here, the count of
coordinated operations drops below six and ADR-013's central trade-off improves materially.

---

## Domain M — Local secrets: offline attack and unattended daemons `[GAP]`

**ADRs:** 019, 020, 058 · **Findings:** F-46, F-47, F-75

Two related problems at opposite ends of the system.

**Owner side (F-46).** ADR-020 stores `pointer_ciphertext` server-side and calls it a blind store. It
is blind — and it is an offline oracle. Anyone with that table can brute-force the passphrase,
verifying each guess against the Poly1305 tag. A hit yields the master secret, hence every file the
owner ever stored, permanently and non-revocably. Argon2id at t=3/m=64 MB is an interactive setting,
not an offline-attack setting.

**Provider side (F-75).** ADR-058 defers *"where the daemon-local passphrase comes from"* as a UX
decision. It is not. Auto-generated and stored locally → the key sits beside the ciphertext and
protects nothing. Operator-entered → the daemon cannot start unattended, breaking ADR-047, ADR-009
and ADR-051 at once. The standard resolution — OS keystore binding — appears nowhere in the corpus.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-34** | Memory-hard KDF parameters against offline attack | `Argon2 parameter selection GPU attack cost` · `memory hard function attack cost estimate 2024` · `password hashing cost parameter recommendation offline` | Gives guesses-per-second per rupee at stated parameters on current hardware — converts directly into policy | The original Argon2 competition paper; we need current attack economics |
| **R-35** | Real-world passphrase entropy distributions | `passphrase entropy measurement user chosen` · `password strength distribution empirical study` · `guessing number distribution real passwords` | Reports a *distribution*, so we can state what fraction of owners are actually protected | Password-policy usability without a guessing-resistance measure |
| **R-36** | Avoiding server-held offline-verifiable blobs | `password authenticated key exchange practical deployment` · `OPAQUE asymmetric PAKE implementation` · `client side encryption backup without server oracle` | The server holds nothing an attacker can test guesses against — removes the attack rather than raising its cost | Requires a trusted hardware module we'd then operate |
| **R-37** | OS-bound secrets for unattended services | `DPAPI CryptProtectData service credential storage` · `OS keychain integration daemon secret unattended` · `secret storage headless service without interactive unlock` | The OS binds the key to the user account rather than the filesystem, with **no elevation required** (ADR-042) — the shape that satisfies F-75's dilemma | Enterprise HSM/KMS patterns; wrong deployment model |

**Cheapest option worth pricing before spending here:** don't store `pointer_ciphertext` server-side
at all. That trades ADR-020's Scenario 4 recovery for removing the entire oracle. It degrades a
recovery path the UX depends on, so it is a council question — but it should be on the table.

---

## Domain N — AEAD nonce handling and RNG failure `[GAP]`

**ADRs:** 019, 020 · **Finding:** F-47

ADR-019 uses `nonce = 0` for the AONT stream cipher, safe only while `SecureRandom` never repeats `K`.
VM cloning, container image reuse, snapshot/restore and early-boot entropy starvation all produce
repeats, silently. ADR-020 requires a durable monotone counter and concedes it cannot enforce
non-reuse if that counter is lost.

**XChaCha20-Poly1305's 192-bit nonce removes both problems.** Not in the ADRs or the corpus — the
corpus has RFC 8439 and nothing newer on AEAD nonce handling.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-38** | Extended-nonce and misuse-resistant AEAD | `XChaCha20-Poly1305 extended nonce construction` · `nonce misuse resistant AEAD deployment` · `AES-GCM-SIV synthetic IV analysis` | Quantifies collision margin at 192 bits **and** states what survives a nonce repeat — the second property is why it's worth adopting even if counters stay | A new AEAD proposal with no deployed implementation; we need something in libsodium today |
| **R-39** | RNG failure on cloned and low-entropy systems | `entropy failure virtual machine cloning keys` · `random number generator weakness embedded devices measurement` · `duplicate keys entropy survey` | Measures how often this happens in the wild — turns "theoretically fragile" into a probability weighable against the fix | RNG design; we need field measurement |

Both are small and close to decision-ready. R-38 especially.

---

## Domain I — Coordination DoS, reputation gaming, relay `[GAP]`

**ADRs:** 001, 005, 008, 021, 025, 043, 045, 053 · **Findings:** F-12, F-31, F-50, F-61

Three things grouped because each is "adversary or cost attacks the coordinator."

**F-12:** ADR-001 and ADR-014 both assert quorum reads mitigate DoS. Quorum gives consistency; under
a flood it raises per-request cost. Nothing in the corpus covers coordination-service DoS at all.

**F-31 / F-61:** ADR-008 defers the scoring weights that determine provider income, with no
adversarial analysis. And ADR-045's disclosure policy is self-defeating — `released/gross` *is* the
multiplier, so two or three releases reveal ADR-024's entire threshold ladder.

**F-50:** ~30% of providers route through Vyomanaut-operated relays (higher in India per ADR-028).
That is operator-paid cloud egress on every byte for a third of the network, capacity sized against
steady-state provider count rather than repair burst, and an on-path position for all their traffic.
ADR-043's "no manual networking step" promise is what forces this cost.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-40** | Application-layer DoS and admission control | `application layer DDoS mitigation API admission control` · `client puzzles denial of service resource asymmetry` · `rate limiting fairness authenticated clients` | Exploits that our clients are **registered** — a lever most DoS work doesn't have | Volumetric network-layer scrubbing; that's a purchase, not a design |
| **R-41** | Reputation manipulation: on-off, oscillation, whitewashing | `on-off attack reputation system resistance` · `oscillation attack trust model peer to peer` · `whitewashing newcomer reputation cost` | Analyses *time-windowed* scores specifically — our three-window design is the structure these attacks exploit | EigenTrust variants; the centralised-scorer decision is settled and correct |
| **R-42** | Relay sizing and cost under burst load | `circuit relay capacity peer-to-peer sizing` · `TURN relay server capacity planning concurrent sessions` · `NAT relay bandwidth cost model` | Sizes against **concurrent connections under burst**, not steady-state peer count — ADR-029's 384 slots were computed against the latter | WebRTC media relay; our traffic is bulk transfer |

---

# Band 4 — Economics and compliance

## Domain J — Payments, Sybil bound, and the carbon ledger `[WEAK]`

**ADRs:** 011, 012, 024, 030, 044, 053, 054 · **Findings:** F-25, F-41, F-42, F-64, F-77

ADR-053 and ADR-054 substantially improved the vetting-income picture (F-77): duration roughly thirds,
the flat 50% cap becomes a rising ramp. **The residual is now entirely ADR-030's 10% storage cap**,
which neither revisits and which is the binding term.

And ADR-054 **removed** the deterrent that partially covered Sybil farming — vetting-stage earnings
are no longer seized on departure. The play is now: register, behave perfectly 45 days, collect a ramp
reaching ~0.98×, depart with no forfeiture, repeat. Both ADRs flag the exposure and both decline to
compute the bound. That is now materially more urgent than when first logged.

| # | Topic | Search strings | Accept if | Reject if |
| --- | --- | --- | --- | --- |
| **R-43** | Micropayment aggregation and payout batching | `micropayment aggregation transaction cost amortization` · `probabilistic micropayments lottery tickets` · `payout batching marketplace settlement frequency` | Quantifies the batching interval that makes sub-rupee accrual viable, and its cost in perceived fairness | Payment-channel work; we have no channel and no chain |
| **R-44** | Regulatory posture of held funds and forfeiture (India) | RBI *Guidelines on Regulation of Payment Aggregators and Payment Gateways* — **primary source, not a search** · `escrow account payment aggregator India nodal` · `consumer forfeiture clause enforceability India` | States whether a non-NBFC marketplace may hold and forfeit earned provider balances, and under what account structure | Crypto/VDA regulation — different regime |
| **R-45** | Supply-side retention under delayed and withheld earnings | `gig platform payout delay supply retention` · `deferred compensation platform worker churn` · `escrow holdback marketplace supplier participation` | Measures churn as a function of payout delay and withholding — the input F-42's residual needs | Purely theoretical contract design; we have Lazear (Paper 56) |
| **R-46** | Embodied carbon, **both sides of the ledger** | `embodied carbon storage device lifecycle assessment` · `manufacturing carbon hard drive SSD per terabyte` — pair with **R-22** | Gives per-TB embodied carbon for consumer-class drives **and** lets us subtract accelerated replacement, not just add avoided datacenter hardware | Datacenter PUE / operational-energy studies; ADR-044 already correctly rejected that framing |

**R-44 is not academic literature.** Primary sources are the RBI PA/PG guidelines and Razorpay's own
escrow/Route documentation on permissible account structures. Research note, not a `paper-NN`. It
matters because **F-41**: ADR-024 §5 justifies seizure as cost recovery while §7 says the operator
bears no repair cost. A forfeiture defended by a cost the same document denies will not survive a
consumer complaint.

**R-46 is rescoped.** ADR-044's claim is *gross* — avoided datacenter hardware. But Vyomanaut raises
duty cycle on consumer drives that would otherwise idle, and F-40's ~1 h/day of seeking accelerates
replacement. The honest accounting subtracts that. It is exactly the check ADR-044 exists to survive.

---

# Band 5 — Carried forward

Priority moved because Bands 1–4 landed above them, not because anything changed about the items.

| Item | ADR | Status | Note |
| --- | --- | --- | --- |
| **Hot/Cold band parameters** | 018 | **Council, not research** | Papers 59–61 closed the literature question. What remains is Q59-1 (does Vyomanaut want runtime re-tiering at all) and deriving `k_hot` once target SLA numbers exist. **Stop searching this.** |
| **NAT traversal real-world success rate** | 043 | Unchanged | The gap is the measured failure rate across Indian ISP/router configurations, not the libp2p choice. **Fold into the R-17 campaign.** |
| **QUIC empirical gaps** | 021 | Band 5 | CGNAT prevalence, UDP-block rates, 0-RTT reconnect penalty, migration false-positive rate. Same campaign. |
| **vLog GC fine/coarse reclaim** | 023 | Unchanged | Real divergence between `build.md` Session 5.1.5 and ADR-023's prose. VGKV and ZoomDB triaged ACCEPT/WATCH. Independent of everything above — can run in parallel by a second reader. |
| **RocksDB vs BadgerDB unification** | 046 | Band 5 | Q49-1, post-launch measurement. **Not literature** — but see F-73: Badger's `SyncWrites: false` default is a correction to make now, not a research item. |
| **DHT necessity and cost at small N** | 001 | **Suspended** | Subordinate to F-09/F-10 — pointless to research a subsystem whose necessity is under council review. Restore if the ruling keeps the DHT. |
| **Upload straggler / Storj `o` parameter** | 003 | Band 5 [V3] | POCache (Paper 61) covers the read-path analogue. Low urgency. |
| **Repair burst admission control** | 004, 055 | Band 5 [V3] | ADR-055 closed the structural half. The remainder falls out of R-16/R-19. |
| **Adaptive polling from score history** | 008 | Band 5 [V3] | Downstream of Domain D — needs a churn model first. |
| **Post-graduation monitoring, forced-ceiling providers** | 053 | Band 5 | Flag exists, response does not. Pairs with R-41. |
| **ISP data-plan sync** | 056 | Band 5 [V3] | Correctly deferred on DEPA telecom rollout. Per-platform network accounting is engineering. |
| **Background execution continuity** | 057 | Band 5 [V3] | Privilege check against ADR-042 (F-63) and managed-device fallback. Engineering, not literature. |
| **Desktop shell cluster** | 038–041, 049–052 | **Re-tagged** | Was *"engineering verification, not literature gaps."* Now: **decisions already made on single-source evidence.** Priority stays low, but ADR-040 makes a community tray library the Provider app's *primary* interface on one anecdotal precedent, with a Wails-v3 trigger that has no date, owner or check interval. Low-because-cheap-to-reverse, not low-because-well-evidenced. |

---

## Recommended sequence

```
R-01 → R-02 → R-03 → R-04       A  PoR primitives                    [F-32 — blocks everything]
R-27 → R-28 → R-29              P  confidentiality-preserving repair [F-69 — independent blocker]
R-30 → R-31                     K  decouple the two thresholds       [F-34 — independent blocker]
      ↓ council: F-32, F-69, F-34, F-03
R-05 → R-06 → R-07 → R-08       B  audit at scale     [R-08 read *with* Domain A]
R-09 → R-10 → R-11              C  wide stripes + small objects      [parallel, independent]
      ↓
R-12 → R-13 → R-14 → R-15       D  churn and lifetime                [highest leverage]
R-16 → R-17 → R-18 → R-19       E  correlated failure, small N       [R-17 = one campaign, five answers]
R-20 → R-21 → R-22              F  compute and I/O budget            [parallel, cheap]
      ↓
R-23 → R-24 → R-25              G  transparency logs
R-26 → R-32 → R-33              L  single-writer election
R-34 → R-35 → R-36 → R-37       M  local secrets
R-38 → R-39                     N  nonce handling                    [small, nearly decision-ready]
R-40 → R-41 → R-42              I  DoS, gaming, relay
R-43 → R-44 → R-45 → R-46       J  economics, compliance, carbon
```

**If only one domain gets attention: A.** The only one where the specification is not merely
under-evidenced but internally impossible, and everything in audit, receipt, scoring and payment sits
downstream.

**If only two: A and P.** P is true on every ordinary operation rather than under attack, and it may
change the V3 code-family decision from a bandwidth question into a confidentiality one.

**If only three: add D.** Where the corpus is genuinely, dangerously stale, upstream of five separate
Accepted ADRs, and now also upstream of F-72's unmeasured logon duty cycle.

---

## Three things that look like research and are not

Stated explicitly so they don't consume search budget. Each is correctly identified as such in its own
ADR:

- **`k_hot` derivation** (ADR-018) — a calculation, once target SLA numbers exist.
- **Secrets-manager product choice** (ADR-027) — infrastructure decision, deferred to deployment.
- **Network-namespace runbook for 1,000-node simulation** (ADR-029) — engineering spec.

And one standing rule, earned twice: **F-43** (Clay `40^16` vs `40²`) and **NFR-044** (an interpolation
formula wrong at its own documented anchor) are the same failure — a formula lifted from a paper into
an ADR without its substitution shown. ADR-031 §5, ADR-048 §6 and ADR-055 all show their working
unprompted. Make that the convention: **any formula imported from a paper carries its substitution
inline.**
